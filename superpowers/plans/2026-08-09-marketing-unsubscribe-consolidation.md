# Marketing Email Unsubscribe Consolidation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make TreatmentPath marketing emails show a single, Postmark-styled unsubscribe link pointing to our own `/marketing-preferences/<token>` page, and remove Postmark's automatic `subscriptions.pstmrk.it` unsubscribe link by switching the broadcast stream to `UnsubscribeHandlingType: "Custom"`.

**Architecture:** The Django backend already appends a preferences footer and mints a per-(practice, person) token. We will change that footer to a centered Postmark-style "Unsubscribe" link and add a matching plain-text line to the `text_body`. The Go service will expose a new Postmark client method that `PATCH`es `/message-streams/broadcast` to set `SubscriptionManagementConfiguration.UnsubscribeHandlingType = "Custom"`, and the domain-verification handler will call it after DKIM/Return-Path pass, logging (but not failing on) errors.

**Tech Stack:** Python 3.12 / Django 5.1, Go 1.24 / Gin / GORM, Postmark API.

---

## File map

| File | Responsibility |
|------|--------------|
| `TreatmentPathBackend/TreatmentPath/marketingBroadcast/delivery_reporting.py` | Builds HTML/text email bodies; contains `_build_preferences_footer` and `send_broadcast_email`. |
| `TreatmentPathBackend/TreatmentPath/marketingBroadcast/tests.py` | Django tests for send path and footer. |
| `EmailServiceGo/internal/marketing/postmark_client.go` | Postmark HTTP client; add `SetCustomUnsubscribeHandling`. |
| `EmailServiceGo/internal/marketing/handlers.go` | HTTP handlers; wire stream config into `VerifyDomain`. |
| `EmailServiceGo/internal/marketing/postmark_client_test.go` | Tests for Postmark client methods. |
| `EmailServiceGo/internal/marketing/handlers_test.go` | Tests for HTTP handlers including `VerifyDomain`. |

---

## Task 1: Update Django footer and text body

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/delivery_reporting.py:28-44`
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/delivery_reporting.py:120-125` and `:154-155`

- [ ] **Step 1.1: Change `_build_preferences_footer` to return Postmark-style HTML**

Current:
```python
return (
    f'<p style="font-size:12px;color:#888888;margin-top:24px;">'
    f"{practice.name} &middot; "
    f'<a href="{url}">Update your email preferences</a>'
    f"</p>"
)
```

Change to:
```python
return (
    f'<p style="font-size:12px;color:#888888;text-align:center;margin-top:24px;">'
    f'<a href="{url}" style="color:#888888;text-decoration:underline;">Unsubscribe</a>'
    f"</p>"
)
```

- [ ] **Step 1.2: Add a plain-text unsubscribe helper and include it in the text body**

Add a new helper next to `_build_preferences_footer`:
```python
def _build_text_unsubscribe_line(practice, person, custom_unsubscribe_url=None):
    """Plain-text opt-out line for the TextBody payload."""
    token, _ = MarketingPreferenceToken.objects.get_or_create(
        practice=practice, person=person
    )
    if custom_unsubscribe_url:
        url = custom_unsubscribe_url
    else:
        url = f"{settings.FRONTEND_URL}/marketing-preferences/{token.token}"
    return f"Unsubscribe: {url}"
```

In `send_broadcast_email`, when `person is not None`, append the line to `text_body`:
```python
if person is None:
    ...
else:
    rendered_html = render_and_validate(...) + _build_preferences_footer(...)
    text_body = render_source.plain_text_fallback + "\n\n" + _build_text_unsubscribe_line(
        practice, person, campaign.custom_unsubscribe_url or None
    )
```

Then pass `text_body=text_body` to `client.send_email(...)` instead of `render_source.plain_text_fallback`.

- [ ] **Step 1.3: Run Django tests for the send path**

Run:
```bash
cd TreatmentPathBackend/TreatmentPath
source venv/bin/activate
python manage.py test marketingBroadcast.tests.SendBroadcastEmailTests marketingBroadcast.tests.SendBroadcastEmailWithSelfAuthoredTemplateTests --keepdb
```

---

## Task 2: Add Go Postmark client method

**Files:**
- Modify: `EmailServiceGo/internal/marketing/postmark_client.go`

- [ ] **Step 2.1: Add request/response types and `SetCustomUnsubscribeHandling`**

Add after `VerifyDomainResponse`:
```go
type SetMessageStreamRequest struct {
    SubscriptionManagementConfiguration struct {
        UnsubscribeHandlingType string `json:"UnsubscribeHandlingType"`
    } `json:"SubscriptionManagementConfiguration"`
}

type SetMessageStreamResponse struct {
    ID string `json:"ID"`
    SubscriptionManagementConfiguration struct {
        UnsubscribeHandlingType string `json:"UnsubscribeHandlingType"`
    } `json:"SubscriptionManagementConfiguration"`
}
```

Add method:
```go
func (c *Client) SetCustomUnsubscribeHandling(ctx context.Context) (*SetMessageStreamResponse, error) {
    body, err := json.Marshal(SetMessageStreamRequest{})
    if err != nil {
        return nil, fmt.Errorf("marshal message-stream body: %w", err)
    }

    req, err := http.NewRequestWithContext(ctx, http.MethodPatch, c.baseURL+"/message-streams/broadcast", bytes.NewReader(body))
    if err != nil {
        return nil, fmt.Errorf("build message-stream request: %w", err)
    }
    req.Header.Set("Accept", "application/json")
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Postmark-Server-Token", c.serverToken)

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("call postmark message-streams: %w", err)
    }
    defer resp.Body.Close()

    if resp.StatusCode >= 300 {
        respBody, _ := io.ReadAll(resp.Body)
        return nil, fmt.Errorf("postmark message-streams returned %d: %s", resp.StatusCode, string(respBody))
    }

    var out SetMessageStreamResponse
    if err := json.NewDecoder(resp.Body).Decode(&out); err != nil {
        return nil, fmt.Errorf("decode postmark message-streams response: %w", err)
    }
    return &out, nil
}
```

- [ ] **Step 2.2: Add a client test for the new method**

Add to `EmailServiceGo/internal/marketing/postmark_client_test.go`:
```go
func TestSetCustomUnsubscribeHandlingSendsPatchAndParsesResponse(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path != "/message-streams/broadcast" {
            t.Fatalf("expected path /message-streams/broadcast, got %s", r.URL.Path)
        }
        if r.Method != http.MethodPatch {
            t.Fatalf("expected PATCH, got %s", r.Method)
        }
        if r.Header.Get("X-Postmark-Server-Token") != "server-token-456" {
            t.Fatalf("expected server token header, got %q", r.Header.Get("X-Postmark-Server-Token"))
        }
        var body SetMessageStreamRequest
        json.NewDecoder(r.Body).Decode(&body)
        if body.SubscriptionManagementConfiguration.UnsubscribeHandlingType != "Custom" {
            t.Fatalf("expected UnsubscribeHandlingType=Custom, got %q", body.SubscriptionManagementConfiguration.UnsubscribeHandlingType)
        }
        resp := SetMessageStreamResponse{ID: "broadcast"}
        resp.SubscriptionManagementConfiguration.UnsubscribeHandlingType = "Custom"
        json.NewEncoder(w).Encode(resp)
    }))
    defer server.Close()

    client := NewClient(&Config{BaseURL: server.URL, ServerToken: "server-token-456"})
    result, err := client.SetCustomUnsubscribeHandling(context.Background())
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if result.ID != "broadcast" {
        t.Errorf("expected ID=broadcast, got %q", result.ID)
    }
    if result.SubscriptionManagementConfiguration.UnsubscribeHandlingType != "Custom" {
        t.Errorf("expected UnsubscribeHandlingType=Custom, got %q", result.SubscriptionManagementConfiguration.UnsubscribeHandlingType)
    }
}
```

- [ ] **Step 2.3: Run Go client tests**

Run:
```bash
cd EmailServiceGo
go test ./internal/marketing/... -run TestSetCustomUnsubscribeHandling
```

---

## Task 3: Wire stream configuration into domain verification

**Files:**
- Modify: `EmailServiceGo/internal/marketing/handlers.go:76-115`

- [ ] **Step 3.1: Call `SetCustomUnsubscribeHandling` after DKIM/Return-Path verify**

After:
```go
if dkimResult.DKIMVerified && returnPathResult.ReturnPathDomainVerified {
    now := time.Now()
    domain.Status = "verified"
    domain.VerifiedAt = &now
}
```

add (inside the same block, before `h.db.Save`):
```go
if _, err := h.client.SetCustomUnsubscribeHandling(ctx); err != nil {
    // Some Postmark accounts are not approved for Custom unsubscribe handling.
    // Log but don't fail verification; the domain is still usable, Postmark will
    // just inject its own unsubscribe link.
    fmt.Printf("warning: failed to set broadcast stream Custom unsubscribe handling: %v\n", err)
}
```

- [ ] **Step 3.2: Update handler tests to expect the stream PATCH call**

In `EmailServiceGo/internal/marketing/handlers_test.go`, update `TestVerifyDomainMarksVerifiedWhenBothChecksPass` and `TestVerifyDomainStaysPendingWhenOnlyOneCheckPasses` so the Postmark mock also handles `/message-streams/broadcast`. For the pending test, the PATCH should still be attempted (verification logic runs after both checks return), but the status should stay pending because not both checks passed.

For `TestVerifyDomainMarksVerifiedWhenBothChecksPass`, extend the mock:
```go
case "/message-streams/broadcast":
    json.NewEncoder(w).Encode(SetMessageStreamResponse{ID: "broadcast"})
```

- [ ] **Step 3.3: Add a test that stream update failure doesn't fail verification**

Add to `handlers_test.go`:
```go
func TestVerifyDomainSucceedsEvenWhenStreamUpdateFails(t *testing.T) {
    postmarkServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        switch r.URL.Path {
        case "/domains/36736/verifyDkim":
            json.NewEncoder(w).Encode(VerifyDomainResponse{ID: 36736, DKIMVerified: true})
        case "/domains/36736/verifyReturnPath":
            json.NewEncoder(w).Encode(VerifyDomainResponse{ID: 36736, ReturnPathDomainVerified: true})
        case "/message-streams/broadcast":
            w.WriteHeader(http.StatusUnprocessableEntity)
            w.Write([]byte(`{"ErrorCode":500,"Message":"Custom unsubscribe handling not enabled for account"}`))
        }
    }))
    defer postmarkServer.Close()

    handler, db := setupTestHandler(t, postmarkServer.URL)
    db.Create(&MarketingDomain{PracticeID: 42, FullDomain: "news.smiledentistry.com", PostmarkDomainID: 36736, Status: "pending"})

    router := gin.New()
    router.POST("/domains/:practice_id/verify", handler.VerifyDomain)

    req := httptest.NewRequest(http.MethodPost, "/domains/42/verify", nil)
    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("expected 200, got %d: %s", w.Code, w.Body.String())
    }

    var stored MarketingDomain
    db.First(&stored, "practice_id = ?", 42)
    if stored.Status != "verified" {
        t.Errorf("expected status=verified, got %q", stored.Status)
    }
}
```

- [ ] **Step 3.4: Run Go handler tests**

Run:
```bash
cd EmailServiceGo
go test ./internal/marketing/... -run TestVerifyDomain
```

---

## Task 4: Update Django tests

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/tests.py`

- [ ] **Step 4.1: Add assertions for new footer style and text body**

In `SendBroadcastEmailTests`:
- Keep `test_email_includes_preferences_footer_link` but add assertions that the body contains the Postmark-style centered link text and styling.
- Add a new test `test_text_body_includes_unsubscribe_link` that inspects the `text_body` argument passed to `send_email` and asserts it contains the token URL and plain "Unsubscribe" text.

Example:
```python
@patch("marketingBroadcast.delivery_reporting.MarketingEmailServiceClient")
def test_email_includes_postmark_style_unsubscribe_footer(self, mock_client_cls):
    mock_client_cls.return_value.send_email.return_value = {"message_id": "msg-3"}
    email_message = real_send_broadcast_email(self.recipient)

    token = MarketingPreferenceToken.objects.get(
        practice=self.practice, person=self.person
    )
    self.assertIn(f"/marketing-preferences/{token.token}", email_message.body)
    self.assertIn(">Unsubscribe</a>", email_message.body)
    self.assertIn("text-align:center", email_message.body)
```

- [ ] **Step 4.2: Remove/update any tests expecting the old footer text**

Search for `"Update your email preferences"` in the test file. Update any assertions to expect the new "Unsubscribe" text or the token URL.

- [ ] **Step 4.3: Run full Django marketingBroadcast tests**

Run:
```bash
cd TreatmentPathBackend/TreatmentPath
source venv/bin/activate
python manage.py test marketingBroadcast.tests --keepdb
```

---

## Task 5: Full verification

- [ ] **Step 5.1: Run Go tests**

```bash
cd EmailServiceGo
go test ./internal/marketing/...
```

- [ ] **Step 5.2: Run Django tests**

```bash
cd TreatmentPathBackend/TreatmentPath
source venv/bin/activate
python manage.py test marketingBroadcast.tests --keepdb
```

- [ ] **Step 5.3: Run formatters/linters**

For Go:
```bash
cd EmailServiceGo
gofmt -w internal/marketing/postmark_client.go internal/marketing/handlers.go internal/marketing/postmark_client_test.go internal/marketing/handlers_test.go
```

For Python (black/isort as configured in `TreatmentPathBackend/TreatmentPath/pyproject.toml`):
```bash
cd TreatmentPathBackend/TreatmentPath
source venv/bin/activate
black marketingBroadcast/delivery_reporting.py marketingBroadcast/tests.py
isort marketingBroadcast/delivery_reporting.py marketingBroadcast/tests.py
```

---

## Self-review

**Spec coverage:**
1. Postmark-style footer in Django — Task 1.1.
2. Plain-text unsubscribe line — Task 1.2.
3. Go `SetCustomUnsubscribeHandling` — Task 2.
4. Wire into verification flow, log errors, don't fail — Task 3.
5. Tests updated — Tasks 2, 3, 4.
6. Full verification — Task 5.

**Placeholder scan:** No placeholders.

**Type consistency:** `SetMessageStreamRequest`/`SetMessageStreamResponse` reused in client and tests.

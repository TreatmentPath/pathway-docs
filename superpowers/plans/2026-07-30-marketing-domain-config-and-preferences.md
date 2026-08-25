# Marketing Domain Config & Patient Preferences Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a practice set their marketing domain's email prefix/reply-to addresses, and make the existing-but-unreachable patient preferences API actually reachable — via a footer link in every marketing email and a public page for patients to land on.

**Architecture:** Two independent backend additions (a `PATCH` endpoint on the existing `MarketingSendingDomain` config, and a footer-injection helper called from the existing `send_broadcast_email`), plus three frontend additions (two new form sections on the existing `MarketingDomainConfigView`, and one new public page + route). No new models — reuses `MarketingSendingDomain`, `GoMarketingDomain`, and `MarketingPreferenceToken` exactly as they exist today.

**Tech Stack:** Django REST Framework (backend), React + TypeScript + Vitest (frontend), existing `useFetchWithAuth`/plain `fetch` conventions.

**Spec:** [`PRD/Broadcasts/specs/2026-07-30-marketing-domain-config-and-preferences-design.md`](../../../PRD/Broadcasts/specs/2026-07-30-marketing-domain-config-and-preferences-design.md)

---

### Task 1: Backend — `PATCH /marketing/domains/config/` endpoint

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/views/domain_views.py`
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/urls.py`
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/tests.py:1152` (insert before line 1154's `from TreatmentPlan.models import ContactChannel, PersonChannel`)

- [ ] **Step 1: Write the failing tests**

Insert this new test class into `tests.py` right after `MarketingDomainViewTests` ends (after line 1152, before the blank line + `from TreatmentPlan.models import ContactChannel, PersonChannel` import at line 1154):

```python
class UpdateMarketingDomainConfigViewTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(name="Config Test Practice")
        self.user = User.objects.create_user(
            email="configstaff@example.com",
            password="password123",
            current_practice=self.practice,
        )
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)
        GoMarketingDomain.objects.create(
            id=uuid.uuid4(),
            practice_id=self.practice.id,
            full_domain="news.example.com",
            status="verified",
        )

    def test_updates_email_prefix(self):
        response = self.client.patch(
            "/api/backend/marketing/domains/config/",
            {"email_prefix": "hello"},
            format="json",
        )
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.data["domain"]["email_prefix"], "hello")
        config = MarketingSendingDomain.objects.get(practice=self.practice)
        self.assertEqual(config.email_prefix, "hello")

    def test_updates_reply_to(self):
        response = self.client.patch(
            "/api/backend/marketing/domains/config/",
            {"reply_to": ["reception@example.com"]},
            format="json",
        )
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.data["domain"]["reply_to"], ["reception@example.com"])

    def test_rejects_more_than_five_reply_to_addresses(self):
        addresses = [f"addr{i}@example.com" for i in range(6)]
        response = self.client.patch(
            "/api/backend/marketing/domains/config/",
            {"reply_to": addresses},
            format="json",
        )
        self.assertEqual(response.status_code, 400)
        self.assertIn("reply_to", response.data["error"])

    def test_rejects_invalid_email_in_reply_to(self):
        response = self.client.patch(
            "/api/backend/marketing/domains/config/",
            {"reply_to": ["not-an-email"]},
            format="json",
        )
        self.assertEqual(response.status_code, 400)
        self.assertIn("not-an-email", response.data["error"])

    def test_requires_registered_domain(self):
        GoMarketingDomain.objects.filter(practice_id=self.practice.id).delete()
        response = self.client.patch(
            "/api/backend/marketing/domains/config/",
            {"email_prefix": "hello"},
            format="json",
        )
        self.assertEqual(response.status_code, 404)

    def test_requires_authentication(self):
        anonymous_client = APIClient()
        response = anonymous_client.patch(
            "/api/backend/marketing/domains/config/",
            {"email_prefix": "hello"},
            format="json",
        )
        self.assertEqual(response.status_code, 401)
```

- [ ] **Step 2: Run tests to verify they fail**

Run (from `TreatmentPathBackend/TreatmentPath/`, with venv activated):
```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
python manage.py test marketingBroadcast.tests.UpdateMarketingDomainConfigViewTests --keepdb -v 2
```
Expected: FAIL — `NoReverseMatch` or 404, since the URL/view don't exist yet.

- [ ] **Step 3: Add the view**

In `domain_views.py`, add this class after `VerifyMarketingDomainView` (at the end of the file):

```python
class UpdateMarketingDomainConfigView(APIView):
    """Updates the practice's own MarketingSendingDomain config
    (email_prefix/reply_to) — never touches GoMarketingDomain."""

    permission_classes = [IsAuthenticated]

    def patch(self, request):
        practice = request.user.current_practice
        go_domain = GoMarketingDomain.objects.filter(practice_id=practice.id).first()
        if not go_domain:
            return Response(
                {"error": "no marketing domain registered for this practice"},
                status=status.HTTP_404_NOT_FOUND,
            )

        reply_to = request.data.get("reply_to")
        if reply_to is not None:
            if len(reply_to) > 5:
                return Response(
                    {"error": "reply_to: maximum 5 addresses allowed"},
                    status=status.HTTP_400_BAD_REQUEST,
                )
            from django.core.exceptions import ValidationError
            from django.core.validators import validate_email

            for address in reply_to:
                try:
                    validate_email(address)
                except ValidationError:
                    return Response(
                        {"error": f"reply_to: '{address}' is not a valid email address"},
                        status=status.HTTP_400_BAD_REQUEST,
                    )

        config, _ = MarketingSendingDomain.objects.get_or_create(practice=practice)
        if "email_prefix" in request.data:
            config.email_prefix = request.data["email_prefix"]
        if reply_to is not None:
            config.reply_to = reply_to
        config.save()

        return Response({"domain": _serialize_domain(go_domain, config)})
```

- [ ] **Step 4: Wire the URL**

In `urls.py`, add the import and the path. Update the import block at the top:

```python
from marketingBroadcast.views.domain_views import (
    MarketingDomainStatusView,
    RegisterMarketingDomainView,
    UpdateMarketingDomainConfigView,
    VerifyMarketingDomainView,
)
```

And add this path right after the `domains/verify/` path:

```python
    # Staff-facing: update email_prefix/reply_to on the practice's marketing
    # send config (MarketingSendingDomain) — never touches GoMarketingDomain.
    path(
        "domains/config/",
        UpdateMarketingDomainConfigView.as_view(),
        name="marketing-domain-config",
    ),
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
python manage.py test marketingBroadcast.tests.UpdateMarketingDomainConfigViewTests --keepdb -v 2
```
Expected: `OK` (6 tests pass).

- [ ] **Step 6: Run the full marketingBroadcast suite to check for regressions**

```bash
python manage.py test marketingBroadcast --keepdb
```
Expected: `OK`, all existing tests still pass.

- [ ] **Step 7: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/marketingBroadcast/views/domain_views.py TreatmentPathBackend/TreatmentPath/marketingBroadcast/urls.py TreatmentPathBackend/TreatmentPath/marketingBroadcast/tests.py
git commit -m "feat(marketing): add PATCH endpoint for domain email_prefix/reply_to config"
```

---

### Task 2: Backend — preferences footer link in outgoing emails

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/delivery_reporting.py`
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/tests.py:2075` (insert before line 2077's `from marketingBroadcast.delivery_reporting import refresh_campaign_counters`)

- [ ] **Step 1: Write the failing tests**

Insert these two test methods into the existing `SendBroadcastEmailTests` class, right after `test_no_verified_marketing_domain_raises` (after line 2075, still inside the class — before the blank lines and next imports at 2077-2079):

```python
    @patch("marketingBroadcast.delivery_reporting.MarketingEmailServiceClient")
    def test_email_includes_preferences_footer_link(self, mock_client_cls):
        mock_client_cls.return_value.send_email.return_value = {"message_id": "msg-3"}
        email_message = real_send_broadcast_email(self.recipient)

        token = MarketingPreferenceToken.objects.get(
            practice=self.practice, person=self.person
        )
        self.assertIn(f"/marketing-preferences/{token.token}", email_message.body)

    @patch("marketingBroadcast.delivery_reporting.MarketingEmailServiceClient")
    def test_reuses_same_token_across_sends(self, mock_client_cls):
        mock_client_cls.return_value.send_email.return_value = {"message_id": "msg-4"}
        real_send_broadcast_email(self.recipient)
        real_send_broadcast_email(self.recipient)

        self.assertEqual(
            MarketingPreferenceToken.objects.filter(
                practice=self.practice, person=self.person
            ).count(),
            1,
        )

    @patch("marketingBroadcast.delivery_reporting.MarketingEmailServiceClient")
    def test_test_send_also_includes_footer(self, mock_client_cls):
        mock_client_cls.return_value.send_email.return_value = {"message_id": "msg-5"}
        email_message = real_send_broadcast_email(self.recipient, is_test=True)
        self.assertIn("/marketing-preferences/", email_message.body)
```

Add the `MarketingPreferenceToken` import. Find this existing import line near the top of `SendBroadcastEmailTests`' section (around line 2001, right before `class SendBroadcastEmailTests`):

```python
from marketingBroadcast.models import GoMarketingDomain
```

Change it to:

```python
from marketingBroadcast.models import GoMarketingDomain, MarketingPreferenceToken
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python manage.py test marketingBroadcast.tests.SendBroadcastEmailTests --keepdb -v 2
```
Expected: FAIL on the 3 new tests — no `/marketing-preferences/` link in `email_message.body` yet.

- [ ] **Step 3: Add the footer helper and wire it in**

In `delivery_reporting.py`, add the import and the helper function. Update the imports at the top:

```python
from django.conf import settings

from activityLog.models import ActivityLogHelper
from marketingBroadcast.marketing_email_client import MarketingEmailServiceClient
from marketingBroadcast.models import (
    BroadcastCampaignCounters,
    BroadcastRecipient,
    GoMarketingDomain,
    MarketingPreferenceToken,
    MarketingSendingDomain,
)
from marketingBroadcast.template_engine import render_and_validate
from messaging.models import EmailMessages
```

Add this new function right after `_resolve_recipient_email`:

```python
def _build_preferences_footer(practice, person):
    """Non-removable — deliberately not a SAFE_PERSONALISATION_FIELDS template
    token, so no template author can omit it. Reuses the same token
    indefinitely per (practice, person), matching MarketingPreferenceToken's
    documented no-expiry design (see consent-ledger spec)."""
    token, _ = MarketingPreferenceToken.objects.get_or_create(
        practice=practice, person=person
    )
    url = f"{settings.FRONTEND_URL}/marketing-preferences/{token.token}"
    return (
        f'<p style="font-size:12px;color:#888888;margin-top:24px;">'
        f"{practice.name} &middot; "
        f'<a href="{url}">Update your email preferences</a>'
        f"</p>"
    )
```

In `send_broadcast_email`, find this line:

```python
    rendered_html = render_and_validate(
        campaign.content_snapshot, campaign.template_version, person, practice
    )
```

Change it to:

```python
    rendered_html = render_and_validate(
        campaign.content_snapshot, campaign.template_version, person, practice
    )
    rendered_html += _build_preferences_footer(practice, person)
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test marketingBroadcast.tests.SendBroadcastEmailTests --keepdb -v 2
```
Expected: `OK` (6 tests pass — 3 existing + 3 new).

- [ ] **Step 5: Run the full marketingBroadcast suite to check for regressions**

```bash
python manage.py test marketingBroadcast --keepdb
```
Expected: `OK`.

- [ ] **Step 6: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/marketingBroadcast/delivery_reporting.py TreatmentPathBackend/TreatmentPath/marketingBroadcast/tests.py
git commit -m "feat(marketing): append non-removable preferences link footer to outgoing emails"
```

---

### Task 3: Frontend — API endpoint config

**Files:**
- Modify: `perfect-pixel-playground-project/src/config/api.ts:1524-1529`

- [ ] **Step 1: Add the new endpoint builders**

Find the existing `domain` block inside `marketing`:

```typescript
    domain: {
      // Read-only — does not call Postmark, unlike register/verify below
      status: () => getApiUrl('/marketing/domains/status/'),
      register: () => getApiUrl('/marketing/domains/register/'),
      verify: () => getApiUrl('/marketing/domains/verify/'),
    },
  },
```

Replace it with:

```typescript
    domain: {
      // Read-only — does not call Postmark, unlike register/verify below
      status: () => getApiUrl('/marketing/domains/status/'),
      register: () => getApiUrl('/marketing/domains/register/'),
      verify: () => getApiUrl('/marketing/domains/verify/'),
      updateConfig: () => getApiUrl('/marketing/domains/config/'),
    },
    preferences: {
      // Public, unauthenticated — token-scoped patient preferences page
      get: (token: string) => getApiUrl(`/marketing/preferences/${token}/`),
      update: (token: string) => getApiUrl(`/marketing/preferences/${token}/`),
    },
  },
```

- [ ] **Step 2: Verify TypeScript compiles**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npm run typecheck 2>&1 | grep -i "config/api.ts"
```
Expected: no output (no errors referencing this file).

- [ ] **Step 3: Commit**

```bash
git add src/config/api.ts
git commit -m "feat(marketing): add domain config and preferences endpoint builders"
```

---

### Task 4: Frontend — email prefix + reply-to editing UI

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/settings/components/practice/MarketingDomainConfigView.tsx`
- Test: `perfect-pixel-playground-project/src/pages/settings/components/practice/MarketingDomainConfigView.test.tsx` (new)

- [ ] **Step 1: Write the failing test**

Create `MarketingDomainConfigView.test.tsx`:

```typescript
import { MemoryRouter } from "react-router-dom";
import { afterEach, describe, expect, it, vi } from "vitest";
import { fireEvent, render, screen, waitFor } from "@testing-library/react";

vi.mock("@/lib/helpers", () => ({
  useFetchWithAuth: () => mockFetchWithAuth,
}));

const mockFetchWithAuth = vi.fn();

import MarketingDomainConfigView from "./MarketingDomainConfigView";

function jsonResponse(body: unknown, ok = true) {
  return Promise.resolve({
    ok,
    json: () => Promise.resolve(body),
  } as Response);
}

const statusResponse = {
  registered: true,
  domain: {
    full_domain: "news.example.com",
    status: "pending",
    dkim_host: "dkim.host",
    dkim_value: "dkim-value",
    return_path_host: "rp.host",
    return_path_value: "rp-value",
    registration_error: "",
    email_prefix: "",
    reply_to: [],
  },
};

function renderView() {
  return render(
    <MemoryRouter>
      <MarketingDomainConfigView onBack={() => {}} />
    </MemoryRouter>,
  );
}

describe("MarketingDomainConfigView sender config", () => {
  afterEach(() => {
    vi.restoreAllMocks();
    mockFetchWithAuth.mockReset();
  });

  it("saves the email prefix", async () => {
    mockFetchWithAuth.mockImplementation((url: string) => {
      if (url.includes("/domains/status/")) return jsonResponse(statusResponse);
      if (url.includes("/emailService/domains") || url.includes("/domains")) return jsonResponse({ domains: [] });
      if (url.includes("/inbox-addresses")) return jsonResponse([]);
      if (url.includes("/domains/config/")) {
        return jsonResponse({
          domain: { ...statusResponse.domain, email_prefix: "hello" },
        });
      }
      return jsonResponse({});
    });

    renderView();

    const prefixInput = await screen.findByLabelText("Email prefix");
    fireEvent.change(prefixInput, { target: { value: "hello" } });
    fireEvent.click(screen.getByRole("button", { name: "Save prefix" }));

    await waitFor(() =>
      expect(mockFetchWithAuth).toHaveBeenCalledWith(
        expect.stringContaining("/domains/config/"),
        expect.objectContaining({ method: "PATCH" }),
      ),
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vitest run src/pages/settings/components/practice/MarketingDomainConfigView.test.tsx
```
Expected: FAIL — no "Email prefix" label / "Save prefix" button exists yet.

- [ ] **Step 3: Add state, fetch, and the email-prefix section**

`useFetchWithAuth` is already imported and `fetchWithAuth` is already declared in this file (`const fetchWithAuth = useFetchWithAuth();`, used by `fetchDomainData`/`handleVerifyDomain`) — reuse it as-is, do not re-import or re-declare it.

The `MarketingSendingDomain` interface already has `email_prefix: string` and `reply_to: string[]` from earlier session work — no change needed there.

Add one new interface near the top, alongside the existing ones:

```typescript
interface ReplyToCandidate {
  address: string;
  label?: string;
}
```

Inside the component, add new state and a save handler right after the existing `copiedValue`/`verifying` state declarations:

```typescript
  const [emailPrefix, setEmailPrefix] = useState("");
  const [savingPrefix, setSavingPrefix] = useState(false);
  const [replyToCandidates, setReplyToCandidates] = useState<ReplyToCandidate[]>([]);
  const [replyTo, setReplyTo] = useState<string[]>([]);
  const [savingReplyTo, setSavingReplyTo] = useState(false);
```

Add a fetch for reply-to candidates (existing verified domains + active inbox addresses), called once on mount alongside `fetchDomainData`. Add this function after `fetchDomainData`:

```typescript
  const fetchReplyToCandidates = async () => {
    // domains.list() response is dual-shape (PascalCase or snake_case depending
    // on the Go endpoint hit) — matches the exact pattern PracticeCommunicationCard's
    // fetchDomainsData already uses. inboxAddresses.list() is always snake_case
    // under a `routing_addresses` key — matches fetchInboxAddressesData there.
    type RawDomain = { Status?: string; status?: string; Type?: string; type?: string; Domain?: string; domain?: string };
    type RawInboxAddress = { is_active?: boolean; full_address?: string };
    try {
      const [domainsRes, inboxRes] = await Promise.all([
        fetchWithAuth(API_ENDPOINTS.emailService.domains.list()),
        fetchWithAuth(API_ENDPOINTS.emailService.inboxAddresses.list()),
      ]);
      const domainsData = domainsRes.ok ? await domainsRes.json() : { domains: [] };
      const inboxData = inboxRes.ok ? await inboxRes.json() : { routing_addresses: [] };

      const domainsList: RawDomain[] = Array.isArray(domainsData) ? domainsData : domainsData.domains || [];
      const inboxList: RawInboxAddress[] = inboxData.routing_addresses || [];

      const candidates: ReplyToCandidate[] = [
        ...domainsList
          .filter((d) => (d.Status || d.status) === "verified" && (d.Type || d.type) !== "inbox_address")
          .map((d) => ({ address: `updates@${d.Domain || d.domain}` })),
        ...inboxList
          .filter((a) => a.is_active && a.full_address)
          .map((a) => ({ address: a.full_address as string, label: "Inbox" })),
      ];
      setReplyToCandidates(candidates);
    } catch (error) {
      console.error("Failed to fetch reply-to candidates:", error);
    }
  };
```

Sync `emailPrefix`/`replyTo` local state whenever `domainData` loads — update the existing `useEffect(() => { fetchDomainData(); }, []);` block:

```typescript
  useEffect(() => {
    fetchDomainData();
    fetchReplyToCandidates();
  }, []);

  useEffect(() => {
    if (domainData) {
      setEmailPrefix(domainData.email_prefix || "");
      setReplyTo(domainData.reply_to || []);
    }
  }, [domainData]);
```

Add the save handlers, right after `handleVerifyDomain`:

```typescript
  const handleSaveEmailPrefix = async () => {
    try {
      setSavingPrefix(true);
      const res = await fetchWithAuth(API_ENDPOINTS.marketing.domain.updateConfig(), {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ email_prefix: emailPrefix }),
      });
      if (!res.ok) {
        const errorData = await res.json();
        throw new Error(extractErrorMessage(errorData, "Failed to save email prefix"));
      }
      toast.success("Email prefix saved");
      await fetchDomainData();
    } catch (err) {
      toast.error(err instanceof Error ? err.message : "Failed to save email prefix");
    } finally {
      setSavingPrefix(false);
    }
  };

  const handleToggleReplyTo = async (address: string) => {
    const isSelected = replyTo.includes(address);
    let newList: string[];
    if (isSelected) {
      newList = replyTo.filter((a) => a !== address);
    } else {
      if (replyTo.length >= 5) {
        toast.error("Maximum 5 reply-to addresses allowed");
        return;
      }
      newList = [...replyTo, address];
    }

    setSavingReplyTo(true);
    try {
      const res = await fetchWithAuth(API_ENDPOINTS.marketing.domain.updateConfig(), {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ reply_to: newList }),
      });
      if (!res.ok) {
        const errorData = await res.json();
        throw new Error(extractErrorMessage(errorData, "Failed to update reply-to addresses"));
      }
      setReplyTo(newList);
      toast.success(isSelected ? `Removed ${address}` : `Added ${address}`);
    } catch (err) {
      toast.error(err instanceof Error ? err.message : "Failed to update reply-to addresses");
    } finally {
      setSavingReplyTo(false);
    }
  };

  const handleClearAllReplyTo = async () => {
    setSavingReplyTo(true);
    try {
      const res = await fetchWithAuth(API_ENDPOINTS.marketing.domain.updateConfig(), {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ reply_to: [] }),
      });
      if (!res.ok) {
        const errorData = await res.json();
        throw new Error(extractErrorMessage(errorData, "Failed to clear reply-to addresses"));
      }
      setReplyTo([]);
      toast.success("Cleared all reply-to addresses");
    } catch (err) {
      toast.error(err instanceof Error ? err.message : "Failed to clear reply-to addresses");
    } finally {
      setSavingReplyTo(false);
    }
  };
```

- [ ] **Step 4: Add the email prefix + reply-to JSX sections**

In the render, right after the Return-Path `Card` closes and before the trailing `{domainData.registration_error && (...)}` block, add:

```tsx
      {/* Email prefix */}
      <Card className="border-0 bg-transparent shadow-none">
        <CardHeader className="pt-0 px-0">
          <CardTitle className="text-sm">Sender email prefix</CardTitle>
          <CardDescription>
            Emails send from {emailPrefix || "updates"}@{domainData.full_domain}
          </CardDescription>
        </CardHeader>
        <CardContent className="px-0">
          <div className="flex items-center gap-3">
            <Label htmlFor="marketing-email-prefix" className="text-xs min-w-[80px]">
              Email prefix
            </Label>
            <Input
              id="marketing-email-prefix"
              value={emailPrefix}
              onChange={(e) => setEmailPrefix(e.target.value)}
              placeholder="updates"
              className="max-w-xs"
            />
            <Button size="sm" onClick={handleSaveEmailPrefix} disabled={savingPrefix}>
              {savingPrefix ? <Loader2 className="w-3.5 h-3.5 animate-spin" /> : "Save prefix"}
            </Button>
          </div>
        </CardContent>
      </Card>

      {/* Reply-to addresses */}
      <Card className="border-0 bg-transparent shadow-none">
        <CardHeader className="pt-0 px-0">
          <div className="flex items-center justify-between">
            <div>
              <CardTitle className="text-sm">Reply-to addresses</CardTitle>
              <CardDescription>Where replies are sent. Up to 5. Leave empty to use the from address.</CardDescription>
            </div>
            {replyTo.length > 0 && (
              <Button
                size="sm"
                variant="ghost"
                onClick={handleClearAllReplyTo}
                disabled={savingReplyTo}
                className="text-xs text-gray-400 hover:text-red-500 hover:bg-red-50"
              >
                {savingReplyTo ? <Loader2 className="w-3 h-3 animate-spin" /> : "Clear all"}
              </Button>
            )}
          </div>
        </CardHeader>
        <CardContent className="px-0">
          {replyToCandidates.length === 0 ? (
            <p className="text-sm text-gray-400">No verified domains or inbox addresses to pick from yet.</p>
          ) : (
            <div className="divide-y border rounded-md overflow-hidden">
              {replyToCandidates.map((candidate) => {
                const isSelected = replyTo.includes(candidate.address);
                return (
                  <div
                    key={candidate.address}
                    className={`flex items-center justify-between px-3 py-2.5 cursor-pointer transition-colors ${isSelected ? "bg-primary/5" : "bg-white hover:bg-gray-50"}`}
                    onClick={() => !savingReplyTo && handleToggleReplyTo(candidate.address)}
                  >
                    <div className="flex items-center gap-2">
                      <span className={`text-sm ${isSelected ? "text-primary font-medium" : "text-gray-700"}`}>
                        {candidate.address}
                      </span>
                      {candidate.label && (
                        <Badge variant="secondary" className="text-xs border-0">
                          {candidate.label}
                        </Badge>
                      )}
                    </div>
                    <div
                      className={`w-4 h-4 border rounded flex items-center justify-center shrink-0 ${isSelected ? "bg-primary border-primary" : "border-gray-300"}`}
                    >
                      {isSelected && <Check className="w-3 h-3 text-white" />}
                    </div>
                  </div>
                );
              })}
            </div>
          )}
        </CardContent>
      </Card>
```

- [ ] **Step 5: Run test to verify it passes**

```bash
npx vitest run src/pages/settings/components/practice/MarketingDomainConfigView.test.tsx
```
Expected: PASS.

- [ ] **Step 6: Typecheck and lint**

```bash
npm run typecheck 2>&1 | grep -i MarketingDomainConfigView
npx eslint src/pages/settings/components/practice/MarketingDomainConfigView.tsx
```
Expected: no output from typecheck grep, 0 problems from eslint.

- [ ] **Step 7: Commit**

```bash
git add src/pages/settings/components/practice/MarketingDomainConfigView.tsx src/pages/settings/components/practice/MarketingDomainConfigView.test.tsx
git commit -m "feat(marketing): add email prefix and reply-to editing to domain config view"
```

---

### Task 5: Frontend — public patient preferences page

**Files:**
- Create: `perfect-pixel-playground-project/src/pages/MarketingPreferencesPage.tsx`
- Test: `perfect-pixel-playground-project/src/pages/MarketingPreferencesPage.test.tsx` (new)
- Modify: `perfect-pixel-playground-project/src/App.tsx`

- [ ] **Step 1: Write the failing test**

Create `MarketingPreferencesPage.test.tsx`:

```typescript
import { MemoryRouter, Route, Routes } from "react-router-dom";
import { afterEach, describe, expect, it, vi } from "vitest";
import { fireEvent, render, screen, waitFor } from "@testing-library/react";

import MarketingPreferencesPage from "./MarketingPreferencesPage";

function renderPage(token = "abc123") {
  return render(
    <MemoryRouter initialEntries={[`/marketing-preferences/${token}`]}>
      <Routes>
        <Route path="/marketing-preferences/:token" element={<MarketingPreferencesPage />} />
      </Routes>
    </MemoryRouter>,
  );
}

describe("MarketingPreferencesPage", () => {
  afterEach(() => {
    vi.restoreAllMocks();
  });

  it("shows the masked email and current state", async () => {
    vi.stubGlobal(
      "fetch",
      vi.fn().mockResolvedValue(
        new Response(
          JSON.stringify({
            masked_email: "j***@example.com",
            state: "on",
            note: "This toggle applies to marketing emails only.",
          }),
          { status: 200, headers: { "Content-Type": "application/json" } },
        ),
      ),
    );

    renderPage();

    expect(await screen.findByText("j***@example.com")).toBeInTheDocument();
    expect(screen.getByText("This toggle applies to marketing emails only.")).toBeInTheDocument();
  });

  it("toggles the state and posts the update", async () => {
    const fetchMock = vi.fn().mockImplementation((url: string, init?: RequestInit) => {
      if (!init || init.method === undefined) {
        return Promise.resolve(
          new Response(
            JSON.stringify({ masked_email: "j***@example.com", state: "on", note: "Note text" }),
            { status: 200, headers: { "Content-Type": "application/json" } },
          ),
        );
      }
      return Promise.resolve(
        new Response(JSON.stringify({ state: "off" }), {
          status: 200,
          headers: { "Content-Type": "application/json" },
        }),
      );
    });
    vi.stubGlobal("fetch", fetchMock);

    renderPage();

    const toggle = await screen.findByRole("switch");
    fireEvent.click(toggle);

    await waitFor(() =>
      expect(fetchMock).toHaveBeenCalledWith(
        expect.stringContaining("/marketing/preferences/abc123/"),
        expect.objectContaining({ method: "POST" }),
      ),
    );
  });

  it("shows an invalid-link message on 404", async () => {
    vi.stubGlobal(
      "fetch",
      vi.fn().mockResolvedValue(new Response(JSON.stringify({ error: "Not found" }), { status: 404 })),
    );

    renderPage();

    expect(await screen.findByText(/no longer valid/i)).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npx vitest run src/pages/MarketingPreferencesPage.test.tsx
```
Expected: FAIL — module doesn't exist yet.

- [ ] **Step 3: Create the page component**

Create `MarketingPreferencesPage.tsx`:

```typescript
import { useEffect, useState } from "react";
import { useParams } from "react-router-dom";
import { Loader2, AlertCircle } from "lucide-react";
import { Switch } from "@/components/ui/switch";
import { Alert, AlertDescription } from "@/components/ui/alert";
import { API_ENDPOINTS } from "@/config/api";

type PageState = "loading" | "ready" | "not_found" | "error";

interface PreferencesData {
  masked_email: string;
  state: "on" | "off";
  note: string;
}

export default function MarketingPreferencesPage() {
  const { token } = useParams<{ token: string }>();
  const [pageState, setPageState] = useState<PageState>("loading");
  const [data, setData] = useState<PreferencesData | null>(null);
  const [saving, setSaving] = useState(false);

  useEffect(() => {
    if (!token) return;
    (async () => {
      try {
        const res = await fetch(API_ENDPOINTS.marketing.preferences.get(token));
        if (res.status === 404) {
          setPageState("not_found");
          return;
        }
        if (!res.ok) {
          setPageState("error");
          return;
        }
        const body: PreferencesData = await res.json();
        setData(body);
        setPageState("ready");
      } catch {
        setPageState("error");
      }
    })();
  }, [token]);

  const handleToggle = async () => {
    if (!token || !data) return;
    const newState = data.state === "on" ? "off" : "on";
    setSaving(true);
    try {
      const res = await fetch(API_ENDPOINTS.marketing.preferences.update(token), {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ state: newState }),
      });
      if (!res.ok) {
        setPageState("error");
        return;
      }
      setData({ ...data, state: newState });
    } finally {
      setSaving(false);
    }
  };

  if (pageState === "loading") {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-50">
        <Loader2 className="w-8 h-8 animate-spin text-gray-400" />
      </div>
    );
  }

  if (pageState === "not_found") {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-50 px-4">
        <Alert className="max-w-md">
          <AlertCircle className="h-4 w-4" />
          <AlertDescription>This link is no longer valid.</AlertDescription>
        </Alert>
      </div>
    );
  }

  if (pageState === "error" || !data) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-gray-50 px-4">
        <Alert variant="destructive" className="max-w-md">
          <AlertCircle className="h-4 w-4" />
          <AlertDescription>Something went wrong. Please try again later.</AlertDescription>
        </Alert>
      </div>
    );
  }

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50 px-4">
      <div className="w-full max-w-md bg-white border rounded-lg shadow-sm p-6 space-y-4">
        <h1 className="text-lg font-semibold text-gray-900">Email preferences</h1>
        <p className="text-sm text-gray-600">
          Marketing emails to <span className="font-medium">{data.masked_email}</span>
        </p>
        <div className="flex items-center justify-between py-3 border-y">
          <span className="text-sm text-gray-700">
            {data.state === "on" ? "Receiving marketing emails" : "Not receiving marketing emails"}
          </span>
          <Switch checked={data.state === "on"} onCheckedChange={handleToggle} disabled={saving} />
        </div>
        <p className="text-xs text-gray-500">{data.note}</p>
        {saving && (
          <div className="flex items-center gap-2 text-xs text-gray-400">
            <Loader2 className="w-3 h-3 animate-spin" />
            Saving...
          </div>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Add the route**

In `App.tsx`, add the import near the other page imports (alongside `import GuestAccess from "./pages/GuestAccess";`):

```typescript
import MarketingPreferencesPage from "./pages/MarketingPreferencesPage";
```

Add the route near the `/guest` route:

```tsx
                  <Route path="/marketing-preferences/:token" element={<MarketingPreferencesPage />} />
```

- [ ] **Step 5: Run test to verify it passes**

```bash
npx vitest run src/pages/MarketingPreferencesPage.test.tsx
```
Expected: PASS (3 tests).

- [ ] **Step 6: Typecheck and lint**

```bash
npm run typecheck 2>&1 | grep -iE "MarketingPreferencesPage|App.tsx"
npx eslint src/pages/MarketingPreferencesPage.tsx
```
Expected: typecheck grep shows no new errors beyond the pre-existing unrelated ones already known in `App.tsx` (import-conflict and `Area`/`Dispatch` errors, confirmed pre-existing earlier in this session); eslint shows 0 problems.

- [ ] **Step 7: Commit**

```bash
git add src/pages/MarketingPreferencesPage.tsx src/pages/MarketingPreferencesPage.test.tsx src/App.tsx
git commit -m "feat(marketing): add public patient preferences page and route"
```

---

### Task 6: Manual end-to-end verification

- [ ] **Step 1: Verify domain config UI**

Start both dev servers if not already running (Django on its usual port, Vite on `:8080`). Log in, go to Settings → Practice → Communication, open the marketing domain detail view (search icon), set an email prefix and toggle a reply-to address, confirm both save and persist across a page refresh (re-fetch from `GET /marketing/domains/status/`).

- [ ] **Step 2: Verify footer link in a real send**

Trigger a campaign test-send (existing "Test send" flow, once template selection is unblocked — if template selection is still not wired up in this session, verify via `python manage.py shell` instead):

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
python manage.py shell -c "
from marketingBroadcast.delivery_reporting import send_broadcast_email
from marketingBroadcast.models import BroadcastRecipient, BroadcastCampaign
from TreatmentPlan.models import Person
recipient = BroadcastRecipient.objects.filter(campaign__practice_id=16).first()
if recipient:
    msg = send_broadcast_email(recipient, is_test=True)
    print('/marketing-preferences/' in msg.body)
"
```
Expected: prints `True`.

- [ ] **Step 3: Verify the public preferences page**

Take the token printed/queried from the `MarketingPreferenceToken` row created in Step 2, visit `http://localhost:8080/marketing-preferences/<token>` in a browser (no login), confirm the masked email and toggle render, and that toggling flips the state (verify via `GET /marketing/preferences/<token>/` directly, or by checking `MarketingConsent.state` in the DB).

- [ ] **Step 4: Note the FRONTEND_URL dev mismatch (no fix required, just confirm)**

`settings.py`'s `_default_frontend_url()` falls back to `http://localhost:5173` in `DEBUG` mode, but this repo's Vite dev server actually runs on `:8080` (confirmed earlier this session). This is a pre-existing inconsistency, out of scope for this plan — the manual verification steps above use `:8080` directly rather than trusting the generated link's port. If `FRONTEND_URL` is unset in your local `.env`, the printed link in Step 2 will show `:5173` — substitute `:8080` when opening it in Step 3.

---

## Self-Review Notes

- **Spec coverage:** §4 (Part A endpoint + UI) → Tasks 1, 3, 4. §5 (Part B footer + page) → Tasks 2, 3, 5. §6 (error handling — 404 unregistered, 400 invalid reply_to, 404 invalid token) → covered in Task 1 tests (404/400) and Task 5 tests (404). §7 (testing) → covered by every task's own test step.
- **No placeholders:** all steps contain complete, runnable code.
- **Type consistency:** `MarketingSendingDomain` interface in `MarketingDomainConfigView.tsx` (Task 4) matches the backend `_serialize_domain` shape exactly (Task 1) — `email_prefix`/`reply_to` already existed in that interface from earlier session work and required no change, just consumption.

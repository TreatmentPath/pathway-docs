# Guest Session Security Patch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Patch three security vulnerabilities in the guest session system: no rate limiting on the verify endpoint, different error codes that allow token enumeration, and raw tokens exposed in the session list endpoint.

**Architecture:** Three focused changes to `Admin/views.py` and `TreatmentPath/settings.py`. Add a `guest_verify` scoped throttle rate to settings, apply `throttle_scope` to the verify view, normalise all token error responses to HTTP 400, and strip the raw `token` field from the list endpoint response.

**Tech Stack:** Django REST Framework, `rest_framework.throttling.ScopedRateThrottle` (already configured globally).

---

## File Map

| File | What changes |
|---|---|
| `TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py` | Add `"guest_verify": "10/minute"` to `DEFAULT_THROTTLE_RATES` |
| `TreatmentPathBackend/TreatmentPath/Admin/views.py` | 3 changes: add `throttle_scope`, normalise error codes, remove raw token from list |
| `TreatmentPathBackend/TreatmentPath/Admin/tests.py` | Add security tests for all three fixes |

---

## Task 1: Rate-limit the verify endpoint (10/minute)

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py:131-138`
- Modify: `TreatmentPathBackend/TreatmentPath/Admin/views.py` (around line 8930)
- Test: `TreatmentPathBackend/TreatmentPath/Admin/tests.py`

### Context

`ScopedRateThrottle` is already in `DEFAULT_THROTTLE_CLASSES`. Adding a named scope in settings and setting `throttle_scope` on the view is all that's needed — no imports required.

- [ ] **Step 1: Write the failing test**

Open `TreatmentPathBackend/TreatmentPath/Admin/tests.py`. Add at the bottom:

```python
from django.test import override_settings
from rest_framework.test import APIClient


class GuestSessionVerifyRateLimitTest(TestCase):
    """verify endpoint must be rate-limited to 10/minute."""

    @override_settings(
        REST_FRAMEWORK={
            **__import__('django.conf', fromlist=['settings']).settings.REST_FRAMEWORK,
            "DEFAULT_THROTTLE_RATES": {
                **__import__('django.conf', fromlist=['settings']).settings.REST_FRAMEWORK.get(
                    "DEFAULT_THROTTLE_RATES", {}
                ),
                "guest_verify": "3/minute",  # Low limit for test speed
            },
        }
    )
    def test_verify_endpoint_has_throttle_scope(self):
        """verify_guest_session_view must have throttle_scope set."""
        from Admin.views import verify_guest_session_view
        self.assertEqual(
            getattr(verify_guest_session_view, "throttle_scope", None),
            "guest_verify",
        )
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
cd TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test Admin.tests.GuestSessionVerifyRateLimitTest -v 2
```

Expected: `FAIL — AssertionError: None != 'guest_verify'`

- [ ] **Step 3: Add `guest_verify` rate to settings**

In `TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py`, find:

```python
    "DEFAULT_THROTTLE_RATES": {
        "anon": "1000/hour",  # Anonymous users: 1000 requests per hour
        "user": "10000/hour",  # Authenticated users: 10000 requests per hour
        "login": "20/minute",  # Login endpoints: 20 attempts per minute
        "otp": "10/minute",  # OTP requests: 10 per minute
        "password_reset": "10/hour",  # Password reset: 10 per hour
        "public_booking": "100/hour",  # Public booking endpoints: 100 per hour
    },
```

Replace with:

```python
    "DEFAULT_THROTTLE_RATES": {
        "anon": "1000/hour",  # Anonymous users: 1000 requests per hour
        "user": "10000/hour",  # Authenticated users: 10000 requests per hour
        "login": "20/minute",  # Login endpoints: 20 attempts per minute
        "otp": "10/minute",  # OTP requests: 10 per minute
        "password_reset": "10/hour",  # Password reset: 10 per hour
        "public_booking": "100/hour",  # Public booking endpoints: 100 per hour
        "guest_verify": "10/minute",  # Guest token verification: 10 per minute per IP
    },
```

- [ ] **Step 4: Add `throttle_scope` to `verify_guest_session_view`**

In `TreatmentPathBackend/TreatmentPath/Admin/views.py`, find:

```python
@api_view(["POST"])
@permission_classes([AllowAny])
def verify_guest_session_view(request):
```

Replace with:

```python
@api_view(["POST"])
@permission_classes([AllowAny])
@throttle_classes([ScopedRateThrottle])
def verify_guest_session_view(request):
    verify_guest_session_view.throttle_scope = "guest_verify"
```

Wait — `@throttle_classes` and `throttle_scope` on `@api_view` functions require a different pattern. Use this instead:

```python
from rest_framework.throttling import ScopedRateThrottle


@api_view(["POST"])
@permission_classes([AllowAny])
def verify_guest_session_view(request):
    verify_guest_session_view.throttle_scope = "guest_verify"
```

Actually the correct DRF pattern for scoped throttling on `@api_view` is to set the attribute on the function directly **before** the decorator, like this. Find the existing import block at the top of `Admin/views.py` where other DRF imports live, and add if not present:

```python
from rest_framework.throttling import ScopedRateThrottle
```

Then find:

```python
@api_view(["POST"])
@permission_classes([AllowAny])
def verify_guest_session_view(request):
    """
    Verify a guest session token and return practice access information.
    No authentication required - uses the guest token.
```

Replace with:

```python
@api_view(["POST"])
@permission_classes([AllowAny])
@throttle_classes([ScopedRateThrottle])
def verify_guest_session_view(request):
    """
    Verify a guest session token and return practice access information.
    No authentication required - uses the guest token.
    Rate limited to 10/minute per IP (guest_verify scope).
```

And immediately after the `def` line (first line of the function body, before the docstring ends), set:

```python
verify_guest_session_view.throttle_scope = "guest_verify"
```

The cleanest way is to set it **after** the function definition, outside the function body. Find:

```python
@api_view(["POST"])
@permission_classes([AllowAny])
def verify_guest_session_view(request):
    """
    Verify a guest session token and return practice access information.
    No authentication required - uses the guest token.
    ...
    """
    from django.utils import timezone
```

Replace the decorator block with:

```python
@api_view(["POST"])
@permission_classes([AllowAny])
@throttle_classes([ScopedRateThrottle])
def verify_guest_session_view(request):
    """
    Verify a guest session token and return practice access information.
    No authentication required - uses the guest token.
    Rate limited to 10/minute per IP (guest_verify scope).
    ...
    """
    from django.utils import timezone
```

Then find the line AFTER the closing of `verify_guest_session_view` (look for the next `@api_view` decorator after the function ends) and insert before it:

```python
verify_guest_session_view.throttle_scope = "guest_verify"
```

Also check if `throttle_classes` is already imported at the top of `Admin/views.py`:

```bash
grep -n "throttle_classes\|ScopedRateThrottle" TreatmentPathBackend/TreatmentPath/Admin/views.py | head -5
```

If not present, find the rest_framework imports block and add:

```python
from rest_framework.decorators import throttle_classes
from rest_framework.throttling import ScopedRateThrottle
```

- [ ] **Step 5: Run test to confirm it passes**

```bash
cd TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test Admin.tests.GuestSessionVerifyRateLimitTest -v 2
```

Expected: PASS.

- [ ] **Step 6: Manually verify the rate limit works**

```bash
# Hit the endpoint 11 times rapidly — the 11th should return 429
for i in $(seq 1 11); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST http://127.0.0.1:8000/api/backend/system-settings/guest-sessions/verify/ \
    -H "Content-Type: application/json" \
    -d '{"token": "invalid-test-token"}'
done
```

Expected: First 10 return `400` (bad token), 11th returns `429 Too Many Requests`.

> Note: Run this only with the dev server running. Skip if server isn't up — the unit test is sufficient.

---

## Task 2: Normalise error responses — no token enumeration

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/Admin/views.py` (lines ~8973-8985)
- Test: `TreatmentPathBackend/TreatmentPath/Admin/tests.py`

### Context

Currently the verify endpoint returns:
- `404` for invalid/inactive token
- `401` for expired token

An attacker can distinguish "token exists but expired" from "token never existed" — this leaks information. Both should return `400` with a generic message.

- [ ] **Step 1: Write the failing test**

Add to `Admin/tests.py`:

```python
class GuestSessionVerifyErrorNormalisationTest(TestCase):
    """verify endpoint must return 400 for all token errors, not 404/401."""

    def setUp(self):
        self.client = APIClient()
        self.url = "/api/backend/system-settings/guest-sessions/verify/"

    def test_invalid_token_returns_400_not_404(self):
        response = self.client.post(
            self.url,
            {"token": "completely-invalid-token-that-does-not-exist"},
            format="json",
        )
        self.assertEqual(response.status_code, 400)

    def test_expired_token_returns_400_not_401(self):
        from django.utils import timezone
        from datetime import timedelta
        from UserAuthentication.models import Practice
        from django.contrib.auth import get_user_model
        from Admin.models import GuestSession

        User = get_user_model()
        practice = Practice.objects.create(
            name="Test Practice Expired",
            slug="test-practice-expired-verify",
            country="GB",
        )
        admin = User.objects.create_user(
            email="admin_expired_verify@test.com",
            password="testpass123",
            user_type="superuser",
        )
        expired_session = GuestSession.objects.create(
            admin_user=admin,
            practice=practice,
            token="expired-test-token-verify-123",
            expires_at=timezone.now() - timedelta(hours=1),
            is_active=True,
        )

        response = self.client.post(
            self.url,
            {"token": "expired-test-token-verify-123"},
            format="json",
        )
        self.assertEqual(response.status_code, 400)

    def test_missing_token_returns_400(self):
        response = self.client.post(self.url, {}, format="json")
        self.assertEqual(response.status_code, 400)
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
cd TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test Admin.tests.GuestSessionVerifyErrorNormalisationTest -v 2
```

Expected: `test_invalid_token_returns_400_not_404` FAILS (gets 404), `test_expired_token_returns_400_not_401` FAILS (gets 401).

- [ ] **Step 3: Update error responses in verify_guest_session_view**

In `TreatmentPathBackend/TreatmentPath/Admin/views.py`, find:

```python
        except GuestSession.DoesNotExist:
            return Response(
                {"error": "Invalid or inactive guest token"},
                status=status.HTTP_404_NOT_FOUND,
            )

        # Check if expired
        if guest_session.is_expired():
            guest_session.revoke()
            return Response(
                {"error": "Guest session has expired"},
                status=status.HTTP_401_UNAUTHORIZED,
            )
```

Replace with:

```python
        except GuestSession.DoesNotExist:
            return Response(
                {"error": "Invalid or inactive guest token"},
                status=status.HTTP_400_BAD_REQUEST,
            )

        # Check if expired
        if guest_session.is_expired():
            guest_session.revoke()
            return Response(
                {"error": "Invalid or inactive guest token"},
                status=status.HTTP_400_BAD_REQUEST,
            )
```

Note: both cases now return the **same message and same status code** — this prevents an attacker from distinguishing expired tokens from never-valid tokens.

- [ ] **Step 4: Run tests to confirm they pass**

```bash
cd TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test Admin.tests.GuestSessionVerifyErrorNormalisationTest -v 2
```

Expected: All 3 tests PASS.

---

## Task 3: Remove raw token from list endpoint

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/Admin/views.py` (around line 9141)
- Test: `TreatmentPathBackend/TreatmentPath/Admin/tests.py`

### Context

`list_guest_sessions_view` currently returns `"token": session.token` — the raw secret token — to superusers. If an admin account is compromised, all active guest tokens are exposed. Remove it entirely. The token is a one-time credential for sharing, not something to display in management UIs.

- [ ] **Step 1: Write the failing test**

Add to `Admin/tests.py`:

```python
class GuestSessionListTokenExposureTest(TestCase):
    """list endpoint must not expose raw tokens."""

    def setUp(self):
        from django.contrib.auth import get_user_model
        User = get_user_model()
        self.client = APIClient()
        self.superuser = User.objects.create_user(
            email="superuser_list_test@test.com",
            password="testpass123",
            user_type="superuser",
        )
        self.client.force_authenticate(user=self.superuser)
        self.url = "/api/backend/system-settings/guest-sessions/"

    def test_list_does_not_expose_raw_token(self):
        from UserAuthentication.models import Practice
        from django.utils import timezone
        from datetime import timedelta
        from Admin.models import GuestSession

        practice = Practice.objects.create(
            name="Test Practice List Token",
            slug="test-practice-list-token",
            country="GB",
        )
        GuestSession.objects.create(
            admin_user=self.superuser,
            practice=practice,
            token="super-secret-raw-token-abc123",
            expires_at=timezone.now() + timedelta(hours=24),
            is_active=True,
        )

        response = self.client.get(self.url)
        self.assertEqual(response.status_code, 200)

        sessions = response.data.get("sessions", [])
        self.assertTrue(len(sessions) > 0, "Expected at least one session in response")

        for session in sessions:
            self.assertNotIn(
                "token", session, "Raw token must not appear in list response"
            )
            # Also verify the actual secret value isn't embedded anywhere
            self.assertNotIn(
                "super-secret-raw-token-abc123",
                str(session),
            )
```

- [ ] **Step 2: Run test to confirm it fails**

```bash
cd TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test Admin.tests.GuestSessionListTokenExposureTest -v 2
```

Expected: FAIL — `AssertionError: 'token' should not be in session dict`.

- [ ] **Step 3: Remove token field from list_guest_sessions_view**

In `TreatmentPathBackend/TreatmentPath/Admin/views.py`, find in `list_guest_sessions_view`:

```python
            sessions_data.append(
                {
                    "id": session.id,
                    "token": session.token,
                    "practice": {
```

Replace with:

```python
            sessions_data.append(
                {
                    "id": session.id,
                    "practice": {
```

- [ ] **Step 4: Run test to confirm it passes**

```bash
cd TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test Admin.tests.GuestSessionListTokenExposureTest -v 2
```

Expected: PASS.

---

## Task 4: Run all security tests together + full regression check

- [ ] **Step 1: Run all three new test classes**

```bash
cd TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py test \
  Admin.tests.GuestSessionVerifyRateLimitTest \
  Admin.tests.GuestSessionVerifyErrorNormalisationTest \
  Admin.tests.GuestSessionListTokenExposureTest \
  -v 2
```

Expected: All 6 tests PASS.

- [ ] **Step 2: Run existing Admin tests to check for regressions**

```bash
python manage.py test Admin --verbosity=1 --keepdb 2>&1 | tail -5
```

Expected: Same pass/fail count as before these changes.

- [ ] **Step 3: Verify settings change didn't break other throttle scopes**

```bash
python manage.py test --verbosity=1 --keepdb 2>&1 | grep -E "^(OK|FAIL|Ran|FAILED)" | tail -3
```

Expected: Same result as baseline (30 failures, 14 errors — all pre-existing, none new).

# Compact Patient-Guide Share Link Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the long patient-guide URL (`/treatment-verification/services?code=EUbpm5&treatment_id=1191`, ~78 chars) with a compact one (`/g/EUbpm5`, ~30 chars) and stop the raw URL from sitting editable inside the Share message body.

**Architecture:** Mirror the existing Daylist confirmation pattern exactly. Daylist uses `CONFIRM_BASE_URL` (default `https://dev.confirm.dental`) plus a short token as the whole path, with `src/confirmationRouting.ts` + `App.tsx` splitting "dedicated domain → `/:token`" from "local dev → `/confirm/:token`". We add `GUIDE_BASE_URL` and the `/g/:code` local equivalent. The 6-char code already exists (`SixDigitVerificationCode.code`) — no new token field and no migration. A new public resolver turns a code into its bound plan id, then the frontend redirects to the UNCHANGED verification page carrying both `code` and `treatment_id`, so the existing security model is untouched.

**Tech Stack:** Django 5.1 + DRF (backend), React 18 + Vite + react-router (frontend), pytest/Django test runner.

**Spec:** No separate spec doc — the requirement came from the user in-session: "can we have it more compact so we can have something like how we have it in daylist confirmation… also note that, the link should not be editable".

## Global Constraints

- **Security model must not change.** `SixDigitVerificationCode` carries audit findings #31/#32 (documented at `TreatmentPlan/models.py:4531`). The code is the credential for a PUBLIC, UNAUTHENTICATED WRITE endpoint (`public_treatment_views.share_availability` appends to `TreatmentPlan.notes`). The redirect must hand the verification page the same `code` + `treatment_id` pair it receives today.
- **`code` is NOT unique** — `models.CharField(max_length=20)`, no `unique=True`. Any lookup by code alone MUST refuse an ambiguous (>1 row) match rather than pick one.
- **Legacy unbound codes must not resolve.** Rows with `treatment_plan_id IS NULL` predate the binding; `is_usable_for` falls back to a practice check that denies by default. The resolver requires `treatment_plan__isnull=False`.
- **Expiry must be honoured** — `expires_at` is nullable on legacy rows; treat `expires_at <= now()` as dead.
- **Backend tests:** run with `--keepdb`. NEVER pass `--noinput` (it destroys the persistent test DB).
- **Do not add a migration.** No model field is added.
- **URL names collide in this repo** (six known duplicates, e.g. `patient-list` in both `Notes/urls.py` and `TreatmentPlan/urls.py`). Verified free before use: `guide-code-resolve`.
- **No `as any`** and no type-bypasses in the frontend.

---

### Task 1: Backend — public code resolver

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/verification_views.py` (append a new view)
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/urls.py:878` (register beside the existing `verification-code` path)
- Test: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_guide_code_resolve.py`

**Interfaces:**
- Consumes: `SixDigitVerificationCode` (`TreatmentPlan/models.py:4531`) fields `code`, `is_active`, `expires_at`, `treatment_plan_id`.
- Produces: `GET /api/backend/treatment-plan/public/guide/<str:code>/` → `200 {"treatment_plan_id": <int>, "code": "<code>"}` or `404 {"error": "..."}`. URL name `guide-code-resolve`.
- **MUST sit under the `public/` prefix.** `SubscriptionMiddleware.EXCLUDED_PATHS` (`payments/middleware.py:40`) exempts `/api/backend/treatment-plan/public/`; anything outside it under `treatment-plan/` hits `SUBSCRIPTION_REQUIRED_PATHS` and returns 401 before DRF permissions are consulted. Proven by a red run that returned 401 rather than 404.

- [ ] **Step 1: Write the failing test**

```python
# TreatmentPlan/tests/test_guide_code_resolve.py
from datetime import timedelta

from django.test import TestCase
from django.urls import reverse
from django.utils import timezone

from TreatmentPlan.models import Practice, SixDigitVerificationCode, TreatmentPlan
from UserAuthentication.models import User

GUIDE_URL = "/api/backend/treatment-plan/public/guide/{code}/"


class GuideCodeResolveTests(TestCase):
    @classmethod
    def setUpTestData(cls):
        # Verified against the models: Practice requires name AND slug;
        # User requires email + password and has NO practice FK.
        cls.practice = Practice.objects.create(
            name="Resolver Practice", slug="resolver-practice"
        )
        cls.user = User.objects.create(
            email="resolver@example.com", password="x"
        )
        cls.plan = TreatmentPlan.objects.create(practice=cls.practice)

    def _code(self, value, **kwargs):
        defaults = dict(
            user=self.user,
            treatment_plan=self.plan,
            is_active=True,
            expires_at=timezone.now() + timedelta(days=7),
        )
        defaults.update(kwargs)
        return SixDigitVerificationCode.objects.create(code=value, **defaults)

    def test_resolves_bound_active_code_to_its_plan(self):
        self._code("AbC123")
        res = self.client.get(GUIDE_URL.format(code="AbC123"))
        self.assertEqual(res.status_code, 200)
        self.assertEqual(res.json()["treatment_plan_id"], self.plan.id)

    def test_unknown_code_is_404(self):
        res = self.client.get(GUIDE_URL.format(code="ZZZZZZ"))
        self.assertEqual(res.status_code, 404)

    def test_inactive_code_is_404(self):
        self._code("Dead01", is_active=False)
        res = self.client.get(GUIDE_URL.format(code="Dead01"))
        self.assertEqual(res.status_code, 404)

    def test_expired_code_is_404(self):
        self._code("Old001", expires_at=timezone.now() - timedelta(seconds=1))
        res = self.client.get(GUIDE_URL.format(code="Old001"))
        self.assertEqual(res.status_code, 404)

    def test_legacy_unbound_code_does_not_resolve(self):
        # treatment_plan=None predates the audit #31 binding; it names no plan.
        self._code("Leg001", treatment_plan=None)
        res = self.client.get(GUIDE_URL.format(code="Leg001"))
        self.assertEqual(res.status_code, 404)

    def test_ambiguous_code_refuses_rather_than_guessing(self):
        # `code` has no unique constraint. Two live rows -> we must not pick one.
        other_plan = TreatmentPlan.objects.create(practice=self.practice)
        self._code("Dup001")
        self._code("Dup001", treatment_plan=other_plan)
        res = self.client.get(GUIDE_URL.format(code="Dup001"))
        self.assertEqual(res.status_code, 404)

    def test_requires_no_authentication(self):
        # The patient clicking the link is not logged in.
        self._code("Pub001")
        self.client.logout()
        res = self.client.get(GUIDE_URL.format(code="Pub001"))
        self.assertEqual(res.status_code, 200)
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPlan.tests.test_guide_code_resolve --keepdb -v 2
```
Expected: FAIL — 404 for every case because the route does not exist yet.

- [ ] **Step 3: Write minimal implementation**

Append to `TreatmentPlan/views/verification_views.py`:

```python
class GuideCodeResolveView(APIView):
    """Resolve a short guide code to the plan it is bound to.

    Public and unauthenticated: the patient following the short link has no
    session. This grants nothing new — a code is already bound to exactly one
    plan (audit #31), so a valid code always implied its plan; this endpoint
    only saves the SMS from having to carry the id as a second query param.

    Refuses rather than guesses when the answer is not unambiguous:
    `code` has no unique constraint, so two live rows mean we cannot know
    which plan was meant.
    """

    permission_classes = [AllowAny]
    authentication_classes = []

    def get(self, request, code):
        now = timezone.now()
        matches = list(
            SixDigitVerificationCode.objects.filter(
                code=code,
                is_active=True,
                treatment_plan__isnull=False,
            )
            .exclude(expires_at__lte=now)[:2]
        )
        if len(matches) != 1:
            # 0 = unknown/dead/legacy-unbound; 2 = ambiguous. Same opaque
            # answer either way so the endpoint does not confirm which codes
            # exist.
            return Response(
                {"error": "Invalid or expired link."},
                status=status.HTTP_404_NOT_FOUND,
            )
        return Response(
            {"treatment_plan_id": matches[0].treatment_plan_id, "code": code},
            status=status.HTTP_200_OK,
        )
```

Ensure these imports exist at the top of the file: `from rest_framework.permissions import AllowAny` and `from django.utils import timezone`.

Register in `TreatmentPlan/urls.py` immediately after the existing `verification-code` path (line ~878), and add `GuideCodeResolveView` to the `verification_views` import list:

```python
    # Short patient-guide link: /g/<code> resolves here before redirecting.
    path(
        "public/guide/<str:code>/",
        GuideCodeResolveView.as_view(),
        name="guide-code-resolve",
    ),
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
python manage.py test TreatmentPlan.tests.test_guide_code_resolve --keepdb -v 2
```
Expected: PASS, 7 tests.

- [ ] **Step 5: Commit**

```bash
git add TreatmentPlan/views/verification_views.py TreatmentPlan/urls.py TreatmentPlan/tests/test_guide_code_resolve.py
git commit -m "feat: resolve short patient-guide codes to their bound plan"
```

---

### Task 2: Backend — GUIDE_BASE_URL and share_url on the code response

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py:488` (beside `CONFIRM_BASE_URL`)
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/verification_views.py:177-186` (the POST response)
- Test: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_guide_share_url.py`

**Interfaces:**
- Consumes: `settings.GUIDE_BASE_URL`.
- Produces: `POST /api/backend/treatment-plan/verification-code/` response gains `share_url` — a string when `GUIDE_BASE_URL` is configured, otherwise `None`. The existing `code` key is unchanged.

- [ ] **Step 1: Write the failing test**

```python
# TreatmentPlan/tests/test_guide_share_url.py
from django.test import TestCase, override_settings

from TreatmentPlan.views.verification_views import build_guide_share_url


class BuildGuideShareUrlTests(TestCase):
    @override_settings(GUIDE_BASE_URL="https://guide.dental")
    def test_uses_configured_domain_with_code_as_whole_path(self):
        # Mirrors CONFIRM_BASE_URL/<token> from the daylist confirmation link.
        self.assertEqual(build_guide_share_url("AbC123"), "https://guide.dental/AbC123")

    @override_settings(GUIDE_BASE_URL="https://guide.dental/")
    def test_trailing_slash_does_not_double(self):
        self.assertEqual(build_guide_share_url("AbC123"), "https://guide.dental/AbC123")

    @override_settings(GUIDE_BASE_URL="")
    def test_returns_none_when_unset_so_frontend_uses_its_own_origin(self):
        self.assertIsNone(build_guide_share_url("AbC123"))
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
python manage.py test TreatmentPlan.tests.test_guide_share_url --keepdb -v 2
```
Expected: FAIL with `ImportError: cannot import name 'build_guide_share_url'`.

- [ ] **Step 3: Write minimal implementation**

In `settings.py`, directly below the existing `CONFIRM_BASE_URL` line:

```python
# Dedicated short domain for patient-guide links, mirroring CONFIRM_BASE_URL.
# Empty means "not configured": the frontend then builds its own-origin /g/<code>
# equivalent, so local dev works without DNS.
GUIDE_BASE_URL = os.environ.get("GUIDE_BASE_URL", "")
```

In `verification_views.py`, above `VerificationCodeView`:

```python
def build_guide_share_url(code):
    """The patient-facing short link for a guide code, or None when no
    dedicated domain is configured (the frontend then uses its own origin)."""
    base = getattr(settings, "GUIDE_BASE_URL", "") or ""
    if not base:
        return None
    return f"{base.rstrip('/')}/{code}"
```

Add `from django.conf import settings` if absent, then change the POST response:

```python
        return Response(
            {
                "code": code,
                "share_url": build_guide_share_url(code),
            },
            status=status.HTTP_201_CREATED,
        )
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
python manage.py test TreatmentPlan.tests.test_guide_share_url --keepdb -v 2
```
Expected: PASS, 3 tests.

- [ ] **Step 5: Commit**

```bash
git add TreatmentPath/settings.py TreatmentPlan/views/verification_views.py TreatmentPlan/tests/test_guide_share_url.py
git commit -m "feat: add GUIDE_BASE_URL and return share_url with guide codes"
```

---

### Task 3: Frontend — /g/:code route that resolves and redirects

**Files:**
- Create: `perfect-pixel-playground-project/src/pages/guide-link/GuideLinkRedirectPage.tsx`
- Modify: `perfect-pixel-playground-project/src/config/api.ts:1662-1665` (add the resolver endpoint)
- Modify: `perfect-pixel-playground-project/src/routes/auth.routes.tsx:47` (add the public route)
- Test: `perfect-pixel-playground-project/src/pages/guide-link/GuideLinkRedirectPage.test.tsx`

Filename collision check (run before Write): `ls src/pages/guide-link 2>/dev/null; grep -rl "GuideLinkRedirectPage" src` — expect no matches.

**Interfaces:**
- Consumes: `GET /api/backend/treatment-plan/public/guide/<code>/` from Task 1 → `{treatment_plan_id, code}`.
- Produces: route `/g/:code`, `access: "public"`, redirecting to `/treatment-verification/services?code=<code>&treatment_id=<id>`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/pages/guide-link/GuideLinkRedirectPage.test.tsx
import { render, screen, waitFor } from "@testing-library/react";
import { MemoryRouter, Route, Routes } from "react-router-dom";
import { beforeEach, describe, expect, it, vi } from "vitest";

import { GuideLinkRedirectPage } from "./GuideLinkRedirectPage";

function renderAt(path: string) {
  return render(
    <MemoryRouter initialEntries={[path]}>
      <Routes>
        <Route path="/g/:code" element={<GuideLinkRedirectPage />} />
        <Route
          path="/treatment-verification/services"
          element={<div>SERVICES PAGE</div>}
        />
      </Routes>
    </MemoryRouter>,
  );
}

describe("GuideLinkRedirectPage", () => {
  beforeEach(() => vi.restoreAllMocks());

  it("redirects to the verification page carrying BOTH code and treatment_id", async () => {
    vi.stubGlobal(
      "fetch",
      vi.fn().mockResolvedValue({
        ok: true,
        json: async () => ({ treatment_plan_id: 1191, code: "EUbpm5" }),
      }),
    );
    renderAt("/g/EUbpm5");
    await waitFor(() => expect(screen.getByText("SERVICES PAGE")).toBeTruthy());
  });

  it("shows an expired-link message instead of redirecting when the code is dead", async () => {
    vi.stubGlobal(
      "fetch",
      vi.fn().mockResolvedValue({ ok: false, status: 404, json: async () => ({}) }),
    );
    renderAt("/g/ZZZZZZ");
    await waitFor(() =>
      expect(screen.getByText(/link has expired/i)).toBeTruthy(),
    );
    expect(screen.queryByText("SERVICES PAGE")).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vitest run src/pages/guide-link/GuideLinkRedirectPage.test.tsx
```
Expected: FAIL — cannot resolve `./GuideLinkRedirectPage`.

- [ ] **Step 3: Write minimal implementation**

Add to `src/config/api.ts` inside the existing `patientTreatmentPlanView` group:

```ts
    resolveGuideCode: (code: string) =>
      getApiUrl(`/treatment-plan/public/guide/${encodeURIComponent(code)}/`),
```

Create `src/pages/guide-link/GuideLinkRedirectPage.tsx`:

```tsx
import { useEffect, useState } from "react";
import { Navigate, useParams } from "react-router-dom";

import { API_ENDPOINTS } from "@/config/api";

type Resolution =
  | { state: "loading" }
  | { state: "ready"; treatmentPlanId: number }
  | { state: "dead" };

/**
 * The short patient-guide link (`/g/<code>`, or `<code>` on a dedicated
 * GUIDE_BASE_URL domain). It resolves the code to its bound plan and then hands
 * the UNCHANGED verification page both `code` and `treatment_id` — the pair it
 * has always validated — so shortening the link changes no security behaviour.
 *
 * Unauthenticated by design: the patient following this link has no session.
 */
export function GuideLinkRedirectPage() {
  const { code } = useParams<{ code: string }>();
  const [resolution, setResolution] = useState<Resolution>({ state: "loading" });

  useEffect(() => {
    if (!code) {
      setResolution({ state: "dead" });
      return;
    }
    let cancelled = false;
    void (async () => {
      try {
        const res = await fetch(API_ENDPOINTS.patientTreatmentPlanView.resolveGuideCode(code));
        if (!res.ok) throw new Error(String(res.status));
        const data = (await res.json()) as { treatment_plan_id?: number };
        if (cancelled) return;
        if (typeof data.treatment_plan_id === "number") {
          setResolution({ state: "ready", treatmentPlanId: data.treatment_plan_id });
        } else {
          setResolution({ state: "dead" });
        }
      } catch {
        if (!cancelled) setResolution({ state: "dead" });
      }
    })();
    return () => {
      cancelled = true;
    };
  }, [code]);

  if (resolution.state === "ready") {
    const target = `/treatment-verification/services?code=${encodeURIComponent(
      code ?? "",
    )}&treatment_id=${resolution.treatmentPlanId}`;
    return <Navigate to={target} replace />;
  }

  return (
    <div className="flex min-h-screen items-center justify-center px-6 text-center">
      {resolution.state === "loading" ? (
        <p className="text-sm text-slate-500">Opening your patient guide…</p>
      ) : (
        <div className="max-w-sm space-y-2">
          <h1 className="text-lg font-semibold text-slate-900">
            This link has expired
          </h1>
          <p className="text-sm text-slate-500">
            Please contact your practice for a new link to your patient guide.
          </p>
        </div>
      )}
    </div>
  );
}

export default GuideLinkRedirectPage;
```

Register the route in `src/routes/auth.routes.tsx`, next to the other public treatment-verification entries, using the file's existing `lazyPage` idiom:

```tsx
const GuideLinkRedirect = lazyPage("GuideLinkRedirect", () =>
  import("../pages/guide-link/GuideLinkRedirectPage").then((m) => ({
    default: m.GuideLinkRedirectPage,
  })),
);
```

```tsx
  {
    path: "/g/:code",
    component: GuideLinkRedirect,
    access: "public",
  },
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
npx vitest run src/pages/guide-link/GuideLinkRedirectPage.test.tsx
```
Expected: PASS, 2 tests.

- [ ] **Step 5: Commit**

```bash
git add src/pages/guide-link src/config/api.ts src/routes/auth.routes.tsx
git commit -m "feat: add /g/:code short patient-guide link route"
```

---

### Task 4: Frontend — emit the short link, and stop it being editable

**Files:**
- Modify: `perfect-pixel-playground-project/src/lib/patientGuideDocuments.ts:178-191` (`buildPatientGuideMessage`)
- Modify: `perfect-pixel-playground-project/src/components/modals/treatmentPlan/TreatmentPlanConfirmationModal.tsx` (`useVerificationLink`)
- Modify: `perfect-pixel-playground-project/src/components/modals/treatmentPlan/SendToPatientSection.tsx` (read-only link row + send-time composition)
- Modify: `perfect-pixel-playground-project/src/components/modals/treatmentPlan/SendToPatientMock.tsx:34` — it also calls `buildPatientGuideMessage({..., link})`. Reachable only via `TreatmentPlanSuccessModal`, which is exported from `src/components/modals/index.ts` but mounted nowhere; it must still compile after the signature change.
- Test: EXTEND `perfect-pixel-playground-project/src/hooks/usePatientGuideDocuments.test.tsx` — the existing `describe("buildPatientGuideMessage")` block at line 67 currently asserts the URL IS in the body, which this task deliberately reverses. Update that assertion rather than writing a parallel suite. Runner is `vitest` with `globals: true` (no describe/it imports needed).

**Interfaces:**
- Consumes: `share_url` from Task 2, `/g/:code` from Task 3.
- Produces: `buildPatientGuideMessage({patientName, practiceName})` returns body text with NO URL in it; `composePatientGuideMessage(body, link)` appends the link at send time.

- [ ] **Step 1: Write the failing test**

```ts
// append to src/lib/patientGuideDocuments.test.ts
import { buildPatientGuideMessage, composePatientGuideMessage } from "./patientGuideDocuments";

describe("patient guide message link handling", () => {
  it("keeps the editable body free of any URL", () => {
    const body = buildPatientGuideMessage({
      patientName: "Guide Testpatient",
      practiceName: "Practice Mannie",
    });
    expect(body).not.toMatch(/https?:\/\//);
    expect(body).toContain("Hi Guide");
  });

  it("appends the real link once at send time", () => {
    const body = buildPatientGuideMessage({
      patientName: "Guide Testpatient",
      practiceName: "Practice Mannie",
    });
    const sent = composePatientGuideMessage(body, "https://guide.dental/EUbpm5");
    expect(sent).toContain("https://guide.dental/EUbpm5");
    expect(sent.match(/guide\.dental/g)).toHaveLength(1);
  });

  it("does not append twice if the staff member pasted the link back in", () => {
    const sent = composePatientGuideMessage(
      "See it here: https://guide.dental/EUbpm5",
      "https://guide.dental/EUbpm5",
    );
    expect(sent.match(/guide\.dental/g)).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
npx vitest run src/lib/patientGuideDocuments.test.ts
```
Expected: FAIL — `composePatientGuideMessage` is not exported, and the body still contains a URL.

- [ ] **Step 3: Write minimal implementation**

In `src/lib/patientGuideDocuments.ts`, replace `buildPatientGuideMessage` and add the composer:

```ts
export function buildPatientGuideMessage({
  patientName,
  practiceName,
}: {
  patientName: string;
  practiceName?: string | null;
}): string {
  const firstName = firstNameFrom(patientName);
  const practice = practiceName?.trim() || "your practice";
  // Deliberately URL-free: the link is shown read-only beside the box and
  // appended at send time, so it cannot be broken by editing.
  return `Hi ${firstName}, here's your personalised patient guide from ${practice}. Any questions, just reply to this message.`;
}

/** The body the patient actually receives: staff copy plus the real link, once. */
export function composePatientGuideMessage(
  body: string,
  link: string | null | undefined,
): string {
  const url = link?.trim();
  if (!url) return body;
  if (body.includes(url)) return body;
  return `${body.trimEnd()}\n\n${url}`;
}
```

In `TreatmentPlanConfirmationModal.tsx`, change `useVerificationLink` to prefer the backend's `share_url` and fall back to the own-origin short path:

```ts
        const data = await res.json();
        const code = data.code || data.verification_code;
        if (!cancelled && code) {
          // Prefer the dedicated GUIDE_BASE_URL domain when the backend is
          // configured with one; otherwise use this origin's /g/<code>.
          setVerificationLink(data.share_url || `${window.location.origin}/g/${code}`);
        }
```

In `SendToPatientSection.tsx`: seed `message` from `buildPatientGuideMessage({patientName, practiceName})` (drop `link` from that call), send `composePatientGuideMessage(message, verificationLink)` as the `content` for BOTH channels, base `smsPartCount` on the composed string, and render the link read-only above the textarea:

```tsx
        <div className="rounded-lg border border-gray-200 bg-gray-50 p-3">
          <p className="mb-1 text-xs font-medium text-gray-600">Patient guide link</p>
          <div className="flex items-center gap-2">
            <code className="min-w-0 flex-1 truncate text-xs text-gray-700">
              {verificationLink ?? "Generating…"}
            </code>
            <Button
              type="button"
              variant="outline"
              size="sm"
              disabled={!verificationLink}
              onClick={() => verificationLink && window.open(verificationLink, "_blank")}
            >
              Test link
            </Button>
          </div>
          <p className="mt-1 text-xs text-gray-400">
            Added automatically when this sends — it can't be edited or broken.
          </p>
        </div>
```

Update the counter line to drop the now-untrue `[link]` sentence, using the composed length:

```tsx
          <p className="mt-1 text-xs text-gray-400">
            {composed.length} characters · {parts} SMS part{parts !== 1 ? "s" : ""} · the link is added automatically.
          </p>
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
npx vitest run src/lib/patientGuideDocuments.test.ts
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/lib/patientGuideDocuments.ts src/lib/patientGuideDocuments.test.ts src/components/modals/treatmentPlan
git commit -m "feat: send the short guide link and keep it out of the editable body"
```

---

### Task 5: End-to-end verification against the running app

**Files:** none modified — this task produces evidence.

- [ ] **Step 1: Confirm the outbound sandbox is ON before sending**

Sends from this flow reach real Twilio and real email otherwise.

```bash
pid=$(ss -ltnp | grep ':8000 ' | grep -oP 'pid=\K[0-9]+' | head -1)
tr '\0' '\n' < /proc/$pid/environ | grep OUTBOUND_SANDBOX
```
Expected: `OUTBOUND_SANDBOX=1`. If absent, restart Django with it set (interceptors install at `AppConfig.ready`, so it is restart-only).

- [ ] **Step 2: Drive the flow and send**

Log in, Patient Guides → "Guide Testpatient" (patient 141032) → add a treatment → Save Guide → Yes, Create Plan → Share → Send to patient.

- [ ] **Step 3: Assert the SHORT link is what actually went out**

```bash
python3 -c "
import json
for line in open('/tmp/treatmentpath-outbox.jsonl'):
    if line.strip():
        e = json.loads(line)
        print(e['channel'], '->', e['to'])
        print('  body:', e['body'])
"
```
Expected: the body contains `/g/<code>` (or `GUIDE_BASE_URL/<code>`) and NOT `/treatment-verification/services?code=`. Record the character count — it should drop the SMS from 2 parts to 1.

- [ ] **Step 4: Prove the short link actually opens the guide**

Visit `http://localhost:8080/g/<code>` in the browser and confirm it lands on the services page with the plan loaded. Then visit `http://localhost:8080/g/ZZZZZZ` and confirm the expired-link message renders rather than a crash or an infinite spinner.

- [ ] **Step 5: Typecheck and lint the touched frontend files**

```bash
npx tsc -p tsconfig.check.tmp.json 2>&1 | grep -cE "error TS"
```
Expected: 492 (the standing baseline) with zero errors naming the touched files. Judge by delta, not absolute count.

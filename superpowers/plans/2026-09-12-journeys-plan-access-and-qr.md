# Journeys Plan Access and QR Consolidation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Put `View treatment plan` and `Show QR code` icons directly on every plan-backed Journeys row across desktop, mobile and board, and collapse four divergent QR implementations into the one shared hook and modal.

**Architecture:** This is mostly a **deletion**. A working hook (`useTreatmentPlanQr`) and modal (`QrCodeModal`) already exist; three hand-rolled copies of QR generation and one intermediate "PIN or QR?" chooser get removed in favour of them. Two real gaps are filled: the modal gains loading/error/retry by owning the hook itself, and the backend starts reporting the QR link's real expiry instead of the UI hardcoding "20 minutes".

**Tech Stack:** React 18 + TypeScript + Vite, vitest + @testing-library/react, Radix UI (`Dialog`, `Popover`), lucide-react icons, Django REST Framework.

**Spec:** `docs/superpowers/specs/2026-09-12-patient-guide-preview-and-journeys-access-design.md` (§4, FR6–FR11)

**Covers:** FR6, FR7, FR8, FR9, FR10, FR11. FR1–FR5 and FR12–FR15 belong to the companion plan (`2026-09-12-draft-plans-and-guide-preview.md`) and are out of scope here.

---

## Global Constraints

- **Branch:** frontend work happens on `mannieJuly` in `perfect-pixel-playground-project`. Verify with `git rev-parse --abbrev-ref HEAD` before starting — a fix landing on the wrong branch has silently happened on this project before.
- **NEVER run `git add`, `git commit`, or `git push`.** The user performs all VCS operations. Tasks end with verification, not commits. Leave changes in the working tree and report what changed.
- **Do not create new functions by default.** Find the existing helper and reuse it. Where a new one is genuinely required it must be pure where possible, defined **once** in a shared module, and imported — never re-implemented per call site.
- **Frontend typecheck:** `NODE_OPTIONS=--max-old-space-size=8192 npx tsc --noEmit -p tsconfig.app.json`.
  Two traps: the bare root form checks nothing and exits 0; and WITHOUT the raised heap the
  `-p` form dies of OOM (`exit=134`, `Aborted (core dumped)`) after printing an ~880-line
  stack dump that `wc -l` happily reports as if it were an error count. **Check the exit code**
  and count with `grep -c "error TS"`, never `wc -l`. Baseline: **501 errors** — judge by
  **delta**, and grep the output for your own file paths.
- **Frontend tests:** `npx vitest run <path>`.
- **Backend tests:** always `--keepdb`. **NEVER `--noinput`** — it destroys the persistent test DB. Activate the venv first: `source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate`; `manage.py` is at `TreatmentPathBackend/TreatmentPath/manage.py`.
- **Accessible names are required copy, verbatim:** `View treatment plan` and `Show QR code` (FR11).
- The working tree already contains an applied-but-uncommitted cherry-pick of `41558f946` (18 files). Do not revert it.

---

## File Structure

**Created:**
- `src/lib/treatmentPlanAccess.ts` — the single pure eligibility predicate (FR7). One responsibility: given a journey record, can its treatment plan be opened?
- `src/lib/treatmentPlanAccess.test.ts` — its tests.
- `src/hooks/useTreatmentPlanView.ts` — the single PIN-and-open-tab flow (FR8), lifted out of `OpenTable`. One responsibility: mint a verification code for a plan and open the patient view.
- `src/hooks/useTreatmentPlanView.test.ts` — its tests.
- `src/components/journeys/PlanRowActions.tsx` — the two icons as one component, used by every surface. One responsibility: render the eligible actions for a row.
- `src/components/journeys/PlanRowActions.test.tsx` — its tests.

**Modified:**
- `src/hooks/useTreatmentPlanQr.ts` — expose `expiresAt` parsed from the response header.
- `src/components/QrCodeModal.tsx` — own the hook; take `planId` instead of a pre-fetched `qrCodeUrl`; render loading/error/retry and real expiry.
- `src/components/compact/OpenTable.tsx` — delete hand-rolled QR + PIN, drop `TreatmentViewMethodDialog`, mount `PlanRowActions`.
- `src/components/compact/ActiveTable.tsx` — mount `PlanRowActions`.
- `src/pages/Journeys.tsx` — delete hand-rolled QR, drop `TreatmentViewMethodDialog`, mount `PlanRowActions`.
- `src/components/openplans/OpenPlansTableRow.tsx` — delete hand-rolled QR, mount `PlanRowActions`.
- `src/components/compact/JourneyMobileCard.tsx` — mount `PlanRowActions`.
- `src/components/journey-board/JourneyBoard.tsx` — mount `PlanRowActions` on the card.
- `TreatmentPath/TreatmentPlan/views/treatment_plan_views.py` — add `X-Expires-At` to the QR response.

**Deleted (usages only; file may remain until nothing imports it):**
- `src/components/TreatmentViewMethodDialog.tsx` — the intermediate chooser FR9 removes.

---

## Task 1: Shared eligibility predicate

The rule for "can this row open a plan?" currently lives inline in `OpenTable.tsx:1696` as `lead.has_treatment_plan !== false && hasIndividualProcedures(lead)`, with `hasIndividualProcedures` defined locally at `OpenTable.tsx:187` and not exported. Every other surface needs the same rule, so it becomes one pure function.

**Files:**
- Create: `src/lib/treatmentPlanAccess.ts`
- Test: `src/lib/treatmentPlanAccess.test.ts`
- Modify: `src/components/compact/OpenTable.tsx:187-191` (delete local `hasIndividualProcedures`)

**Interfaces:**
- Consumes: nothing.
- Produces: `canAccessTreatmentPlan(record: PlanAccessRecord): boolean` and `type PlanAccessRecord = { id?: string | number | null; has_treatment_plan?: boolean; procedures?: Array<{ procedures?: Array<{ id?: number | string | null; name?: string | null } | null> | null } | null> | null }`. Tasks 6–10 import both.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/treatmentPlanAccess.test.ts
import { describe, expect, it } from "vitest";

import { canAccessTreatmentPlan } from "@/lib/treatmentPlanAccess";

describe("canAccessTreatmentPlan", () => {
  it("allows a record with at least one identifiable procedure", () => {
    expect(
      canAccessTreatmentPlan({
        id: "1",
        procedures: [{ procedures: [{ id: 7, name: "Crown" }] }],
      }),
    ).toBe(true);
  });

  it("allows a procedure carrying only a name", () => {
    expect(
      canAccessTreatmentPlan({
        id: "1",
        procedures: [{ procedures: [{ name: "Crown" }] }],
      }),
    ).toBe(true);
  });

  it("rejects a record explicitly flagged as having no plan", () => {
    expect(
      canAccessTreatmentPlan({
        id: "1",
        has_treatment_plan: false,
        procedures: [{ procedures: [{ id: 7 }] }],
      }),
    ).toBe(false);
  });

  it("rejects a record with no procedures at all", () => {
    expect(canAccessTreatmentPlan({ id: "1", procedures: [] })).toBe(false);
  });

  it("rejects a record whose categories carry empty procedure lists", () => {
    expect(
      canAccessTreatmentPlan({ id: "1", procedures: [{ procedures: [] }] }),
    ).toBe(false);
  });

  it("rejects a record with no id, since no plan can be addressed", () => {
    expect(
      canAccessTreatmentPlan({
        id: null,
        procedures: [{ procedures: [{ id: 7 }] }],
      }),
    ).toBe(false);
  });

  it("tolerates null entries without throwing", () => {
    expect(
      canAccessTreatmentPlan({ id: "1", procedures: [null, { procedures: [null] }] }),
    ).toBe(false);
  });

  it("tolerates missing fields entirely", () => {
    expect(canAccessTreatmentPlan({})).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/treatmentPlanAccess.test.ts`
Expected: FAIL — cannot resolve `@/lib/treatmentPlanAccess`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/treatmentPlanAccess.ts

/**
 * Shape shared by every journey row that can point at a treatment plan —
 * Lead (OpenTable), Patient (ActiveTable), JourneyBoardCard and the mobile card
 * all structurally satisfy it.
 */
export type PlanAccessRecord = {
  id?: string | number | null;
  has_treatment_plan?: boolean;
  procedures?: Array<{
    procedures?: Array<{ id?: number | string | null; name?: string | null } | null> | null;
  } | null> | null;
};

/**
 * The single answer to "can this row open its treatment plan?" (FR7).
 *
 * A row qualifies only when it is addressable (has an id), is not explicitly
 * flagged as plan-less, and carries at least one procedure we can identify.
 * A plan with categories but no procedures renders an empty guide, so it does
 * not qualify.
 *
 * Pure — no fetching, no permission call. Server-side scoping remains the real
 * access control; this only decides whether to offer the action.
 */
export function canAccessTreatmentPlan(record: PlanAccessRecord): boolean {
  if (record.id === null || record.id === undefined || record.id === "") return false;
  if (record.has_treatment_plan === false) return false;

  return (record.procedures ?? []).some((category) =>
    (category?.procedures ?? []).some((procedure) =>
      Boolean(procedure?.id || procedure?.name),
    ),
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/treatmentPlanAccess.test.ts`
Expected: PASS, 8 tests.

- [ ] **Step 5: Remove the now-duplicated local helper**

In `src/components/compact/OpenTable.tsx`, delete the local function at lines 187–191:

```ts
function hasIndividualProcedures(lead: Lead): boolean {
  return (lead.procedures ?? []).some((category) =>
    (category.procedures ?? []).some((procedure) => Boolean(procedure?.id || procedure?.name))
  );
}
```

Add the import at the top of the file:

```ts
import { canAccessTreatmentPlan } from "@/lib/treatmentPlanAccess";
```

Replace the call site at what was line 1696:

```tsx
{lead.has_treatment_plan !== false && hasIndividualProcedures(lead) ? (
```

with:

```tsx
{canAccessTreatmentPlan(lead) ? (
```

- [ ] **Step 6: Verify nothing else referenced the deleted helper**

Run: `grep -rn "hasIndividualProcedures" src`
Expected: no matches.

- [ ] **Step 7: Typecheck**

Run: `NODE_OPTIONS=--max-old-space-size=8192 npx tsc --noEmit -p tsconfig.app.json 2>&1 | grep -c "error TS"`
Expected: no increase over the baseline you recorded before starting. Record the baseline first with the same command.

---

## Task 2: Backend reports the QR link's real expiry

`generate_qr_code` already computes `expires_at` but returns a bare PNG, so the UI hardcodes "This QR code expires in 20 minutes from generation time." in `QrCodeModal.tsx`. That is a second source of truth for a value only the server knows (FR9).

**Files:**
- Modify: `TreatmentPath/TreatmentPlan/views/treatment_plan_views.py:1739-1803`
- Test: `TreatmentPath/TreatmentPlan/tests/test_treatment_plan_qr_expiry.py`

**Interfaces:**
- Consumes: nothing.
- Produces: the `generate-qr` response carries header `X-Expires-At`, an ISO-8601 UTC timestamp equal to the persisted `TemporaryTreatmentPlanLink.expires_at`. Task 3 parses it.

- [ ] **Step 1: Write the failing test**

```python
# TreatmentPath/TreatmentPlan/tests/test_treatment_plan_qr_expiry.py
from datetime import datetime

from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase

from TreatmentPlan.models import TemporaryTreatmentPlanLink


class GenerateQrExpiryHeaderTests(GuidePlanTestCase):
    """The QR response must tell the client when the link dies (FR9).

    Without this the frontend can only hardcode the 20-minute lifetime, which
    silently lies the moment the server-side timedelta changes.
    """

    def setUp(self):
        super().setUp()
        self.plan = self.make_plan()

    def test_response_carries_x_expires_at_matching_the_persisted_link(self):
        url = reverse("treatment-plan-generate-qr", kwargs={"pk": self.plan.pk})
        response = self.client.post(url)

        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertEqual(response["Content-Type"], "image/png")

        header = response.get("X-Expires-At")
        self.assertIsNotNone(header, "X-Expires-At header missing")

        link = TemporaryTreatmentPlanLink.objects.filter(
            treatment_plan=self.plan
        ).latest("created_at")

        self.assertEqual(
            datetime.fromisoformat(header),
            link.expires_at,
            "header must report the same instant that was persisted",
        )
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPlan.tests.test_treatment_plan_qr_expiry --keepdb
```
Expected: FAIL — `X-Expires-At header missing`. If it instead errors on the fixture import, build `guide_fixtures.py` from the Shared Test Fixture section first.

- [ ] **Step 3: Write minimal implementation**

In `treatment_plan_views.py`, immediately after the existing `Content-Disposition` assignment inside `generate_qr_code`:

```python
        response = HttpResponse(img_buffer.getvalue(), content_type="image/png")
        response["Content-Disposition"] = (
            f'inline; filename="treatment_plan_{pk}_qr.png"'
        )
        # The link's lifetime is computed here and nowhere else; report it so the
        # UI never has to restate the 20-minute window as its own constant (FR9).
        response["X-Expires-At"] = temp_link.expires_at.isoformat()

        return response
```

- [ ] **Step 4: Run test to verify it passes**

Run the same command as Step 2.
Expected: PASS.

- [ ] **Step 5: Confirm the header survives CORS to the browser**

Custom response headers are invisible to cross-origin JS unless exposed. Check the CORS config:

Run: `grep -rn "CORS_ALLOW_HEADERS\|CORS_EXPOSE_HEADERS" TreatmentPath/settings/`
- If `CORS_EXPOSE_HEADERS` exists, append `"X-Expires-At"`.
- If it does not exist, add `CORS_EXPOSE_HEADERS = ["X-Expires-At"]` to the same settings module that defines the other `CORS_*` values.

This is not optional — without it Task 3 reads `null` in the browser while passing in tests.

---

## Task 3: Hook exposes the expiry

**Files:**
- Modify: `src/hooks/useTreatmentPlanQr.ts`
- Test: `src/hooks/useTreatmentPlanQr.test.ts`

**Interfaces:**
- Consumes: `X-Expires-At` from Task 2.
- Produces: `useTreatmentPlanQr(planId, enabled)` returns its existing `{ qrCodeUrl, isLoading, error, retry, copyImage, download }` **plus `expiresAt: Date | null`**. Task 4 renders it.

- [ ] **Step 1: Write the failing test**

```ts
// src/hooks/useTreatmentPlanQr.test.ts
import { renderHook, waitFor } from "@testing-library/react";
import { beforeEach, describe, expect, it, vi } from "vitest";

const { mockFetchWithAuth } = vi.hoisted(() => ({ mockFetchWithAuth: vi.fn() }));

vi.mock("@/lib/helpers", () => ({ useFetchWithAuth: () => mockFetchWithAuth }));
vi.mock("sonner", () => ({
  toast: Object.assign(vi.fn(), { success: vi.fn(), error: vi.fn() }),
}));

import { useTreatmentPlanQr } from "@/hooks/useTreatmentPlanQr";

describe("useTreatmentPlanQr", () => {
  beforeEach(() => {
    mockFetchWithAuth.mockReset();
    global.URL.createObjectURL = vi.fn(() => "blob:qr");
    global.URL.revokeObjectURL = vi.fn();
  });

  it("exposes the expiry reported by the server", async () => {
    mockFetchWithAuth.mockResolvedValue({
      ok: true,
      blob: async () => new Blob(["qr"], { type: "image/png" }),
      headers: { get: (name: string) => (name === "X-Expires-At" ? "2026-09-12T10:20:00+00:00" : null) },
    });

    const { result } = renderHook(() => useTreatmentPlanQr("42", true));

    await waitFor(() => expect(result.current.isLoading).toBe(false));
    expect(result.current.expiresAt?.toISOString()).toBe("2026-09-12T10:20:00.000Z");
  });

  it("leaves expiry null when the header is absent", async () => {
    mockFetchWithAuth.mockResolvedValue({
      ok: true,
      blob: async () => new Blob(["qr"], { type: "image/png" }),
      headers: { get: () => null },
    });

    const { result } = renderHook(() => useTreatmentPlanQr("42", true));

    await waitFor(() => expect(result.current.isLoading).toBe(false));
    expect(result.current.expiresAt).toBeNull();
  });

  it("leaves expiry null when the header is unparseable", async () => {
    mockFetchWithAuth.mockResolvedValue({
      ok: true,
      blob: async () => new Blob(["qr"], { type: "image/png" }),
      headers: { get: () => "not-a-date" },
    });

    const { result } = renderHook(() => useTreatmentPlanQr("42", true));

    await waitFor(() => expect(result.current.isLoading).toBe(false));
    expect(result.current.expiresAt).toBeNull();
  });

  it("surfaces an error when generation fails", async () => {
    mockFetchWithAuth.mockResolvedValue({ ok: false, headers: { get: () => null } });

    const { result } = renderHook(() => useTreatmentPlanQr("42", true));

    await waitFor(() => expect(result.current.isLoading).toBe(false));
    expect(result.current.error).toBe("Couldn't generate QR");
    expect(result.current.qrCodeUrl).toBe("");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/hooks/useTreatmentPlanQr.test.ts`
Expected: FAIL — `expiresAt` does not exist on the hook's return value.

- [ ] **Step 3: Write minimal implementation**

The hook currently duplicates its fetch in two places — `generate` (the retry callback) and the `useEffect`. Collapse them: the effect calls `generate`. This removes the duplication rather than adding a third copy of the parse.

Add the pure parser at module scope, above the hook:

```ts
/** The server's `X-Expires-At`, or null when absent/unparseable. Pure. */
function parseExpiresAt(response: { headers?: { get(name: string): string | null } }): Date | null {
  const raw = response.headers?.get("X-Expires-At");
  if (!raw) return null;
  const parsed = new Date(raw);
  return Number.isNaN(parsed.getTime()) ? null : parsed;
}
```

Add the state beside the existing state declarations:

```ts
  const [expiresAt, setExpiresAt] = useState<Date | null>(null);
```

Rewrite `generate` to record it, and to be the single fetch path:

```ts
  const generate = useCallback(async () => {
    if (!planId) return;
    setIsLoading(true);
    setError(null);
    try {
      const response = await fetchWithAuth(
        API_ENDPOINTS.patientTreatmentPlanView.generateQrCode(planId),
        { method: "POST", headers: { "Content-Type": "application/json" } },
      );
      if (!response.ok) throw new Error("Failed to generate QR code");
      const blob = await response.blob();
      const url = window.URL.createObjectURL(blob);
      if (urlRef.current) window.URL.revokeObjectURL(urlRef.current);
      urlRef.current = url;
      setQrCodeUrl(url);
      setExpiresAt(parseExpiresAt(response));
    } catch {
      setError("Couldn't generate QR");
      setQrCodeUrl("");
      setExpiresAt(null);
    } finally {
      setIsLoading(false);
    }
  }, [fetchWithAuth, planId]);
```

Replace the whole body of the existing `useEffect` — the one that repeats the fetch inline — with a call to `generate`, keeping the disabled-state cleanup:

```ts
  useEffect(() => {
    if (!enabled || !planId) {
      if (urlRef.current) {
        window.URL.revokeObjectURL(urlRef.current);
        urlRef.current = "";
      }
      setQrCodeUrl("");
      setError(null);
      setExpiresAt(null);
      return;
    }
    void generate();
    return () => {
      if (urlRef.current) {
        window.URL.revokeObjectURL(urlRef.current);
        urlRef.current = "";
      }
    };
  }, [enabled, generate, planId]);
```

Add `expiresAt` to the returned object.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/hooks/useTreatmentPlanQr.test.ts`
Expected: PASS, 4 tests.

- [ ] **Step 5: Verify existing consumers still pass**

Run: `npx vitest run src/components/modals/treatmentPlan/TreatmentPlanSuccessModal.test.tsx`
Expected: PASS. This modal and `SendToPatientSection` already consume the hook; the return type only gained a field, so they must be unaffected. If they fail, the effect rewrite changed fetch timing — fix that before continuing.

---

## Task 4: QrCodeModal owns the hook

Today callers pre-fetch the image and hand the modal a ready `qrCodeUrl`, which is exactly why three of them hand-rolled the fetch. Inverting this makes the modal self-sufficient and lets Tasks 6–10 be pure deletions. It also removes the modal's duplicated copy/download toast logic, which restates what the hook's `copyImage` / `download` already do.

**Files:**
- Modify: `src/components/QrCodeModal.tsx`
- Test: `src/components/QrCodeModal.test.tsx`

**Interfaces:**
- Consumes: `useTreatmentPlanQr(planId, enabled)` including `expiresAt` (Task 3).
- Produces: `<QrCodeModal isOpen onClose planId patientName />` where `planId: string | null`. The old `qrCodeUrl` and `treatmentPlanId` props are **gone**; Tasks 6–10 must not pass them.

- [ ] **Step 1: Write the failing test**

```tsx
// src/components/QrCodeModal.test.tsx
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { beforeEach, describe, expect, it, vi } from "vitest";

const { mockFetchWithAuth } = vi.hoisted(() => ({ mockFetchWithAuth: vi.fn() }));

vi.mock("@/lib/helpers", () => ({ useFetchWithAuth: () => mockFetchWithAuth }));
vi.mock("sonner", () => ({
  toast: Object.assign(vi.fn(), { success: vi.fn(), error: vi.fn() }),
}));
vi.mock("@/hooks/use-toast", () => ({ toast: vi.fn() }));

import QrCodeModal from "@/components/QrCodeModal";

const okResponse = () => ({
  ok: true,
  blob: async () => new Blob(["qr"], { type: "image/png" }),
  headers: {
    get: (name: string) =>
      name === "X-Expires-At" ? "2026-09-12T10:20:00+00:00" : null,
  },
});

describe("QrCodeModal", () => {
  beforeEach(() => {
    mockFetchWithAuth.mockReset();
    global.URL.createObjectURL = vi.fn(() => "blob:qr");
    global.URL.revokeObjectURL = vi.fn();
  });

  it("generates the QR itself when opened", async () => {
    mockFetchWithAuth.mockResolvedValue(okResponse());

    render(
      <QrCodeModal isOpen onClose={vi.fn()} planId="42" patientName="Ada Lovelace" />,
    );

    await waitFor(() =>
      expect(screen.getByAltText(/QR code for Ada Lovelace/i)).toBeInTheDocument(),
    );
    expect(mockFetchWithAuth).toHaveBeenCalledTimes(1);
    expect(String(mockFetchWithAuth.mock.calls[0][0])).toContain("generate-qr");
  });

  it("does not fetch while closed", () => {
    mockFetchWithAuth.mockResolvedValue(okResponse());

    render(
      <QrCodeModal isOpen={false} onClose={vi.fn()} planId="42" patientName="Ada" />,
    );

    expect(mockFetchWithAuth).not.toHaveBeenCalled();
  });

  it("shows a loading state before the image arrives", async () => {
    let release: (value: unknown) => void = () => {};
    mockFetchWithAuth.mockReturnValue(new Promise((resolve) => { release = resolve; }));

    render(<QrCodeModal isOpen onClose={vi.fn()} planId="42" patientName="Ada" />);

    expect(screen.getByRole("status", { name: /generating qr code/i })).toBeInTheDocument();

    release(okResponse());
    await waitFor(() =>
      expect(screen.queryByRole("status", { name: /generating qr code/i })).not.toBeInTheDocument(),
    );
  });

  it("shows an actionable error and retries", async () => {
    mockFetchWithAuth.mockResolvedValueOnce({ ok: false, headers: { get: () => null } });

    render(<QrCodeModal isOpen onClose={vi.fn()} planId="42" patientName="Ada" />);

    await waitFor(() => expect(screen.getByText(/couldn't generate qr/i)).toBeInTheDocument());

    mockFetchWithAuth.mockResolvedValueOnce(okResponse());
    await userEvent.click(screen.getByRole("button", { name: /try again/i }));

    await waitFor(() =>
      expect(screen.getByAltText(/QR code for Ada/i)).toBeInTheDocument(),
    );
  });

  it("reports the server's expiry rather than a hardcoded window", async () => {
    mockFetchWithAuth.mockResolvedValue(okResponse());

    render(<QrCodeModal isOpen onClose={vi.fn()} planId="42" patientName="Ada" />);

    await waitFor(() => expect(screen.getByTestId("qr-expiry")).toBeInTheDocument());
    expect(screen.getByTestId("qr-expiry").textContent).toMatch(/expires/i);
    expect(screen.queryByText(/20 minutes/i)).not.toBeInTheDocument();
  });

  it("falls back to generic expiry copy when the server sends no header", async () => {
    mockFetchWithAuth.mockResolvedValue({
      ok: true,
      blob: async () => new Blob(["qr"], { type: "image/png" }),
      headers: { get: () => null },
    });

    render(<QrCodeModal isOpen onClose={vi.fn()} planId="42" patientName="Ada" />);

    await waitFor(() => expect(screen.getByTestId("qr-expiry")).toBeInTheDocument());
    expect(screen.getByTestId("qr-expiry").textContent).toMatch(/expires after a short time/i);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/components/QrCodeModal.test.tsx`
Expected: FAIL — the component still requires `qrCodeUrl` and never fetches.

- [ ] **Step 3: Write minimal implementation**

Replace `src/components/QrCodeModal.tsx` entirely:

```tsx
import React from "react";
import {
  Dialog,
  DialogContent,
  DialogDescription,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import { Button } from "@/components/ui/button";
import { Download, Copy, X, Loader2, AlertCircle } from "lucide-react";

import { useTreatmentPlanQr } from "@/hooks/useTreatmentPlanQr";

interface QrCodeModalProps {
  isOpen: boolean;
  onClose: () => void;
  /** Null suppresses generation entirely — used while no row is selected. */
  planId: string | null;
  patientName: string;
}

/**
 * The one QR dialog. It owns generation via `useTreatmentPlanQr`, so callers
 * pass a plan id and nothing else — this is what lets the Journeys surfaces
 * drop their hand-rolled fetches (FR9, FR10).
 */
const QrCodeModal: React.FC<QrCodeModalProps> = ({
  isOpen,
  onClose,
  planId,
  patientName,
}) => {
  const { qrCodeUrl, isLoading, error, expiresAt, retry, copyImage, download } =
    useTreatmentPlanQr(planId, isOpen);

  const expiryLabel = expiresAt
    ? `Expires at ${expiresAt.toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" })}`
    : "This QR code expires after a short time.";

  return (
    <Dialog open={isOpen} onOpenChange={onClose}>
      <DialogContent hideClose={true} className="sm:max-w-md">
        <DialogHeader>
          <DialogTitle className="flex items-center justify-between">
            QR code for patient guide
            <Button variant="ghost" size="icon" onClick={onClose} className="h-6 w-6">
              <X size={16} />
            </Button>
          </DialogTitle>
          <DialogDescription>
            QR code for {patientName}'s patient guide. Patients can scan this to view
            their patient guide without logging in.
          </DialogDescription>
        </DialogHeader>

        <div className="flex flex-col items-center space-y-4 py-4">
          {isLoading ? (
            <div
              role="status"
              aria-label="Generating QR code"
              className="flex h-64 w-64 items-center justify-center rounded-lg border bg-white"
            >
              <Loader2 className="h-8 w-8 animate-spin text-gray-400" />
            </div>
          ) : error ? (
            <div className="flex h-64 w-64 flex-col items-center justify-center gap-3 rounded-lg border bg-white px-4 text-center">
              <AlertCircle className="h-8 w-8 text-red-500" />
              <p className="text-sm text-gray-700">{error}</p>
              <Button variant="outline" size="sm" onClick={() => void retry()}>
                Try again
              </Button>
            </div>
          ) : (
            <div className="rounded-lg border bg-white p-4 shadow-sm">
              <img
                src={qrCodeUrl}
                alt={`QR code for ${patientName}'s patient guide`}
                className="h-64 w-64 object-contain"
              />
            </div>
          )}

          <div className="flex w-full space-x-2">
            <Button
              variant="outline"
              onClick={() => void copyImage()}
              disabled={!qrCodeUrl}
              className="flex-1"
            >
              <Copy size={16} className="mr-2" />
              Copy Image
            </Button>
            <Button onClick={download} disabled={!qrCodeUrl} className="flex-1">
              <Download size={16} className="mr-2" />
              Download
            </Button>
          </div>
        </div>

        <div data-testid="qr-expiry" className="text-center text-xs text-gray-500">
          {expiryLabel}
        </div>
      </DialogContent>
    </Dialog>
  );
};

export default QrCodeModal;
```

Note what this deletes: the local `handleDownload` / `handleCopyImage` and their `@/hooks/use-toast` calls. The hook's `copyImage` and `download` already raise the equivalent toasts through `sonner`; keeping both was the duplication.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/components/QrCodeModal.test.tsx`
Expected: PASS, 6 tests.

- [ ] **Step 5: Find every caller that must now change**

Run: `grep -rn "QrCodeModal" src --include=*.tsx`
Expected: matches in `OpenTable.tsx`, `Journeys.tsx`, `OpenPlansTableRow.tsx`. These will not typecheck until Tasks 6–8. That is expected; note the list and proceed.

---

## Task 5: Shared view-plan flow

`OpenTable.handleGeneratePin` (`OpenTable.tsx:893`) mints a verification code and opens the patient view in a new tab. `Journeys.tsx` has the same flow. FR8 needs it on every surface, so it gets lifted once.

**Files:**
- Create: `src/hooks/useTreatmentPlanView.ts`
- Test: `src/hooks/useTreatmentPlanView.test.ts`

**Interfaces:**
- Consumes: `API_ENDPOINTS.patientTreatmentPlanView.generateSixDigitOneTimePassword`.
- Produces: `useTreatmentPlanView()` returning `{ openPlan(planId: string): Promise<void>; isOpening: boolean; openingPlanId: string | null }`. Tasks 6–10 call `openPlan`.

- [ ] **Step 1: Write the failing test**

```ts
// src/hooks/useTreatmentPlanView.test.ts
import { act, renderHook, waitFor } from "@testing-library/react";
import { beforeEach, describe, expect, it, vi } from "vitest";

const { mockFetchWithAuth, mockToast } = vi.hoisted(() => ({
  mockFetchWithAuth: vi.fn(),
  mockToast: Object.assign(vi.fn(), { success: vi.fn(), error: vi.fn() }),
}));

vi.mock("@/lib/helpers", () => ({ useFetchWithAuth: () => mockFetchWithAuth }));
vi.mock("sonner", () => ({ toast: mockToast }));

import { useTreatmentPlanView } from "@/hooks/useTreatmentPlanView";

describe("useTreatmentPlanView", () => {
  beforeEach(() => {
    mockFetchWithAuth.mockReset();
    mockToast.error.mockReset();
    window.open = vi.fn();
  });

  it("opens the patient view in a new tab with the minted code", async () => {
    mockFetchWithAuth.mockResolvedValue({ ok: true, json: async () => ({ code: "123456" }) });

    const { result } = renderHook(() => useTreatmentPlanView());
    await act(async () => { await result.current.openPlan("42"); });

    expect(window.open).toHaveBeenCalledWith(
      "/verify-password-view-treatment?code=123456&treatment_id=42",
      "_blank",
    );
  });

  it("accepts the verification_code spelling", async () => {
    mockFetchWithAuth.mockResolvedValue({
      ok: true,
      json: async () => ({ verification_code: "654321" }),
    });

    const { result } = renderHook(() => useTreatmentPlanView());
    await act(async () => { await result.current.openPlan("42"); });

    expect(window.open).toHaveBeenCalledWith(
      "/verify-password-view-treatment?code=654321&treatment_id=42",
      "_blank",
    );
  });

  it("never opens a tab when code generation fails", async () => {
    mockFetchWithAuth.mockResolvedValue({ ok: false });

    const { result } = renderHook(() => useTreatmentPlanView());
    await act(async () => { await result.current.openPlan("42"); });

    expect(window.open).not.toHaveBeenCalled();
    expect(mockToast.error).toHaveBeenCalled();
  });

  it("never opens a tab when the response carries no code", async () => {
    mockFetchWithAuth.mockResolvedValue({ ok: true, json: async () => ({}) });

    const { result } = renderHook(() => useTreatmentPlanView());
    await act(async () => { await result.current.openPlan("42"); });

    expect(window.open).not.toHaveBeenCalled();
    expect(mockToast.error).toHaveBeenCalled();
  });

  it("ignores a second click while the first is in flight", async () => {
    let release: (value: unknown) => void = () => {};
    mockFetchWithAuth.mockReturnValue(new Promise((resolve) => { release = resolve; }));

    const { result } = renderHook(() => useTreatmentPlanView());

    act(() => { void result.current.openPlan("42"); });
    await waitFor(() => expect(result.current.isOpening).toBe(true));
    act(() => { void result.current.openPlan("42"); });

    expect(mockFetchWithAuth).toHaveBeenCalledTimes(1);

    release({ ok: true, json: async () => ({ code: "123456" }) });
    await waitFor(() => expect(result.current.isOpening).toBe(false));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/hooks/useTreatmentPlanView.test.ts`
Expected: FAIL — cannot resolve `@/hooks/useTreatmentPlanView`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/hooks/useTreatmentPlanView.ts
import { useCallback, useRef, useState } from "react";
import { toast } from "sonner";

import { API_ENDPOINTS } from "@/config/api";
import { useFetchWithAuth } from "@/lib/helpers";

/**
 * The one "open this plan as the patient sees it" flow (FR8).
 *
 * Mints a six-digit verification code bound to the plan, then opens the
 * verification route in a new tab. A tab is opened ONLY once a usable code is
 * in hand — never speculatively — so a failure can't leave the user staring at
 * a blank or misleading patient page.
 */
export function useTreatmentPlanView() {
  const fetchWithAuth = useFetchWithAuth();
  const [openingPlanId, setOpeningPlanId] = useState<string | null>(null);
  // A ref, not the state value: two clicks in the same tick both read stale state.
  const inFlight = useRef(false);

  const openPlan = useCallback(
    async (planId: string) => {
      if (inFlight.current || !planId) return;
      inFlight.current = true;
      setOpeningPlanId(planId);
      try {
        const response = await fetchWithAuth(
          API_ENDPOINTS.patientTreatmentPlanView.generateSixDigitOneTimePassword(),
          {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ treatment_plan_id: planId }),
          },
        );
        if (!response.ok) throw new Error("Failed to generate verification code");

        const data = await response.json();
        const code = data?.code || data?.verification_code;
        if (!code) throw new Error("No verification code returned");

        window.open(
          `/verify-password-view-treatment?code=${code}&treatment_id=${planId}`,
          "_blank",
        );
      } catch {
        toast.error("Couldn't open the treatment plan. Please try again.");
      } finally {
        inFlight.current = false;
        setOpeningPlanId(null);
      }
    },
    [fetchWithAuth],
  );

  return { openPlan, isOpening: openingPlanId !== null, openingPlanId };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/hooks/useTreatmentPlanView.test.ts`
Expected: PASS, 5 tests.

---

## Task 6: The row actions component

**Files:**
- Create: `src/components/journeys/PlanRowActions.tsx`
- Test: `src/components/journeys/PlanRowActions.test.tsx`

**Interfaces:**
- Consumes: `canAccessTreatmentPlan` / `PlanAccessRecord` (Task 1), `useTreatmentPlanView` (Task 5), `QrCodeModal` (Task 4).
- Produces: `<PlanRowActions record={…} patientName="…" />`. Tasks 7–10 mount exactly this.

- [ ] **Step 1: Write the failing test**

```tsx
// src/components/journeys/PlanRowActions.test.tsx
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { beforeEach, describe, expect, it, vi } from "vitest";

const { mockFetchWithAuth } = vi.hoisted(() => ({ mockFetchWithAuth: vi.fn() }));

vi.mock("@/lib/helpers", () => ({ useFetchWithAuth: () => mockFetchWithAuth }));
vi.mock("sonner", () => ({
  toast: Object.assign(vi.fn(), { success: vi.fn(), error: vi.fn() }),
}));

import PlanRowActions from "@/components/journeys/PlanRowActions";

const eligible = { id: "42", procedures: [{ procedures: [{ id: 7, name: "Crown" }] }] };

describe("PlanRowActions", () => {
  beforeEach(() => {
    mockFetchWithAuth.mockReset();
    global.URL.createObjectURL = vi.fn(() => "blob:qr");
    global.URL.revokeObjectURL = vi.fn();
    window.open = vi.fn();
  });

  it("renders both actions with their accessible names", () => {
    render(<PlanRowActions record={eligible} patientName="Ada" />);

    expect(screen.getByRole("button", { name: "View treatment plan" })).toBeInTheDocument();
    expect(screen.getByRole("button", { name: "Show QR code" })).toBeInTheDocument();
  });

  it("renders nothing for a record with no usable plan", () => {
    const { container } = render(
      <PlanRowActions record={{ id: "42", procedures: [] }} patientName="Ada" />,
    );

    expect(container).toBeEmptyDOMElement();
  });

  it("opens the QR dialog without an intermediate chooser", async () => {
    mockFetchWithAuth.mockResolvedValue({
      ok: true,
      blob: async () => new Blob(["qr"], { type: "image/png" }),
      headers: { get: () => null },
    });

    render(<PlanRowActions record={eligible} patientName="Ada" />);
    await userEvent.click(screen.getByRole("button", { name: "Show QR code" }));

    await waitFor(() =>
      expect(screen.getByAltText(/QR code for Ada/i)).toBeInTheDocument(),
    );
    expect(screen.queryByText(/how would you like to view/i)).not.toBeInTheDocument();
  });

  it("does not generate a QR until the icon is clicked", () => {
    render(<PlanRowActions record={eligible} patientName="Ada" />);
    expect(mockFetchWithAuth).not.toHaveBeenCalled();
  });

  it("opens the patient view via the access-code flow", async () => {
    mockFetchWithAuth.mockResolvedValue({ ok: true, json: async () => ({ code: "123456" }) });

    render(<PlanRowActions record={eligible} patientName="Ada" />);
    await userEvent.click(screen.getByRole("button", { name: "View treatment plan" }));

    await waitFor(() =>
      expect(window.open).toHaveBeenCalledWith(
        "/verify-password-view-treatment?code=123456&treatment_id=42",
        "_blank",
      ),
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/components/journeys/PlanRowActions.test.tsx`
Expected: FAIL — cannot resolve `@/components/journeys/PlanRowActions`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/components/journeys/PlanRowActions.tsx
import React, { useState } from "react";
import { ExternalLink, QrCode } from "lucide-react";

import { Button } from "@/components/ui/button";
import QrCodeModal from "@/components/QrCodeModal";
import { useTreatmentPlanView } from "@/hooks/useTreatmentPlanView";
import { canAccessTreatmentPlan, type PlanAccessRecord } from "@/lib/treatmentPlanAccess";

interface PlanRowActionsProps {
  record: PlanAccessRecord;
  patientName: string;
}

/**
 * `View treatment plan` + `Show QR code`, side by side on a journey row
 * (FR6, FR9, FR11). Mounted by every surface — desktop tables, mobile cards and
 * board cards — so the behaviour cannot drift between them.
 *
 * Renders nothing at all when the record has no usable plan (FR7): the journey
 * row itself stays visible, only these actions disappear.
 */
const PlanRowActions: React.FC<PlanRowActionsProps> = ({ record, patientName }) => {
  const [isQrOpen, setIsQrOpen] = useState(false);
  const { openPlan, openingPlanId } = useTreatmentPlanView();

  if (!canAccessTreatmentPlan(record)) return null;

  const planId = String(record.id);

  return (
    <>
      <Button
        variant="ghost"
        size="sm"
        className="h-8 px-2"
        aria-label="View treatment plan"
        title="View treatment plan"
        disabled={openingPlanId === planId}
        onClick={(event) => {
          event.stopPropagation();
          void openPlan(planId);
        }}
      >
        <ExternalLink className="h-4 w-4" />
      </Button>
      <Button
        variant="ghost"
        size="sm"
        className="h-8 px-2"
        aria-label="Show QR code"
        title="Show QR code"
        onClick={(event) => {
          event.stopPropagation();
          setIsQrOpen(true);
        }}
      >
        <QrCode className="h-4 w-4" />
      </Button>
      <QrCodeModal
        isOpen={isQrOpen}
        onClose={() => setIsQrOpen(false)}
        planId={isQrOpen ? planId : null}
        patientName={patientName}
      />
    </>
  );
};

export default PlanRowActions;
```

`stopPropagation` matters: these icons sit inside rows and cards whose own click handlers open a detail panel.

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/components/journeys/PlanRowActions.test.tsx`
Expected: PASS, 5 tests.

---

## Task 7: OpenTable adopts the shared actions

**Files:**
- Modify: `src/components/compact/OpenTable.tsx` — delete `handleViewLead` (`:887`), `handleGeneratePin` (`:893`), `handleGenerateQr` (`:932`), `handleQrModalClose` (`:963`), the `TreatmentViewMethodDialog` import (`:55`) and element (`:1840`), the `QrCodeModal` element, and the `selectedLeadForView` / `isViewMethodDialogOpen` / `isQrModalOpen` / `qrCodeUrl` state.

**Interfaces:**
- Consumes: `PlanRowActions` (Task 6).
- Produces: nothing for later tasks.

- [ ] **Step 1: Mount the icons in the row's action cell**

In the actions `<td>` (around `:1273`), put the icons **before** the existing `Popover`, so they sit next to the existing action icon rather than inside the menu (FR6):

```tsx
                <td className="px-2 py-4 text-right overflow-hidden">
                  <div className="flex items-center justify-end">
                    <PlanRowActions record={lead} patientName={lead.name} />
                    <Popover>
```

Add the import:

```ts
import PlanRowActions from "@/components/journeys/PlanRowActions";
```

- [ ] **Step 2: Remove the now-redundant popover entry**

Delete the `View treatment plan` button inside the popover (the `canAccessTreatmentPlan(lead) ? (…) : null` block from Task 1 Step 5) — the action is now on the row itself, and FR6 requires it not be menu-only.

- [ ] **Step 3: Delete the hand-rolled QR and PIN flows**

Remove `handleViewLead`, `handleGeneratePin`, `handleGenerateQr`, `handleQrModalClose`, the `TreatmentViewMethodDialog` import and element, the `QrCodeModal` element, and the four pieces of state listed under **Files**. Also remove the `onViewPlan={() => handleViewLead(lead)}` prop passed to `JourneyMobileCard` at `:1284` — Task 9 replaces it.

- [ ] **Step 4: Verify nothing dangles**

Run:
```bash
grep -n "handleViewLead\|handleGeneratePin\|handleGenerateQr\|TreatmentViewMethodDialog\|isViewMethodDialogOpen\|selectedLeadForView" src/components/compact/OpenTable.tsx
```
Expected: no matches.

- [ ] **Step 5: Typecheck and run the suite**

Run: `NODE_OPTIONS=--max-old-space-size=8192 npx tsc --noEmit -p tsconfig.app.json 2>&1 | grep -c "error TS"` — no increase over the 501-error baseline.
Run: `npx vitest run src/components/compact` — PASS.

---

## Task 8: Journeys page adopts the shared actions

**Files:**
- Modify: `src/pages/Journeys.tsx` — the hand-rolled QR fetch at `:1682`, the `TreatmentViewMethodDialog` import (`:64`) and element (`:3388`), and the `QrCodeModal` element (`:3395`).

**Interfaces:**
- Consumes: `PlanRowActions` (Task 6).
- Produces: nothing for later tasks.

- [ ] **Step 1: Read the surrounding handler before editing**

Run: `sed -n '1660,1710p' src/pages/Journeys.tsx` and `sed -n '3380,3410p' src/pages/Journeys.tsx`.
This file is 3,451 lines; identify every piece of state the QR flow owns before deleting, or you will leave orphans.

- [ ] **Step 2: Mount the icons on the row**

Add the import and render `<PlanRowActions record={card} patientName={card.name} />` in the row's action area, beside the existing action icon:

```ts
import PlanRowActions from "@/components/journeys/PlanRowActions";
```

The board card's record shape must satisfy `PlanAccessRecord`. If `JourneyBoardCard` spells the plan id differently from `id`, map it at the call site:

```tsx
<PlanRowActions
  record={{ id: card.recordId, has_treatment_plan: card.has_treatment_plan, procedures: card.procedures }}
  patientName={card.name}
/>
```

- [ ] **Step 3: Delete the hand-rolled QR flow and the chooser**

Remove the `generateQrCode` fetch at `:1682` with its surrounding handler, the `TreatmentViewMethodDialog` import and element, the `QrCodeModal` element, and their state.

- [ ] **Step 4: Verify nothing dangles**

Run: `grep -n "TreatmentViewMethodDialog\|generate-qr\|generateQrCode" src/pages/Journeys.tsx`
Expected: no matches.

- [ ] **Step 5: Confirm the chooser is now unused everywhere**

Run: `grep -rn "TreatmentViewMethodDialog" src --include=*.tsx`
Expected: only `src/components/TreatmentViewMethodDialog.tsx` itself. Delete that file.

- [ ] **Step 6: Typecheck and run the suite**

Run: `NODE_OPTIONS=--max-old-space-size=8192 npx tsc --noEmit -p tsconfig.app.json 2>&1 | grep -c "error TS"` — no increase over the 501-error baseline.
Run: `npx vitest run src/pages/Journeys.test.tsx` — PASS.

---

## Task 9: Mobile card and board card

**Files:**
- Modify: `src/components/compact/JourneyMobileCard.tsx:160-170` (props) and `:420-432` (the `onViewPlan` button)
- Modify: `src/components/journey-board/JourneyBoard.tsx:60-70` (props) and the card's action area

**Interfaces:**
- Consumes: `PlanRowActions` (Task 6).
- Produces: nothing for later tasks.

- [ ] **Step 1: Replace the mobile card's `onViewPlan` button**

In `JourneyMobileCard.tsx`, delete the `onViewPlan` prop from the interface and the `{onViewPlan && (…)}` button at `:423`, and render the shared component in the card's action row instead:

```tsx
<PlanRowActions record={record} patientName={name} />
```

The card must receive the full record. If it currently takes only scalar props, add a single `record: PlanAccessRecord` prop rather than threading `procedures` and `has_treatment_plan` separately.

- [ ] **Step 2: Do the same for the board card**

In `JourneyBoard.tsx`, delete the `onViewPlan: (card: JourneyBoardCard) => void` prop (`:64`) and render `<PlanRowActions record={card} patientName={card.name} />` on the card.

- [ ] **Step 3: Update the callers that pass `onViewPlan`**

Run: `grep -rn "onViewPlan" src --include=*.tsx`
Remove every remaining `onViewPlan={…}` prop, including in `JourneyBoard.test.tsx:75` and `JourneyBoardEditDialog.test.tsx`. In the tests, delete the prop and its assertion rather than leaving a mock that asserts nothing.

- [ ] **Step 4: Typecheck and run the board suite**

Run: `NODE_OPTIONS=--max-old-space-size=8192 npx tsc --noEmit -p tsconfig.app.json 2>&1 | grep -c "error TS"` — no increase over the 501-error baseline.
Run: `npx vitest run src/components/journey-board src/components/compact` — PASS.

---

## Task 10: ActiveTable and OpenPlansTableRow

**Files:**
- Modify: `src/components/compact/ActiveTable.tsx` — mount `PlanRowActions` in the row action cell
- Modify: `src/components/openplans/OpenPlansTableRow.tsx:329` — delete the hand-rolled QR fetch, mount `PlanRowActions`

**Interfaces:**
- Consumes: `PlanRowActions` (Task 6).
- Produces: nothing for later tasks.

- [ ] **Step 1: Mount the icons in ActiveTable**

Mirror Task 7 Step 1 exactly — icons before the row's existing `Popover`, `record={patient}`, `patientName={patient.name}`. FR6 requires Active Plan rows to carry the same actions as Open Plan rows.

- [ ] **Step 2: Delete OpenPlansTableRow's hand-rolled QR**

Run: `sed -n '300,360p' src/components/openplans/OpenPlansTableRow.tsx` first to see what state it owns. Remove the `generateQrCode` fetch and its state, and render `<PlanRowActions record={patient} patientName={patient.name} />` in the row's action area.

- [ ] **Step 3: Confirm every hand-rolled QR call site is gone**

Run: `grep -rn "generateQrCode" src --include=*.tsx --include=*.ts`
Expected: exactly two matches — `src/hooks/useTreatmentPlanQr.ts` and `src/config/api.ts`. Any other match is a copy that survived.

- [ ] **Step 4: Full typecheck and test run**

Run: `NODE_OPTIONS=--max-old-space-size=8192 npx tsc --noEmit -p tsconfig.app.json 2>&1 | grep -c "error TS"` — no increase over the 501-error baseline.
Run: `npx vitest run` — PASS, or only pre-existing failures. Record which failures pre-date this work before starting, so you can tell them apart.

- [ ] **Step 5: Report for the user to commit**

Run: `git status --porcelain` and `git rev-parse --abbrev-ref HEAD`.
Confirm the branch is `mannieJuly`, summarise what changed, and hand off. **Do not commit.**

---

## Manual verification (FR6–FR11)

Do this in the running app before declaring the work done — the tests cover behaviour, not placement.

- [ ] Both icons appear on an Open Plan row in the desktop table, next to the existing action icon, not inside its menu.
- [ ] Both icons appear on an Active Plan row.
- [ ] Both appear on a mobile card at narrow width, and on a board card.
- [ ] A record with no linked plan shows neither icon, and the row is still visible and usable.
- [ ] `Show QR code` opens the dialog directly — no "PIN or QR?" step.
- [ ] The dialog shows a spinner, then the QR, then a real expiry time rather than "20 minutes".
- [ ] Kill the network, click `Show QR code`: an error with a working `Try again`.
- [ ] Double-click `View treatment plan` rapidly: one tab, not two.
- [ ] Hovering each icon shows its tooltip; a screen reader announces `View treatment plan` / `Show QR code`.

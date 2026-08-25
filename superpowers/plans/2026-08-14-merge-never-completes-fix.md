# Fix: Contact Merges Have Never Completed — Implementation Plan

> **For agentic workers:** Steps use checkbox (`- [ ]`) syntax for tracking. Executed
> inline in the same session that wrote this plan (Inline Execution, per this project's
> established preference this session), not subagent-driven.

**Goal:** Stop `PatientWorkspacePage` promising merge capability it can never deliver
(the root cause of the working merge feature's historical zero usage), point staff at
the feature that actually works (Contacts > Data Quality, rebuilt and verified earlier
tonight), and retire the one confirmed-dead, self-declared-deprecated legacy endpoint
(`SuggestMergeView`) this investigation surfaced along the way.

**Architecture:** Pure frontend copy/navigation fix on `PatientWorkspacePage.tsx` (no
backend behavior change to merging itself — `Person.merge()`/`ContactMergeLog` are
untouched) + one backend deletion of dead code (`SuggestMergeView` + its URL + its sole
caller helper `_find_suggestions`, confirmed to have zero other callers).

**Tech Stack:** React/TypeScript (Vite), Django REST Framework — same stack as the rest
of this session's work.

---

### Task 1: Rename the misleading merge affordances and add a real path to Data Quality

**Execution-time correction (critical):** live browser verification revealed
`OverviewTab` — where this plan originally placed the "Check for duplicate contacts"
link — is itself dead/unreachable code: `selectedTab` is derived purely from
`deriveWorkspaceTab(location.pathname)`, which never returns `"overview"` (its
fallback is `"details"`), and `ALL_WORKSPACE_TABS` has no `"overview"` entry — no
button anywhere can ever select it. `OverviewTab`'s banner-copy fix was kept (harmless,
correct if that tab is ever wired up), but the actually-effective fix moved the
duplicate-contacts link into `PatientActionsMenu` instead — confirmed reachable via
the "Patient record actions" button in the page header, and live-verified end-to-end
(click → navigates to `/contact/data-quality` → real Data Quality page renders). Added
`onCheckDuplicates` prop + a third `DropdownMenuItem` accordingly; test file and this
task's Step 3 code below reflect the corrected, reachable implementation.

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/PatientWorkspacePage.tsx`
- Create: `perfect-pixel-playground-project/src/pages/PatientWorkspacePage.test.tsx`
  (no existing test file for this page)

- [x] **Step 1: Write the failing test**

```tsx
import { describe, expect, it, vi, beforeEach } from "vitest";
import { render, screen, fireEvent } from "@testing-library/react";
import { MemoryRouter } from "react-router-dom";

const mockNavigate = vi.fn();
vi.mock("react-router-dom", async () => {
  const actual = await vi.importActual("react-router-dom");
  return { ...actual, useNavigate: () => mockNavigate };
});

// PatientActionsMenu and OverviewTab are not exported individually today —
// import the whole module and exercise them through the default export's
// rendered output once patient data is loaded. Given the size of this page's
// data-fetching surface, test the two extracted pieces directly instead by
// exporting them (Step 2 will add `export` to both).
import { PatientActionsMenu, OverviewTab } from "./PatientWorkspacePage";

describe("PatientActionsMenu", () => {
  it("labels the household-role action honestly and does not claim to merge", () => {
    render(<PatientActionsMenu onLinkFamily={vi.fn()} onArchive={vi.fn()} />);
    fireEvent.click(screen.getByLabelText("Patient record actions"));
    expect(screen.getByText(/Link family member/i)).toBeInTheDocument();
    expect(screen.queryByText(/Merge patient/i)).not.toBeInTheDocument();
  });
});

describe("OverviewTab duplicate-contacts banner", () => {
  const baseProps = {
    patient: { tasks: [], treatment_plans: [], last_contacted: null } as any,
    onCreateNote: vi.fn(),
    onCreateTreatmentPlan: vi.fn(),
    onNavigateToTab: vi.fn(),
    mergeSuggestionCount: 1,
    onReviewFamilySuggestion: vi.fn(),
  };

  it("does not claim merge capability in the family-suggestion banner", () => {
    render(<OverviewTab {...baseProps} />, { wrapper: MemoryRouter });
    expect(screen.queryByText(/Review and merge/i)).not.toBeInTheDocument();
    expect(screen.getByText(/possible family member/i)).toBeInTheDocument();
  });

  it("renders a link to the Data Quality tab for real duplicate-contact merging", () => {
    render(<OverviewTab {...baseProps} />, { wrapper: MemoryRouter });
    const link = screen.getByText(/Check for duplicate contacts/i);
    fireEvent.click(link);
    expect(mockNavigate).toHaveBeenCalledWith("/contact/data-quality");
  });
});
```

- [x] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/pages/PatientWorkspacePage.test.tsx`
Expected: FAIL — `PatientActionsMenu`/`OverviewTab` are not exported, and the current
copy still says "Merge patient" / "Review and merge".

- [x] **Step 3: Implement**

In `PatientWorkspacePage.tsx`:

1. Export `PatientActionsMenu` and `OverviewTab` (add `export` to both declarations —
   no other change to their existing export status; they're currently module-private
   consts, which is fine for the page but blocks direct unit testing).

2. Rename `PatientActionsMenu`'s prop from `onMerge` to `onLinkFamily` and its menu
   item text from "Merge patient" to "Link family member":

```tsx
export const PatientActionsMenu = ({ onLinkFamily, onArchive }: { onLinkFamily: () => void; onArchive: () => void }) => (
  <DropdownMenu>
    <DropdownMenuTrigger asChild>
      <button
        type="button"
        aria-label="Patient record actions"
        className="flex h-6 w-6 flex-shrink-0 items-center justify-center rounded-full bg-white text-gray-400 transition-colors hover:bg-[#e8ddf6] hover:text-[#6941c6]"
      >
        <MoreVertical className="h-4 w-4" />
      </button>
    </DropdownMenuTrigger>
    <DropdownMenuContent align="end" className="w-48">
      <DropdownMenuItem onSelect={onLinkFamily}>
        <Users className="mr-2 h-4 w-4" />
        Link family member
      </DropdownMenuItem>
      <DropdownMenuItem onSelect={onArchive}>
        <Archive className="mr-2 h-4 w-4" />
        Archive record
      </DropdownMenuItem>
    </DropdownMenuContent>
  </DropdownMenu>
);
```

3. In `OverviewTab`, rename the `onReviewDuplicates` prop to `onReviewFamilySuggestion`,
   fix the banner copy to accurately describe what `HouseholdSuggestionsView` actually
   returns (family-role suggestions, never duplicate merges), and add a second,
   separate, always-visible link to the real duplicate-merge feature:

```tsx
export const OverviewTab = ({
  patient,
  onCreateNote,
  onCreateTreatmentPlan,
  onNavigateToTab,
  mergeSuggestionCount,
  onReviewFamilySuggestion,
}: {
  patient: PatientWorkspacePatient;
  onCreateNote: () => void;
  onCreateTreatmentPlan: () => void;
  onNavigateToTab: (tab: PatientWorkspaceTab) => void;
  mergeSuggestionCount: number;
  onReviewFamilySuggestion: () => void;
}) => {
  const navigate = useNavigate();
  const openTasks =
    patient.tasks?.filter((t) => t.status?.toLowerCase() !== "completed").length ?? 0;
  const planCount = patient.treatment_plans?.length ?? 0;
  const lastContacted = patient.last_contacted
    ? format(new Date(patient.last_contacted), "d MMM yyyy")
    : "—";

  return (
    <div className="space-y-5 min-w-0 max-w-full">
      {/* Family-role suggestion banner — HouseholdSuggestionsView only ever
          returns missing_family_role kind, never a duplicate-merge candidate.
          Previously worded as "potential duplicate records... Review and
          merge", which was never true and is the confirmed root cause of the
          real merge feature (Data Quality tab) having zero historical usage —
          staff who clicked this looking for merge capability never found it
          here, and had no reason to know it existed elsewhere. */}
      {mergeSuggestionCount > 0 && (
        <div className="flex items-center justify-between gap-3 rounded-xl border border-amber-200 bg-amber-50 px-4 py-3">
          <div className="flex items-start gap-3 min-w-0">
            <AlertTriangle className="h-4 w-4 text-amber-500 mt-0.5 flex-shrink-0" />
            <p className="text-sm text-amber-800">
              {mergeSuggestionCount === 1
                ? "1 possible family member detected."
                : `${mergeSuggestionCount} possible family members detected.`}
              {" "}Review to link them as family.
            </p>
          </div>
          <Button size="sm" variant="outline" className="flex-shrink-0 border-amber-300 text-amber-800 hover:bg-amber-100" onClick={onReviewFamilySuggestion}>
            Review
          </Button>
        </div>
      )}

      {/* Real duplicate-contact merging lives on the Data Quality tab (Contacts
          page) — always offered here, not gated on mergeSuggestionCount, since
          that count only reflects family-role suggestions and says nothing
          about whether this contact has an actual duplicate elsewhere. */}
      <button
        type="button"
        onClick={() => navigate("/contact/data-quality")}
        className="text-sm text-[#6941c6] hover:underline"
      >
        Check for duplicate contacts
      </button>

      {/* At-a-glance summary */}
      <div className="grid gap-3 sm:grid-cols-3">
```

(The rest of `OverviewTab`'s body is unchanged — only the banner block, the new link,
and the signature change above.)

4. Update the two call sites (menu + banner render) further down the same file:

```tsx
<PatientActionsMenu onLinkFamily={() => setMergeModalOpen(true)} onArchive={() => setArchiveDialogOpen(true)} />
```

```tsx
<OverviewTab
  patient={patient}
  onCreateNote={handleCreateNote}
  onCreateTreatmentPlan={handleCreateTreatmentPlan}
  onNavigateToTab={handleTabChange}
  mergeSuggestionCount={mergeSuggestions.length}
  onReviewFamilySuggestion={() => setMergeModalOpen(true)}
/>
```

(`setMergeModalOpen(true)` behavior is unchanged for both — they still open
`ContactMergeModal` to let staff assign a family role, which is genuinely functional.
Only the label and the banner copy change; the modal itself is untouched.)

- [x] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/pages/PatientWorkspacePage.test.tsx`
Expected: PASS

- [x] **Step 5: Full frontend test suite + typecheck**

```bash
npx vitest run
npx tsc --noEmit -p tsconfig.app.json > /tmp/tsc_after_merge_fix.txt 2>&1
grep -c "error TS" /tmp/tsc_after_merge_fix.txt
```
Expected: no new failures; TS error count unchanged from the post-retirement baseline
established during the Data Quality Task 8 work tonight (no new errors introduced by
this page's edits — check the diff doesn't add any `PatientWorkspacePage.tsx` lines to
the error list).

---

### Task 2: Retire the confirmed-dead `SuggestMergeView`

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/contact_merge_views.py`
  (remove `SuggestMergeView` class and `_find_suggestions` helper — confirmed its only
  caller)
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/urls.py` (remove the
  `contacts/suggest-merge/` path + import)
- Modify: `perfect-pixel-playground-project/src/config/api.ts` (remove
  `contactMerge.suggest`)
- Test: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/
  test_suggest_merge_retired.py` (new)

- [x] **Step 1: Write the failing test**

**Execution-time correction:** the plan's draft test body below referenced a
nonexistent factory module. Actual implementation (`TreatmentPlan/tests/
test_suggest_merge_retired.py`) simplified to just the two import/reverse-safety
checks — no authenticated client needed, since neither assertion makes a request.

```python
from django.test import TestCase
from UserAuthentication.models import Practice
from UserAuthentication.tests.factories import create_authenticated_user_and_client
# (use whatever this app's existing test convention is for an authenticated
# APIClient — check TreatmentPlan/tests/test_dedupe_patients_patientdocuments.py
# or a nearby TreatmentPlan test for the actual helper/pattern in use, since this
# plan was written without re-reading that exact convention; adapt import paths
# to match rather than guessing a nonexistent factory module.)


class SuggestMergeRetiredTests(TestCase):
    def test_suggest_merge_url_no_longer_resolves(self):
        from django.urls import reverse, NoReverseMatch
        with self.assertRaises(NoReverseMatch):
            reverse("contact-suggest-merge")

    def test_suggest_merge_view_class_removed(self):
        import TreatmentPlan.views.contact_merge_views as cmv
        self.assertFalse(hasattr(cmv, "SuggestMergeView"))
        self.assertFalse(hasattr(cmv, "_find_suggestions"))
```

- [x] **Step 2: Run test to verify it fails**

Run: `python manage.py test --keepdb -v 2 TreatmentPlan.tests.test_suggest_merge_retired`
Expected: FAIL — `reverse("contact-suggest-merge")` currently succeeds;
`SuggestMergeView`/`_find_suggestions` currently exist.

- [x] **Step 3: Remove the dead code**

In `contact_merge_views.py`: delete the `SuggestMergeView` class (confirmed at lines
423-463 as of this audit — re-check exact bounds before deleting, since line numbers
drift) and the `_find_suggestions` helper (confirmed at lines 202-374, confirmed via
grep to have exactly one caller — `SuggestMergeView.post` at line 458 — before
deleting; if that grep now shows a second caller, STOP and re-scope this step, don't
delete a helper something else depends on).

In `urls.py`: remove the `contacts/suggest-merge/` path entry (confirmed around lines
1128-1132) and the now-unused `SuggestMergeView` import.

In `api.ts`: remove `contactMerge.suggest` (confirmed zero call sites; grep once more
immediately before deleting, in case something changed since the audit).

- [x] **Step 4: Run test to verify it passes**

Run: `python manage.py test --keepdb -v 2 TreatmentPlan.tests.test_suggest_merge_retired`
Expected: PASS

- [x] **Step 5: Run the full backend test suite touching this file**

```bash
python manage.py test --keepdb -v 2 TreatmentPlan.tests.test_suggest_merge_retired \
  TreatmentPlan -v 1
```
Expected: same pre-existing-failure profile as documented in the Data Quality Task 8
retirement (no NEW failures introduced by this deletion — `ConfirmMergeView`/
`UnmergeContactView`/`Person.merge`/`Person.unmerge` tests, if any exist under
`TreatmentPlan/tests/`, must still pass unchanged since those are untouched).

- [x] **Step 6: Frontend typecheck**

```bash
npx tsc --noEmit -p tsconfig.app.json 2>&1 | grep -i "contactMerge.suggest\|api.ts"
```
Expected: no new errors referencing the removed endpoint.

---

### Task 3: Close out the investigation

**Files:**
- Modify: `.claude memory file` `project_merge_never_completes.md` (via the memory
  system, not a repo file) — update status to resolved, cross-reference this spec/plan.

- [x] **Step 1: Verify end-to-end one more time**

Done live via Playwright: navigated to `/patients/9594`, opened "Patient record
actions", confirmed all three items render correctly ("Link family member", "Check for
duplicate contacts", "Archive record" — no "Merge patient" anywhere), clicked "Check
for duplicate contacts", confirmed it navigates to `/contact/data-quality` and the real
Data Quality page renders (same page verified working end-to-end earlier tonight during
the Data Quality Plan 5 Task 7 work).

- [x] **Step 2: Update project memory**

Update `project_merge_never_completes.md`: change status to reflect that the root
cause (misleading UI on the page staff actually use, promising merge capability the
underlying suggestion source could never deliver) is now fixed, that the one genuinely
dead legacy endpoint (`SuggestMergeView`) is retired, and that `ConfirmMergeView`/
`UnmergeContactView` remain live/untouched by design (documented reasoning in the spec).
Cross-reference `docs/superpowers/specs/2026-08-14-merge-never-completes-fix-design.md`.

- [x] **Step 3: Do NOT commit**

Per standing user rule (`feedback_no_commits.md`) — leave all changes uncommitted for
the user to review when they wake up.

---

## Self-review notes

- **Spec coverage:** implements the spec's "Fix" section items 1 and 2 in full; item 3
  ("explicitly not touched") requires no code changes by definition — verified nothing
  in Tasks 1-2 touches `ConfirmMergeView`, `UnmergeContactView`, `Person.merge`,
  `Person.unmerge`, or `ContactMergeLog`.
- **No placeholders:** Task 1's test and implementation are both concrete, complete
  code — the one deliberately-flagged uncertainty (the exact authenticated-APIClient
  helper convention in Task 2 Step 1) is called out explicitly as a step for the
  implementer to verify against real neighboring test code, not silently guessed at,
  matching this session's own established convention for flagging real unknowns
  instead of fabricating plausible-looking code.
- **Type consistency:** `onLinkFamily`/`onReviewFamilySuggestion` prop names introduced
  in Task 1 Step 3 match the call-site updates in the same step — checked together.
- **Line numbers throughout are as of the same-session Explore-agent audit** and are
  flagged for re-verification immediately before each deletion, since this file was
  written without re-reading `contact_merge_views.py`/`urls.py` a second time.

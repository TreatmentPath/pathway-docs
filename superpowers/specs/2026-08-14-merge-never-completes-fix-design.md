# Fix: Contact Merges Have Never Completed in Production

**Status:** Approved for autonomous implementation per explicit user instruction
(2026-08-13 session: "so after testing everything... we will write a spec and a plan
for this and also implement so I want you to help me to do this end to end
[[project-merge-never-completes]], I will be sleeping"). Written and implemented
without the normal interactive brainstorming dialogue, since the user pre-authorized
end-to-end autonomous work while asleep. Investigation and design decisions below are
traceable to the prior investigation recorded in project memory
`project_merge_never_completes.md` and a same-session Explore-agent audit of current code.

## Background — what was already known

Prior investigation (recorded in memory, not repeated here) established:

- `TreatmentPlan_person.merged_into_id` is NULL for all 83,008 Persons in prod — zero
  merges have ever completed, while 56 dismissals recorded fine in the same session
  (proving staff DO reach the merge UI, they just never complete a merge).
- Three frontend paths could reach a Person merge:
  - **Path 1** (six journey tables — Intake/Nurture/Open/Active/CustomJourney):
    confirmed dead-by-design, not a bug. `ContactMergeModal`'s `onResolveSuggestion`
    prop short-circuits `handleConfirm` before it can reach the real merge endpoint —
    correct behavior for that flow's actual purpose (linking a new record to an
    existing patient), which was never meant to be a Person merge. **Out of scope,
    left untouched.**
  - **Path 2** (`PatientPanelSheet`'s auto-triggered modal, `PatientWorkspacePage`'s
    "Merge patient" menu item and "Review duplicates" banner): structurally alive
    (no short-circuit — `onResolveSuggestion` is never passed) but functionally dead,
    because its only suggestion source, `HouseholdSuggestionsView`, hardcodes
    `"kind": "missing_family_role"` on every response and can never emit
    `"contact_match"`. `ContactMergeModal.handleConfirm`'s real-merge branch
    (`contact_match`/`unlinked_match`/`existing_patient_review`) is therefore
    unreachable code on this path.
  - **Path 3** (Contacts page → Data Quality → Duplicates): the one path with no
    code-level dead end, calling the real `Person.merge()`. Open question at the time:
    why zero usage despite being fully wired.

## What changed since that investigation (this session, same conversation)

Path 3's entire backend and frontend were independently rebuilt as part of a separate,
already-completed "unified Data Quality model" project (5 phases, `docs/superpowers/
plans/2026-08-13-*.md`), for unrelated reasons (consolidating three fragmented
duplicate-detection/import-error systems into one `DataQualityIssue` model). The old
`DataQualityMergeView` this investigation referenced as "Path 3" was retired in that
project's final task and replaced by `DataQualityIssueViewSet.merge`
(`dataQuality/views.py`) + `DataQualityIssuesScreen.tsx`, which:

- Calls `Person.merge(winner, loser, performed_by=...)` directly — same underlying
  mechanism, no behavior change to the merge itself.
- Is practice-scoped (`Person.objects.get(pk=winner_id, practice=practice)` on both
  sides) — satisfies the standing practice-scoped-merging rule.
- Was manually verified end-to-end in a live browser session this same night: login →
  Data Quality tab loads → duplicate_contact issue expands to a winner-selection
  picker → merge action → real backend write confirmed via direct DB query.
- Is reachable by default: `feature: "contacts"` gates the whole `/contact/*` route,
  and `"contacts"` defaults to `True` for `practice_admin`/`practice_dentist`/
  `practice_staff`/`practice_member` roles (`access_control.py`) — not admin-only, no
  additional gate found on the Data Quality tab specifically.

**This means Path 3 is no longer a hypothesis — it is a verified-working merge path
as of tonight.** The remaining open question from the investigation — "why did the
one working path have zero usage" — now has a concrete, evidence-backed answer,
established via a same-session code audit (see below), not a data investigation.

## Root cause of Path 3's historical zero-usage

`PatientWorkspacePage.tsx` — the page staff actually work from when looking at a
specific patient — has a "Merge patient" menu item (`PatientActionsMenu`, dropdown)
and a "Review duplicates" banner ("N potential duplicate records detected... Review
and merge or link as family to keep data clean") that both open `ContactMergeModal`
fed by `HouseholdSuggestionsView`. Per the confirmed dead-Path-2 finding, that
suggestion source can **only ever** produce `missing_family_role` items — it has never
been able to detect or offer a real duplicate merge. A staff member who clicks "Merge
patient" or "Review duplicates" on this page — the natural, most-visible place to look
for this functionality — either sees nothing (no household suggestions) or a
family-role assignment prompt, never an actual duplicate-merge option. The banner's own
copy ("potential duplicate records detected... Review and merge") is not just dead —
it is actively **incorrect**: it never fires for genuine duplicates, only for
missing-family-role cases, and even then the word "merge" in its copy is never
actionable.

**This is a plausible, sufficient explanation for zero merges in production**: the
one working feature (now on the Data Quality tab) was never advertised from the page
where staff would actually notice a duplicate, while the page that DID advertise
"merge" capability could never deliver it. Staff who wanted to merge a duplicate had
no reason to know a separate, differently-labeled tab elsewhere in the app could do it.

## Fix

### 1. Stop `PatientWorkspacePage` promising merge capability it cannot deliver

- Rename `PatientActionsMenu`'s "Merge patient" menu item to accurately describe what
  it does — assign a family relationship to a household suggestion — and rename the
  banner copy to drop the false "merge" promise. These are the only two UI surfaces
  that currently claim merge capability without being able to deliver it.
- Add a real, honest path from this page to the feature that DOES work: a "Check for
  duplicate contacts" link that navigates to the Contacts page's Data Quality tab
  (`/contact` with the Data Quality tab pre-selected), where staff can actually
  complete a merge. This directly closes the discoverability gap the original
  investigation flagged as the remaining open question.

### 2. Retire `SuggestMergeView` (`/contacts/suggest-merge/`)

Confirmed via the same-session audit: this view is self-declared deprecated in its own
docstring, still registered in `urls.py`, and has **zero live callers** anywhere in the
frontend (the one endpoint config entry that points to it, `API_ENDPOINTS.contactMerge.
suggest()`, has zero call sites; the only other frontend reference is a test asserting
it is NOT called). It is the sole code path that could ever construct a `contact_match`
suggestion for Path 2's already-dead modal usage. Deleting it removes confirmed,
self-declared-dead code with no live consumer — consistent with this session's
established "no orphaned dead code" practice (same pattern as Task 8 of the Data
Quality retirement).

### 3. Explicitly NOT touched, and why

- **`ConfirmMergeView`** (`/contacts/confirm-merge/`) — also currently unreachable by
  any live UI path (nothing feeds it a real `contact_match` payload today, since its
  only suggestion source is the now-deleted `SuggestMergeView`), but it implements a
  materially different code path (Case 1: reassigning a person-less record's `person`
  FK directly, with no `Person.merge()` call and no `ContactMergeLog` at all) that the
  new Data Quality flow does not replicate. There is no positive evidence it is safe to
  delete outright — it may have non-UI consumers (scripts, a mobile client, direct API
  use) that this audit cannot rule out. Leaving it live is zero-risk; deleting it
  without more certainty is not. Out of scope for tonight.
- **`UnmergeContactView`** (`/contacts/unmerge/`) — must stay regardless of anything
  else in this document. It is the only endpoint that can undo a `Person.merge()`,
  including merges performed through the new, now-primary Data Quality flow. Retiring
  it would remove the undo capability for the feature this fix is trying to make
  people actually use.
- **Path 1** (journey-table flows) — confirmed correct-by-design in the prior
  investigation, not a merge path at all. No change.

## Testing

- Frontend: `PatientWorkspacePage.tsx` currently has no test file. Add one covering:
  the renamed menu item / banner copy, and that the new "Check for duplicate contacts"
  link navigates to the Data Quality tab.
- Backend: a test asserting `/contacts/suggest-merge/` no longer resolves (404) after
  removal, and that `SuggestMergeView`/`_find_suggestions` are gone from
  `contact_merge_views.py` (import-safety — nothing else in the codebase imports them).
- No changes to `Person.merge()`/`ContactMergeLog`/`ConfirmMergeView`/
  `UnmergeContactView` — no new tests needed there; existing coverage (and tonight's
  Data Quality merge-action tests) already exercises the shared merge machinery this
  fix relies on without modifying it.

## Out of scope (tracked separately, not part of this fix)

- Item 4 from `project_prod_duplicate_findings.md` (Go's Dentally sync drops phone
  numbers when they can't reach E.164, unlike Django's fallback) — a real, separate,
  already-documented open bug, unrelated to why merges don't complete.
- The 148 genuine cross-Dentally-record duplicate pairs identified in the prod
  investigation — a data cleanup task that becomes actionable once this fix ships and
  staff can actually find and use the merge feature, not a code fix itself.
- Any further UI/UX polish of the Data Quality tab beyond what Phase 5 already built
  and verified tonight.

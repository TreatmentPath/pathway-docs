# Patient guide preview, draft plans, and Journeys access — design

**Date:** 2026-09-12
**Status:** Approved design, not yet implemented
**Source PRD:** "Patient guide preview and Journeys access — functional requirements" (FR1–FR15)
**Repos:** `perfect-pixel-playground-project` (branch `mannieJuly`), `TreatmentPathBackend`

---

## 1. Summary

Three changes ship together:

1. **Draft treatment plans + guide preview.** A new `draft` plan state lets staff save an
   unfinished patient guide, preview it exactly as the patient will see it, leave, and come
   back to finish it.
2. **Journeys row access.** View-plan and QR icons on every plan-backed journey row, with the
   intermediate "PIN or QR?" chooser removed.
3. **Create-only treatment plans.** Issued plans can no longer be edited through the guide
   create flow; only drafts can.

---

## 2. Decisions taken during design

| Question | Decision |
|---|---|
| How the draft is saved | **Preview auto-saves a draft**, then renders it. FR3 is amended accordingly. |
| Can a draft be edited? | **Yes — drafts are editable, issued plans are not.** FR13's create-only rule applies only after a plan leaves draft. (Briefly reversed on 2026-09-12, then reinstated by the user the same day — the PRD stands.) |
| Where drafts appear | **Docs page and the patient's record**, marked as drafts. Hidden everywhere else. |
| Draft lifetime | **30 days**, then cleaned up. |
| `41558f946` (guide + patient-layout polish) | **Port that commit alone** onto `mannieJuly`; leave the consent rework on `jbSeptember`. Applied cleanly, no conflicts. |
| What the preview shows | **The full patient-facing guide** — treatments, pricing, welcome message, photos, reviews, and attached documents (retaining inline image/PDF preview and the file-detail/download fallback). Not documents alone. |

### 2.1 Amendments to the PRD

- **FR3** previously forbade creating or updating *any* treatment plan on preview. Amended:
  opening the preview **may write a draft plan**. It must still not issue a plan, upload staged
  documents, send email or SMS, generate a verification code, generate a QR code, or record
  patient-guide engagement.
- **FR13** previously forbade entering update mode at all. Amended: a create session must not
  update an **issued** plan. Resuming a draft loads it by id and updates it — that is the
  feature, not a violation.

FR12, FR13 and FR15 were briefly withdrawn on 2026-09-12 in favour of an "editable until active"
rule, then reinstated by the user the same day. **The PRD's create-only requirement stands as
written**, with the draft carve-out above as its only exception.

### 2.2 Standing constraint

Per the user's rule (2026-09-12): **do not add new functions by default.** Find the existing
helper and reuse it. Where a new one is genuinely required it must be pure where possible,
defined once in a shared module, and imported — never re-implemented per call site. This design
deliberately removes more duplicated logic than it adds.

---

## 3. Section 1 — Draft state and preview

### 3.1 Why not the obvious shortcut

The patient-facing view is a **public, unauthenticated** endpoint —
`/treatment-plan/public/plans/<id>/?code=…` — deliberately outside the subscription gate so a
patient who is not logged in can open their guide. Previewing through it would require minting
a `SixDigitVerificationCode` for a draft, making an unfinished guide publicly reachable by
anyone holding that code. Rejected.

### 3.2 Chosen approach: staff preview endpoint + the patient page in preview mode

- A new **authenticated, practice-scoped** endpoint returns the **same payload shape** as the
  public view, for draft and issued plans alike. It must **reuse the public view's existing
  serializer**, not fork it — the two payloads diverging is precisely the failure the
  no-duplication rule exists to prevent. The endpoint differs from the public one only in how
  the caller is authorised (session auth + practice scope, instead of a verification code).
- `Services.tsx` gains a second data source — fetch-by-auth instead of fetch-by-code — and
  renders identically from it. **One renderer**, so the preview *is* the guide rather than a
  lookalike that drifts.
- No code minted, no public exposure, no engagement tracking.

Two approaches were rejected:

- *Mint a code and open the public URL* — reintroduces exactly the exposure FR3 guarded against.
- *Render client-side from the create form's state* — was the right answer before drafts
  existed; now it means maintaining a second form-state→guide-shape mapping that will silently
  drift. Rejected on the no-duplication rule.

### 3.3 Backend

- Add `("draft", "Draft")` to `TreatmentPlan.status` choices + migration.
  Current choices: `pending / active / processing / processed / completed / cancelled`
  (`TreatmentPath/TreatmentPlan/models.py:3787`).
- `pending` **is** "Open Plan" and is filtered for explicitly across the backend, so drafts stay
  out of those lists for free. **The real work is the leakage sweep**: enumerate every query
  that fetches treatment plans with *no* status filter — counts, reporting, patient workspace,
  journey boards, the Go sync — and exclude drafts from each. This must be enumerated during
  implementation, not estimated now.
- Note: `status` has no DB-level constraint (Django `choices` is Python-only), so adding a value
  does not require Go-side changes. Whether Go *reads* plans unfiltered is part of the sweep.
- Confirming creation flips `draft` → `pending`. That is the moment the plan becomes real and,
  per FR13, stops being editable.
- A scheduled cleanup removes drafts older than **30 days**.

### 3.4 Frontend

- **Preview** saves/updates the draft, then opens a full-screen in-app view with **Back to edit**
  and **Continue to create**. Because the draft is persisted, nothing is lost even on a hard
  refresh — satisfying FR2 without holding state in memory.
- FR4 validation (patient and treatments present) gates the preview using the **same** check the
  create action already uses — reused, not restated.
- Drafts listed on the Docs page and the patient's record, clearly marked.

---

## 4. Section 2 — Journeys row actions and QR (FR6–FR11)

### 4.1 Existing duplication to remove

QR generation exists four times:

| Location | Status |
|---|---|
| `src/hooks/useTreatmentPlanQr.ts` | The proper shared hook — keep |
| `src/components/compact/OpenTable.tsx:932` | Hand-rolled duplicate — delete |
| `src/components/openplans/OpenPlansTableRow.tsx:329` | Hand-rolled duplicate — delete |
| `src/pages/Journeys.tsx:1682` | Hand-rolled duplicate — delete |

FR9/FR10 are therefore largely a **deletion**. All surfaces route through the existing hook plus
the existing `QrCodeModal`. `TreatmentViewMethodDialog` (the intermediate PIN-or-QR chooser)
stops being used.

### 4.2 The two real gaps

- **`QrCodeModal` has no loading / error / retry state** — callers currently pre-fetch the image
  and hand it a ready URL. Move the hook **inside** the modal so it takes a `planId`. One
  component owns QR display; call sites reduce to `<QrCodeModal planId … />`. This also deletes
  the copy/download toast logic the modal currently duplicates against the hook's own
  `copyImage` / `download`.
- **Expiry is not knowable by the frontend.** The link expires 20 minutes after generation, but
  that is hardcoded server-side and the endpoint returns a bare PNG
  (`views/treatment_plan_views.py:1750`). FR9 requires showing expiry. Rather than hardcode
  "20 minutes" in the UI as a second source of truth, the endpoint gains an **`X-Expires-At`
  response header** from the `expires_at` it already computes. Backwards-compatible.

### 4.3 Row actions

- View-plan and QR icons together on the row, next to the existing action icon, across **desktop
  tables, mobile cards, and board views** for eligible Open Plan and Active Plan records. Not
  behind an overflow menu (FR6).
- View-plan (FR8) reuses the existing PIN flow currently written inline in
  `OpenTable.handleGeneratePin` — lifted to one shared place rather than re-implemented per
  surface.
- Eligibility (FR7): no plan link, or a plan that is missing/stale/out of practice scope →
  action suppressed or disabled, journey row still visible, clear error if attempted.
- Accessible names and tooltips: `View treatment plan`, `Show QR code` (FR11).
- Overlapping requests for the same record are prevented (FR10) by the hook's single in-flight
  state rather than per-call-site guards.

---

## 5. Section 3 — Create-only treatment plans (FR12–FR15)

- Remove the `Quick Edit` / `Comprehensive` dialog from `OpenTable.tsx` (`:1880`) and
  `ActiveTable.tsx` (`:1687`), and the equivalent Journeys plan-edit chooser.
- `planId` is **not** banned outright — it is **allowed only when the referenced plan is a
  draft**. A `planId` pointing at an issued plan, and any `editMode=comprehensive`, redirects to
  a fresh guide with a clear create-only message.
- `updateTreatmentPlan` survives **solely** as the draft-save path
  (`src/hooks/useTreatmentPlanCreation.ts`).
- The rule is enforced **server-side as well**, not only by hiding buttons: the update endpoint
  must reject content edits to a plan that is no longer a draft. A hidden button is not access
  control.
- FR14: moving records, assigning staff, editing journey notes, and creating tasks are
  untouched. This removes treatment-plan editing, not operational journey actions.
- FR15: update-specific confirmation, success, and button copy removed; create language only —
  except within the draft flow, where "save draft" wording is correct.

### 5.1 Consequence worth noting

Removing Quick Edit also removes the only route that currently edits an open plan's priority,
assignee and treatment categories in one place (`OpenTable.handleEditSubmit`, `:603`). Priority
and assignee remain reachable through the separate operational actions FR14 preserves. **Changing
an issued plan's treatments becomes impossible by design** — the replacement is to create a new
guide. Confirm that is intended before the code is deleted.

This also retires a latent risk rather than inheriting it: that handler maps treatments to
`{ category_id, procedures: [] }`, an empty procedure list per category, which would discard the
priced procedures beneath them if the endpoint replaces categories wholesale. Unverified, and now
moot on this path.

## 6. Section 4 — Testing

**Backend**
- Draft plans excluded from every list that previously returned all plans (one test per query
  found in the §3.3 sweep).
- `draft` → `pending` on confirmed creation.
- 30-day cleanup removes only drafts, only past the threshold.
- Preview endpoint refuses cross-practice access and unauthenticated access.
- `X-Expires-At` matches the persisted `TemporaryTreatmentPlanLink.expires_at`.

**Frontend**
- Preview renders without issuing a plan, sending, or minting a code.
- Back-to-edit preserves every draft selection.
- QR dialog: loading state, error + retry, no overlapping requests for one record.
- Edit entry points gone; a legacy `planId`/`editMode` URL lands on a fresh create flow.
- Draft `planId` resumes rather than redirecting.
- The update endpoint rejects a content edit to a non-draft plan even when called directly.

Run backend tests with `--keepdb`, never `--noinput`. Frontend typecheck must use
`-p tsconfig.app.json` (root `tsc --noEmit` checks nothing); judge by delta against the
~491 pre-existing errors.

---

## 7. Risks and open items

- **The leakage sweep is the main unknown.** If a plan query without a status filter is missed,
  half-finished guides appear in production counts or reporting. Every such query needs a test.
- **`41558f946` is uncommitted.** It is applied to the working tree on `mannieJuly` and awaits
  the user's commit. All VCS operations are the user's.
- **Draft plans and the Go sync.** Go writes to `TreatmentPlan`. Whether it reads plans without
  a status filter is unverified and belongs in the sweep.
- **`Services.tsx` is 722 lines** and is being given a second data source. If the change makes
  it unwieldy, extracting the data-loading concern is in scope; a broader rewrite is not.

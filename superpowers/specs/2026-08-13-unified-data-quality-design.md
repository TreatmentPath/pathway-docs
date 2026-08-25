# Unified Data Quality — Design

**Status:** Approved for planning, 2026-08-13
**Author:** Claude (brainstormed with user)

## Problem

The Data Quality screen (`/contact/data-quality/*`, `DataQualityContent.tsx`) has four
sub-tabs — Duplicates, Non-Duplicates, Duplicate Contacts, Missing Info — that look like
one feature but are backed by four structurally independent systems:

| Sub-tab (route) | UI label | Frontend | Backend | Model |
|---|---|---|---|---|
| `duplicates` (default) | "conflicts" | `DuplicatesTab.tsx` (standalone fetch) | `DentallyImportFailureViewSet` | `dentallyIntegration.DentallyImportFailure` |
| `non-duplicates` | "errors" | `NonDuplicatesTab.tsx` (standalone fetch) | `DentallyImportErrorViewSet` | `dentallyIntegration.DentallyImportError` |
| `duplicate-contacts` | "duplicates" | `DuplicateContactsTab.tsx` via `useDataQuality` | `DataQualityDuplicatesView`/`MergeView`/`DismissDuplicateView` | `TreatmentPlan.Person`/`PersonChannel`/`ContactChannel`/`DuplicateClusterDismissal` |
| `missing-info` | "Missing Info" | `MissingInfoTab.tsx` via `useDataQuality` | `DataQualityMissingInfoView`/`DeleteView` | `TreatmentPlan.Patient`/`Intake` |

Concretely, this causes:

- **Two separate Django apps** power one screen (`dentallyIntegration` vs `TreatmentPlan`),
  with no shared model, no shared dismissal mechanism, no shared classifier vocabulary.
- **Confusing labels**: the default tab ("conflicts") and the tab that actually merges
  duplicate Persons ("duplicates", i.e. `duplicate-contacts`) look like near-synonyms but
  hit entirely different systems. This is a likely contributor to
  [[project-merge-never-completes]] — staff may simply never find the working merge path.
- **A real cross-service dual-writer risk**: both Django's Celery sync task
  (`dentallyIntegration/tasks.py`) AND Go's migration importer
  (`EmailServiceGo/internal/dentally/migration/service.go persistImportIssue`) write
  directly into `dentallyintegration_dentallyimportfailure` /
  `dentallyintegration_dentallyimporterror` via independently hand-maintained column
  maps (Go via raw `.Table()` calls, no shared ORM model). This is exactly the shape of
  bug the project's Go↔Django schema-parity tooling exists to catch, but today it isn't
  registered as a single source of truth either side can drift against consistently.
- **Two of the four issue types are computed live, not stored**: `duplicate-contacts` and
  `missing-info` run an expensive scan (raw SQL + Python fuzzy-name-matching, in one case
  18,305 candidate channel-groups pre-filter on local data) on every page load, with
  nothing materialized — no history, no way to track resolution over time, no way for a
  future consumer (e.g. a dashboard, an alert) to query "open issues" cheaply.
- **Dead code**: `useDataQuality.fetchDuplicates()` still fetches and transforms
  `dentallyImportFailures` into `dentally-group-*` items that `DuplicateContactsTab`
  immediately filters out — vestigial from before `DuplicatesTab.tsx` got its own direct
  fetch.

## Goal

Replace all four systems with **one unified, classifier-driven data-quality model**:
one Django app, one table, one set of writers (Django + Go, both event-driven for
import-time issues; Go alone, nightly, for scan-based issues), one resolution/dismissal
mechanism, one frontend screen.

**Out of scope**: `ContactMergeModal`, the journey-table identity-review flow
(`handleResolveIdentityReview`), and the Patient Workspace "Merge" button. These solve a
different problem (linking a new record to an existing patient during conversion) and
share some backend plumbing (`Person.merge`) but are not part of this redesign. See
[[project-merge-never-completes]] for that separate, still-open investigation.

## Design

### 1. Data model

New Django app **`dataQuality`** (camelCase, matching `dentallyIntegration`/`teamChat`
convention), one model:

```python
class DataQualityIssue(models.Model):
    practice = models.ForeignKey(Practice, on_delete=models.CASCADE)

    classifier = models.CharField(max_length=32, choices=[
        # import-time — event-driven, written by both Django Celery + Go migration importer
        ("duplicate_import", "Duplicate (import)"),
        ("validation_error", "Validation error"),
        ("missing_data_import", "Missing data (import)"),
        ("invalid_phone_import", "Invalid phone (import)"),
        ("other_import_error", "Other import error"),
        # scan-based — written nightly by Go's cron sweep
        ("duplicate_contact", "Duplicate contact"),
        ("missing_info", "Missing info"),
    ])
    status = models.CharField(max_length=16, choices=[
        ("open", "Open"), ("dismissed", "Dismissed"), ("resolved", "Resolved"),
    ], default="open")
    source = models.CharField(max_length=16, choices=[
        ("django_sync", "Django sync"), ("go_migration", "Go migration"), ("go_sweep", "Go sweep"),
    ])

    # Generic subject reference. "patient"/"intake" for import-issue and missing_info
    # rows (record_id = that row's pk); "person_cluster" for duplicate_contact rows
    # (record_id = a synthetic cluster key, e.g. the shared channel id — member Person
    # ids live in `detail`, there being no single natural FK for an N-member cluster).
    record_type = models.CharField(max_length=16)
    record_id = models.CharField(max_length=64)

    detail = models.JSONField(default=dict)  # conflict_details, missing_fields, cluster members, etc.

    # ONLY meaningful for the five import-time classifiers — null for duplicate_contact
    # and missing_info, which are not Dentally-import-specific (they scan every
    # Patient/Intake regardless of origin, including manually-created contacts).
    # Doubles as the upsert key Go/Django's Celery task already use today
    # (practice_id + dentally_patient_id) to update-in-place rather than duplicate a
    # row on retry, for those five classifiers.
    dentally_patient_id = models.BigIntegerField(null=True, blank=True)

    resolved_by = models.ForeignKey(User, null=True, blank=True, on_delete=models.SET_NULL)
    resolved_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        indexes = [
            models.Index(fields=["practice", "classifier", "status"]),
            models.Index(fields=["practice", "dentally_patient_id"]),
        ]
```

This one table retires **four** things: `DentallyImportFailure`, `DentallyImportError`,
`DuplicateClusterDismissal`, and the ad hoc `discard`/delete-based dismissal used by the
import-error tabs — `status` replaces all of them with one field.

**Upsert keys** (how a writer avoids creating duplicate rows on retry):
- Import-time classifiers: `(practice_id, dentally_patient_id)`.
- `missing_info`: `(practice_id, classifier, record_type, record_id)`.
- `duplicate_contact`: `(practice_id, classifier, record_id)` where `record_id` is the
  cluster's shared-channel id (stable across sweeps as long as the cluster's membership
  doesn't change the grouping channel).

### 2. Writers

**Import-time (event-driven, trigger point unchanged, target table changes):**
- Django's Celery task (`dentallyIntegration/tasks.py`) — currently `update_or_create`s
  into `DentallyImportFailure`/`DentallyImportError` keyed on `is_duplicate`. Repoint to
  `DataQualityIssue`, same key, classifier chosen the same way.
- Go's migration importer (`persistImportIssue`, `service.go` ~line 1959) — same change:
  the two raw `.Table("dentallyintegration_dentallyimport...")` calls become one
  `.Table("dataquality_dataqualityissue")` call; `failure_type` becomes `classifier`
  directly (no more if/else routing to two table names).
- This closes the dual-writer drift risk: both languages now write the exact same
  columns into the exact same table instead of two independently hand-maintained column
  maps into two legacy tables.

**Scan-time (new, nightly, Go-owned — not Django):**

Decision: implemented **entirely in Go**, not as a Go-triggered/Django-computed call.
Rationale: this is a full per-practice table scan (18,305 candidate channel-groups
pre-filter on local data alone) with an O(n²) pairwise name-comparison inside each
channel group — exactly the workload Django/gunicorn (synchronous WSGI workers) is
worst at and Go (goroutine worker pool) is best at. Calling into Django from Go would
relocate the bottleneck and add network overhead for no benefit. It also matches the
codebase's existing division of labor: Go already owns direct, heavy read/write access
to `PersonChannel`/`ContactChannel` (`canonicalTwinID`, `linkPatientToPersonAndChannels`)
for the same reason.

- New cron entry in `EmailServiceGo/internal/dentally/scheduler/scheduler.go` (alongside
  the existing `0 2 * * *` daily sync), running **nightly** per active practice:
  - `duplicate_contact` detector: port `DataQualityDuplicatesView`'s logic (shared-channel
    grouping + `_name_similar` fuzzy match + DOB-conflict exclusion) from Python to Go.
  - `missing_info` detector: `Patient`/`Intake` rows with blank email AND phone — same
    predicate as today's Django view, trivial to port.
  - Reconciliation: any `open` row whose underlying condition no longer holds (person
    now has an email, cluster no longer shares a channel) flips to `resolved`
    automatically, not only via manual dismiss.
- **Drift guard for the ported heuristic**: `_name_similar` gets duplicated into Go. To
  prevent silent drift between the two copies (the project has a known unguarded case of
  exactly this — the `9`-minimum-suffix canonical-twin constant, duplicated with no
  shared fixture), add a committed JSON truth-table fixture
  (`{a, b, expectedMatch}` cases) that both a Python test and a Go test assert against.

**Cadence**: nightly (matches the existing 2:00 UTC slot and the project's snapshot-
reconciliation full-sweep pattern). Rejected hourly — these are slow-moving issues, and a
full per-practice scan at that frequency is unnecessary DB load.

### 3. Schema-guard / parity-check integration

No changes needed to the guard or agent themselves — both already generalize:

- `EmailServiceGo/internal/db/schemaguard.go`'s `requiredColumns` gets a new entry for
  `dataquality_dataqualityissue`, generated via
  `DATABASE_URL=... go run ./cmd/schemaguard_gen dataquality_dataqualityissue` (do not
  hand-write it). Once Go's raw SQL/GORM writes to this table exist, the existing
  `TestSchemaGuard_EveryGoWrittenTableIsRegistered` coverage test automatically fails if
  the entry is missing — it derives the Go-written table set from source, not a
  hand-maintained list.
- `.claude/agents/go-django-parity-guard.md` walks "all six failure modes" against
  whatever tables Go currently writes, re-derived from source each run — it already
  covers this table once Go writes to it. No edit needed to the agent definition.
- Rollout checklist item: run `/parity-check` after the Go writers land, to confirm
  registration explicitly rather than assuming it "just works."

### 4. Resolution actions

One `status` transition per action, replacing five different per-classifier removal
mechanisms (two dismissal tables, one discard flag, one bare delete, one
filtered-out-of-response):

| Classifier(s) | Actions | Effect |
|---|---|---|
| `duplicate_contact` | Merge, Dismiss | Merge → `Person.merge()` (unchanged, already correct) + `status="resolved"`. Dismiss → `status="dismissed"`. |
| `duplicate_import` | Force-import, Dismiss | Existing force-import logic, now flips this row's status instead of a separate dismissal table. |
| `validation_error`, `other_import_error` | Retry, Edit-and-retry, Delete source row | Existing `DentallyImportErrorViewSet` retry/create-patient logic, repointed. |
| `missing_data_import`, `invalid_phone_import` | Edit-and-retry, Discard | Same. |
| `missing_info` | Edit contact, Delete | Existing `DataQualityDeleteView` logic, now also flips status to `resolved` on success. |

No change to the actual fix logic anywhere in this table — only the bookkeeping.

**Open item, flagged for the implementation plan, not resolved here**: `DuplicatesTab`/
`NonDuplicatesTab` have additional confirm-dialog nuance today (family-import,
replace-existing, override-replace) that wasn't fully enumerated during design. The
implementation plan should re-read both components in full before building the unified
action handlers, to make sure none of that nuance is dropped.

### 5. Frontend

One `DataQualityContent` screen, one table, driven by `classifier` + `status` instead of
four routed sub-tabs hitting four different hooks:
- Filter chips (All / Duplicate Contacts / Import Duplicates / Validation Errors /
  Missing Data / Invalid Phone / Missing Info) replace the current tab bar — fixes the
  "conflicts" vs "duplicates" label confusion, since every row now carries its own
  classifier label directly.
- One row-action area per row, keyed off `classifier` per the Section 4 table.
- `useDataQuality` collapses to one hook: `fetchIssues({classifier?, status?})`,
  `resolveIssue(id, action, payload)`, `dismissIssue(id)`.

### 6. Rollout order

1. `dataQuality` app + `DataQualityIssue` model + migration (Django).
2. Repoint Django's Celery task writer to the new model. Direct cutover (no dual-write
   window) — this is internal staff tooling, not patient-facing, so the risk of a brief
   gap is low relative to the complexity of maintaining a temporary dual-write path.
3. Repoint Go's `persistImportIssue` writer; register the table in `schemaguard.go` via
   `schemaguard_gen`; run `/parity-check` to confirm the coverage tests catch it.
4. Build the Go nightly sweep (duplicate_contact + missing_info detectors, ported
   `_name_similar` + shared fixture, cron entry).
5. Rewire the four frontend tabs into one; retire `DentallyImportFailure`/
   `DentallyImportError`/`DuplicateClusterDismissal` models once nothing reads them.

## Non-goals / explicitly deferred

- Fixing the `ContactMergeModal`/journey-table merge dead-ends
  ([[project-merge-never-completes]]) — separate, still-open investigation.
- Retrying the removed `dentally-group-*` dead code path in `useDataQuality` — it's
  deleted as part of this rewrite, not preserved or fixed.
- A shared Go/Django ORM layer for `DataQualityIssue` — Go continues to write via raw
  SQL/GORM `.Table()`, matching the existing pattern for every other Go-written table in
  this codebase (no new abstraction introduced for this one model).

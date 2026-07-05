# Daylist Confirmations — Phase 4: Cohort/Bucket Targeting Engine

## Context

Phases 1-3 built the confirmation data model, the appointment-anchored sequence engine, and UI surfacing. None of them let staff control *which* appointments actually get enrolled in a confirmation sequence — `enroll_eligible_appointments` (`dentallyIntegration/confirmation_automation.py`) currently enrolls every eligible future appointment unconditionally. This phase adds that targeting layer (PRD FR4/FR4a-d, FR5, FR6, FR8a, FR12e), plus a related gap discovered while designing it: confirmation-automation (and daylist-automation) message sends currently show up in the staff Inbox, when they should be hidden the same way Recall automation sends already are.

This phase spans three repositories: `TreatmentPathBackend` (Django), `perfect-pixel-playground-project` (React), and `EmailServiceGo` (the Go sync service).

## Part A: Cohort/bucket targeting engine

### The bucket catalog

A curated, non-generic catalog of exactly the 7 named buckets FR5 requires — not a generic field/operator/value rule builder (FR5 asks for specific named presets, and a fully generic builder would be scope beyond what's actually required). Each bucket is a small, pure, testable predicate function.

New file: `dentallyIntegration/cohort_buckets.py`

```python
BUCKET_REGISTRY = {
    "high_risk": {
        "label": "High risk",
        "group": "risk",
        "apply": lambda qs: qs.filter(risk_tier="High"),
    },
    "medium_risk": {
        "label": "Medium/Elevated risk",
        "group": "risk",
        "apply": lambda qs: qs.filter(risk_tier="Elevated"),
    },
    "booked_under_30d": {
        "label": "Booked less than 30 days ago",
        "group": "booking_age",
        "apply": lambda qs: qs.filter(booking_age_days__lt=30),
    },
    "booked_over_30d": {
        "label": "Booked over 30 days ago",
        "group": "booking_age",
        "apply": lambda qs: qs.filter(booking_age_days__gte=30),
    },
    "booked_over_60d": {
        "label": "Booked over 60 days ago",
        "group": "booking_age",
        "apply": lambda qs: qs.filter(booking_age_days__gte=60),
    },
    "long_and_old_booking": {
        "label": "Over 45 minutes and booked over 7 days ago",
        "group": "compound",
        "apply": lambda qs: qs.filter(duration__gt=45, booking_age_days__gt=7),
    },
    "zero_balance": {
        "label": "GBP 0 balance",
        "group": "balance",
        "apply": lambda qs: qs.filter(patient_outstanding_balance=0),
    },
}
```

`booking_age_days` is a computed value, `now - DentallyAppointment.booked_at` (an existing field, `models.py:830`, documented as "When the appointment was originally booked in Dentally — source of truth for lead-time scoring" — already the correct field for this, no new field needed), expressed as a queryset annotation rather than a stored column, since it changes continuously with the passage of time and shouldn't be a stale synced value.

**Combining rule:** buckets selected within the same `group` combine with OR (multi-select, per FR4a's "High and Elevated risk" example); buckets from different groups combine with AND. `"compound"` is its own single-entry group, so it never combines with anything — selecting it is an atomic yes/no choice, matching how FR5 phrases it as one bucket, not two.

### Targeting configuration

One new JSON field on `ConfirmationSequence`: `targeting` (`default=dict`), shaped as:

```json
{
  "base_audience": { "bucket_keys": ["booked_over_30d"] },
  "sub_cohorts": [
    {
      "name": "VIP opt-in",
      "priority": 1,
      "bucket_keys": ["high_risk"],
      "confirm_dental_opt_in": true,
      "template_override_id": null
    }
  ]
}
```

`base_audience.bucket_keys` empty means "all eligible appointments" — today's unconditional behavior stays the default for any sequence that doesn't configure targeting. Sub-cohorts are matched in ascending `priority` order against appointments already in the base audience; the first sub-cohort an appointment matches wins (FR4c's precedence) and its `confirm_dental_opt_in`/`template_override_id` override the sequence's default treatment for that appointment. An appointment matching no sub-cohort gets the base treatment.

A serializer-level validator (not a DB constraint, matching how `steps` JSON is validated today) checks: every `bucket_keys` entry exists in `BUCKET_REGISTRY`; `priority` values are unique within `sub_cohorts`.

### Enrollment integration

`enroll_eligible_appointments` (`confirmation_automation.py`) gains one new filtering step: after building `candidates` (today's window + already-enrolled exclusion), apply `targeting.base_audience.bucket_keys` via the registry before the per-appointment eligibility loop. Inside that loop, a new helper `resolve_sub_cohort(appointment, targeting)` returns the matching sub-cohort dict or `None`; this result is stored on `ConfirmationSequenceEnrollment` via one new nullable field, `sub_cohort_name` (`CharField`), so downstream send logic (which template, whether to include the opt-in link) knows the treatment for that enrollment without re-resolving it.

### Preview-count endpoint

New action on the confirmation-sequence viewset, `GET .../preview-count/`, modeled directly on `RecallRecordViewSet.preview_count` (`recall_views.py:1086`): accepts candidate (not-yet-saved) `targeting` JSON as a query param, applies it against the same eligibility base `enroll_eligible_appointments` uses, and returns the FR4d/FR6 breakdown:

```json
{
  "base_audience_count": 42,
  "sub_cohorts": [{"name": "VIP opt-in", "count": 10}],
  "overlap_deduped_count": 10,
  "final_send_count": 42,
  "sendable_sms": 38,
  "sendable_email": 40,
  "excluded": 3,
  "already_confirmed": 5,
  "already_cancelled": 2,
  "missing_contact": 1
}
```

Stateless, read-only, no persistence — same as its Recall precedent.

### Risk tier: new synced field

`risk_tier` is currently computed only by Go (`EmailServiceGo/internal/dentally/daylist/noshow/risk.go`), on-request, never persisted. Add:

- **Django migration**: new column `DentallyAppointment.risk_tier` (`CharField`, nullable or blank-default, with a `db_default` matching this project's Go↔Django parity rule — Go writes this column via raw SQL with an explicit column list, so any NOT-NULL column needs a `db_default` or the sync breaks).
- **Go change**: `risk.go`'s existing per-appointment risk computation gets one more write, alongside the other risk-enrichment fields it already writes each sync cycle (`patient_outstanding_balance` et al.) — same column set pattern, just one more field.

## Part B: Inbox visibility + Daylist activity panel

### The problem

Recall automation sends are hidden from the staff Inbox today via `messaging/views/session_views.py`'s session queryset, which excludes sessions whose only messages have `message_purpose="recall"`. Confirmation-automation and daylist-automation sends both use a different, generic label (`message_purpose="automation"`), which this exclusion does **not** catch — so both currently show up in the regular Inbox as if they were normal staff-initiated conversations.

### The fix: allow-list instead of exclude-list

Change `session_views.py`'s filtering logic from "hide sessions where the only messages are `recall`" to "show sessions where at least one message is `manual`". This is a one-time change that automatically hides every current and future automated-send label (`recall`, `automation`, anything added later) without needing a repeated code change per new label — the tradeoff being this also newly hides daylist-automation's currently-visible sends, which is an accepted, intentional side effect (daylist-automation messages are one-way automated sends, same category as recall/confirmation, and arguably shouldn't have been visible in the general Inbox either).

Apply the equivalent allow-list logic across all three message types the session queryset already checks (`EmailMessages`, `SMSMessage`, `WhatsAppMessage`).

### New Daylist activity panel

Since this hides both daylist-automation and confirmation-automation messages from the Inbox, staff need somewhere to still see that history — mirroring how `RecallActivityPanel.tsx` exists for Recall.

**Backend**: new `DaylistReportingViewSet` (`dentallyIntegration/views/`), modeled directly on `RecallReportingViewSet` (`recall_sequence_views.py:165`) — same `mode=count`/`mode=events` shape, same patient/contact + time-window scoping — but filtered on `source_type__in=["daylist_automation", "confirmation_automation"]` rather than `message_purpose="recall"`, since both message types share the generic `"automation"` purpose and are distinguished only by `source_type`.

**Frontend**: new `DaylistActivityPanel.tsx` (`src/components/patients/patient-panel/`), modeled directly on `RecallActivityPanel.tsx` — same summary-row (total contact attempts, per-channel counts) + message-feed structure (reusing the shared `ActivityFeed`/`ActivityItem` components), calling the new endpoint. Shows daylist-reminder and confirmation-automation messages together in one panel, since both are conceptually part of the Daylist feature area — not two separate panels.

### Scorecard opt-in indicator (FR12e)

A small badge on the appointment card (alongside Phase 3's confirmation-status corner badge) indicating an appointment was sent through the confirm.dental opt-in sub-cohort treatment — sourced from `ConfirmationSequenceEnrollment.sub_cohort_name` being non-null and that sub-cohort's `confirm_dental_opt_in` being true.

## Frontend: bucket/cohort setup UI (FR4b-d)

A setup screen for each confirmation sequence:
- A checklist of the 7 buckets, visually grouped by `group` (so it's clear which checkboxes are either/or vs. must-all-match).
- A way to add a sub-cohort: its own bucket checklist, an opt-in toggle, a priority/rank, and (optionally) a template override.
- A live preview panel showing the `preview-count` endpoint's breakdown, updating as staff change selections, before anything is saved.

## Out of scope for this phase

- A fully generic rule builder (arbitrary field/operator/value) — FR5's v1 list is fixed and named; a generic builder is future work if ever needed.
- Historical backfill of `risk_tier` for past appointments — only appointments synced going forward get the field populated by Go.
- Changing Recall's own Inbox-hiding behavior — it already works correctly and is untouched by this phase.

## Testing

- Unit tests for each of the 7 bucket predicates in isolation.
- Tests for group-based OR/AND combination logic (same-group multi-select = OR; cross-group = AND).
- Tests for sub-cohort priority resolution (an appointment matching two sub-cohorts gets the lower-priority-number one).
- Tests for `enroll_eligible_appointments` with targeting configured (only matching appointments enrolled; sub-cohort tagging correct).
- Tests for the preview-count endpoint (counts match what enrollment would actually do if saved).
- A Go↔Django parity test for the new `risk_tier` column (matching this project's existing parity-guard pattern for shared-DB fields).
- Tests for the Inbox allow-list change (a session with only automation/recall messages is hidden; a session with any manual message is shown; mixed sessions with both are shown).
- Tests for the new `DaylistReportingViewSet` (`mode=count`/`mode=events` return correct counts/message lists, scoped correctly by patient/contact and source_type).

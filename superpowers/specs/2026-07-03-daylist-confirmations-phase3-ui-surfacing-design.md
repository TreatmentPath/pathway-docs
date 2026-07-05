# Daylist Confirmations — Phase 3: UI Surfacing

## Context

Phase 1 (`2026-07-02-daylist-confirmations-phase1-data-model-design.md`) added a confirmation audit trail to `DentallyAppointment` and a derivation function, `compute_confirmation_display_status`, returning one of 5 states. Phase 2 (`2026-07-03-daylist-confirmations-phase2-sequence-engine-design.md`) added the automatic enrollment/sending engine (`ConfirmationSequence`/`ConfirmationSequenceEnrollment`). Neither phase exposed anything to the Daylist frontend. This phase makes confirmation state visible to practice staff: on the appointment card, in the filter, at the practice-scorecard level, and in a patient's history.

This phase spans both repos: `TreatmentPathBackend` (serializer exposure + activity-log integration) and `perfect-pixel-playground-project` (all UI surfaces).

## The 6th state: "Suppressed"

The original gap analysis deferred a 6th FR11 state, "Suppressed," pending the sequence engine (Phase 2) existing. Re-examined now that it does: every stop reason Phase 2 records (`confirmed`/`cancelled`/`cancellation_requested`/`no_valid_contact`) already maps onto one of the 5 existing display statuses — so a naive "enrollment.status == 'stopped'" check would never fire as a genuinely new state. The one case that IS new: the appointment is still `Awaiting confirmation` by its own fields, but its most recent confirmation enrollment has already `completed` (sent every step, no reply) or `stopped` for some other reason — meaning no more automated messages are coming. That's the real, useful 6th state: **"automation is done trying — this needs a human."**

Staff-facing label: **"Needs follow-up"** (not the internal term "Suppressed" — clearer at a glance on a card).

## Backend changes (`TreatmentPathBackend`)

### 1. Extend `compute_confirmation_display_status` (`dentallyIntegration/confirmation_status.py`)

```python
def compute_confirmation_display_status(appointment):
    if appointment.state == "cancelled":
        return "Cancelled"
    if appointment.cancellation_requested:
        return "Cancellation requested"
    if appointment.patient_confirmed:
        return "Confirmed"
    if not has_valid_contact(appointment):
        return "No valid contact"
    latest_enrollment = appointment.confirmation_enrollments.order_by("-created_at").first()
    if latest_enrollment and latest_enrollment.status in ("completed", "stopped"):
        return "Needs follow-up"
    return "Awaiting confirmation"
```

`NOT_ELIGIBLE_STATUSES` (Phase 2's shared constant, used by `is_appointment_eligible_for_confirmation`) needs `"Needs follow-up"` added — an appointment already flagged as needing manual follow-up shouldn't be re-enrolled in another automated sequence pass.

### 2. Serializer exposure (`dentallyIntegration/serializers.py`, `DayListAppointmentSerializer`)

Two new fields:
- `confirmation_status = serializers.SerializerMethodField()` → `compute_confirmation_display_status(obj)`.
- `confirmation_enrollment = serializers.SerializerMethodField()` → `None` if no enrollment exists, else `{"sequence_name": ..., "status": ..., "next_due_at": ..., "stopped_reason": ..., "last_sent_step": ...}` from the most recent `ConfirmationSequenceEnrollment` (same `order_by("-created_at").first()` lookup as above — implementer should factor this into a shared helper rather than querying twice per appointment in the serializer, given this runs per-row on a list endpoint).

### 3. Activity-log integration

**Confirmation-automation sends** (`dentallyIntegration/confirmation_automation.py`, `send_sms`/`send_email`): resolve a `ContactIdentity` via `ContactIdentity.get_or_create_for_contact(practice=practice, email=appointment.patient_email, phone=appointment.patient_phone, name=appointment.patient_name)` (same call recall_automation.py already makes), set it as `contact=` on the created `SMSMessage`/`EmailMessages` row, then call `on_record_created(entity=message, user=None)` — mirroring `recall_automation.py`'s own `_log_activity` wrapper exactly (best-effort, swallow exceptions, never let activity-log failure break a send).

**Patient confirms / requests cancellation** (`dentallyIntegration/views/appointment_confirm_views.py`, `AppointmentConfirmViewSet.create`): `DentallyAppointment` has no `contact` FK of its own, so this can't use `on_record_created` directly (which requires the entity to carry `contact`). Use the lower-level `ActivityLogHelper.log(practice, activity_type, entity=appointment, contact=contact, is_system_generated=False, action_family="appointment", source_system="pathway_automation")` instead — it takes `entity` and `contact` as separate parameters, which fits this case (the "thing that happened" is the appointment; the "whose log it appears under" is the resolved contact). `action_family="appointment"` (not the method's default `"patient_edit"`) matches the existing `ACTION_FAMILY_CHOICES` set and correctly categorizes these as appointment-related events.

**New `ActivityLog.ACTIVITY_TYPE_CHOICES` values** (`activityLog/models.py`): `confirmation_sent`, `patient_confirmed`, `cancellation_requested`. **New `entity_type` label**: `"dentallyappointment"` added to `ENTITY_TYPE_LABELS` (`record_events.py`). **`source_system`**: reuse existing `pathway_automation` — no new choice needed, matches how the confirmation-automation `source_type` tag already labels these sends elsewhere.

**No backfill.** Only appointments confirmed/messaged going forward get activity-log entries — nothing to backfill since none of Phases 1-2 shipped to production yet.

## Frontend changes (`perfect-pixel-playground-project`)

### 1. Types (`src/pages/day-list/types.ts`)

```ts
export type ConfirmationStatus =
  | 'Confirmed'
  | 'Awaiting confirmation'
  | 'Cancellation requested'
  | 'Cancelled'
  | 'No valid contact'
  | 'Needs follow-up';

export interface ConfirmationEnrollment {
  sequence_name: string;
  status: 'active' | 'completed' | 'stopped';
  next_due_at: string | null;
  stopped_reason: string;
  last_sent_step: number | null;
}
```

Add `confirmation_status: ConfirmationStatus` and `confirmation_enrollment: ConfirmationEnrollment | null` to the `Appointment` interface, alongside the existing `patient_confirmed?: boolean` (left in place — `confirmation_status` supersedes it for display purposes, but the raw boolean stays for anything still reading it directly).

### 2. Appointment card (`components/AppointmentCard.tsx`) — FR11, FR12a

The corner slot (currently `showRiskCorner`-gated risk badge) becomes a single status indicator. The backend already computes the single correct `confirmation_status` value per appointment (per the priority order baked into `compute_confirmation_display_status` above) — the frontend does NOT re-derive or re-prioritize anything, it's a direct switch on that one value:

- `'Cancelled'` → grey "Cancelled" badge
- `'Cancellation requested'` → amber "Cancellation requested" badge
- `'Confirmed'` → green "Confirmed" badge
- `'No valid contact'` → grey "No valid contact" badge
- `'Needs follow-up'` → purple "Needs follow-up" badge
- `'Awaiting confirmation'` → fall back to today's risk logic — red "High risk"/amber "Elevated risk" badge if `no_show_risk.status` is High/Elevated, otherwise nothing renders in this slot (this is the ONLY case where the risk badge still shows, satisfying FR12a: any other confirmation_status value means risk is never shown, even if the appointment happens to be High risk).

The existing inline `Confirmed`/`Cancelled` badges near the patient name (lines ~644-661 today) are removed — this corner slot is now the single place confirmation state shows. The existing `Completed` badge (driven by Dentally appointment `status`, not confirmation state) is unaffected and stays where it is.

### 3. Filter (`components/FilterDialog.tsx`) — FR12, FR12b

New `confirmationFilter: ConfirmationStatus[]` prop (defaults to all 6 selected — no filtering), rendered as a multi-select checkbox group, following the same pattern as the existing risk-tier multi-select used elsewhere in this codebase. Independent of the existing `statusFilter` (Dentally appointment status) — a different dimension, not merged into the same dropdown.

### 4. Practice-level coverage (`components/SummaryCards.tsx`) — FR12c, FR12d

New `SummaryCardKey` value `'confirmed'`, with a `CARD_STYLE` entry (green accent, matching the existing 4-key `Record` shape). Card shows "Confirmed: X / Y" for the currently-filtered date/list. Clicking it toggles the list filter to show only non-confirmed appointments (`confirmation_status !== 'Confirmed'`), the same click-to-filter behavior the existing opportunity cards already have. `PractitionerScorecards`' `MixDonut` (hardcoded to the 4 opportunity categories) is NOT touched — confirmation coverage is a different dimension and doesn't belong in that visualization.

### 5. History (`components/patients/patient-panel/PatientActivityFeed.tsx`) — FR13

Add 3 new `ActivityType` entries (`confirmation_sent`, `patient_confirmed`, `cancellation_requested`) to the existing icon-by-type map, following the same pattern as the existing `consent`/`sms`/`email` entries. In `PatientPanelAccordion.tsx`'s communication-vs-journey grouping (`COMM_TYPES`), `confirmation_sent` groups under communication (it's a message send, like `sms_sent`/`email_sent`); `patient_confirmed`/`cancellation_requested` group under journey (they're patient actions/milestones, like consent events). No changes needed to the data-fetching path itself — once the backend emits these as `Activity` rows under the appointment's resolved `ContactIdentity`'s `ActivityLog`, the existing `PatientPanelSheet` → `PatientPanelAccordion` → `ActivityFeed` chain picks them up automatically (same activity-log session the patient's other events already appear in).

## Out of scope for this phase

- Cohort/bucket targeting (Phase 4) — unaffected by this phase.
- Follow-up task creation (Phase 5) — the "Needs follow-up" card state is a passive visual signal only in this phase; FR21-25's actual task-creation automation is separate, later work.
- The confirmation-sequence editor/admin UI (creating/editing `ConfirmationSequence` rows, FR1/FR2's remaining CRUD surface, FR19/FR20's manual controls) — not part of this phase's scope, which is read-only status surfacing.
- Historical backfill of activity-log entries for appointments that already have confirmation data from Phases 1-2's testing.

## Testing

- Backend: `compute_confirmation_display_status` unit tests for the new "Needs follow-up" branch (enrollment completed with appointment still awaiting; enrollment stopped with appointment still awaiting; enrollment still active → stays "Awaiting confirmation", not "Needs follow-up"; no enrollment at all → "Awaiting confirmation" as before).
- Backend: serializer test confirming `confirmation_status`/`confirmation_enrollment` fields appear correctly for a few representative appointments (confirmed, awaiting with active enrollment, awaiting with completed enrollment).
- Backend: activity-log integration test — sending a confirmation SMS creates a `SMSMessage` with `contact` set and a corresponding `Activity` row; confirming an appointment via `AppointmentConfirmViewSet.create` creates an `Activity` row via `ActivityLogHelper.log`.
- Frontend: `AppointmentCard` renders the correct badge per `confirmation_status` value, including the priority-order interaction with `no_show_risk` (confirmed status suppresses risk display even when risk is High).
- Frontend: `FilterDialog`'s confirmation filter correctly narrows the appointment list.
- Frontend: `SummaryCards`' new confirmed-coverage tile computes the right X/Y count and its click handler correctly toggles the non-confirmed filter.
- Manual verification: confirm an appointment via the public confirm page, check it appears in the patient's activity feed in the panel.

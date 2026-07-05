# Daylist Confirmations — Phase 2: Appointment-Anchored Sequence Engine

## Context

Phase 1 (`docs/superpowers/specs/2026-07-02-daylist-confirmations-phase1-data-model-design.md`, implemented) added a confirmation audit trail to `DentallyAppointment` (`patient_confirmed_at`, `confirmation_source`, `confirmation_channel`, `cancellation_requested*`) plus a derived display-status function `compute_confirmation_display_status`. This phase builds the actual outreach engine: automatically enrolling eligible future appointments into a configurable email/SMS confirmation sequence, sending steps on schedule, and stopping when the appointment becomes ineligible.

A gap analysis (`PRD/Daylist/summary2.md`) found the closest existing precedent, `RecallSequence`/`RecallSequenceEnrollment` (`dentallyIntegration/models.py:3236-3399`, engine in `recall_automation.py`), is **not directly reusable** for two structural reasons:
1. Recall's due-date math counts *forward* from enrollment date (`enrolled_at + offset_days`); confirmations need to count *backward* from the appointment's start time (`appointment_start - offset_days`).
2. Recall's enrollment is **manual and per-patient** (staff selects patients from a list, POSTs to an `/enroll/` action — see `recall_sequence_views.py:37-89`); this PRD's workflow diagram ("Pathway finds eligible future appointments from Dentally sync → Pathway sends SMS/email steps") requires **automatic, per-appointment** enrollment, closer in spirit to `DayListAutomation`'s scheduled scan-and-target pattern (`dentallyIntegration/daylist_automation.py`).

This phase is a new, bespoke module (`confirmation_automation.py`) and two new models — not a fork of `RecallSequence` — reusing only what genuinely transfers: the step JSON shape (channel/template/offset/send-time), the template-resolution pattern, and the send helpers (`send_sms`/`send_email` from `daylist_automation.py`, reused as-is).

## Scope boundary

Out of scope for this phase (later phases in the build order from `PRD/Daylist/summary2.md`):
- Cohort/bucket targeting (risk tier, booking age, duration, balance, clinician, site) — Phase 4. This phase enrolls **all** future appointments within a configured day-window that have valid contact info and aren't already confirmed/cancelled/cancellation-requested — no risk-based filtering yet.
- Daylist UI surfacing of sequence/enrollment state — Phase 3 (frontend), plus the sequence editor UI itself.
- Non-confirmation follow-up tasks (FR21-25) — Phase 5.

## Models

### `ConfirmationSequence`

```python
class ConfirmationSequence(models.Model):
    STATUS_CHOICES = [
        ("draft", "Draft"),
        ("active", "Active"),
        ("paused", "Paused"),
        ("archived", "Archived"),
    ]
    practice = models.ForeignKey(
        "UserAuthentication.Practice",
        on_delete=models.CASCADE,
        related_name="confirmation_sequences",
    )
    name = models.CharField(max_length=120)
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default="draft")
    days_ahead = models.PositiveSmallIntegerField(
        default=7,
        help_text="Only appointments within this many days count as eligible for enrollment.",
    )
    send_time = models.TimeField(default="09:00:00")
    steps = models.JSONField(default=list, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "confirmation_sequence"
        verbose_name = "Confirmation Sequence"
        verbose_name_plural = "Confirmation Sequences"

    def __str__(self):
        return f"Confirmation sequence '{self.name}' (practice {self.practice_id})"
```

`status` uses the 4-value draft/active/paused/archived enum (FR1) rather than Recall's single `is_active` boolean — `draft` and `archived` sequences are never scanned for enrollment; `paused` sequences keep existing enrollments but don't enroll new appointments or advance existing ones (a paused sequence's enrollments simply stop advancing until resumed, they aren't force-stopped).

`steps` shape: `[{"channel": "email"|"sms", "template_id": <int>, "offset_days": <int>, "send_time"?: "HH:MM"}, ...]` — `offset_days` means **days before the appointment**, the opposite direction from Recall's steps. Default v1 sequence (FR2) is expressed as two steps: `{"channel": "email", "offset_days": 7}` then `{"channel": "sms", "offset_days": 3}` — practice-editable, not hardcoded.

### `ConfirmationSequenceEnrollment`

```python
class ConfirmationSequenceEnrollment(models.Model):
    STATUS_CHOICES = [
        ("active", "Active"),
        ("completed", "Completed"),
        ("stopped", "Stopped"),
    ]
    practice = models.ForeignKey(
        "UserAuthentication.Practice",
        on_delete=models.CASCADE,
        related_name="confirmation_sequence_enrollments",
    )
    sequence = models.ForeignKey(
        ConfirmationSequence, on_delete=models.CASCADE, related_name="enrollments"
    )
    appointment = models.ForeignKey(
        DentallyAppointment, on_delete=models.CASCADE, related_name="confirmation_enrollments"
    )
    status = models.CharField(max_length=12, choices=STATUS_CHOICES, default="active")
    current_step = models.IntegerField(default=0)
    last_sent_step = models.IntegerField(null=True, blank=True)
    enrolled_at = models.DateTimeField()
    enrolled_appointment_start = models.DateTimeField(
        help_text="Snapshot of appointment.start_time at enrollment time, used to "
        "detect a reschedule (FR17) — if the appointment's current start_time "
        "differs from this, remaining steps' due dates must be recomputed.",
    )
    next_due_at = models.DateTimeField()
    stopped_reason = models.CharField(max_length=60, blank=True, default="")
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "confirmation_sequence_enrollment"
        verbose_name = "Confirmation Sequence Enrollment"
        verbose_name_plural = "Confirmation Sequence Enrollments"
        constraints = [
            models.UniqueConstraint(
                fields=["appointment"],
                condition=models.Q(status="active"),
                name="one_active_confirmation_enrollment_per_appointment",
            )
        ]
        indexes = [
            models.Index(fields=["practice", "status", "next_due_at"]),
        ]
```

`last_sent_step` is distinct from `current_step` (FR7: "last sent step" vs "next planned step") — `current_step` always points at the step that will fire next; `last_sent_step` is `None` until the first send, then tracks the most recently fired index.

The partial unique constraint (`status="active"`, one row per `appointment`) is what enforces the PRD's "prevent duplicate confirmation enrollments for the same appointment" rule (Edge Cases §5) at the DB level, so it holds even under concurrent scan runs — not just an application-level check that a race could slip past.

## Engine (`dentallyIntegration/confirmation_automation.py`)

New bespoke module, matching the `recall_automation.py`/`daylist_automation.py` one-file-per-feature convention.

### Backward due-date math

```python
def step_send_time(step, sequence):
    """Effective send time for one step: the step's own 'send_time' ("HH:MM")
    if set, otherwise the sequence-level default. Identical logic to Recall's
    step_send_time (recall_automation.py:31-42), copied rather than imported
    since it takes a ConfirmationSequence, not a RecallSequence."""


def compute_step_due_at(practice, appointment_start, offset_days, send_time):
    """The UTC instant a step is due: `offset_days` before appointment_start's
    calendar date (in the practice's timezone), at `send_time`."""
```

`compute_step_due_at` is the opposite of Recall's same-named function (`recall_automation.py:45-60`), which adds `offset_days` to `enrolled_at`. This one subtracts `offset_days` from `appointment_start`'s local date.

### Eligibility check

```python
def is_appointment_eligible_for_confirmation(appointment):
    """False if the appointment is Cancelled, Confirmed, or Cancellation
    requested (per compute_confirmation_display_status), or has no valid
    contact. True otherwise (including plain 'Awaiting confirmation')."""
```

Reuses `confirmation_status.compute_confirmation_display_status`/`has_valid_contact` from Phase 1 rather than re-deriving eligibility logic — this is the single source of truth for "is this appointment still a candidate."

### Periodic scan: `process_confirmation_enrollments(now=None)`

Two passes, mirroring the shape of `daylist_automation.run_daylist_automation` (targeting) combined with `recall_automation.process_recall_enrollments` (advancing):

**Pass 1 — enroll new appointments.** For each `ConfirmationSequence` with `status="active"`: query `DentallyAppointment`s for that practice with `start_time` between now and `now + days_ahead` days, excluding any with an existing active enrollment (any sequence) and any ineligible per `is_appointment_eligible_for_confirmation`. For each match:
- Compute every step's due date via `compute_step_due_at(practice, appointment.start_time, step["offset_days"], send_time)`.
- **FR3a late-booking catch-up**: set `current_step` to the index of the first step whose due date is still `>= now`. If ALL steps' due dates have already passed (e.g. booked same-day, both the 7-day and 3-day steps are already in the past), **do not create an enrollment at all** — matches PRD Edge Cases §5 ("if booked 2 days before, skip both default scheduled steps unless the sequence has an explicit immediate catch-up step" — Phase 2 doesn't add an immediate-catch-up step type, so "skip entirely" is the correct default behavior).
- Create the `ConfirmationSequenceEnrollment` with `enrolled_at=now`, `enrolled_appointment_start=appointment.start_time`, `next_due_at` = the chosen first step's due date.

**Pass 2 — advance due enrollments.** For each `active` enrollment whose `sequence.status == "active"` and `next_due_at <= now`:
- **Reschedule check (FR17)** — if `appointment.start_time != enrollment.enrolled_appointment_start`, recompute `next_due_at` (and all later steps, implicitly, since each is computed fresh from `appointment.start_time` when its turn comes) from the new start time, update `enrolled_appointment_start` to the new value, and re-evaluate `is_appointment_eligible_for_confirmation` before proceeding (a reschedule might have also changed cancellation state).
- **Eligibility re-check (FR16)** — if `is_appointment_eligible_for_confirmation` is now False, set `status="stopped"` with a `stopped_reason` matching the specific cause (`"confirmed"`, `"cancelled"`, `"cancellation_requested"`, `"no_valid_contact"`) and stop — no message sent this cycle.
- Otherwise, send `steps[current_step]` via the existing `send_sms`/`send_email` helpers (imported from `daylist_automation.py`, not reimplemented) with content resolved via a new `_resolve_confirmation_template(practice, channel, template_id)` scoped to `template_type="appointment_confirmation"` (mirrors `daylist_automation._resolve_daylist_template`'s `template_type` scoping, applied to the `appointment_confirmation` type added in Phase 1). Placeholders: `patient_name`, `appointment_date`, `appointment_time`, `practitioner_name`, `practice_name`, `confirmation_link`, `practice_phone` (FR14's exact list) — `confirmation_link` generated by calling the same token-generation logic `AppointmentConfirmLinkViewSet.retrieve` uses, internally (not over HTTP), passing `channel=` the step's channel so Phase 1's `confirmation_channel` stamping stays accurate.
- On success: `last_sent_step = current_step`, advance `current_step`; if that was the last step, `status="completed"`; otherwise compute the next step's `next_due_at`.
- On failure (send error, e.g. no phone/email — shouldn't happen given the eligibility check, but defensive): log and leave the enrollment `active` at the same step to retry next cycle, matching `daylist_automation`'s "never crash the batch" convention.

### Scheduling

A Celery Beat task calling `process_confirmation_enrollments()` on a short interval (e.g. every 15 minutes, matching `DayListAutomation`'s existing cadence pattern) — added to `TreatmentPath/settings.py`'s beat schedule alongside the existing `dentallyIntegration.tasks.process_recall_sequences` entry.

## Testing

- Model: the partial unique constraint actually blocks a second active enrollment for the same appointment (integrity error), but allows a new active enrollment once the first is `stopped`/`completed`.
- `compute_step_due_at`: backward math produces the correct UTC instant given a practice timezone with a non-zero offset (mirror the shape of `RecallStepDueTimeTests.test_day_one_uses_send_time` in `tests.py:1163-1173` — a `Europe/London` practice in BST, asserting the resulting UTC hour accounts for the +1 offset). Note there is no "day 0 fires immediately" special case here the way Recall's `compute_step_due_at` has (`tests.py:1152-1161`) — every confirmation step has a concrete `offset_days` counted backward from a known appointment time, there's no "day 0 = enrollment instant" concept to replicate.
- `is_appointment_eligible_for_confirmation`: one test per Phase 1 display-status branch that should exclude (Confirmed, Cancelled, Cancellation requested, No valid contact), one that should include (Awaiting confirmation).
- Catch-up index: appointment booked well outside the window before any step is due (starts at step 0); booked between the 7-day and 3-day steps (starts at step 1); booked same-day/after all steps' due dates (no enrollment created at all).
- Pass 2 stop reasons: one test per eligibility-loss cause (confirmed, cancelled, cancellation-requested, contact removed) confirming `stopped_reason` matches and no message is sent that cycle.
- Reschedule: enrollment's `next_due_at` recomputes correctly when `appointment.start_time` changes, and `enrolled_appointment_start` updates to match.
- End-to-end: enroll → step 0 sent → step 1 due later → step 1 sent → `completed`. Separately: enroll → appointment cancelled before step 0 fires → `stopped` with reason `"cancelled"`, zero messages sent.

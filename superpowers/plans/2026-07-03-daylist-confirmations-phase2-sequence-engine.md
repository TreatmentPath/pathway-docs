# Daylist Confirmations Phase 2 — Sequence Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **Do NOT run `git add`/`git commit`/`git push` at any step — the user handles all VCS operations themselves. Leave all changes uncommitted for review.**

**Goal:** Build the automatic, appointment-anchored confirmation-sequence engine: enroll eligible future appointments, send scheduled email/SMS steps counting backward from the appointment date, and stop enrollments when the appointment becomes ineligible.

**Architecture:** Two new models (`ConfirmationSequence`, `ConfirmationSequenceEnrollment`) and a new bespoke module `confirmation_automation.py` (matching the existing `recall_automation.py`/`daylist_automation.py` one-file-per-feature convention) — not a fork of `RecallSequence`, since the scheduling basis (backward from appointment date, not forward from enrollment) and enrollment key (per-appointment, not per-patient) are structurally different. Reuses Phase 1's `compute_confirmation_display_status`/`has_valid_contact` for eligibility and its `appointment_confirmation` template type for content; writes new (not reused) `send_sms`/`send_email` helpers so audit-trail rows are correctly attributed to `source_type="confirmation_automation"` rather than borrowing `daylist_automation`'s hardcoded `source_type="daylist_automation"`.

**Tech Stack:** Django (TreatmentPathBackend), Celery Beat

**Spec:** `docs/superpowers/specs/2026-07-03-daylist-confirmations-phase2-sequence-engine-design.md`

**Deviation from the spec, found while grounding this plan in the actual codebase:** the spec says sending "reuses `daylist_automation.py`'s `send_sms`/`send_email` helpers... as-is." On inspection, both functions hardcode `source_type="daylist_automation"` in the `SMSMessage`/`EmailMessages` row they create (`daylist_automation.py:154-164`, `196-204`) — there's no parameter to override it. Reusing them literally would mislabel every confirmation-sequence message as a Day List Automation send in the audit trail/reporting. Task 4 below writes new, analogous functions in `confirmation_automation.py` instead (same Twilio/EmailServiceClient mechanism, same "never crash the batch" error handling, `source_type="confirmation_automation"`).

---

### Task 1: Models + migration

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/models.py` (add two new classes, after `RecallSequenceEnrollment` which ends around line 3416, before `ReconciliationStatus`)
- Create: `TreatmentPath/dentallyIntegration/migrations/0147_confirmationsequence_confirmationsequenceenrollment.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class ConfirmationSequenceModelTests(TestCase):
    """ConfirmationSequence: status enum, steps JSON, sane defaults."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Confirmation Sequence Dental")

    def test_defaults_and_round_trip(self):
        sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="Default confirmation sequence",
        )
        self.assertEqual(sequence.status, "draft")
        self.assertEqual(sequence.days_ahead, 7)
        self.assertEqual(sequence.steps, [])

        sequence.status = "active"
        sequence.steps = [
            {"channel": "email", "template_id": 1, "offset_days": 7},
            {"channel": "sms", "template_id": 2, "offset_days": 3},
        ]
        sequence.save()
        sequence.refresh_from_db()
        self.assertEqual(sequence.status, "active")
        self.assertEqual(len(sequence.steps), 2)


class ConfirmationSequenceEnrollmentModelTests(TestCase):
    """ConfirmationSequenceEnrollment: fields, and the partial unique
    constraint that enforces one active enrollment per appointment."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Confirmation Enrollment Dental")
        self.sequence = ConfirmationSequence.objects.create(
            practice=self.practice, name="Test sequence", status="active",
        )
        self.appointment = DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=9300,
            dentally_patient_id=501,
            patient_name="Enrollment Test Patient",
            patient_phone="+447700900000",
            state="pending",
            duration=30,
            start_time=timezone.make_aware(
                datetime(2026, 7, 10, 10, 0, 0), timezone.get_current_timezone(),
            ),
        )

    def test_defaults_and_round_trip(self):
        now = timezone.now()
        enrollment = ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=self.appointment,
            enrolled_at=now,
            enrolled_appointment_start=self.appointment.start_time,
            next_due_at=now,
        )
        self.assertEqual(enrollment.status, "active")
        self.assertEqual(enrollment.current_step, 0)
        self.assertIsNone(enrollment.last_sent_step)
        self.assertEqual(enrollment.stopped_reason, "")

    def test_only_one_active_enrollment_per_appointment(self):
        now = timezone.now()
        ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=self.appointment,
            enrolled_at=now,
            enrolled_appointment_start=self.appointment.start_time,
            next_due_at=now,
        )
        with self.assertRaises(IntegrityError):
            with transaction.atomic():
                ConfirmationSequenceEnrollment.objects.create(
                    practice=self.practice,
                    sequence=self.sequence,
                    appointment=self.appointment,
                    enrolled_at=now,
                    enrolled_appointment_start=self.appointment.start_time,
                    next_due_at=now,
                )

    def test_second_active_enrollment_allowed_after_first_stops(self):
        now = timezone.now()
        first = ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=self.appointment,
            enrolled_at=now,
            enrolled_appointment_start=self.appointment.start_time,
            next_due_at=now,
        )
        first.status = "stopped"
        first.stopped_reason = "cancelled"
        first.save()

        second = ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=self.appointment,
            enrolled_at=now,
            enrolled_appointment_start=self.appointment.start_time,
            next_due_at=now,
        )
        self.assertEqual(second.status, "active")
```

Add `ConfirmationSequence`, `ConfirmationSequenceEnrollment` to the existing `from .models import (...)` block at the top of `tests.py` (alphabetical order), and confirm `IntegrityError`/`transaction` are already imported at the top of the file (they are, per the existing `from django.db import IntegrityError, connection, transaction` line — no change needed there).

- [ ] **Step 2: Run tests to verify they fail**

From `TreatmentPath/`, in ONE chained command (each terminal call is a fresh shell — chaining `source .../activate &&` is required every time or you'll hit a misleading `ModuleNotFoundError: No module named 'django_migration_linter'` from falling back to the wrong Python environment):

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && python manage.py test dentallyIntegration.tests.ConfirmationSequenceModelTests dentallyIntegration.tests.ConfirmationSequenceEnrollmentModelTests --keepdb -v 2
```

Expected: FAIL — `ImportError` (the models don't exist yet).

**IMPORTANT: always use `--keepdb`, never `--noinput`** — a fresh test-DB rebuild is broken in this repo; `--noinput` destroys the persistent test DB.

- [ ] **Step 3: Add the two models**

In `dentallyIntegration/models.py`, insert right after `RecallSequenceEnrollment` ends (around line 3416) and before `class ReconciliationStatus`:

```python
class ConfirmationSequence(models.Model):
    """A practice-configured appointment-confirmation sequence (Daylist
    Confirmations Phase 2). Unlike RecallSequence, steps count backward from
    the APPOINTMENT date, not forward from enrollment — see
    confirmation_automation.compute_step_due_at.

    steps = [
      {"channel": "sms"|"email", "template_id": <int>, "offset_days": <int>,
       "send_time"?: "HH:MM"},
      ...
    ]
    offset_days means days BEFORE the appointment.
    """

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


class ConfirmationSequenceEnrollment(models.Model):
    """One appointment enrolled in a ConfirmationSequence — tracks progress.

    Keyed on `appointment`, not patient (unlike RecallSequenceEnrollment) —
    confirmations are inherently per-appointment. The partial unique
    constraint below enforces "one active enrollment per appointment" at the
    DB level, per the PRD's duplicate-enrollment-prevention rule.
    """

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
        DentallyAppointment,
        on_delete=models.CASCADE,
        related_name="confirmation_enrollments",
    )
    status = models.CharField(max_length=12, choices=STATUS_CHOICES, default="active")
    current_step = models.IntegerField(default=0)
    last_sent_step = models.IntegerField(
        null=True,
        blank=True,
        help_text="Most recently fired step index, distinct from current_step "
        "(the NEXT step due). None until the first send.",
    )
    enrolled_at = models.DateTimeField()
    enrolled_appointment_start = models.DateTimeField(
        help_text="Snapshot of appointment.start_time at enrollment time, used "
        "to detect a reschedule — if the appointment's current start_time "
        "differs from this, remaining steps' due dates are recomputed.",
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

    def __str__(self):
        return (
            f"Confirmation enrollment for appointment {self.appointment_id} "
            f"(sequence {self.sequence_id}, {self.status})"
        )
```

- [ ] **Step 4: Generate and inspect the migration**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py makemigrations dentallyIntegration
```

Expected: creates `dentallyIntegration/migrations/0147_confirmationsequence_confirmationsequenceenrollment.py` (or a similar auto-generated name — fine to leave as-is) depending on `0146_dentallyappointment_confirmation_fields`, containing two `CreateModel` operations plus the `AddConstraint`/`AddIndex` for the enrollment model. If `makemigrations` sweeps in any unrelated pending change (e.g. the known `OpportunityTreatmentMap` index-rename drift), remove that operation from the generated file by hand so only these two new models' operations remain.

- [ ] **Step 5: Apply the migration**

```bash
python manage.py migrate dentallyIntegration
```

Expected: `Applying dentallyIntegration.0147_confirmationsequence_confirmationsequenceenrollment... OK`

- [ ] **Step 6: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationSequenceModelTests dentallyIntegration.tests.ConfirmationSequenceEnrollmentModelTests --keepdb -v 2
```

Expected: PASS (4 tests).

**Do NOT commit.**

---

### Task 2: Backward due-date math

**Files:**
- Create: `TreatmentPath/dentallyIntegration/confirmation_automation.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class ConfirmationStepDueTimeTests(TestCase):
    """compute_step_due_at: `offset_days` before the appointment's calendar
    date (in the practice timezone), at send_time — the opposite direction
    from RecallSequence's own compute_step_due_at (which counts forward from
    enrollment)."""

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Confirmation Due Time Dental", timezone="Europe/London"
        )

    def test_offset_days_before_appointment(self):
        from datetime import time as dtime
        from datetime import timezone as dt_tz

        from .confirmation_automation import compute_step_due_at

        # Appointment at 2026-06-25 10:00 UTC (June = BST, UTC+1 in London).
        appointment_start = datetime(2026, 6, 25, 10, 0, 0, tzinfo=dt_tz.utc)
        due = compute_step_due_at(self.practice, appointment_start, 7, dtime(9, 0))
        # 7 days before 2026-06-25 (local date) = 2026-06-18, 09:00 London (BST) = 08:00 UTC.
        self.assertEqual(due.date().isoformat(), "2026-06-18")
        self.assertEqual((due.hour, due.minute), (8, 0))

    def test_zero_offset_is_appointment_day(self):
        from datetime import time as dtime
        from datetime import timezone as dt_tz

        from .confirmation_automation import compute_step_due_at

        appointment_start = datetime(2026, 6, 25, 10, 0, 0, tzinfo=dt_tz.utc)
        due = compute_step_due_at(self.practice, appointment_start, 0, dtime(9, 0))
        self.assertEqual(due.date().isoformat(), "2026-06-25")
        self.assertEqual((due.hour, due.minute), (8, 0))


class ConfirmationStepSendTimeTests(TestCase):
    """step_send_time: per-step 'send_time' override, else the sequence default."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Send Time Override Dental")
        self.sequence = ConfirmationSequence.objects.create(
            practice=self.practice, name="Send time test", send_time="09:00:00",
        )

    def test_uses_sequence_default_when_step_has_no_override(self):
        from .confirmation_automation import step_send_time

        result = step_send_time({"channel": "email", "offset_days": 7}, self.sequence)
        self.assertEqual((result.hour, result.minute), (9, 0))

    def test_uses_step_override_when_present(self):
        from datetime import time as dtime

        from .confirmation_automation import step_send_time

        result = step_send_time(
            {"channel": "sms", "offset_days": 3, "send_time": "14:30"}, self.sequence
        )
        self.assertEqual(result, dtime(14, 30))
```

Add `ConfirmationSequence` to the `.models` import block if not already added by Task 1 (it will be).

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && python manage.py test dentallyIntegration.tests.ConfirmationStepDueTimeTests dentallyIntegration.tests.ConfirmationStepSendTimeTests --keepdb -v 2
```

Expected: FAIL — `ModuleNotFoundError: No module named 'dentallyIntegration.confirmation_automation'`.

- [ ] **Step 3: Create the module with these two functions**

Create `dentallyIntegration/confirmation_automation.py`:

```python
"""Daylist Confirmations — Phase 2 sequence engine (bespoke, own model, own
engine, NOT the generic Workflow/canvas engine — same pattern as
recall_automation.py / daylist_automation.py).

Unlike RecallSequence, a ConfirmationSequence's steps count backward from the
APPOINTMENT's start time, not forward from enrollment — see
compute_step_due_at below (the mirror-image of
recall_automation.compute_step_due_at).

See docs/superpowers/specs/2026-07-03-daylist-confirmations-phase2-sequence-engine-design.md.
"""

import logging
from datetime import datetime, timedelta

import pytz


logger = logging.getLogger(__name__)


def step_send_time(step, sequence):
    """Effective send time for one step: the step's own 'send_time' ("HH:MM")
    if set, otherwise the sequence-level default. Mirrors
    recall_automation.step_send_time's logic (copied, not imported — that one
    takes a RecallSequence, not a ConfirmationSequence)."""
    raw = (step or {}).get("send_time")
    if raw:
        try:
            parts = str(raw).split(":")
            from datetime import time as _time

            return _time(int(parts[0]), int(parts[1]))
        except (ValueError, IndexError):
            pass
    return sequence.send_time


def compute_step_due_at(practice, appointment_start, offset_days, send_time):
    """The UTC instant a step is due: `offset_days` before appointment_start's
    calendar date (in the practice's timezone), at `send_time`. The mirror
    image of recall_automation.compute_step_due_at, which ADDS offset_days to
    an enrollment date; this SUBTRACTS offset_days from an appointment date."""
    tz = pytz.timezone(getattr(practice, "timezone", None) or "UTC")
    local_appt_date = appointment_start.astimezone(tz).date()
    due_date = local_appt_date - timedelta(days=offset_days)
    naive_due = datetime.combine(due_date, send_time)
    return tz.localize(naive_due).astimezone(pytz.UTC)
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationStepDueTimeTests dentallyIntegration.tests.ConfirmationStepSendTimeTests --keepdb -v 2
```

Expected: PASS (4 tests).

**Do NOT commit.**

---

### Task 3: Eligibility check

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/confirmation_automation.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class IsAppointmentEligibleForConfirmationTests(TestCase):
    """is_appointment_eligible_for_confirmation: False for Confirmed,
    Cancelled, Cancellation requested, or No valid contact (per Phase 1's
    compute_confirmation_display_status); True for Awaiting confirmation."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Eligibility Test Dental")

    def _appointment(self, **overrides):
        defaults = dict(
            practice=self.practice,
            dentally_id=9400,
            dentally_patient_id=601,
            patient_name="Eligibility Test Patient",
            patient_phone="+447700900000",
            state="pending",
            duration=30,
        )
        defaults.update(overrides)
        return DentallyAppointment.objects.create(**defaults)

    def test_confirmed_is_not_eligible(self):
        from .confirmation_automation import is_appointment_eligible_for_confirmation

        appointment = self._appointment(patient_confirmed=True)
        self.assertFalse(is_appointment_eligible_for_confirmation(appointment))

    def test_cancelled_is_not_eligible(self):
        from .confirmation_automation import is_appointment_eligible_for_confirmation

        appointment = self._appointment(state="cancelled")
        self.assertFalse(is_appointment_eligible_for_confirmation(appointment))

    def test_cancellation_requested_is_not_eligible(self):
        from .confirmation_automation import is_appointment_eligible_for_confirmation

        appointment = self._appointment(cancellation_requested=True)
        self.assertFalse(is_appointment_eligible_for_confirmation(appointment))

    def test_no_valid_contact_is_not_eligible(self):
        from .confirmation_automation import is_appointment_eligible_for_confirmation

        appointment = self._appointment(patient_phone="", patient_email="")
        self.assertFalse(is_appointment_eligible_for_confirmation(appointment))

    def test_awaiting_confirmation_is_eligible(self):
        from .confirmation_automation import is_appointment_eligible_for_confirmation

        appointment = self._appointment()
        self.assertTrue(is_appointment_eligible_for_confirmation(appointment))
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && python manage.py test dentallyIntegration.tests.IsAppointmentEligibleForConfirmationTests --keepdb -v 2
```

Expected: FAIL — `ImportError: cannot import name 'is_appointment_eligible_for_confirmation'`.

- [ ] **Step 3: Add the function**

Append to `dentallyIntegration/confirmation_automation.py` (after `compute_step_due_at`):

```python
def is_appointment_eligible_for_confirmation(appointment):
    """False if the appointment is Cancelled, Confirmed, or Cancellation
    requested (per compute_confirmation_display_status), or has no valid
    contact. True for 'Awaiting confirmation' (the only state that should
    still receive confirmation messages)."""
    from .confirmation_status import compute_confirmation_display_status

    status = compute_confirmation_display_status(appointment)
    return status == "Awaiting confirmation"
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.IsAppointmentEligibleForConfirmationTests --keepdb -v 2
```

Expected: PASS (5 tests).

**Do NOT commit.**

---

### Task 4: Sending primitives (send_sms, send_email, template resolution, confirmation link)

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/confirmation_automation.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class ConfirmationAutomationSendingTests(TestCase):
    """send_sms/send_email log to the audit trail with source_type=
    'confirmation_automation' (NOT 'daylist_automation' — a real bug this
    plan's own spec caught: reusing daylist_automation's send_sms/send_email
    would mislabel confirmation messages). _resolve_confirmation_template
    only resolves active 'appointment_confirmation'-typed templates.
    get_confirmation_link never regenerates an existing token and stamps
    confirmation_channel."""

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Confirmation Sending Dental", twilio_phone_number="+447700900000"
        )
        self.appointment = DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=9500,
            dentally_patient_id=701,
            patient_name="Sending Test Patient",
            patient_phone="+447700900111",
            state="pending",
            duration=30,
        )

    @override_settings(TWILIO_ACCOUNT_SID="ACtest", TWILIO_AUTH_TOKEN="tok")
    def test_send_sms_logs_confirmation_automation_source_type(self):
        from messaging.models import SMSMessage

        from .confirmation_automation import send_sms

        with patch("twilio.rest.Client") as MockTwilio:
            MockTwilio.return_value.messages.create.return_value.sid = "SMfake"
            ok, _ = send_sms(self.practice, "+447700900111", "Please confirm.")

        self.assertTrue(ok)
        msg = SMSMessage.objects.get(practice=self.practice)
        self.assertEqual(msg.source_type, "confirmation_automation")

    def test_resolve_confirmation_template_requires_correct_type(self):
        from messaging.models import SMSMessageTemplate

        from .confirmation_automation import _resolve_confirmation_template

        wrong_type = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="Wrong type",
            template_type="daylist",
            content="Should not resolve",
            is_active=True,
        )
        subject, body = _resolve_confirmation_template(
            self.practice, "sms", wrong_type.id
        )
        self.assertIsNone(body)

        right_type = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="Right type",
            template_type="appointment_confirmation",
            content="Please confirm {{ patient_name }}",
            is_active=True,
        )
        subject, body = _resolve_confirmation_template(
            self.practice, "sms", right_type.id
        )
        self.assertEqual(body, "Please confirm {{ patient_name }}")

    def test_get_confirmation_link_generates_token_once(self):
        from .confirmation_automation import get_confirmation_link

        first_url = get_confirmation_link(self.appointment, "sms")
        self.appointment.refresh_from_db()
        self.assertEqual(self.appointment.confirmation_channel, "sms")
        first_token = self.appointment.confirmation_token

        second_url = get_confirmation_link(self.appointment, "email")
        self.appointment.refresh_from_db()
        self.assertEqual(self.appointment.confirmation_channel, "email")
        self.assertEqual(self.appointment.confirmation_token, first_token)
        self.assertEqual(first_url, second_url)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && python manage.py test dentallyIntegration.tests.ConfirmationAutomationSendingTests --keepdb -v 2
```

Expected: FAIL — `ImportError` for each missing function.

- [ ] **Step 3: Add the sending primitives**

Append to `dentallyIntegration/confirmation_automation.py`:

```python
def get_confirmation_link(appointment, channel):
    """(url) for the appointment's confirmation page. Never regenerates an
    existing token (mirrors AppointmentConfirmLinkViewSet.retrieve's
    contract exactly — a link already sent to a patient must stay valid).
    Always stamps confirmation_channel to the given channel."""
    from django.conf import settings

    from .confirm_utils import generate_short_token

    update_fields = ["confirmation_channel"]
    appointment.confirmation_channel = channel
    if not appointment.confirmation_token:
        appointment.confirmation_token = generate_short_token()
        update_fields.append("confirmation_token")
    appointment.save(update_fields=update_fields)

    confirm_base_url = getattr(settings, "CONFIRM_BASE_URL", "https://dev.confirm.dental")
    return f"{confirm_base_url}/{appointment.confirmation_token}"


def _resolve_confirmation_template(practice, channel, template_id):
    """(subject, body) for an active 'appointment_confirmation'-typed
    template, scoped to practice + is_active=True. Mirrors
    daylist_automation._resolve_daylist_template's template_type scoping
    (added there after a code review caught the same gap this function
    avoids from the start). Returns (None, None) when missing/inactive/
    wrong-type — never raises."""
    from messaging.models import EmailMessageTemplate, SMSMessageTemplate

    if channel == "email":
        t = EmailMessageTemplate.objects.filter(
            id=template_id,
            practice=practice,
            is_active=True,
            template_type="appointment_confirmation",
        ).first()
        if not t:
            return None, None
        return (t.subject or ""), (t.content or "")
    t = SMSMessageTemplate.objects.filter(
        id=template_id,
        practice=practice,
        is_active=True,
        template_type="appointment_confirmation",
    ).first()
    if not t:
        return None, None
    return "", (t.content or "")


def render_message(template_str, appointment, practice, confirmation_link):
    """Renders `template_str` against one appointment + practice + its
    confirmation link. autoescape=False — plain-text SMS/email body, not
    HTML (same rationale as daylist_automation.render_message)."""
    from django.template import Context, Template

    import pytz as _pytz

    tz = _pytz.timezone(getattr(practice, "timezone", None) or "UTC")
    local_start = appointment.start_time.astimezone(tz) if appointment.start_time else None
    ctx = Context(
        {
            "patient_name": appointment.patient_name or "",
            "appointment_date": local_start.strftime("%d %b %Y") if local_start else "",
            "appointment_time": local_start.strftime("%H:%M") if local_start else "",
            "practitioner_name": appointment.practitioner_name or "",
            "practice_name": practice.name if practice else "",
            "confirmation_link": confirmation_link,
            "practice_phone": getattr(practice, "phone_number", "") or "",
        },
        autoescape=False,
    )
    return Template(template_str).render(ctx)


def send_sms(practice, to_phone, body):
    """Sends one SMS via Twilio and logs an SMSMessage row with
    source_type='confirmation_automation' — a NEW function, not a reuse of
    daylist_automation.send_sms, because that one hardcodes
    source_type='daylist_automation' with no override, which would mislabel
    confirmation-sequence sends. Returns (ok: bool, detail: str)."""
    from django.conf import settings

    from messaging.models import SMSMessage

    if not getattr(practice, "twilio_phone_number", None):
        return False, "practice has no twilio_phone_number configured"

    to = to_phone if to_phone.startswith("+") else f"+{to_phone}"
    sid, status_, error = None, "queued", ""
    try:
        from twilio.rest import Client as TwilioClient

        client = TwilioClient(settings.TWILIO_ACCOUNT_SID, settings.TWILIO_AUTH_TOKEN)
        tw = client.messages.create(body=body, from_=practice.twilio_phone_number, to=to)
        sid, status_ = tw.sid, "sent"
    except Exception as e:  # noqa: BLE001 — log + report, never crash the batch
        status_, error = "failed", str(e)[:200]
        logger.warning("[ConfirmationAutomation] SMS send failed to %s: %s", to, e)

    SMSMessage.objects.create(
        practice=practice,
        phone_number=to_phone,
        content=body,
        status=status_,
        twilio_sid=sid,
        error_message=error,
        message_purpose="automation",
        source_type="confirmation_automation",
        direction="outgoing",
    )
    return status_ == "sent", error or "sent"


def send_email(practice, to_email, subject, body):
    """Sends one email via the same EmailServiceClient + sending-domain path
    as daylist_automation.send_email, but a NEW function so the audit-trail
    row is correctly tagged source_type='confirmation_automation'. Returns
    (ok: bool, detail: str)."""
    from dentallyIntegration.recall_automation import _resolve_sending_domain
    from messaging.models import EmailMessages
    from UserAuthentication.alert_email_utils import _mint_practice_auth_token

    from_email, sending_domain_id = _resolve_sending_domain(practice)
    if not from_email:
        return False, "practice has no sending domain configured"

    status_, error = "sent", ""
    try:
        from messaging.email_service import EmailServiceClient

        client = EmailServiceClient(auth_token=_mint_practice_auth_token(practice))
        client.send_email(
            domain_id=sending_domain_id,
            from_email=from_email,
            to=[to_email],
            subject=subject,
            body_html=body,
        )
    except Exception as e:  # noqa: BLE001 — log + report, never crash the batch
        status_, error = "failed", str(e)[:200]
        logger.warning("[ConfirmationAutomation] Email send failed to %s: %s", to_email, e)

    EmailMessages.objects.create(
        practice=practice,
        direction="outgoing",
        message_purpose="automation",
        source_type="confirmation_automation",
        to_patient=to_email,
        subject=subject or "Please confirm your appointment",
        body=body,
    )
    return status_ == "sent", error or "sent"
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationAutomationSendingTests --keepdb -v 2
```

Expected: PASS (3 tests).

**Do NOT commit.**

---

### Task 5: Enrollment scan (Pass 1: enroll, Pass 2: advance) + orchestrator

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/confirmation_automation.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class EnrollEligibleAppointmentsTests(TestCase):
    """enroll_eligible_appointments: enrolls future appointments within
    days_ahead that are eligible and not already actively enrolled (any
    sequence). FR3a: starts current_step at the first step whose due date is
    still in the future; if ALL steps are already past-due, skips enrollment
    entirely rather than creating a dead-on-arrival enrollment."""

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Enroll Scan Dental", timezone="UTC"
        )
        self.sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="7-and-3-day sequence",
            status="active",
            days_ahead=14,
            steps=[
                {"channel": "email", "template_id": 1, "offset_days": 7},
                {"channel": "sms", "template_id": 2, "offset_days": 3},
            ],
        )

    def _appointment(self, start_time, **overrides):
        defaults = dict(
            practice=self.practice,
            dentally_id=9600,
            dentally_patient_id=801,
            patient_name="Enroll Scan Patient",
            patient_phone="+447700900000",
            state="pending",
            duration=30,
            start_time=start_time,
        )
        defaults.update(overrides)
        return DentallyAppointment.objects.create(**defaults)

    def test_enrolls_appointment_well_outside_first_step(self):
        from .confirmation_automation import enroll_eligible_appointments

        now = timezone.now()
        appointment = self._appointment(now + timedelta(days=10))
        enroll_eligible_appointments(self.sequence, now=now)

        enrollment = ConfirmationSequenceEnrollment.objects.get(appointment=appointment)
        self.assertEqual(enrollment.current_step, 0)
        self.assertEqual(enrollment.status, "active")

    def test_late_booking_starts_at_second_step(self):
        from .confirmation_automation import enroll_eligible_appointments

        now = timezone.now()
        # Booked with only 5 days' notice — the 7-day-before step is already past,
        # the 3-day-before step is still ahead.
        appointment = self._appointment(now + timedelta(days=5))
        enroll_eligible_appointments(self.sequence, now=now)

        enrollment = ConfirmationSequenceEnrollment.objects.get(appointment=appointment)
        self.assertEqual(enrollment.current_step, 1)

    def test_same_day_booking_skips_enrollment_entirely(self):
        from .confirmation_automation import enroll_eligible_appointments

        now = timezone.now()
        # Booked same-day — both the 7-day and 3-day steps are already past.
        appointment = self._appointment(now + timedelta(hours=2))
        enroll_eligible_appointments(self.sequence, now=now)

        self.assertFalse(
            ConfirmationSequenceEnrollment.objects.filter(appointment=appointment).exists()
        )

    def test_appointment_outside_days_ahead_window_not_enrolled(self):
        from .confirmation_automation import enroll_eligible_appointments

        now = timezone.now()
        appointment = self._appointment(now + timedelta(days=30))
        enroll_eligible_appointments(self.sequence, now=now)

        self.assertFalse(
            ConfirmationSequenceEnrollment.objects.filter(appointment=appointment).exists()
        )

    def test_already_confirmed_appointment_not_enrolled(self):
        from .confirmation_automation import enroll_eligible_appointments

        now = timezone.now()
        appointment = self._appointment(now + timedelta(days=10), patient_confirmed=True)
        enroll_eligible_appointments(self.sequence, now=now)

        self.assertFalse(
            ConfirmationSequenceEnrollment.objects.filter(appointment=appointment).exists()
        )

    def test_already_actively_enrolled_appointment_skipped(self):
        from .confirmation_automation import enroll_eligible_appointments

        now = timezone.now()
        appointment = self._appointment(now + timedelta(days=10))
        ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=appointment,
            enrolled_at=now,
            enrolled_appointment_start=appointment.start_time,
            next_due_at=now,
        )
        enroll_eligible_appointments(self.sequence, now=now)

        self.assertEqual(
            ConfirmationSequenceEnrollment.objects.filter(appointment=appointment).count(), 1
        )


class AdvanceConfirmationEnrollmentsTests(TestCase):
    """advance_confirmation_enrollments (Pass 2): re-checks eligibility before
    sending, stops on ineligibility with the right reason, sends + advances
    otherwise, completes after the last step, and detects a reschedule."""

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Advance Dental", timezone="UTC", twilio_phone_number="+447700900000"
        )
        self.sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="Two-step sequence",
            status="active",
            steps=[
                {"channel": "sms", "template_id": None, "offset_days": 7},
                {"channel": "sms", "template_id": None, "offset_days": 3},
            ],
        )
        from messaging.models import SMSMessageTemplate

        self.template = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="Confirmation SMS",
            template_type="appointment_confirmation",
            content="Hi {{ patient_name }}, confirm: {{ confirmation_link }}",
            is_active=True,
        )
        self.sequence.steps = [
            {"channel": "sms", "template_id": self.template.id, "offset_days": 7},
            {"channel": "sms", "template_id": self.template.id, "offset_days": 3},
        ]
        self.sequence.save()

    def _enrollment(self, appointment_start, current_step=0, next_due_at=None):
        now = timezone.now()
        appointment = DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=9700,
            dentally_patient_id=901,
            patient_name="Advance Test Patient",
            patient_phone="+447700900222",
            state="pending",
            duration=30,
            start_time=appointment_start,
        )
        enrollment = ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=appointment,
            current_step=current_step,
            enrolled_at=now,
            enrolled_appointment_start=appointment_start,
            next_due_at=next_due_at or (now - timedelta(minutes=5)),
        )
        return enrollment

    @override_settings(TWILIO_ACCOUNT_SID="ACtest", TWILIO_AUTH_TOKEN="tok")
    def test_sends_due_step_and_advances(self):
        from .confirmation_automation import advance_confirmation_enrollments

        enrollment = self._enrollment(timezone.now() + timedelta(days=7))
        with patch("twilio.rest.Client") as MockTwilio:
            MockTwilio.return_value.messages.create.return_value.sid = "SMfake"
            stats = advance_confirmation_enrollments(now=timezone.now())

        enrollment.refresh_from_db()
        self.assertEqual(stats["fired"], 1)
        self.assertEqual(enrollment.last_sent_step, 0)
        self.assertEqual(enrollment.current_step, 1)
        self.assertEqual(enrollment.status, "active")

    @override_settings(TWILIO_ACCOUNT_SID="ACtest", TWILIO_AUTH_TOKEN="tok")
    def test_completes_after_last_step(self):
        from .confirmation_automation import advance_confirmation_enrollments

        enrollment = self._enrollment(timezone.now() + timedelta(days=3), current_step=1)
        with patch("twilio.rest.Client") as MockTwilio:
            MockTwilio.return_value.messages.create.return_value.sid = "SMfake"
            advance_confirmation_enrollments(now=timezone.now())

        enrollment.refresh_from_db()
        self.assertEqual(enrollment.status, "completed")
        self.assertEqual(enrollment.last_sent_step, 1)

    def test_stops_when_appointment_becomes_confirmed(self):
        from .confirmation_automation import advance_confirmation_enrollments

        enrollment = self._enrollment(timezone.now() + timedelta(days=7))
        enrollment.appointment.patient_confirmed = True
        enrollment.appointment.save(update_fields=["patient_confirmed"])

        stats = advance_confirmation_enrollments(now=timezone.now())

        enrollment.refresh_from_db()
        self.assertEqual(enrollment.status, "stopped")
        self.assertEqual(enrollment.stopped_reason, "confirmed")
        self.assertEqual(stats["fired"], 0)

    def test_stops_when_appointment_cancelled(self):
        from .confirmation_automation import advance_confirmation_enrollments

        enrollment = self._enrollment(timezone.now() + timedelta(days=7))
        enrollment.appointment.state = "cancelled"
        enrollment.appointment.save(update_fields=["state"])

        advance_confirmation_enrollments(now=timezone.now())

        enrollment.refresh_from_db()
        self.assertEqual(enrollment.status, "stopped")
        self.assertEqual(enrollment.stopped_reason, "cancelled")

    def test_reschedule_pushed_later_defers_instead_of_firing_early(self):
        """Step 0 (offset_days=7) was due under the OLD start_time. After a
        reschedule pushes the appointment further out, the step's TRUE due
        date (recomputed against the new start_time) is now in the future —
        the enrollment must defer (update next_due_at, send nothing) rather
        than fire against the stale schedule."""
        from .confirmation_automation import advance_confirmation_enrollments, compute_step_due_at

        original_start = timezone.now() + timedelta(days=7)
        enrollment = self._enrollment(original_start)
        new_start = timezone.now() + timedelta(days=10)
        enrollment.appointment.start_time = new_start
        enrollment.appointment.save(update_fields=["start_time"])

        with patch("twilio.rest.Client") as MockTwilio:
            MockTwilio.return_value.messages.create.return_value.sid = "SMfake"
            stats = advance_confirmation_enrollments(now=timezone.now())

        MockTwilio.assert_not_called()
        self.assertEqual(stats["fired"], 0)
        self.assertEqual(stats["skipped"], 1)
        enrollment.refresh_from_db()
        self.assertEqual(enrollment.enrolled_appointment_start, new_start)
        self.assertEqual(enrollment.status, "active")
        self.assertEqual(enrollment.current_step, 0)
        expected_due = compute_step_due_at(
            self.practice, new_start, 7, self.sequence.send_time
        )
        self.assertEqual(enrollment.next_due_at, expected_due)

    def test_reschedule_pulled_earlier_still_fires_if_now_due(self):
        """Step 0 was due under the OLD start_time. After a reschedule pulls
        the appointment CLOSER (not further away), the step's recomputed due
        date is still in the past, so it fires normally this cycle."""
        from .confirmation_automation import advance_confirmation_enrollments

        original_start = timezone.now() + timedelta(days=30)
        enrollment = self._enrollment(original_start, next_due_at=timezone.now() - timedelta(minutes=5))
        new_start = timezone.now() + timedelta(days=6)  # 7-day-before step is now in the past
        enrollment.appointment.start_time = new_start
        enrollment.appointment.save(update_fields=["start_time"])

        with patch("twilio.rest.Client") as MockTwilio:
            MockTwilio.return_value.messages.create.return_value.sid = "SMfake"
            stats = advance_confirmation_enrollments(now=timezone.now())

        self.assertEqual(stats["fired"], 1)
        enrollment.refresh_from_db()
        self.assertEqual(enrollment.enrolled_appointment_start, new_start)
        self.assertEqual(enrollment.current_step, 1)


class ProcessConfirmationEnrollmentsTests(TestCase):
    """process_confirmation_enrollments: the top-level orchestrator running
    both passes, returning a combined stats dict."""

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Orchestrator Dental", timezone="UTC", twilio_phone_number="+447700900000"
        )
        from messaging.models import SMSMessageTemplate

        self.template = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="Confirmation SMS",
            template_type="appointment_confirmation",
            content="Hi {{ patient_name }}, confirm: {{ confirmation_link }}",
            is_active=True,
        )
        self.sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="Orchestrator sequence",
            status="active",
            days_ahead=14,
            steps=[{"channel": "sms", "template_id": self.template.id, "offset_days": 0}],
        )

    def test_returns_combined_stats(self):
        from .confirmation_automation import process_confirmation_enrollments

        DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=9800,
            dentally_patient_id=1001,
            patient_name="Orchestrator Patient",
            patient_phone="+447700900333",
            state="pending",
            duration=30,
            start_time=timezone.now() + timedelta(days=5),
        )
        stats = process_confirmation_enrollments()
        self.assertIn("enrolled", stats)
        self.assertIn("due", stats)
        self.assertIn("fired", stats)
```

Add `ConfirmationSequence`, `ConfirmationSequenceEnrollment` to the top-of-file `.models` import block (should already be there from Task 1).

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && python manage.py test dentallyIntegration.tests.EnrollEligibleAppointmentsTests dentallyIntegration.tests.AdvanceConfirmationEnrollmentsTests dentallyIntegration.tests.ProcessConfirmationEnrollmentsTests --keepdb -v 2
```

Expected: FAIL — `ImportError` for each missing function.

- [ ] **Step 3: Add the scan functions**

Append to `dentallyIntegration/confirmation_automation.py`:

```python
def enroll_eligible_appointments(sequence, now=None):
    """Pass 1: enroll every eligible future appointment (within
    sequence.days_ahead, valid contact, not already confirmed/cancelled/
    cancellation-requested, no existing active enrollment for ANY sequence)
    that doesn't have one yet. FR3a: starts current_step at the first step
    whose due date is still >= now; if every step's due date has already
    passed, does not enroll at all (nothing left to catch up to)."""
    from django.utils import timezone as dj_timezone

    from .models import DentallyAppointment, ConfirmationSequenceEnrollment

    now = now or dj_timezone.now()
    practice = sequence.practice
    window_end = now + timedelta(days=sequence.days_ahead)
    steps = sequence.steps or []
    if not steps:
        return 0

    already_enrolled_ids = set(
        ConfirmationSequenceEnrollment.objects.filter(
            practice=practice, status="active"
        ).values_list("appointment_id", flat=True)
    )

    candidates = DentallyAppointment.objects.filter(
        practice=practice,
        start_time__gte=now,
        start_time__lte=window_end,
    ).exclude(id__in=already_enrolled_ids)

    enrolled_count = 0
    for appointment in candidates:
        if not is_appointment_eligible_for_confirmation(appointment):
            continue

        due_dates = [
            compute_step_due_at(
                practice, appointment.start_time, int(step.get("offset_days", 0)),
                step_send_time(step, sequence),
            )
            for step in steps
        ]
        start_index = next(
            (i for i, due in enumerate(due_dates) if due >= now), None
        )
        if start_index is None:
            continue  # every step already past-due — nothing to catch up to

        ConfirmationSequenceEnrollment.objects.create(
            practice=practice,
            sequence=sequence,
            appointment=appointment,
            current_step=start_index,
            enrolled_at=now,
            enrolled_appointment_start=appointment.start_time,
            next_due_at=due_dates[start_index],
        )
        enrolled_count += 1

    return enrolled_count


# Maps compute_confirmation_display_status's non-"Awaiting confirmation"
# values to a stored stopped_reason — single source of truth for WHY an
# enrollment stopped, kept in lockstep with confirmation_status.py's own
# priority order rather than re-deriving it from the raw model fields here.
_STOP_REASON_BY_DISPLAY_STATUS = {
    "Cancelled": "cancelled",
    "Cancellation requested": "cancellation_requested",
    "Confirmed": "confirmed",
    "No valid contact": "no_valid_contact",
}


def advance_confirmation_enrollments(now=None):
    """Pass 2: advance every due active enrollment (whose sequence is still
    'active' — a 'paused' sequence's enrollments simply don't advance until
    resumed). Re-checks eligibility and reschedule status before sending.
    Returns a stats dict."""
    from django.utils import timezone as dj_timezone

    from .confirmation_status import compute_confirmation_display_status
    from .models import ConfirmationSequenceEnrollment

    now = now or dj_timezone.now()
    stats = {"due": 0, "fired": 0, "completed": 0, "stopped": 0, "skipped": 0, "failed": 0}

    due = list(
        ConfirmationSequenceEnrollment.objects.filter(
            status="active", next_due_at__lte=now, sequence__status="active"
        ).select_related("sequence", "practice", "appointment")
    )
    stats["due"] = len(due)

    for enrollment in due:
        appointment = enrollment.appointment
        sequence = enrollment.sequence
        steps = sequence.steps or []

        # Reschedule check (FR17): if the appointment's start_time has moved
        # since enrollment/last check, recompute the CURRENT step's due date
        # against the new start_time. If that pushes the step's true due
        # date into the future, defer (update next_due_at, send nothing this
        # cycle) rather than firing early against a stale schedule.
        if (
            appointment.start_time != enrollment.enrolled_appointment_start
            and enrollment.current_step < len(steps)
        ):
            enrollment.enrolled_appointment_start = appointment.start_time
            recomputed_due = compute_step_due_at(
                sequence.practice, appointment.start_time,
                int(steps[enrollment.current_step].get("offset_days", 0)),
                step_send_time(steps[enrollment.current_step], sequence),
            )
            if recomputed_due > now:
                enrollment.next_due_at = recomputed_due
                enrollment.save(update_fields=[
                    "next_due_at", "enrolled_appointment_start", "updated_at",
                ])
                stats["skipped"] += 1
                continue

        display_status = compute_confirmation_display_status(appointment)
        if display_status != "Awaiting confirmation":
            enrollment.status = "stopped"
            enrollment.stopped_reason = _STOP_REASON_BY_DISPLAY_STATUS[display_status]
            enrollment.save(update_fields=[
                "status", "stopped_reason", "enrolled_appointment_start", "updated_at",
            ])
            stats["stopped"] += 1
            continue

        if enrollment.current_step >= len(steps):
            enrollment.status = "completed"
            enrollment.save(update_fields=["status", "enrolled_appointment_start", "updated_at"])
            stats["completed"] += 1
            continue

        step = steps[enrollment.current_step]
        channel = step.get("channel")
        subject, body = _resolve_confirmation_template(practice=sequence.practice, channel=channel, template_id=step.get("template_id"))
        if body is None:
            stats["failed"] += 1
            enrollment.save(update_fields=["enrolled_appointment_start", "updated_at"])
            continue

        confirmation_link = get_confirmation_link(appointment, channel)
        rendered_body = render_message(body, appointment, sequence.practice, confirmation_link)

        if channel == "sms":
            if not appointment.patient_phone:
                stats["skipped"] += 1
                enrollment.save(update_fields=["enrolled_appointment_start", "updated_at"])
                continue
            ok, _ = send_sms(sequence.practice, appointment.patient_phone, rendered_body)
        else:
            if not appointment.patient_email:
                stats["skipped"] += 1
                enrollment.save(update_fields=["enrolled_appointment_start", "updated_at"])
                continue
            rendered_subject = render_message(subject, appointment, sequence.practice, confirmation_link) if subject else "Please confirm your appointment"
            ok, _ = send_email(sequence.practice, appointment.patient_email, rendered_subject, rendered_body)

        if ok:
            stats["fired"] += 1
        else:
            stats["failed"] += 1

        enrollment.last_sent_step = enrollment.current_step
        enrollment.current_step += 1
        if enrollment.current_step >= len(steps):
            enrollment.status = "completed"
            stats["completed"] += 1
        else:
            enrollment.next_due_at = compute_step_due_at(
                sequence.practice, appointment.start_time,
                int(steps[enrollment.current_step].get("offset_days", 0)),
                step_send_time(steps[enrollment.current_step], sequence),
            )
        enrollment.save()

    return stats


def process_confirmation_enrollments(now=None):
    """Top-level orchestrator: Pass 1 (enroll new appointments for every
    active sequence) then Pass 2 (advance due enrollments). Thin wrapper —
    the actual logic lives in enroll_eligible_appointments/
    advance_confirmation_enrollments. Scheduled via CELERY_BEAT_SCHEDULE."""
    from django.utils import timezone as dj_timezone

    from .models import ConfirmationSequence

    now = now or dj_timezone.now()
    enrolled = 0
    for sequence in ConfirmationSequence.objects.filter(status="active"):
        enrolled += enroll_eligible_appointments(sequence, now=now)

    stats = advance_confirmation_enrollments(now=now)
    stats["enrolled"] = enrolled
    return stats
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.EnrollEligibleAppointmentsTests dentallyIntegration.tests.AdvanceConfirmationEnrollmentsTests dentallyIntegration.tests.ProcessConfirmationEnrollmentsTests --keepdb -v 2
```

Expected: PASS (13 tests: 6 in `EnrollEligibleAppointmentsTests` + 6 in `AdvanceConfirmationEnrollmentsTests` + 1 in `ProcessConfirmationEnrollmentsTests` — all green, no failures).

- [ ] **Step 5: Run the full confirmation-automation test suite together**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationSequenceModelTests dentallyIntegration.tests.ConfirmationSequenceEnrollmentModelTests dentallyIntegration.tests.ConfirmationStepDueTimeTests dentallyIntegration.tests.ConfirmationStepSendTimeTests dentallyIntegration.tests.IsAppointmentEligibleForConfirmationTests dentallyIntegration.tests.ConfirmationAutomationSendingTests dentallyIntegration.tests.EnrollEligibleAppointmentsTests dentallyIntegration.tests.AdvanceConfirmationEnrollmentsTests dentallyIntegration.tests.ProcessConfirmationEnrollmentsTests --keepdb -v 2
```

Expected: PASS, all tests from Tasks 1-5 green together, no regressions.

**Do NOT commit.**

---

### Task 6: Celery task wrapper + beat schedule

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/tasks.py`
- Modify: `TreatmentPath/TreatmentPath/settings.py`

- [ ] **Step 1: Add the Celery task**

In `dentallyIntegration/tasks.py`, add right after `process_daylist_automations` (which ends around line 1260 — find the end of that function and insert after it):

```python
@shared_task(bind=True, max_retries=3, default_retry_delay=120)
def process_confirmation_sequences(self):
    """Thin Celery wrapper around
    confirmation_automation.process_confirmation_enrollments — enrolls newly
    eligible future appointments and advances due confirmation-sequence
    steps. Scheduled via CELERY_BEAT_SCHEDULE."""
    from dentallyIntegration.confirmation_automation import (
        process_confirmation_enrollments,
    )

    try:
        stats = process_confirmation_enrollments()
        logger.info(
            "process_confirmation_sequences: enrolled=%s due=%s fired=%s "
            "completed=%s stopped=%s skipped=%s failed=%s",
            stats["enrolled"],
            stats["due"],
            stats["fired"],
            stats["completed"],
            stats["stopped"],
            stats["skipped"],
            stats["failed"],
        )
        return stats
    except Exception as exc:  # noqa: BLE001 — retry transient failures
        logger.exception("process_confirmation_sequences task failed")
        raise self.retry(exc=exc)
```

- [ ] **Step 2: Add the beat schedule entry**

In `TreatmentPath/settings.py`, inside `CELERY_BEAT_SCHEDULE = {...}`, add right after the `"process-recall-sequences"` entry (around line 961):

```python
    # Daylist Confirmations: enroll eligible future appointments and fire any
    # due confirmation-sequence steps. Runs every 15 min so each sequence's
    # own send_time (in the practice's timezone) is honored to within ~15
    # minutes — same cadence as process-recall-sequences.
    "process-confirmation-sequences": {
        "task": "dentallyIntegration.tasks.process_confirmation_sequences",
        "schedule": crontab(minute="*/15"),
        "options": {"queue": "default"},
    },
```

- [ ] **Step 3: Verify the task registers and runs manually**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && python manage.py shell -c "
from dentallyIntegration.tasks import process_confirmation_sequences
print(process_confirmation_sequences.name)
"
```

Expected output: `dentallyIntegration.tasks.process_confirmation_sequences` (confirms the task is importable and correctly named — this does NOT execute it via Celery, just confirms the function/decorator wiring is correct).

- [ ] **Step 4: Run the full dentallyIntegration confirmation test suite one more time to confirm nothing broke**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationSequenceModelTests dentallyIntegration.tests.ConfirmationSequenceEnrollmentModelTests dentallyIntegration.tests.ConfirmationStepDueTimeTests dentallyIntegration.tests.ConfirmationStepSendTimeTests dentallyIntegration.tests.IsAppointmentEligibleForConfirmationTests dentallyIntegration.tests.ConfirmationAutomationSendingTests dentallyIntegration.tests.EnrollEligibleAppointmentsTests dentallyIntegration.tests.AdvanceConfirmationEnrollmentsTests dentallyIntegration.tests.ProcessConfirmationEnrollmentsTests --keepdb -v 2
```

Expected: PASS, all green.

**Do NOT commit.**

---

## Summary of spec coverage

- Models (`ConfirmationSequence`, `ConfirmationSequenceEnrollment`, partial unique constraint) → Task 1.
- Backward due-date math + `step_send_time` → Task 2.
- Eligibility check (reusing Phase 1's `compute_confirmation_display_status`) → Task 3.
- Sending primitives, corrected to use a NEW `source_type="confirmation_automation"` (spec deviation caught and fixed during plan-writing) → Task 4.
- Pass 1 enrollment (FR3a catch-up, duplicate-prevention respecting the DB constraint) + Pass 2 advancement (eligibility re-check/stop reasons FR16, reschedule detection FR17) + orchestrator → Task 5.
- Celery Beat scheduling → Task 6.
- Out-of-scope items (cohort targeting, UI, follow-up tasks, sequence editor) — correctly not covered; they belong to Phases 3-5.

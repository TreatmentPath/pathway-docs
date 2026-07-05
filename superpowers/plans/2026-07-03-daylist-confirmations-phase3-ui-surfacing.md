# Daylist Confirmations Phase 3 — UI Surfacing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **Do NOT run `git add`/`git commit`/`git push` at any step — the user handles all VCS operations themselves. Leave all changes uncommitted for review.**

**Goal:** Expose confirmation state (including a new "Needs follow-up" 6th state) to the Daylist frontend — on the appointment card, in the filter, at the practice-scorecard level — and wire confirmation events into the existing patient activity-log/history feed.

**Architecture:** Backend (Tasks 1-4, `TreatmentPathBackend`) extends Phase 1's derivation function with enrollment-aware logic, exposes it plus an enrollment summary via the existing Daylist serializer, and wires confirmation-automation sends + patient confirm/cancel actions into the existing generic activity-log system (reusing the Patient→ContactIdentity resolution the serializer already does, NOT creating new/duplicate contacts). Frontend (Tasks 5-9, `perfect-pixel-playground-project`) consumes these across the card, filter, summary cards, and the existing patient panel's activity feed.

**Tech Stack:** Django (TreatmentPathBackend), React/TypeScript (perfect-pixel-playground-project)

**Spec:** `docs/superpowers/specs/2026-07-03-daylist-confirmations-phase3-ui-surfacing-design.md`

**Deviation from the spec, found while grounding this plan in the actual codebase:** the spec's activity-log section suggested a new `confirmation_sent` `ACTIVITY_TYPE_CHOICES` value for confirmation-automation SMS/email sends. On inspection, `record_events.on_record_created` already auto-derives `activity_type` as `sms_sent`/`email_sent` for any `SMSMessage`/`EmailMessages` entity based on its `direction` field (`TreatmentPath/record_events.py:157-176`) — confirmation-automation's sends already set `direction="outgoing"`, so they'll be correctly labeled with the EXISTING choices with zero new choices needed for that path. Only the confirm/cancellation-request path (which calls the lower-level `ActivityLogHelper.log` directly, bypassing that auto-derivation) needs the two new choices (`patient_confirmed`, `cancellation_requested`) the spec already called for. Task 3 below reflects this — no new choice added there, only in Task 4.

**Second deviation:** the spec's contact-resolution approach (`ContactIdentity.get_or_create_for_contact` from raw phone/email) risked creating a contact identity divergent from the one already associated with the patient via the existing `Patient.meta_data->>'id'` → `Patient.contact` lookup path (`dentallyIntegration/serializers.py:613-660`, `DayListAppointmentSerializer._get_patient`) — this is a known, actively-tracked problem area in this project (contact/patient duplication). Task 3 below instead reuses that exact resolution path.

---

## Backend (`TreatmentPathBackend`)

### Task 1: Extend `compute_confirmation_display_status` with "Needs follow-up"

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/confirmation_status.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py` (after `ComputeConfirmationDisplayStatusTests`, which ends around line 1710):

```python
class ComputeConfirmationDisplayStatusNeedsFollowUpTests(TestCase):
    """compute_confirmation_display_status: the new 'Needs follow-up' state —
    appointment still Awaiting confirmation by its own fields, but its most
    recent confirmation enrollment has already completed or stopped (no more
    automated messages coming)."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Needs Follow Up Dental")
        self.sequence = ConfirmationSequence.objects.create(
            practice=self.practice, name="Test sequence", status="active",
        )

    def _appointment(self, **overrides):
        defaults = dict(
            practice=self.practice,
            dentally_id=9900,
            dentally_patient_id=1101,
            patient_name="Follow Up Test Patient",
            patient_phone="+447700900000",
            state="pending",
            duration=30,
        )
        defaults.update(overrides)
        return DentallyAppointment.objects.create(**defaults)

    def test_completed_enrollment_with_awaiting_appointment_needs_follow_up(self):
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment()
        now = timezone.now()
        ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=appointment,
            status="completed",
            enrolled_at=now,
            enrolled_appointment_start=now,
            next_due_at=now,
        )
        self.assertEqual(
            compute_confirmation_display_status(appointment), "Needs follow-up"
        )

    def test_stopped_enrollment_with_awaiting_appointment_needs_follow_up(self):
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment()
        now = timezone.now()
        ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=appointment,
            status="stopped",
            stopped_reason="no_valid_contact",
            enrolled_at=now,
            enrolled_appointment_start=now,
            next_due_at=now,
        )
        self.assertEqual(
            compute_confirmation_display_status(appointment), "Needs follow-up"
        )

    def test_active_enrollment_stays_awaiting_confirmation(self):
        from .confirmation_automation import compute_step_due_at
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment()
        now = timezone.now()
        ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=appointment,
            status="active",
            enrolled_at=now,
            enrolled_appointment_start=now,
            next_due_at=now,
        )
        self.assertEqual(
            compute_confirmation_display_status(appointment), "Awaiting confirmation"
        )

    def test_no_enrollment_at_all_stays_awaiting_confirmation(self):
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment()
        self.assertEqual(
            compute_confirmation_display_status(appointment), "Awaiting confirmation"
        )

    def test_most_recent_enrollment_wins_when_multiple_exist(self):
        """An appointment can have more than one non-active enrollment over
        time (e.g. re-enrolled after a reschedule). The MOST RECENT one
        determines the display status."""
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment()
        now = timezone.now()
        older = ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=appointment,
            status="stopped",
            stopped_reason="no_valid_contact",
            enrolled_at=now - timedelta(days=2),
            enrolled_appointment_start=now,
            next_due_at=now,
        )
        older.created_at = now - timedelta(days=2)
        older.save(update_fields=["created_at"])

        newer = ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=appointment,
            status="active",
            enrolled_at=now,
            enrolled_appointment_start=now,
            next_due_at=now,
        )
        self.assertEqual(
            compute_confirmation_display_status(appointment), "Awaiting confirmation"
        )
```

Add `ConfirmationSequence`, `ConfirmationSequenceEnrollment` to the `.models` import block at the top of `tests.py` if not already there (they should be, from the Phase 2 plan).

- [ ] **Step 2: Run tests to verify they fail**

From `TreatmentPath/`, in ONE chained command (each terminal call is a fresh shell — chaining `source .../activate &&` is required every time or you'll hit a misleading `ModuleNotFoundError: No module named 'django_migration_linter'` from falling back to the wrong Python environment):

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && python manage.py test dentallyIntegration.tests.ComputeConfirmationDisplayStatusNeedsFollowUpTests --keepdb -v 2
```

Expected: FAIL — `test_completed_enrollment_with_awaiting_appointment_needs_follow_up` and `test_stopped_enrollment_with_awaiting_appointment_needs_follow_up` fail (function currently returns "Awaiting confirmation" for both, not "Needs follow-up"); the other 3 tests pass already since they don't exercise the new branch.

**IMPORTANT: always use `--keepdb`, never `--noinput`** — a fresh test-DB rebuild is broken in this repo; `--noinput` destroys the persistent test DB.

- [ ] **Step 3: Update the function**

Replace `compute_confirmation_display_status` in `dentallyIntegration/confirmation_status.py`:

```python
NOT_ELIGIBLE_STATUSES = {
    "Cancelled",
    "Cancellation requested",
    "Confirmed",
    "No valid contact",
    "Needs follow-up",
}


def compute_confirmation_display_status(appointment):
    """One of: 'Cancelled', 'Cancellation requested', 'Confirmed',
    'No valid contact', 'Needs follow-up', 'Awaiting confirmation'. Checked
    in this priority order so Dentally's own cancellation status always
    wins, per the PRD edge case: "if Dentally says cancelled, Daylist should
    show cancelled even if Pathway previously had confirmed".

    'Needs follow-up' is the appointment's most recent confirmation
    enrollment having already completed or stopped (no more automated
    messages coming) while the appointment itself is still otherwise
    'Awaiting confirmation' — signals staff should follow up manually."""
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

Update the module docstring's mention of the deferred 6th state (currently says "A 6th state, 'Suppressed'... is deferred to Phase 2") to reflect that it's now implemented as "Needs follow-up":

```python
"""Daylist Confirmations — derived (never stored) confirmation display status.

Phase 1 of the Daylist Confirmations PRD (PRD/Daylist/daylist-confirmations-prd.md).
See docs/superpowers/specs/2026-07-02-daylist-confirmations-phase1-data-model-design.md
and docs/superpowers/specs/2026-07-03-daylist-confirmations-phase3-ui-surfacing-design.md.

`compute_confirmation_display_status` is intentionally NOT a stored field —
it's derived from data that's already source-of-truth elsewhere (Dentally's
own appointment state, the confirmation/cancellation-request fields, contact
info, and the appointment's most recent ConfirmationSequenceEnrollment), so
it can never drift out of sync with the underlying facts.
"""
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.ComputeConfirmationDisplayStatusNeedsFollowUpTests dentallyIntegration.tests.ComputeConfirmationDisplayStatusTests dentallyIntegration.tests.IsAppointmentEligibleForConfirmationTests --keepdb -v 2
```

Expected: PASS (all tests, including the pre-existing Phase 1/2 ones — `IsAppointmentEligibleForConfirmationTests` re-run to confirm `NOT_ELIGIBLE_STATUSES`'s new member doesn't break anything, since it's imported by that module too).

**Do NOT commit.**

---

### Task 2: Serializer exposure

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/serializers.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class DayListAppointmentSerializerConfirmationFieldsTests(TestCase):
    """DayListAppointmentSerializer exposes confirmation_status and
    confirmation_enrollment, backed by Phase 1/3's derivation function and
    Phase 2's enrollment model."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Serializer Confirmation Dental")

    def _appointment(self, **overrides):
        defaults = dict(
            practice=self.practice,
            dentally_id=10000,
            dentally_patient_id=1201,
            patient_name="Serializer Test Patient",
            patient_phone="+447700900000",
            state="pending",
            duration=30,
        )
        defaults.update(overrides)
        return DentallyAppointment.objects.create(**defaults)

    def test_confirmed_appointment_exposes_confirmed_status(self):
        from .serializers import DayListAppointmentSerializer

        appointment = self._appointment(patient_confirmed=True)
        data = DayListAppointmentSerializer(appointment).data
        self.assertEqual(data["confirmation_status"], "Confirmed")
        self.assertIsNone(data["confirmation_enrollment"])

    def test_appointment_with_active_enrollment_exposes_enrollment_summary(self):
        from .serializers import DayListAppointmentSerializer

        appointment = self._appointment()
        sequence = ConfirmationSequence.objects.create(
            practice=self.practice, name="Test sequence", status="active",
        )
        now = timezone.now()
        ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=sequence,
            appointment=appointment,
            status="active",
            current_step=1,
            last_sent_step=0,
            enrolled_at=now,
            enrolled_appointment_start=now,
            next_due_at=now + timedelta(days=1),
        )
        data = DayListAppointmentSerializer(appointment).data
        self.assertEqual(data["confirmation_status"], "Awaiting confirmation")
        self.assertIsNotNone(data["confirmation_enrollment"])
        self.assertEqual(data["confirmation_enrollment"]["sequence_name"], "Test sequence")
        self.assertEqual(data["confirmation_enrollment"]["status"], "active")
        self.assertEqual(data["confirmation_enrollment"]["last_sent_step"], 0)

    def test_appointment_with_no_enrollment_exposes_null_enrollment(self):
        from .serializers import DayListAppointmentSerializer

        appointment = self._appointment()
        data = DayListAppointmentSerializer(appointment).data
        self.assertIsNone(data["confirmation_enrollment"])
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && python manage.py test dentallyIntegration.tests.DayListAppointmentSerializerConfirmationFieldsTests --keepdb -v 2
```

Expected: FAIL — `KeyError: 'confirmation_status'` (field doesn't exist in serialized output yet).

- [ ] **Step 3: Add the fields**

In `dentallyIntegration/serializers.py`, `DayListAppointmentSerializer`: add these two field declarations near the existing `consent_summary` field declaration (just above `class Meta:`):

```python
    # Confirmation state (Phase 1-3 of the Daylist Confirmations PRD) — derived,
    # never stored. See confirmation_status.compute_confirmation_display_status.
    confirmation_status = serializers.SerializerMethodField()
    # Most recent ConfirmationSequenceEnrollment summary, or None if the
    # appointment has never been enrolled in a confirmation sequence.
    confirmation_enrollment = serializers.SerializerMethodField()
```

Add `"confirmation_status"` and `"confirmation_enrollment"` to `Meta.fields`, right after the existing `"patient_confirmed"` entry.

**Avoiding a double query per appointment** (flagged during Task 1's code review): a naive `get_confirmation_status` calling `compute_confirmation_display_status(obj)` (which internally queries `obj.confirmation_enrollments.order_by("-created_at").first()`) and a naive `get_confirmation_enrollment` independently re-running that same query would issue it TWICE per appointment on a list endpoint — on top of the N+1 risk across many appointments. Fix this with two small, backward-compatible changes:

First, in `dentallyIntegration/confirmation_status.py`, give `compute_confirmation_display_status` an optional `latest_enrollment` parameter defaulting to `None` (preserves the exact existing behavior for every current caller/test, which don't pass it — this is purely additive):

```python
def compute_confirmation_display_status(appointment, latest_enrollment=None):
    """One of: 'Cancelled', 'Cancellation requested', 'Confirmed',
    'No valid contact', 'Needs follow-up', 'Awaiting confirmation'. Checked
    in this priority order so Dentally's own cancellation status always
    wins, per the PRD edge case: "if Dentally says cancelled, Daylist should
    show cancelled even if Pathway previously had confirmed".

    'Needs follow-up' is the appointment's most recent confirmation
    enrollment having already completed or stopped (no more automated
    messages coming) while the appointment itself is still otherwise
    'Awaiting confirmation' — signals staff should follow up manually.

    `latest_enrollment` is an optional pre-fetched
    ConfirmationSequenceEnrollment (or None) — pass it when the caller
    already looked it up (e.g. DayListAppointmentSerializer, to avoid
    querying twice) to skip this function's own lookup. When omitted
    (the default, used by every pre-existing caller), the lookup happens
    here exactly as before."""
    if appointment.state == "cancelled":
        return "Cancelled"
    if appointment.cancellation_requested:
        return "Cancellation requested"
    if appointment.patient_confirmed:
        return "Confirmed"
    if not has_valid_contact(appointment):
        return "No valid contact"
    if latest_enrollment is None:
        latest_enrollment = appointment.confirmation_enrollments.order_by("-created_at").first()
    if latest_enrollment and latest_enrollment.status in ("completed", "stopped"):
        return "Needs follow-up"
    return "Awaiting confirmation"
```

Run Task 1's own tests after this change to confirm nothing regressed (they call the function with only one positional argument, so the new parameter's default applies and behavior is identical):

```bash
python manage.py test dentallyIntegration.tests.ComputeConfirmationDisplayStatusNeedsFollowUpTests dentallyIntegration.tests.ComputeConfirmationDisplayStatusTests dentallyIntegration.tests.IsAppointmentEligibleForConfirmationTests --keepdb -v 2
```
Expected: PASS, all 17, unchanged from Task 1.

Second, add these two methods to `DayListAppointmentSerializer`, near `get_opportunity_detail` (around line 561-564) — both share ONE cached lookup, following the exact same `hasattr`-instance-caching pattern this same serializer already uses for `_get_patient` (lines 613-660):

```python
    def _get_latest_confirmation_enrollment(self, obj):
        """Cached lookup of the appointment's most recent
        ConfirmationSequenceEnrollment — shared by get_confirmation_status
        and get_confirmation_enrollment so a single appointment only queries
        this once, not twice, per serialization. Mirrors _get_patient's own
        hasattr-based per-instance caching pattern."""
        if not hasattr(obj, "_cached_latest_confirmation_enrollment"):
            obj._cached_latest_confirmation_enrollment = (
                obj.confirmation_enrollments.order_by("-created_at").first()
            )
        return obj._cached_latest_confirmation_enrollment

    def get_confirmation_status(self, obj):
        from .confirmation_status import compute_confirmation_display_status

        return compute_confirmation_display_status(
            obj, latest_enrollment=self._get_latest_confirmation_enrollment(obj)
        )

    def get_confirmation_enrollment(self, obj):
        enrollment = self._get_latest_confirmation_enrollment(obj)
        if not enrollment:
            return None
        return {
            "sequence_name": enrollment.sequence.name,
            "status": enrollment.status,
            "next_due_at": enrollment.next_due_at,
            "stopped_reason": enrollment.stopped_reason,
            "last_sent_step": enrollment.last_sent_step,
        }
```

This still leaves one query per appointment (not per appointment per field), matching the cost of any other single per-row lookup already in this serializer (e.g. `_get_patient` itself) — full N+1 elimination via `Prefetch`/view-level batching across the whole page is a further optimization this task doesn't attempt, consistent with this task's scope (expose the fields correctly; the existing `_get_patient` per-row lookup this serializer already does isn't batched either, so this doesn't regress anything relative to the current baseline).

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.DayListAppointmentSerializerConfirmationFieldsTests --keepdb -v 2
```

Expected: PASS (3 tests).

- [ ] **Step 5: Run the day-list view's existing test class to confirm no regression**

```bash
python manage.py test dentallyIntegration.tests.DentallyDayListViewSetTests --keepdb -v 2
```

Expected: same pass/fail status as before this task (this class has pre-existing unrelated failures per prior sessions' notes — confirm the count/names of any failures are unchanged, not that everything passes).

**Do NOT commit.**

---

### Task 3: Activity-log integration for confirmation-automation sends

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/confirmation_automation.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class ConfirmationAutomationActivityLogTests(TestCase):
    """send_sms/send_email resolve the appointment's Patient→ContactIdentity
    (the SAME path DayListAppointmentSerializer already uses, NOT a new
    contact created from raw phone/email) and emit an activity-log entry —
    reusing record_events.on_record_created, which auto-derives
    'sms_sent'/'email_sent' from the message's own `direction` field, so no
    new ACTIVITY_TYPE_CHOICES value is needed for this path."""

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Confirmation Activity Log Dental", twilio_phone_number="+447700900000"
        )
        self.appointment = DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=10100,
            dentally_patient_id=1301,
            patient_name="Activity Log Test Patient",
            patient_phone="+447700900111",
            state="pending",
            duration=30,
        )

    def _create_matching_patient(self):
        from TreatmentPlan.models import ContactIdentity, Patient

        contact = ContactIdentity.objects.create(
            practice=self.practice, normalized_phone="7700900111", display_name="Activity Log Test Patient",
        )
        Patient.objects.create(
            practice=self.practice,
            contact=contact,
            first_name="Activity",
            last_name="Log",
            meta_data={"id": self.appointment.dentally_patient_id},
        )
        return contact

    @override_settings(TWILIO_ACCOUNT_SID="ACtest", TWILIO_AUTH_TOKEN="tok")
    def test_send_sms_creates_activity_log_entry_when_patient_resolves(self):
        from activityLog.models import ActivityLog

        from .confirmation_automation import send_sms

        contact = self._create_matching_patient()
        with patch("twilio.rest.Client") as MockTwilio:
            MockTwilio.return_value.messages.create.return_value.sid = "SMfake"
            send_sms(self.practice, self.appointment.patient_phone, "Please confirm.",
                     appointment=self.appointment)

        self.assertTrue(ActivityLog.objects.filter(contact=contact).exists())

    @override_settings(TWILIO_ACCOUNT_SID="ACtest", TWILIO_AUTH_TOKEN="tok")
    def test_send_sms_does_not_crash_when_no_matching_patient(self):
        """If no Patient row matches (e.g. patient not yet synced from
        Dentally), the send must still succeed — activity-log linkage is
        best-effort, never a hard requirement for sending."""
        from .confirmation_automation import send_sms

        with patch("twilio.rest.Client") as MockTwilio:
            MockTwilio.return_value.messages.create.return_value.sid = "SMfake"
            ok, _ = send_sms(self.practice, self.appointment.patient_phone, "Please confirm.",
                             appointment=self.appointment)

        self.assertTrue(ok)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && python manage.py test dentallyIntegration.tests.ConfirmationAutomationActivityLogTests --keepdb -v 2
```

Expected: FAIL — `TypeError: send_sms() got an unexpected keyword argument 'appointment'` (the function doesn't accept this parameter yet).

- [ ] **Step 3: Add appointment-aware contact resolution and activity logging**

In `dentallyIntegration/confirmation_automation.py`, add a new helper function (place it right before `send_sms`):

```python
def _get_contact_for_appointment(appointment):
    """Resolve the ContactIdentity already associated with this appointment's
    patient — the SAME Patient.meta_data->>'id' lookup path
    DayListAppointmentSerializer._get_patient already uses (serializers.py),
    NOT a new contact created from raw phone/email (that would risk creating
    a contact divergent from the one already linked elsewhere — a known,
    actively-tracked problem area in this project). Returns None if no
    matching Patient row exists yet (e.g. not synced) or it has no contact —
    activity-log linkage is best-effort, never required for a send to
    succeed."""
    from TreatmentPlan.models import Patient

    if not appointment.dentally_patient_id:
        return None
    try:
        patient = Patient.objects.select_related("contact").get(
            practice_id=appointment.practice_id,
            meta_data__id=appointment.dentally_patient_id,
        )
    except (Patient.DoesNotExist, Patient.MultipleObjectsReturned):
        return None
    return patient.contact


def _log_confirmation_activity(message, appointment):
    """Best-effort activity-log entry for a confirmation-automation send —
    mirrors recall_automation.py's own `_log_activity` wrapper exactly (never
    let activity-log failure break a send). No-ops if no ContactIdentity
    resolves for this appointment's patient."""
    contact = _get_contact_for_appointment(appointment)
    if not contact:
        return
    message.contact = contact
    message.save(update_fields=["contact"])
    try:
        from TreatmentPath.record_events import on_record_created

        on_record_created(entity=message, user=None)
    except Exception:  # noqa: BLE001 — best-effort, never break a send
        logger.warning(
            "[ConfirmationAutomation] activity-log entry failed for appointment %s",
            appointment.id,
        )
```

Then update `send_sms`'s signature and body to accept and use an optional `appointment` parameter:

```python
def send_sms(practice, to_phone, body, appointment=None):
    """Sends one SMS via Twilio and logs an SMSMessage row with
    source_type='confirmation_automation' — a NEW function, not a reuse of
    daylist_automation.send_sms, because that one hardcodes
    source_type='daylist_automation' with no override, which would mislabel
    confirmation-sequence sends. If `appointment` is given, best-effort logs
    an activity-log entry against the appointment's resolved patient contact.
    Returns (ok: bool, detail: str)."""
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

    message = SMSMessage.objects.create(
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
    if appointment is not None:
        _log_confirmation_activity(message, appointment)
    return status_ == "sent", error or "sent"
```

Apply the identical `appointment=None` parameter and `_log_confirmation_activity` call to `send_email` (same pattern — add the parameter, keep everything else, log after the `EmailMessages.objects.create(...)` call, only if `appointment is not None`):

```python
def send_email(practice, to_email, subject, body, appointment=None):
    """Sends one email via the same EmailServiceClient + sending-domain path
    as daylist_automation.send_email, but a NEW function so the audit-trail
    row is correctly tagged source_type='confirmation_automation'. If
    `appointment` is given, best-effort logs an activity-log entry against
    the appointment's resolved patient contact. Returns (ok: bool, detail: str)."""
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

    message = EmailMessages.objects.create(
        practice=practice,
        direction="outgoing",
        message_purpose="automation",
        source_type="confirmation_automation",
        to_patient=to_email,
        subject=subject or "Please confirm your appointment",
        body=body,
    )
    if appointment is not None:
        _log_confirmation_activity(message, appointment)
    return status_ == "sent", error or "sent"
```

Finally, update the two call sites in `advance_confirmation_enrollments` to pass `appointment=appointment`:

```python
            ok, _ = send_sms(sequence.practice, appointment.patient_phone, rendered_body, appointment=appointment)
```
and
```python
            ok, _ = send_email(sequence.practice, appointment.patient_email, rendered_subject, rendered_body, appointment=appointment)
```

(These are the exact two `send_sms`/`send_email` calls inside `advance_confirmation_enrollments`, added in Phase 2's plan — find them by searching for `ok, _ = send_sms(` and `ok, _ = send_email(` in the file.)

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationAutomationActivityLogTests --keepdb -v 2
```

Expected: PASS (2 tests).

- [ ] **Step 5: Run the full confirmation-automation suite to confirm no regressions**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationAutomationSendingTests dentallyIntegration.tests.AdvanceConfirmationEnrollmentsTests dentallyIntegration.tests.EnrollEligibleAppointmentsTests dentallyIntegration.tests.ProcessConfirmationEnrollmentsTests --keepdb -v 2
```

Expected: PASS, all green — the existing `ConfirmationAutomationSendingTests` calls `send_sms`/`send_email` WITHOUT the new `appointment` kwarg (it's optional, defaults to `None`), so those calls must be unaffected.

**Do NOT commit.**

---

### Task 4: Activity-log integration for confirm/cancellation-request

**Files:**
- Modify: `TreatmentPath/activityLog/models.py`
- Modify: `TreatmentPath/activityLog/serializers.py`
- Modify: `TreatmentPath/dentallyIntegration/views/appointment_confirm_views.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class AppointmentConfirmActivityLogTests(TestCase):
    """AppointmentConfirmViewSet.create emits an activity-log entry (via the
    low-level ActivityLogHelper.log, since DentallyAppointment has no
    `contact` FK of its own — record_events.on_record_created requires
    the entity to already carry one) when the appointment's patient resolves
    to a ContactIdentity. Best-effort — never blocks the confirm/cancel
    action itself if logging fails or no contact resolves."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Confirm Activity Log Dental")
        self.appointment = DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=10200,
            dentally_patient_id=1401,
            patient_name="Confirm Activity Test Patient",
            state="pending",
            duration=30,
        )
        self.token = self.appointment.confirmation_token = "abc123"
        self.appointment.save(update_fields=["confirmation_token"])

    def _create_matching_patient(self):
        from TreatmentPlan.models import ContactIdentity, Patient

        contact = ContactIdentity.objects.create(
            practice=self.practice, normalized_phone="7700900999", display_name="Confirm Activity Test Patient",
        )
        Patient.objects.create(
            practice=self.practice,
            contact=contact,
            first_name="Confirm",
            last_name="Activity",
            meta_data={"id": self.appointment.dentally_patient_id},
        )
        return contact

    def test_confirm_creates_activity_log_entry_when_patient_resolves(self):
        from activityLog.models import Activity, ActivityLog

        contact = self._create_matching_patient()
        response = self.client.post(
            "/api/backend/dentally/confirm/", {"token": self.token}, format="json",
        )
        self.assertEqual(response.status_code, 200)
        self.assertTrue(ActivityLog.objects.filter(contact=contact).exists())
        self.assertTrue(
            Activity.objects.filter(activity_type="patient_confirmed").exists()
        )

    def test_request_cancellation_creates_activity_log_entry(self):
        from activityLog.models import Activity

        self._create_matching_patient()
        response = self.client.post(
            "/api/backend/dentally/confirm/",
            {"token": self.token, "action": "request_cancellation"},
            format="json",
        )
        self.assertEqual(response.status_code, 200)
        self.assertTrue(
            Activity.objects.filter(activity_type="cancellation_requested").exists()
        )

    def test_confirm_does_not_fail_when_no_matching_patient(self):
        response = self.client.post(
            "/api/backend/dentally/confirm/", {"token": self.token}, format="json",
        )
        self.assertEqual(response.status_code, 200)
        self.appointment.refresh_from_db()
        self.assertTrue(self.appointment.patient_confirmed)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && python manage.py test dentallyIntegration.tests.AppointmentConfirmActivityLogTests --keepdb -v 2
```

Expected: FAIL — `test_confirm_creates_activity_log_entry_when_patient_resolves` and `test_request_cancellation_creates_activity_log_entry` fail (no `ActivityLog`/`Activity` rows created yet); the third test may already pass (confirming doesn't currently fail, just doesn't log).

- [ ] **Step 3: Add the two new `ACTIVITY_TYPE_CHOICES` values**

In `activityLog/models.py`, `ActivityLog.ACTIVITY_TYPE_CHOICES`, add two new entries under the existing `# Appointments` section (currently lines 86-90):

```python
        # Appointments
        ('appointment_scheduled', 'Appointment Scheduled'),
        ('appointment_cancelled', 'Appointment Cancelled'),
        ('appointment_completed', 'Appointment Completed'),
        ('appointment_missed', 'Appointment Missed'),
        ('patient_confirmed', 'Patient Confirmed'),
        ('cancellation_requested', 'Cancellation Requested'),
```

(`Activity.ACTIVITY_TYPE_CHOICES = ActivityLog.ACTIVITY_TYPE_CHOICES` already re-exports the same list — no separate change needed there.)

- [ ] **Step 4: Add `dentallyappointment` to `ENTITY_TYPE_MAP`**

In `activityLog/serializers.py`, `ENTITY_TYPE_MAP` (currently lines 12-23):

```python
ENTITY_TYPE_MAP = {
    "patient": "Patient",
    "intake": "Intake",
    "nurture": "Nurture",
    "treatmentplan": "Treatment Plan",
    "calllog": "Call Log",
    "smsmessage": "SMS Message",
    "emailmessage": "Email Message",
    "emailmessages": "Email Message",
    "messagesession": "Message Session",
    "archive": "Archive",
    "dentallyappointment": "Appointment",
}
```

- [ ] **Step 5: Wire the confirm/cancel view**

In `dentallyIntegration/views/appointment_confirm_views.py`, `AppointmentConfirmViewSet.create` (the method from Phase 1's Task 3), add activity logging. Replace the method with:

```python
    def create(self, request):
        """POST /confirm/ — {"token": ..., "action": "confirm" | "request_cancellation"}.
        `action` defaults to "confirm" for backward compatibility with existing
        outstanding links / the current frontend, which POSTs {"token": ...} alone."""
        from django.utils import timezone

        token = request.data.get("token")
        action = request.data.get("action", "confirm")
        appointment, err = self._resolve_appointment(token)
        if err:
            return err

        if action == "request_cancellation":
            if not appointment.cancellation_requested:
                appointment.cancellation_requested = True
                appointment.cancellation_requested_at = timezone.now()
                appointment.cancellation_requested_source = "confirm_link"
                appointment.save(update_fields=[
                    "cancellation_requested",
                    "cancellation_requested_at",
                    "cancellation_requested_source",
                ])
                self._log_confirmation_activity(appointment, "cancellation_requested")
            return Response({"cancellation_requested": True})

        if not appointment.patient_confirmed:
            appointment.patient_confirmed = True
            appointment.patient_confirmed_at = timezone.now()
            appointment.confirmation_source = "confirm_link"
            appointment.save(update_fields=[
                "patient_confirmed",
                "patient_confirmed_at",
                "confirmation_source",
            ])
            self._log_confirmation_activity(appointment, "patient_confirmed")
        return Response({"confirmed": True})

    def _log_confirmation_activity(self, appointment, activity_type):
        """Best-effort activity-log entry for a patient confirm/cancellation-
        request action. Uses the low-level ActivityLogHelper.log (not
        record_events.on_record_created) because DentallyAppointment has no
        `contact` FK of its own — this helper takes entity and contact as
        separate parameters, which fits. Never blocks the confirm/cancel
        action if logging fails or no contact resolves."""
        from dentallyIntegration.confirmation_automation import _get_contact_for_appointment

        contact = _get_contact_for_appointment(appointment)
        if not contact:
            return
        try:
            from activityLog.models import ActivityLogHelper

            ActivityLogHelper.log(
                practice=appointment.practice,
                activity_type=activity_type,
                entity=appointment,
                contact=contact,
                is_system_generated=False,
                action_family="appointment",
                source_system="pathway_automation",
            )
        except Exception:  # noqa: BLE001 — best-effort, never break confirm/cancel
            pass
```

- [ ] **Step 6: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.AppointmentConfirmActivityLogTests --keepdb -v 2
```

Expected: PASS (3 tests).

- [ ] **Step 7: Run the full Phase 1 confirm-endpoint suite to confirm no regressions**

```bash
python manage.py test dentallyIntegration.tests.AppointmentConfirmViewSetTests dentallyIntegration.tests.AppointmentConfirmLinkViewSetTests --keepdb -v 2
```

Expected: same pass/fail status as before this task (the one known pre-existing `test_returns_signed_url` failure is unrelated and unaffected — everything else should still pass).

- [ ] **Step 8: Generate and apply a migration for the new `ACTIVITY_TYPE_CHOICES` entries**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py makemigrations activityLog
python manage.py migrate activityLog
```

Expected: a small migration with `AlterField` operations on `activity_type` for `ActivityLog`/`Activity` (choices-only change, matching the same no-op-at-the-DB-level pattern as the `appointment_confirmation` template-type migration from an earlier phase). If it sweeps in anything unrelated, remove that operation by hand so only the choices change remains.

**Do NOT commit.**

---

## Frontend (`perfect-pixel-playground-project`)

### Task 5: Types

**Files:**
- Modify: `src/pages/day-list/types.ts`

- [ ] **Step 1: Add the new types**

Add near the existing `NoShowRisk` type definitions:

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

- [ ] **Step 2: Add the two new fields to `Appointment`**

In the `Appointment` interface, add right after the existing `patient_confirmed?: boolean` field:

```ts
  confirmation_status: ConfirmationStatus;
  confirmation_enrollment: ConfirmationEnrollment | null;
```

- [ ] **Step 3: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit
```

Expected: no NEW errors introduced by this change (the two fields are additive to an existing interface — if `Appointment` objects are constructed anywhere with an object literal missing these required fields, `tsc` will flag it; if so, make those fields have sensible defaults at the construction site rather than making them optional here, since the backend always returns them).

**Do NOT commit.**

---

### Task 6: Appointment card — unified status corner slot

**Files:**
- Modify: `src/pages/day-list/components/AppointmentCard.tsx`

- [ ] **Step 1: Replace the risk-corner logic with a unified status slot**

Replace the existing risk-corner variables (currently around lines 443-447):

```ts
  const riskStatus = appointment.no_show_risk?.status;
  const showRiskCorner = riskStatus === 'High' || riskStatus === 'Elevated';
  const riskCornerClass = riskStatus === 'High'
    ? 'bg-red-100 text-red-800'
    : 'bg-amber-50 text-amber-800';
```

with:

```ts
  const riskStatus = appointment.no_show_risk?.status;
  const showRiskCorner = riskStatus === 'High' || riskStatus === 'Elevated';
  const riskCornerClass = riskStatus === 'High'
    ? 'bg-red-100 text-red-800'
    : 'bg-amber-50 text-amber-800';

  // Unified status corner (FR11/FR12a) — the backend already computes the
  // single correct confirmation_status value; this is a direct switch, not a
  // re-derivation. Risk only ever shows in the 'Awaiting confirmation' case —
  // every other confirmation_status value takes over this slot entirely,
  // satisfying FR12a (confirmed/etc. appointments never show the risk alert,
  // even if the underlying risk score is High).
  const CONFIRMATION_STATUS_STYLE: Record<
    Exclude<typeof appointment.confirmation_status, 'Awaiting confirmation'>,
    { label: string; className: string }
  > = {
    'Cancelled': { label: 'Cancelled', className: 'bg-gray-100 text-gray-600' },
    'Cancellation requested': { label: 'Cancellation requested', className: 'bg-amber-100 text-amber-800' },
    'Confirmed': { label: 'Confirmed', className: 'bg-green-100 text-green-700' },
    'No valid contact': { label: 'No valid contact', className: 'bg-gray-100 text-gray-500' },
    'Needs follow-up': { label: 'Needs follow-up', className: 'bg-purple-100 text-purple-700' },
  };
  const statusCornerEntry = appointment.confirmation_status !== 'Awaiting confirmation'
    ? CONFIRMATION_STATUS_STYLE[appointment.confirmation_status]
    : null;
```

- [ ] **Step 2: Remove the inline Confirmed/Cancelled badges near the patient name**

Remove these two blocks (currently around lines 650-661 — the `{appointment.status === 'cancelled' && (...)}` and `{appointment.patient_confirmed && ... && (...)}` blocks). Leave the `Completed` badge (driven by `appointment.status === 'completed'`, a different concept from confirmation state) untouched.

- [ ] **Step 3: Replace the corner-slot render logic**

Replace the existing risk-corner render block (currently around lines 769-778):

```tsx
              {showRiskCorner && (
                <button
                  type="button"
                  onClick={() => onOpenAIReport(appointment.id)}
                  className={`flex min-w-[72px] flex-shrink-0 items-center justify-center gap-1.5 rounded-lg px-2.5 py-2 text-xs font-semibold transition-opacity hover:opacity-85 ${riskCornerClass}`}
                  title={`No-show risk: ${riskStatus}`}
                >
                  <AlertTriangle className="h-4 w-4 flex-shrink-0" />
                  <span>{riskStatus} risk</span>
                </button>
              )}
```

with:

```tsx
              {statusCornerEntry ? (
                <span
                  className={`flex min-w-[72px] flex-shrink-0 items-center justify-center gap-1.5 rounded-lg px-2.5 py-2 text-xs font-semibold ${statusCornerEntry.className}`}
                  title={`Confirmation status: ${statusCornerEntry.label}`}
                >
                  <span>{statusCornerEntry.label}</span>
                </span>
              ) : showRiskCorner && (
                <button
                  type="button"
                  onClick={() => onOpenAIReport(appointment.id)}
                  className={`flex min-w-[72px] flex-shrink-0 items-center justify-center gap-1.5 rounded-lg px-2.5 py-2 text-xs font-semibold transition-opacity hover:opacity-85 ${riskCornerClass}`}
                  title={`No-show risk: ${riskStatus}`}
                >
                  <AlertTriangle className="h-4 w-4 flex-shrink-0" />
                  <span>{riskStatus} risk</span>
                </button>
              )}
```

- [ ] **Step 4: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit
```

Expected: no new errors.

- [ ] **Step 5: Manual verification**

Start the frontend dev server (or use whatever `run`-skill/dev-server convention this project uses) and navigate to the Daylist page:
1. Find (or set up via the API/admin) an appointment with `confirmation_status: 'Confirmed'` — confirm the corner slot shows a green "Confirmed" badge, and the inline badge near the name no longer shows a separate "Confirmed" pill.
2. Find/set up an appointment with High risk AND `confirmation_status: 'Awaiting confirmation'` — confirm the risk badge still shows.
3. Find/set up an appointment with High risk AND `confirmation_status: 'Confirmed'` — confirm the risk badge does NOT show (replaced by the green Confirmed badge) — this is the FR12a behavior the whole task exists for.

**Do NOT commit.**

---

### Task 7: Filter — confirmation status multi-select

**Files:**
- Modify: `src/pages/day-list/components/FilterDialog.tsx`
- Modify: `src/pages/DayListPage.tsx`

- [ ] **Step 1: Add the new prop types and a multi-select component**

In `FilterDialog.tsx`, add to `FilterDialogProps` (after the existing `riskSort`/`onRiskSortChange` props):

```ts
  confirmationFilter: ConfirmationStatus[];
  onConfirmationFilterChange: (value: ConfirmationStatus[]) => void;
```

Import the type at the top of the file:

```ts
import type { Clinician, ConfirmationStatus } from '../types';
```

Add a new component in the same file, modeled directly on the existing `ClinicianMultiSelect` (same Popover + checkmark-list structure, just over the 6 confirmation states instead of clinicians):

```tsx
const ALL_CONFIRMATION_STATUSES: ConfirmationStatus[] = [
  'Confirmed',
  'Awaiting confirmation',
  'Cancellation requested',
  'Cancelled',
  'No valid contact',
  'Needs follow-up',
];

/** Multi-select confirmation-status dropdown — same Popover dropdown style as
 *  ClinicianMultiSelect above, over the 6 confirmation states (empty = All). */
function ConfirmationStatusMultiSelect({
  selected,
  onChange,
}: {
  selected: ConfirmationStatus[];
  onChange: (value: ConfirmationStatus[]) => void;
}) {
  const toggle = (status: ConfirmationStatus) => {
    onChange(
      selected.includes(status)
        ? selected.filter((s) => s !== status)
        : [...selected, status],
    );
  };
  const label = selected.length === 0 || selected.length === ALL_CONFIRMATION_STATUSES.length
    ? 'All'
    : selected.length === 1
    ? selected[0]
    : `${selected.length} selected`;

  return (
    <Popover>
      <PopoverTrigger asChild>
        <Button variant="outline" className="w-full justify-between font-normal">
          {label}
          <ChevronDown className="h-4 w-4 opacity-50" />
        </Button>
      </PopoverTrigger>
      <PopoverContent className="w-64 p-1" align="start">
        {ALL_CONFIRMATION_STATUSES.map((status) => (
          <div
            key={status}
            onClick={() => toggle(status)}
            className="flex cursor-pointer items-center justify-between rounded-md px-2 py-1.5 text-sm hover:bg-gray-100"
          >
            <span>{status}</span>
            {(selected.length === 0 || selected.includes(status)) && (
              <Check className="h-4 w-4 text-primary" />
            )}
          </div>
        ))}
      </PopoverContent>
    </Popover>
  );
}
```

- [ ] **Step 2: Render it in the dialog body**

Add a new filter section after the existing "Sort by Risk" block (right before the closing `</div>` of the filter body, before `<DialogFooter>`):

```tsx
          <div className="space-y-2">
            <label className="text-sm font-medium text-gray-700">Confirmation Status</label>
            <ConfirmationStatusMultiSelect
              selected={confirmationFilter}
              onChange={onConfirmationFilterChange}
            />
          </div>
```

Destructure the two new props in the component's function signature alongside the existing ones.

- [ ] **Step 3: Wire state in `DayListPage.tsx`**

Add near the existing `opportunityFilter`/`statusFilter` state declarations:

```ts
  const [confirmationFilter, setConfirmationFilter] = useState<ConfirmationStatus[]>([]);
```

Import `ConfirmationStatus` from `./day-list/types` alongside the other type imports already pulled from there.

In the `baseFilteredAppointments` useMemo (the one that already filters by `selectedProviderIds`/`riskSort`/`statusFilter`), add confirmation filtering. Find the existing filter chain and add this condition (an empty `confirmationFilter` array means no filtering, matching the multi-select's "empty = All" convention):

```ts
    if (confirmationFilter.length > 0) {
      result = result.filter((apt) => confirmationFilter.includes(apt.confirmation_status));
    }
```

Add `confirmationFilter` to that `useMemo`'s dependency array.

Pass the two new props to `<FilterDialog>` wherever it's rendered:

```tsx
              confirmationFilter={confirmationFilter}
              onConfirmationFilterChange={setConfirmationFilter}
```

- [ ] **Step 4: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit
```

Expected: no new errors.

- [ ] **Step 5: Manual verification**

Open the Filter dialog on the Daylist page, select only "Confirmed" in the new Confirmation Status filter, confirm the list narrows to only confirmed appointments. Clear the selection, confirm the list returns to showing all.

**Do NOT commit.**

---

### Task 8: Practice-level confirmation coverage summary card

**Files:**
- Modify: `src/pages/day-list/components/SummaryCards.tsx`
- Modify: `src/pages/DayListPage.tsx`

- [ ] **Step 1: Add the new card key and style**

In `SummaryCards.tsx`, update the `SummaryCardKey` type:

```ts
type SummaryCardKey = 'hygiene' | 'implants' | 'crowns' | 'exam' | 'confirmed';
```

Add a new `CARD_STYLE` entry:

```ts
  confirmed: {
    accent: 'from-green-400 via-green-200 to-transparent',
    active: 'border-green-300 ring-1 ring-green-300',
    hover: 'hover:border-green-200',
  },
```

- [ ] **Step 2: Extend `DayListPage.tsx`'s `OpportunityFilter` type and `summaryCards` computation**

Update the `OpportunityFilter` type (currently `'hygiene' | 'implants' | 'crowns' | 'exam'`):

```ts
type OpportunityFilter = 'hygiene' | 'implants' | 'crowns' | 'exam' | 'confirmed';
```

In the `summaryCards` useMemo (currently computing `hygiene`/`implants`/`crowns`/`exam` counts), add a confirmed/total tally and push a new card:

```ts
  const summaryCards = useMemo(() => {
    // PRD opportunity tallies = the three opportunity badges only: Crown,
    // Implant, Hygiene. Xrays Due is a due-state, NOT an opportunity (PRD line
    // 426), so it isn't counted here. Cosmetic/Orthodontic are deprecated.
    let hygiene = 0, implants = 0, crowns = 0, exam = 0, confirmed = 0;
    for (const apt of baseFilteredAppointments) {
      if (apt.hygiene_due) hygiene++;
      if (apt.implant_opportunity) implants++;
      if (apt.crown_opportunity) crowns++;
      if (apt.exam_due) exam++;
      if (apt.confirmation_status === 'Confirmed') confirmed++;
    }
    const cards: Array<{ title: string; count: number; key: OpportunityFilter }> = [];
    if (hygiene > 0) cards.push({ title: 'Due Hygiene', count: hygiene, key: 'hygiene' });
    if (implants > 0) cards.push({ title: 'Implants', count: implants, key: 'implants' });
    if (crowns > 0) cards.push({ title: 'Crowns', count: crowns, key: 'crowns' });
    if (exam > 0) cards.push({ title: 'Exam', count: exam, key: 'exam' });
    if (baseFilteredAppointments.length > 0) {
      cards.push({
        title: `Confirmed: ${confirmed} / ${baseFilteredAppointments.length}`,
        count: confirmed,
        key: 'confirmed',
      });
    }
    return cards;
  }, [baseFilteredAppointments]);
```

- [ ] **Step 3: Wire the click-to-filter behavior**

In the `filteredAppointments` useMemo's `switch (opportunityFilter)` block, add a case:

```ts
        case 'confirmed':
          return baseFilteredAppointments.filter((apt) => apt.confirmation_status !== 'Confirmed');
```

(This card's click behavior is "show me who's NOT confirmed yet" per FR12d, unlike the opportunity cards which filter TO the matching set — the card's own title already states the X/Y count, so clicking it drills into the *incomplete* set, which is the actionable one.)

- [ ] **Step 4: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit
```

Expected: no new errors.

- [ ] **Step 5: Manual verification**

On the Daylist page, confirm a new "Confirmed: X / Y" card appears in the summary row. Click it — confirm the list narrows to non-confirmed appointments only. Click it again — confirm it toggles back to the full list.

**Do NOT commit.**

---

### Task 9: Activity feed — new confirmation event type

**Files:**
- Modify: `src/components/patients/patient-panel/PatientActivityFeed.tsx`
- Modify: `src/components/patients/patient-panel/PatientPanelAccordion.tsx`

- [ ] **Step 1: Add the new `ActivityType` value + icon**

In `PatientActivityFeed.tsx`, add `"confirmation"` to the `ActivityType` union:

```ts
export type ActivityType =
  | "call"
  | "email"
  | "sms"
  | "note"
  | "update"
  | "task"
  | "contact"
  | "consent"
  | "confirmation";
```

The icon config lives in the `ActivityIcon` component (`PatientActivityFeed.tsx:64-74`), an `iconConfig` object keyed by `ActivityType`, each value shaped `{ icon: LucideIcon, bg: string, text: string }`. Add `BadgeCheck` to the existing `lucide-react` import at the top of the file (alongside `ShieldCheck` etc. — it's already used elsewhere in this codebase, e.g. `AppointmentCard.tsx`, so it's a safe, already-established icon choice), then add a new entry to `iconConfig` right after the existing `consent` entry:

```ts
const ActivityIcon: React.FC<{ type: ActivityType }> = ({ type }) => {
  const iconConfig = {
    call: { icon: Phone, bg: "bg-gray-100", text: "text-gray-600" },
    email: { icon: Mail, bg: "bg-[#a695eb]", text: "text-white" },
    sms: { icon: MessageSquare, bg: "bg-[#a695eb]", text: "text-white" },
    note: { icon: FileText, bg: "bg-amber-100", text: "text-amber-600" },
    update: { icon: Pencil, bg: "bg-blue-100", text: "text-blue-600" },
    task: { icon: ClipboardList, bg: "bg-indigo-100", text: "text-indigo-600" },
    contact: { icon: UserPlus, bg: "bg-green-100", text: "text-green-600" },
    consent: { icon: ShieldCheck, bg: "bg-teal-100", text: "text-teal-700" },
    confirmation: { icon: BadgeCheck, bg: "bg-green-100", text: "text-green-700" },
  };
```

(This is the complete `iconConfig` object with the one new line added — the rest of `ActivityIcon`'s body below it, e.g. `const config = iconConfig[type]; const Icon = config.icon;`, is unchanged.)

- [ ] **Step 2: Map the two new backend activity types to it**

In `PatientPanelAccordion.tsx`'s `mapActivityType` function (`typeMap` object, currently lines 192-215), add two entries in the existing "Consent activities" section (renaming that comment slightly since it's no longer consent-only, or adding a new comment line — either is fine):

```ts
    // Consent / confirmation milestone activities
    'consent_sent': 'consent',
    'consent_signed': 'consent',
    'patient_confirmed': 'confirmation',
    'cancellation_requested': 'confirmation',
```

- [ ] **Step 3: Verify `COMM_TYPES` grouping is correct**

Confirm (no code change needed, just verify by reading) that `COMM_TYPES = new Set(['call', 'email', 'sms'])` at line 756 does NOT include `'confirmation'` — this is correct as-is, since `patient_confirmed`/`cancellation_requested` are patient milestones (journey), not message sends, and should group under "journey" alongside `consent`, matching the design spec.

- [ ] **Step 4: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit
```

Expected: no new errors.

- [ ] **Step 5: Manual end-to-end verification**

This is the one step that exercises the full backend-to-frontend chain built across this whole phase:
1. Using the Django shell or the public confirm page, confirm a test appointment whose patient has a matching `Patient`/`ContactIdentity` row (per Task 3/4's contact-resolution logic).
2. Open that patient's panel in the Daylist frontend (click the patient on their appointment card).
3. In the Overview tab's activity feed, confirm a new "Confirmation ... Patient Confirmed" entry appears (title built from `entity_type_display` + `activity_type_display`, i.e. "Appointment Patient Confirmed") under the Journey category.

**Do NOT commit.**

---

## Summary of spec coverage

- "Needs follow-up" 6th state, `NOT_ELIGIBLE_STATUSES` update → Task 1.
- Serializer exposure (`confirmation_status`, `confirmation_enrollment`) → Task 2.
- Activity-log integration for automated sends (reusing existing Patient→Contact resolution, not creating duplicate contacts; no new activity-type choices needed for this path, per the corrected deviation) → Task 3.
- Activity-log integration for confirm/cancellation-request (new `patient_confirmed`/`cancellation_requested` choices, `ENTITY_TYPE_MAP` entry) → Task 4.
- Frontend types → Task 5.
- Card badge unification (FR11, FR12a) → Task 6.
- Filter (FR12, FR12b) → Task 7.
- Practice-level coverage card + drill-down (FR12c, FR12d) → Task 8.
- History via existing activity feed (FR13) → Task 9.
- Out-of-scope items from the spec (cohort/bucket targeting, follow-up task automation, sequence editor UI, historical backfill) — correctly not covered by any task here.

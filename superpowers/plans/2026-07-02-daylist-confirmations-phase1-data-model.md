# Daylist Confirmations Phase 1 — Data Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **Do NOT run `git add`/`git commit`/`git push` at any step — the user handles all VCS operations themselves. Leave all changes uncommitted for review.**

**Goal:** Add a confirmation audit trail (timestamp/source/channel + cancellation-request state) to `DentallyAppointment`, a derived display-status function for Daylist to consume later, and fix the `appointment_confirmation` template-type gap — the foundation every later Daylist Confirmations phase depends on.

**Architecture:** Six new nullable, Django-only fields on `DentallyAppointment` (verified safe against the Go sync — Go's raw-SQL upsert never references them, following the existing `confirmation_token`/`quick_note` precedent). A new pure function `compute_confirmation_display_status` in a new small module (matching this app's existing `confirm_utils.py`/`daylist_automation.py` convention of one bespoke module per concern) derives the FR11 display state from these fields plus existing data — never stored, so it can't drift out of sync. The existing public confirm endpoints are extended to populate the new fields.

**Tech Stack:** Django (TreatmentPathBackend), Go (EmailServiceGo, one additive map-literal edit only)

**Spec:** `docs/superpowers/specs/2026-07-02-daylist-confirmations-phase1-data-model-design.md`

---

### Task 1: Model fields + migration

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/models.py:764-770` (`DentallyAppointment`, right after `confirmation_token`)
- Create: `TreatmentPath/dentallyIntegration/migrations/0146_dentallyappointment_confirmation_fields.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing test**

Append to `dentallyIntegration/tests.py` (uses the already-imported `DentallyAppointment`, `Practice`):

```python
class DentallyAppointmentConfirmationFieldsTests(TestCase):
    """DentallyAppointment must have the new confirmation-audit-trail fields,
    all nullable/blank-default so they're safe for Go's raw-SQL upsert to
    never reference (see confirmation_token/quick_note precedent)."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Confirmation Fields Dental")

    def test_new_fields_default_and_are_settable(self):
        appointment = DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=9101,
            dentally_patient_id=301,
            patient_name="Field Test Patient",
            state="pending",
            duration=30,
        )
        self.assertIsNone(appointment.patient_confirmed_at)
        self.assertEqual(appointment.confirmation_source, "")
        self.assertEqual(appointment.confirmation_channel, "")
        self.assertFalse(appointment.cancellation_requested)
        self.assertIsNone(appointment.cancellation_requested_at)
        self.assertEqual(appointment.cancellation_requested_source, "")

        now = timezone.now()
        appointment.patient_confirmed_at = now
        appointment.confirmation_source = "confirm_link"
        appointment.confirmation_channel = "sms"
        appointment.cancellation_requested = True
        appointment.cancellation_requested_at = now
        appointment.cancellation_requested_source = "confirm_link"
        appointment.save()
        appointment.refresh_from_db()

        self.assertEqual(appointment.patient_confirmed_at, now)
        self.assertEqual(appointment.confirmation_source, "confirm_link")
        self.assertEqual(appointment.confirmation_channel, "sms")
        self.assertTrue(appointment.cancellation_requested)
        self.assertEqual(appointment.cancellation_requested_at, now)
        self.assertEqual(appointment.cancellation_requested_source, "confirm_link")
```

- [ ] **Step 2: Run test to verify it fails**

From `TreatmentPath/`, venv active (`source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate`):

```bash
python manage.py test dentallyIntegration.tests.DentallyAppointmentConfirmationFieldsTests --keepdb -v 2
```

Expected: FAIL — the fields don't exist yet (`TypeError` on `objects.create` is not hit since none of the new fields are passed to `create()`; instead expect `AttributeError` when the test tries to set/read `appointment.patient_confirmed_at` etc.).

**IMPORTANT: always use `--keepdb`, never `--noinput`** — a fresh test-DB rebuild is broken in this repo; `--noinput` destroys the persistent test DB.

- [ ] **Step 3: Add the fields to the model**

In `dentallyIntegration/models.py`, right after the existing `confirmation_token` field block (ends around line 770, right before `workflow_sms_sent_at`):

```python
    confirmation_token = models.CharField(
        max_length=12,
        null=True,
        blank=True,
        unique=True,
        help_text="Short random token used in the patient-facing confirmation URL",
    )
    patient_confirmed_at = models.DateTimeField(
        null=True,
        blank=True,
        help_text="When the patient confirmed via the confirmation link. Distinct "
        "from `confirmed_at` above, which is Dentally's own appointment-status "
        "timestamp, not a patient-confirmation event.",
    )
    confirmation_source = models.CharField(
        max_length=30,
        blank=True,
        default="",
        db_default="",
        help_text="How the confirmation/cancellation-request happened, e.g. "
        "'confirm_link' (the public confirm.dental page — the only mechanism "
        "today). Free-form, not a choices field, so a future distinct "
        "mechanism doesn't need a migration.",
    )
    confirmation_channel = models.CharField(
        max_length=10,
        blank=True,
        default="",
        db_default="",
        help_text="'sms' or 'email' — which channel the confirmation link was "
        "sent through. Stamped at send time by AppointmentConfirmLinkViewSet, "
        "not derived at confirm time.",
    )
    cancellation_requested = models.BooleanField(
        default=False,
        db_default=False,
        help_text="Patient requested cancellation via the public confirm page. "
        "v1 records intent only — does not cancel in Dentally.",
    )
    cancellation_requested_at = models.DateTimeField(
        null=True,
        blank=True,
        help_text="When the cancellation-request in cancellation_requested was made.",
    )
    cancellation_requested_source = models.CharField(
        max_length=30,
        blank=True,
        default="",
        db_default="",
        help_text="Same meaning as confirmation_source, for the cancellation-request path.",
    )
    workflow_sms_sent_at = models.DateTimeField(
```

- [ ] **Step 4: Generate and inspect the migration**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py makemigrations dentallyIntegration
```

Expected: creates `dentallyIntegration/migrations/0146_dentallyappointment_confirmation_fields.py` (or a similar auto-generated name — fine to leave as-is) depending on `0145_daylistautomation_template_id`, containing exactly 6 `AddField` operations for `DentallyAppointment`, no unrelated operations swept in. If `makemigrations` pulls in an unrelated pending change (e.g. the known `OpportunityTreatmentMap` index-rename drift from prior sessions), remove that operation from the generated file by hand so only the 6 new fields remain.

- [ ] **Step 5: Apply the migration**

```bash
python manage.py migrate dentallyIntegration
```

Expected: `Applying dentallyIntegration.0146_dentallyappointment_confirmation_fields... OK`

- [ ] **Step 6: Run test to verify it passes**

```bash
python manage.py test dentallyIntegration.tests.DentallyAppointmentConfirmationFieldsTests --keepdb -v 2
```

Expected: PASS (1 test).

---

### Task 2: Display-status derivation function

**Files:**
- Create: `TreatmentPath/dentallyIntegration/confirmation_status.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class ComputeConfirmationDisplayStatusTests(TestCase):
    """compute_confirmation_display_status: one of Cancelled, Cancellation
    requested, Confirmed, No valid contact, Awaiting confirmation — checked
    in that priority order so Dentally's own cancellation always wins."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Display Status Dental")

    def _appointment(self, **overrides):
        defaults = dict(
            practice=self.practice,
            dentally_id=9200,
            dentally_patient_id=401,
            patient_name="Status Test Patient",
            patient_phone="+447700900000",
            patient_email="",
            state="pending",
            duration=30,
        )
        defaults.update(overrides)
        return DentallyAppointment.objects.create(**defaults)

    def test_dentally_cancelled_wins_even_if_confirmed(self):
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment(state="cancelled", patient_confirmed=True)
        self.assertEqual(
            compute_confirmation_display_status(appointment), "Cancelled"
        )

    def test_cancellation_requested(self):
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment(cancellation_requested=True)
        self.assertEqual(
            compute_confirmation_display_status(appointment),
            "Cancellation requested",
        )

    def test_cancellation_requested_wins_over_confirmed(self):
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment(
            cancellation_requested=True, patient_confirmed=True
        )
        self.assertEqual(
            compute_confirmation_display_status(appointment),
            "Cancellation requested",
        )

    def test_confirmed(self):
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment(patient_confirmed=True)
        self.assertEqual(
            compute_confirmation_display_status(appointment), "Confirmed"
        )

    def test_no_valid_contact(self):
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment(patient_phone="", patient_email="")
        self.assertEqual(
            compute_confirmation_display_status(appointment), "No valid contact"
        )

    def test_awaiting_confirmation_is_the_default(self):
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment()
        self.assertEqual(
            compute_confirmation_display_status(appointment),
            "Awaiting confirmation",
        )

    def test_email_only_contact_counts_as_valid(self):
        from .confirmation_status import compute_confirmation_display_status

        appointment = self._appointment(
            patient_phone="", patient_email="patient@example.com"
        )
        self.assertEqual(
            compute_confirmation_display_status(appointment),
            "Awaiting confirmation",
        )
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python manage.py test dentallyIntegration.tests.ComputeConfirmationDisplayStatusTests --keepdb -v 2
```

Expected: FAIL — `ModuleNotFoundError: No module named 'dentallyIntegration.confirmation_status'`.

- [ ] **Step 3: Implement the module**

Create `dentallyIntegration/confirmation_status.py`:

```python
"""Daylist Confirmations — derived (never stored) confirmation display status.

Phase 1 of the Daylist Confirmations PRD (PRD/Daylist/daylist-confirmations-prd.md).
See docs/superpowers/specs/2026-07-02-daylist-confirmations-phase1-data-model-design.md.

`compute_confirmation_display_status` is intentionally NOT a stored field —
it's derived from data that's already source-of-truth elsewhere (Dentally's
own appointment state, the confirmation/cancellation-request fields, contact
info), so it can never drift out of sync with the underlying facts.

A 6th state, "Suppressed" (a confirmation sequence stopped sending to this
appointment), is deferred to Phase 2 — that concept doesn't exist until the
sequence engine ships.
"""


def has_valid_contact(appointment):
    """True if the appointment has a usable phone or email to send a
    confirmation message to. Mirrors the same blank-string-is-invalid
    convention already used inline in daylist_automation.py's send loops."""
    return bool(appointment.patient_phone or appointment.patient_email)


def compute_confirmation_display_status(appointment):
    """One of: 'Cancelled', 'Cancellation requested', 'Confirmed',
    'No valid contact', 'Awaiting confirmation'. Checked in this priority
    order so Dentally's own cancellation status always wins, per the PRD
    edge case: "if Dentally says cancelled, Daylist should show cancelled
    even if Pathway previously had confirmed"."""
    if appointment.state == "cancelled":
        return "Cancelled"
    if appointment.cancellation_requested:
        return "Cancellation requested"
    if appointment.patient_confirmed:
        return "Confirmed"
    if not has_valid_contact(appointment):
        return "No valid contact"
    return "Awaiting confirmation"
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.ComputeConfirmationDisplayStatusTests --keepdb -v 2
```

Expected: PASS (7 tests).

---

### Task 3: Wire the confirm/cancellation-request endpoint

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/views/appointment_confirm_views.py:125-136` (`AppointmentConfirmViewSet.create`)
- Test: `TreatmentPath/dentallyIntegration/tests.py` (extends existing `AppointmentConfirmViewSetTests`)

- [ ] **Step 1: Write the failing tests**

Add these test methods inside the existing `AppointmentConfirmViewSetTests` class in `dentallyIntegration/tests.py` (insert after `test_confirm_is_idempotent`, which ends around line 459):

```python
    def test_confirm_sets_confirmed_at_and_source(self):
        response = self.client.post(
            "/api/backend/dentally/confirm/",
            {"token": self.token},
            format="json",
        )
        self.assertEqual(response.status_code, 200)
        self.appointment.refresh_from_db()
        self.assertIsNotNone(self.appointment.patient_confirmed_at)
        self.assertEqual(self.appointment.confirmation_source, "confirm_link")

    def test_confirm_is_idempotent_on_confirmed_at(self):
        self.client.post(
            "/api/backend/dentally/confirm/",
            {"token": self.token},
            format="json",
        )
        self.appointment.refresh_from_db()
        first_confirmed_at = self.appointment.patient_confirmed_at

        self.client.post(
            "/api/backend/dentally/confirm/",
            {"token": self.token},
            format="json",
        )
        self.appointment.refresh_from_db()
        self.assertEqual(self.appointment.patient_confirmed_at, first_confirmed_at)

    def test_confirm_does_not_touch_confirmation_channel(self):
        self.appointment.confirmation_channel = "sms"
        self.appointment.save(update_fields=["confirmation_channel"])

        self.client.post(
            "/api/backend/dentally/confirm/",
            {"token": self.token},
            format="json",
        )
        self.appointment.refresh_from_db()
        self.assertEqual(self.appointment.confirmation_channel, "sms")

    def test_request_cancellation_sets_fields(self):
        response = self.client.post(
            "/api/backend/dentally/confirm/",
            {"token": self.token, "action": "request_cancellation"},
            format="json",
        )
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.data["cancellation_requested"], True)
        self.appointment.refresh_from_db()
        self.assertTrue(self.appointment.cancellation_requested)
        self.assertIsNotNone(self.appointment.cancellation_requested_at)
        self.assertEqual(self.appointment.cancellation_requested_source, "confirm_link")
        # Requesting cancellation must not also mark the patient confirmed.
        self.assertFalse(self.appointment.patient_confirmed)

    def test_request_cancellation_is_idempotent(self):
        self.client.post(
            "/api/backend/dentally/confirm/",
            {"token": self.token, "action": "request_cancellation"},
            format="json",
        )
        self.appointment.refresh_from_db()
        first_requested_at = self.appointment.cancellation_requested_at

        response = self.client.post(
            "/api/backend/dentally/confirm/",
            {"token": self.token, "action": "request_cancellation"},
            format="json",
        )
        self.assertEqual(response.status_code, 200)
        self.appointment.refresh_from_db()
        self.assertEqual(self.appointment.cancellation_requested_at, first_requested_at)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python manage.py test dentallyIntegration.tests.AppointmentConfirmViewSetTests.test_confirm_sets_confirmed_at_and_source dentallyIntegration.tests.AppointmentConfirmViewSetTests.test_request_cancellation_sets_fields --keepdb -v 2
```

Expected: FAIL — `patient_confirmed_at` stays `None` (not set yet), and `action="request_cancellation"` currently does the exact same thing as no action (sets `patient_confirmed=True`), so `response.data["cancellation_requested"]` (KeyError) and `assertFalse(patient_confirmed)` both fail.

- [ ] **Step 3: Update the view**

Replace `AppointmentConfirmViewSet.create` (currently lines 125-136 of `dentallyIntegration/views/appointment_confirm_views.py`) with:

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
        return Response({"confirmed": True})
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.AppointmentConfirmViewSetTests --keepdb -v 2
```

Expected: PASS (all tests in the class, including the 5 new ones — 11 total).

---

### Task 4: Wire the confirmation-link endpoint to stamp channel

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/views/appointment_confirm_views.py:22-52` (`AppointmentConfirmLinkViewSet.retrieve`)
- Test: `TreatmentPath/dentallyIntegration/tests.py` (extends existing `AppointmentConfirmLinkViewSetTests`)

- [ ] **Step 1: Write the failing tests**

Add these test methods inside the existing `AppointmentConfirmLinkViewSetTests` class (insert after `test_returns_signed_url`, which ends around line 377):

```python
    def test_defaults_channel_to_sms(self):
        self.client.force_authenticate(user=self.user)
        self.client.get(
            f"/api/backend/dentally/appointments/{self.appointment.id}/confirmation-link/"
        )
        self.appointment.refresh_from_db()
        self.assertEqual(self.appointment.confirmation_channel, "sms")

    def test_channel_query_param_is_respected(self):
        self.client.force_authenticate(user=self.user)
        self.client.get(
            f"/api/backend/dentally/appointments/{self.appointment.id}/confirmation-link/?channel=email"
        )
        self.appointment.refresh_from_db()
        self.assertEqual(self.appointment.confirmation_channel, "email")

    def test_existing_token_is_not_regenerated(self):
        self.client.force_authenticate(user=self.user)
        first = self.client.get(
            f"/api/backend/dentally/appointments/{self.appointment.id}/confirmation-link/"
        )
        first_token = self.appointment_token_from_url(first.data["url"])

        second = self.client.get(
            f"/api/backend/dentally/appointments/{self.appointment.id}/confirmation-link/?channel=email"
        )
        second_token = self.appointment_token_from_url(second.data["url"])
        self.assertEqual(first_token, second_token)

    def appointment_token_from_url(self, url):
        return url.rstrip("/").rsplit("/", 1)[-1]
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python manage.py test dentallyIntegration.tests.AppointmentConfirmLinkViewSetTests.test_defaults_channel_to_sms dentallyIntegration.tests.AppointmentConfirmLinkViewSetTests.test_channel_query_param_is_respected --keepdb -v 2
```

Expected: FAIL — `confirmation_channel` stays `""` (never set by the current `retrieve()`).

- [ ] **Step 3: Update the view**

Replace `AppointmentConfirmLinkViewSet.retrieve` (currently lines 22-52 of `dentallyIntegration/views/appointment_confirm_views.py`) with:

```python
    def retrieve(self, request, pk=None):
        from django.conf import settings

        from ..confirm_utils import generate_short_token
        from ..models import DentallyAppointment

        practice = getattr(request.user, "practice", None)
        if not practice:
            return Response(
                {"error": "User not associated with a practice"},
                status=status.HTTP_400_BAD_REQUEST,
            )

        try:
            appointment = DentallyAppointment.objects.get(id=pk, practice=practice)
        except DentallyAppointment.DoesNotExist:
            return Response(
                {"error": "Appointment not found"},
                status=status.HTTP_404_NOT_FOUND,
            )

        channel = request.query_params.get("channel", "sms")
        update_fields = ["confirmation_channel"]
        appointment.confirmation_channel = channel
        if not appointment.confirmation_token:
            appointment.confirmation_token = generate_short_token()
            update_fields.append("confirmation_token")
        appointment.save(update_fields=update_fields)

        confirm_base_url = getattr(
            settings, "CONFIRM_BASE_URL", "https://dev.confirm.dental"
        )
        url = f"{confirm_base_url}/{appointment.confirmation_token}"

        return Response({"url": url, "practice_name": practice.name})
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.AppointmentConfirmLinkViewSetTests --keepdb -v 2
```

Expected: PASS (all tests in the class, including the 3 new ones — 6 total).

---

### Task 5: FR14 fix — `appointment_confirmation` template type

**Files:**
- Modify: `TreatmentPath/messaging/models.py:519-529` (`EmailMessageTemplate.TEMPLATE_TYPES`)
- Modify: `TreatmentPath/messaging/models.py:590-599` (`SMSMessageTemplate.TEMPLATE_TYPES`)
- Create: `TreatmentPath/messaging/migrations/00XX_add_appointment_confirmation_template_type.py` (check the next available number — see Step 3)
- Test: `TreatmentPath/messaging/tests.py`

- [ ] **Step 1: Write the failing test**

Append to `messaging/tests.py` (check the top of that file for existing imports of `EmailMessageTemplate`/`SMSMessageTemplate`/`Practice`/`TestCase` and reuse them; add any that are missing):

```python
class AppointmentConfirmationTemplateTypeTests(TestCase):
    """The 'appointment_confirmation' template_type must exist on both
    template models — the frontend (confirmationActions.ts) already requests
    this exact value and silently falls back to name-matching because it
    doesn't exist yet (FR14 of the Daylist Confirmations PRD)."""

    def test_email_template_accepts_appointment_confirmation_type(self):
        practice = Practice.objects.create(name="Template Type Test Dental")
        template = EmailMessageTemplate.objects.create(
            practice=practice,
            name="Appointment Confirmation Email",
            subject="Please confirm your appointment",
            template_type="appointment_confirmation",
            content="Hi {{ patient_name }}, please confirm.",
        )
        self.assertEqual(template.template_type, "appointment_confirmation")
        self.assertIn(
            "appointment_confirmation",
            dict(EmailMessageTemplate.TEMPLATE_TYPES),
        )

    def test_sms_template_accepts_appointment_confirmation_type(self):
        practice = Practice.objects.create(name="SMS Template Type Test Dental")
        template = SMSMessageTemplate.objects.create(
            practice=practice,
            name="Appointment Confirmation SMS",
            template_type="appointment_confirmation",
            content="Hi {{ patient_name }}, please confirm.",
        )
        self.assertEqual(template.template_type, "appointment_confirmation")
        self.assertIn(
            "appointment_confirmation",
            dict(SMSMessageTemplate.TEMPLATE_TYPES),
        )
```

If `messaging/tests.py` doesn't already import `Practice`, `EmailMessageTemplate`, `SMSMessageTemplate`, or `TestCase` at the top, add:

```python
from django.test import TestCase

from UserAuthentication.models import Practice

from .models import EmailMessageTemplate, SMSMessageTemplate
```

(adjusting for whatever's already imported — don't duplicate existing imports).

- [ ] **Step 2: Run test to verify it fails**

```bash
python manage.py test messaging.tests.AppointmentConfirmationTemplateTypeTests --keepdb -v 2
```

Expected: FAIL — `template.template_type` will still save as `"appointment_confirmation"` (Django `CharField` with `choices` doesn't enforce membership at the DB layer by default), but `self.assertIn("appointment_confirmation", dict(EmailMessageTemplate.TEMPLATE_TYPES))` fails since the choice isn't in the list yet.

- [ ] **Step 3: Add the new choice to both models**

In `messaging/models.py`, update `EmailMessageTemplate.TEMPLATE_TYPES` (currently lines 519-529):

```python
    TEMPLATE_TYPES = [
        ("appointment_reminder", "Appointment Reminder"),
        ("appointment_confirmation", "Appointment Confirmation"),
        ("treatment_plan_followup", "Treatment Plan Follow-up"),
        ("post_treatment_care", "Post-Treatment Care"),
        ("treatment_completion", "Treatment Completion"),
        ("daylist", "Day List"),
        ("consent", "Consent"),
        ("aftercare", "Aftercare"),
        ("recall", "Recall"),
        ("custom", "Custom"),
    ]
```

And `SMSMessageTemplate.TEMPLATE_TYPES` (currently lines 590-599):

```python
    TEMPLATE_TYPES = [
        ("appointment_reminder", "Appointment Reminder"),
        ("appointment_confirmation", "Appointment Confirmation"),
        ("treatment_plan_followup", "Treatment Plan Follow-up"),
        ("post_treatment_care", "Post-Treatment Care"),
        ("treatment_completion", "Treatment Completion"),
        ("treatment_plan_acceptance", "Treatment Plan Acceptance"),
        ("daylist", "Day List"),
        ("recall", "Recall"),
        ("custom", "Custom"),
    ]
```

- [ ] **Step 4: Generate and apply the migration**

Django's `choices` on an existing `CharField` doesn't require a schema migration (no column change), but Django still generates a no-op migration to record the change to model state — this is expected and required so `makemigrations --check` stays clean.

```bash
python manage.py makemigrations messaging
python manage.py migrate messaging
```

Expected: a new migration file with an `AlterField` operation on `template_type` for both models (choices list only, no column type/constraint change), applied cleanly.

- [ ] **Step 5: Run test to verify it passes**

```bash
python manage.py test messaging.tests.AppointmentConfirmationTemplateTypeTests --keepdb -v 2
```

Expected: PASS (2 tests).

---

### Task 6: Go snapshot-guard housekeeping

**Files:**
- Modify: `EmailServiceGo/internal/db/snapshotguard.go:71-76`

- [ ] **Step 1: Add the 6 new field names to `mirrorExcludedColumns`**

Current code (`snapshotguard.go:71-76`):

```go
var mirrorExcludedColumns = map[string]map[string]bool{
	// quick_note is Django-owned (free-text staff note), never part of any Dentally
	// sync payload, so the snapshot mirror can never populate it — exclude it from
	// the "missing from mirror" check.
	"dentally_appointment": {"quick_note": true},
}
```

Replace with:

```go
var mirrorExcludedColumns = map[string]map[string]bool{
	// quick_note is Django-owned (free-text staff note), never part of any Dentally
	// sync payload, so the snapshot mirror can never populate it — exclude it from
	// the "missing from mirror" check. The 6 confirmation-audit-trail fields below
	// are the same: Django-only (Daylist Confirmations Phase 1), never written by
	// the Go sync, so the mirror can't populate them either.
	"dentally_appointment": {
		"quick_note":                     true,
		"patient_confirmed_at":           true,
		"confirmation_source":            true,
		"confirmation_channel":           true,
		"cancellation_requested":         true,
		"cancellation_requested_at":      true,
		"cancellation_requested_source":  true,
	},
}
```

- [ ] **Step 2: Run the Go test suite for this package**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
go test ./internal/db/... -v
```

Expected: PASS, no new failures. (If pre-existing unrelated failures exist in this package from before this change, confirm via `git stash` that they're not caused by this edit, same as the backend testing convention established in prior sessions.)

- [ ] **Step 3: Format the Go file**

```bash
gofmt -l internal/db/snapshotguard.go
```

Expected: no output (already correctly formatted). If it lists the file, run `gofmt -w internal/db/snapshotguard.go` to fix alignment.

---

## Summary of spec coverage

- Go-sync safety confirmed before any code was written (prior session, informs this whole plan) → no task needed, already verified.
- 6 new `DentallyAppointment` fields → Task 1.
- `compute_confirmation_display_status` + `has_valid_contact` derivation → Task 2.
- Wiring `AppointmentConfirmViewSet.create` (confirm + request_cancellation actions, idempotent, doesn't touch `confirmation_channel`) → Task 3.
- Wiring `AppointmentConfirmLinkViewSet.retrieve` (channel stamping at send time, default `sms`, doesn't regenerate an existing token) → Task 4.
- FR14 `appointment_confirmation` template type fix → Task 5.
- `snapshotguard.go` housekeeping → Task 6.
- Out-of-scope items from the spec (Suppressed state, API/serializer exposure, UI, sequence engine, cohort engine, follow-up tasks) — correctly not covered by any task here; they belong to Phases 2-5.

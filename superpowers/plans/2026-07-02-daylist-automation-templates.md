# Daylist Automation Template Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a `DayListAutomation` reference a reusable `EmailMessageTemplate`/`SMSMessageTemplate` (`template_type='daylist'`), with custom free text still overriding the template when present — mirroring how `RecallSequence` resolves templates.

**Architecture:** Add a plain `template_id` IntegerField to `DayListAutomation` (no FK, matches Recall's own convention — the model already knows which template table to check via its `channel` field). A new `_resolve_daylist_template` + `_resolve_content` pair in `daylist_automation.py` implements "custom body wins, else resolve template" and is called once at the top of both `run_daylist_automation` and `test_send_daylist_automation`, replacing their direct `automation.message_body`/`message_subject` reads. The existing flat placeholder context in `render_message` is unchanged. Frontend adds a template `<Select>` (fetching `templateType='daylist'` templates) above the existing message fields, relabeled as an optional override.

**Tech Stack:** Django (TreatmentPathBackend), React/TypeScript (perfect-pixel-playground-project)

**Spec:** `docs/superpowers/specs/2026-07-02-daylist-automation-templates-design.md`

---

### Task 1: Model field + migration

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/models.py:3240-3299` (`DayListAutomation`)
- Create: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/migrations/0145_daylistautomation_template_id.py`
- Test: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing test**

Add to `dentallyIntegration/tests.py` (new imports go in the existing `from .models import (...)` block at the top — add `DayListAutomation` to that import list, alphabetically):

```python
from .models import (
    DayListAutomation,
    DentallyAppointment,
    RecallAppointment,
    RecallPatient,
    RecallPayment,
    RecallRecord,
    RecallSandboxConfig,
    RecallSegmentConfig,
    RecallSequence,
    RecallSequenceEnrollment,
    RecallSpendConfig,
)
```

Then append this test class at the end of the file:

```python
class DayListAutomationTemplateFieldTests(TestCase):
    """DayListAutomation must have a plain (non-FK) template_id column, so an
    automation can reference an EmailMessageTemplate/SMSMessageTemplate by id
    without needing two separate nullable FK columns."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Template Field Dental")

    def test_template_id_defaults_to_none_and_is_settable(self):
        automation = DayListAutomation.objects.create(
            practice=self.practice,
            name="Field test",
            channel="sms",
            message_body="Hi {{ patient_name }}",
        )
        self.assertIsNone(automation.template_id)

        automation.template_id = 42
        automation.save(update_fields=["template_id"])
        automation.refresh_from_db()
        self.assertEqual(automation.template_id, 42)
```

- [ ] **Step 2: Run test to verify it fails**

Run (from `TreatmentPathBackend/TreatmentPath/`, with the venv active per project convention:
`source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate`):

```bash
python manage.py test dentallyIntegration.tests.DayListAutomationTemplateFieldTests --keepdb -v 2
```

Expected: FAIL — `TypeError: DayListAutomation() got unexpected keyword arguments: 'template_id'` (field doesn't exist yet), or an `AttributeError` on `automation.template_id`.

**IMPORTANT: always use `--keepdb`, never `--noinput`** — a fresh test-DB rebuild is broken in this repo; `--noinput` destroys the persistent test DB.

- [ ] **Step 3: Add the field to the model**

In `dentallyIntegration/models.py`, inside `DayListAutomation` (after `message_body`, before `created_at`, i.e. right after line 3288 `message_body = models.TextField(blank=True, default="")`):

```python
    message_body = models.TextField(blank=True, default="")
    template_id = models.IntegerField(
        null=True,
        blank=True,
        help_text="Optional id of an active EmailMessageTemplate or "
        "SMSMessageTemplate (template_type='daylist'), matched to `channel`. "
        "Not a FK — mirrors RecallSequence's own template_id convention, since "
        "which table to check depends on `channel`. When message_body is "
        "non-blank it always overrides the template; see "
        "daylist_automation._resolve_content.",
    )
    created_at = models.DateTimeField(auto_now_add=True)
```

Also update the model's class docstring (lines 3241-3254) to mention the new field — replace the existing docstring with:

```python
    """A scheduled day-list outreach automation (bespoke, Recall-sequence-style —
    own model, own simple admin form — NOT the generic Workflow/canvas engine).

    On a schedule (`run_every_minutes`), finds at-risk appointments for the
    practice (reusing the existing Go no-show risk query via the internal
    `/api/internal/daylist/at-risk-appointments` endpoint — the risk scoring
    itself is never duplicated here) and sends one message per matching
    appointment, rendered from `message_body` (or `template_id` when
    `message_body` is blank — see `daylist_automation._resolve_content`).

    `message_body`/resolved template placeholders (rendered per-appointment,
    see `daylist_automation.render_message`):
      {{patient_name}} {{appointment_date}} {{appointment_time}} {{practitioner_name}}
      {{practice_name}} {{confirmation_url}} {{risk_tier}}
    """
```

- [ ] **Step 4: Generate and inspect the migration**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py makemigrations dentallyIntegration
```

Expected output: creates `dentallyIntegration/migrations/0145_daylistautomation_template_id.py`. Open it and confirm it contains a single `AddField` operation for `template_id` on `DayListAutomation`, depending on `0144_dentallypatientaisummary_raw_notes`. It should look like:

```python
from django.db import migrations, models


class Migration(migrations.Migration):

    dependencies = [
        ("dentallyIntegration", "0144_dentallypatientaisummary_raw_notes"),
    ]

    operations = [
        migrations.AddField(
            model_name="daylistautomation",
            name="template_id",
            field=models.IntegerField(
                blank=True,
                help_text="Optional id of an active EmailMessageTemplate or SMSMessageTemplate (template_type='daylist'), matched to `channel`. Not a FK — mirrors RecallSequence's own template_id convention, since which table to check depends on `channel`. When message_body is non-blank it always overrides the template; see daylist_automation._resolve_content.",
                null=True,
            ),
        ),
    ]
```

If Django names the migration file differently (e.g. `0145_daylistautomation_template_id_and_more`), that's fine — leave the auto-generated name as-is.

- [ ] **Step 5: Apply the migration**

```bash
python manage.py migrate dentallyIntegration
```

Expected: `Applying dentallyIntegration.0145_daylistautomation_template_id... OK`

- [ ] **Step 6: Run test to verify it passes**

```bash
python manage.py test dentallyIntegration.tests.DayListAutomationTemplateFieldTests --keepdb -v 2
```

Expected: PASS (1 test).

- [ ] **Step 7: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/dentallyIntegration/models.py \
        TreatmentPathBackend/TreatmentPath/dentallyIntegration/migrations/0145_daylistautomation_template_id.py \
        TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests.py
git commit -m "feat(daylist): add template_id field to DayListAutomation"
```

---

### Task 2: Template resolution in the run engine

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/daylist_automation.py`
- Test: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class DayListAutomationResolveContentTests(TestCase):
    """_resolve_content: custom message_body always wins over template_id;
    falls back to resolving an active EmailMessageTemplate/SMSMessageTemplate
    (template_type='daylist') when message_body is blank; returns ("", "")
    when neither is usable — mirrors recall_automation._resolve_template's
    "never raise" contract."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Resolve Content Dental")

    def test_custom_body_overrides_template(self):
        from messaging.models import SMSMessageTemplate

        from .daylist_automation import _resolve_content

        tmpl = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="Daylist SMS",
            template_type="daylist",
            content="Templated: hi {{ patient_name }}",
            is_active=True,
        )
        automation = DayListAutomation.objects.create(
            practice=self.practice,
            name="Custom wins",
            channel="sms",
            message_body="Custom: hi {{ patient_name }}",
            template_id=tmpl.id,
        )
        subject, body = _resolve_content(automation)
        self.assertEqual(body, "Custom: hi {{ patient_name }}")
        self.assertEqual(subject, "")

    def test_falls_back_to_sms_template(self):
        from messaging.models import SMSMessageTemplate

        from .daylist_automation import _resolve_content

        tmpl = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="Daylist SMS",
            template_type="daylist",
            content="Templated: hi {{ patient_name }}",
            is_active=True,
        )
        automation = DayListAutomation.objects.create(
            practice=self.practice,
            name="Template fallback",
            channel="sms",
            message_body="",
            template_id=tmpl.id,
        )
        subject, body = _resolve_content(automation)
        self.assertEqual(body, "Templated: hi {{ patient_name }}")

    def test_falls_back_to_email_template_with_subject(self):
        from messaging.models import EmailMessageTemplate

        from .daylist_automation import _resolve_content

        tmpl = EmailMessageTemplate.objects.create(
            practice=self.practice,
            name="Daylist Email",
            template_type="daylist",
            subject="Your appointment",
            content="Templated body for {{ patient_name }}",
            is_active=True,
        )
        automation = DayListAutomation.objects.create(
            practice=self.practice,
            name="Email template fallback",
            channel="email",
            message_body="",
            message_subject="",
            template_id=tmpl.id,
        )
        subject, body = _resolve_content(automation)
        self.assertEqual(subject, "Your appointment")
        self.assertEqual(body, "Templated body for {{ patient_name }}")

    def test_inactive_template_and_no_custom_body_returns_empty(self):
        from messaging.models import SMSMessageTemplate

        from .daylist_automation import _resolve_content

        tmpl = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="Inactive",
            template_type="daylist",
            content="Should not be used",
            is_active=False,
        )
        automation = DayListAutomation.objects.create(
            practice=self.practice,
            name="Inactive template",
            channel="sms",
            message_body="",
            template_id=tmpl.id,
        )
        subject, body = _resolve_content(automation)
        self.assertEqual((subject, body), ("", ""))

    def test_no_body_and_no_template_id_returns_empty(self):
        from .daylist_automation import _resolve_content

        automation = DayListAutomation.objects.create(
            practice=self.practice,
            name="Nothing set",
            channel="sms",
            message_body="",
            template_id=None,
        )
        subject, body = _resolve_content(automation)
        self.assertEqual((subject, body), ("", ""))
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python manage.py test dentallyIntegration.tests.DayListAutomationResolveContentTests --keepdb -v 2
```

Expected: FAIL — `ImportError: cannot import name '_resolve_content' from 'dentallyIntegration.daylist_automation'`.

- [ ] **Step 3: Implement `_resolve_daylist_template` and `_resolve_content`**

In `dentallyIntegration/daylist_automation.py`, add these two functions right after `render_message` (i.e. after line 81, before `def send_sms`):

```python
def _resolve_daylist_template(practice, channel, template_id):
    """(subject, body) for an active daylist template, scoped to `practice` +
    `is_active=True`. Mirrors recall_automation._resolve_template exactly —
    returns (None, None) when missing/inactive, never raises, so a bad
    template_id degrades to "no content" rather than crashing a scheduled
    run for other practices."""
    from messaging.models import EmailMessageTemplate, SMSMessageTemplate

    if channel == "email":
        t = EmailMessageTemplate.objects.filter(
            id=template_id, practice=practice, is_active=True
        ).first()
        if not t:
            return None, None
        return (t.subject or ""), (t.content or "")
    t = SMSMessageTemplate.objects.filter(
        id=template_id, practice=practice, is_active=True
    ).first()
    if not t:
        return None, None
    return "", (t.content or "")


def _resolve_content(automation):
    """(subject, body) template strings (pre-Django-Template-rendering) for one
    automation. Custom `message_body` text always wins when non-blank —
    matches Recall's `custom_body or _resolve_template(...)` rule. Falls back
    to `template_id` when set; returns ("", "") when neither yields usable
    content (caller treats that the same as a missing phone/email — skip and
    count as failed)."""
    if (automation.message_body or "").strip():
        return automation.message_subject or "", automation.message_body
    if automation.template_id:
        subject, body = _resolve_daylist_template(
            automation.practice, automation.channel, automation.template_id
        )
        if body:
            return subject or "", body
    return "", ""
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.DayListAutomationResolveContentTests --keepdb -v 2
```

Expected: PASS (5 tests).

- [ ] **Step 5: Wire `_resolve_content` into `run_daylist_automation` and `test_send_daylist_automation`**

In `dentallyIntegration/daylist_automation.py`, replace the body of `run_daylist_automation` (currently lines 161-198) with:

```python
def run_daylist_automation(automation):
    """Runs one automation for real: queries at-risk appointments and sends a
    message to EACH matching patient. Updates last_run_at regardless of
    outcome, so a failing automation doesn't retry every Beat tick."""
    practice = automation.practice
    risk_tiers = automation.risk_tiers or ["High"]
    appointments = fetch_at_risk_appointments(
        practice, automation.days_ahead, risk_tiers
    )

    subject_template, body_template = _resolve_content(automation)

    sent, failed = 0, 0
    for appt in appointments:
        if not body_template:
            failed += 1
            continue
        body = render_message(body_template, appt, practice)
        if automation.channel == "sms":
            phone = appt.get("patient_phone", "")
            if not phone:
                failed += 1
                continue
            country = appt.get("patient_phone_country_code", "") or ""
            full_phone = f"{country}{phone}" if not phone.startswith("+") else phone
            ok, _ = send_sms(practice, full_phone, body)
        else:
            email = appt.get("patient_email", "")
            if not email:
                failed += 1
                continue
            subject = (
                render_message(subject_template, appt, practice)
                if subject_template
                else "Appointment reminder"
            )
            ok, _ = send_email(practice, email, subject, body)
        sent += 1 if ok else 0
        failed += 0 if ok else 1

    automation.last_run_at = timezone.now()
    automation.save(update_fields=["last_run_at"])
    return {"found": len(appointments), "sent": sent, "failed": failed}
```

Then replace the body of `test_send_daylist_automation` (currently lines 201-243) with:

```python
def test_send_daylist_automation(automation, destination):
    """Safe test: runs the REAL at-risk query (read-only) but sends exactly
    ONE message, to `destination` only (a phone number for SMS automations, an
    email address for email automations) — never to a real patient's contact
    info, even though the query itself returns real patient data. If a real
    appointment matches, its data renders the message (realistic test); if
    none match, a synthetic placeholder appointment is used instead so the
    template can still be verified."""
    practice = automation.practice
    risk_tiers = automation.risk_tiers or ["High"]
    appointments = fetch_at_risk_appointments(
        practice, automation.days_ahead, risk_tiers
    )

    if appointments:
        sample = appointments[0]
    else:
        sample = {
            "patient_name": "Test Patient",
            "appointment_date": timezone.now().date().isoformat(),
            "appointment_time": "10:00",
            "practitioner_name": "Dr. Example",
            "confirmation_url": "https://example.com/confirm/test",
            "risk_tier": "High",
        }

    subject_template, body_template = _resolve_content(automation)
    if not body_template:
        return {
            "found": len(appointments),
            "used_synthetic_sample": not appointments,
            "sent_ok": False,
            "detail": "No message content: message_body is blank and template_id "
            "did not resolve to an active template.",
            "rendered_message": "",
        }

    body = render_message(body_template, sample, practice)
    if automation.channel == "sms":
        ok, detail = send_sms(practice, destination, body)
    else:
        subject = (
            render_message(subject_template, sample, practice)
            if subject_template
            else "Appointment reminder"
        )
        ok, detail = send_email(practice, destination, subject, body)
    return {
        "found": len(appointments),
        "used_synthetic_sample": not appointments,
        "sent_ok": ok,
        "detail": detail,
        "rendered_message": body,
    }
```

- [ ] **Step 6: Write tests for the wired-up behavior**

Append to `dentallyIntegration/tests.py`:

```python
class DayListAutomationTemplateSendTests(TestCase):
    """run_daylist_automation and test_send_daylist_automation must resolve
    template_id into real message content when message_body is blank."""

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Template Send Dental", twilio_phone_number="+447700900000"
        )

    @override_settings(TWILIO_ACCOUNT_SID="ACtest", TWILIO_AUTH_TOKEN="tok")
    def test_test_send_uses_template_when_body_blank(self):
        from messaging.models import SMSMessageTemplate

        from .daylist_automation import test_send_daylist_automation

        tmpl = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="Daylist SMS",
            template_type="daylist",
            content="Hi {{ patient_name }}, templated reminder.",
            is_active=True,
        )
        automation = DayListAutomation.objects.create(
            practice=self.practice,
            name="Template test-send",
            channel="sms",
            message_body="",
            template_id=tmpl.id,
        )

        with patch(
            "dentallyIntegration.daylist_automation.fetch_at_risk_appointments",
            return_value=[],
        ), patch("twilio.rest.Client") as MockTwilio:
            MockTwilio.return_value.messages.create.return_value.sid = "SMfake"
            result = test_send_daylist_automation(automation, "+447700900111")

        self.assertTrue(result["sent_ok"])
        self.assertIn("templated reminder", result["rendered_message"])

    def test_test_send_reports_no_content_when_body_blank_and_no_template(self):
        from .daylist_automation import test_send_daylist_automation

        automation = DayListAutomation.objects.create(
            practice=self.practice,
            name="No content",
            channel="sms",
            message_body="",
            template_id=None,
        )

        with patch(
            "dentallyIntegration.daylist_automation.fetch_at_risk_appointments",
            return_value=[],
        ):
            result = test_send_daylist_automation(automation, "+447700900111")

        self.assertFalse(result["sent_ok"])
        self.assertIn("No message content", result["detail"])
```

- [ ] **Step 7: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.DayListAutomationTemplateSendTests --keepdb -v 2
```

Expected: PASS (2 tests).

- [ ] **Step 8: Run the full Task 1+2 test coverage together**

```bash
python manage.py test dentallyIntegration.tests.DayListAutomationTemplateFieldTests dentallyIntegration.tests.DayListAutomationResolveContentTests dentallyIntegration.tests.DayListAutomationTemplateSendTests --keepdb -v 2
```

Expected: PASS (8 tests total).

- [ ] **Step 9: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/dentallyIntegration/daylist_automation.py \
        TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests.py
git commit -m "feat(daylist): resolve template_id into message content at send time"
```

---

### Task 3: Serializer support

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/serializers.py:1507-1555`
- Test: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class DayListAutomationSerializerTemplateTests(TestCase):
    """DayListAutomationSerializer must accept/expose template_id, and
    validate() must require EITHER a non-blank message_body OR a template_id
    (previously message_body alone was mandatory)."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Serializer Template Dental")

    def test_template_id_only_is_valid_for_sms(self):
        from .serializers import DayListAutomationSerializer

        serializer = DayListAutomationSerializer(
            data={
                "name": "Template only",
                "channel": "sms",
                "message_body": "",
                "template_id": 99,
                "risk_tiers": ["High"],
            }
        )
        self.assertTrue(serializer.is_valid(), serializer.errors)

    def test_neither_body_nor_template_id_is_invalid(self):
        from .serializers import DayListAutomationSerializer

        serializer = DayListAutomationSerializer(
            data={
                "name": "Nothing set",
                "channel": "sms",
                "message_body": "",
                "risk_tiers": ["High"],
            }
        )
        self.assertFalse(serializer.is_valid())
        self.assertIn("message_body", serializer.errors)

    def test_template_id_round_trips_on_save(self):
        from .serializers import DayListAutomationSerializer

        serializer = DayListAutomationSerializer(
            data={
                "name": "Round trip",
                "channel": "sms",
                "message_body": "",
                "template_id": 7,
                "risk_tiers": ["High"],
            }
        )
        self.assertTrue(serializer.is_valid(), serializer.errors)
        automation = serializer.save(practice=self.practice)
        self.assertEqual(automation.template_id, 7)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python manage.py test dentallyIntegration.tests.DayListAutomationSerializerTemplateTests --keepdb -v 2
```

Expected: FAIL on `test_template_id_only_is_valid_for_sms` and `test_template_id_round_trips_on_save` — `validate()` currently raises "Message body is required." even with `template_id` set. `test_neither_body_nor_template_id_is_invalid` should already pass (no code change needed for that one, but it documents/pins existing behavior).

- [ ] **Step 3: Update the serializer**

In `dentallyIntegration/serializers.py`, replace the `DayListAutomationSerializer` class (currently lines 1507-1555) with:

```python
class DayListAutomationSerializer(serializers.ModelSerializer):
    """A scheduled day-list outreach automation (bespoke, Recall-sequence-style
    — see dentallyIntegration.daylist_automation for the run engine)."""

    class Meta:
        model = DayListAutomation
        fields = [
            "id",
            "name",
            "is_active",
            "run_every_minutes",
            "last_run_at",
            "days_ahead",
            "risk_tiers",
            "channel",
            "template_id",
            "message_subject",
            "message_body",
            "created_at",
            "updated_at",
        ]
        read_only_fields = ["id", "last_run_at", "created_at", "updated_at"]

    def validate_risk_tiers(self, value):
        allowed = {"High", "Elevated", "Moderate", "Low", "Very Low"}
        if not isinstance(value, list):
            raise serializers.ValidationError("risk_tiers must be a list.")
        for tier in value:
            if tier not in allowed:
                raise serializers.ValidationError(f"Unknown risk tier '{tier}'.")
        return value

    def validate(self, attrs):
        channel = attrs.get("channel", getattr(self.instance, "channel", "sms"))
        message_body = attrs.get(
            "message_body", getattr(self.instance, "message_body", "")
        )
        template_id = attrs.get(
            "template_id", getattr(self.instance, "template_id", None)
        )
        if not (message_body or "").strip() and not template_id:
            raise serializers.ValidationError(
                {"message_body": "Either a message body or a template is required."}
            )
        if channel == "email":
            subject = attrs.get(
                "message_subject", getattr(self.instance, "message_subject", "")
            )
            # A template supplies its own subject at send time (see
            # daylist_automation._resolve_content) — only require a subject
            # here when there's custom body text and no template to fall
            # back on.
            if (message_body or "").strip() and not template_id and not (subject or "").strip():
                raise serializers.ValidationError(
                    {"message_subject": "Email automations need a subject."}
                )
        return attrs
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.DayListAutomationSerializerTemplateTests --keepdb -v 2
```

Expected: PASS (3 tests).

- [ ] **Step 5: Run the existing daylist automation view tests to check for regressions**

```bash
python manage.py test dentallyIntegration.tests -k Daylist --keepdb -v 2
```

(If `-k` isn't supported by this Django test runner version, instead run:)

```bash
python manage.py test dentallyIntegration.tests.DayListAutomationTemplateFieldTests dentallyIntegration.tests.DayListAutomationResolveContentTests dentallyIntegration.tests.DayListAutomationTemplateSendTests dentallyIntegration.tests.DayListAutomationSerializerTemplateTests --keepdb -v 2
```

Expected: PASS (11 tests total, no regressions).

- [ ] **Step 6: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/dentallyIntegration/serializers.py \
        TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests.py
git commit -m "feat(daylist): serializer accepts template_id, require body-or-template"
```

---

### Task 4: Frontend types + template fetch

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/day-list/components/administration/DayListAutomationSection.tsx`

- [ ] **Step 1: Add `template_id` to the `DayListAutomation` interface**

In `DayListAutomationSection.tsx`, replace the interface (currently lines 119-130):

```ts
interface DayListAutomation {
  id: number;
  name: string;
  is_active: boolean;
  run_every_minutes: number;
  last_run_at: string | null;
  days_ahead: number;
  risk_tiers: string[];
  channel: 'sms' | 'email';
  template_id: number | null;
  message_subject: string;
  message_body: string;
}

interface Tmpl {
  id: number;
  name: string;
}

const asTmplList = (d: unknown): Tmpl[] => {
  if (Array.isArray(d)) return d as Tmpl[];
  const o = d as { results?: Tmpl[]; templates?: Tmpl[] };
  return o?.results ?? o?.templates ?? [];
};
```

(`Tmpl`/`asTmplList` mirror `AutomationSection.tsx`'s own `Tmpl`/`asList` — kept local here rather than imported, since these two admin sections don't currently share a components module and this plan doesn't introduce one.)

- [ ] **Step 2: Add `template_id: null` to `blankAutomation()`**

Replace the current `blankAutomation` (lines 143-155):

```ts
const blankAutomation = (): Omit<DayListAutomation, 'id' | 'last_run_at'> => ({
  name: '',
  // Off by default — an automation only starts messaging real patients once
  // an admin deliberately flips it on, after reviewing it (and ideally test-
  // sending it first). Never opt a brand-new automation into live sends.
  is_active: false,
  run_every_minutes: 1440,
  days_ahead: 3,
  risk_tiers: ['High'],
  channel: 'sms',
  template_id: null,
  message_subject: '',
  message_body: '',
});
```

- [ ] **Step 3: Fetch active daylist templates in the component**

Inside `export function DayListAutomationSection() {`, add new state right after the existing `useState` declarations (after line 170, `const subjectRef = useRef<HTMLInputElement>(null);`):

```ts
  const [emailTemplates, setEmailTemplates] = useState<Tmpl[]>([]);
  const [smsTemplates, setSmsTemplates] = useState<Tmpl[]>([]);
```

Then add a new `useEffect` right after the existing one that calls `load()` (after lines 183-185):

```ts
  useEffect(() => {
    (async () => {
      try {
        const [eRes, sRes] = await Promise.all([
          fetchWithAuth(API_ENDPOINTS.messages.fetchActiveEmailTemplates({ templateType: 'daylist' })),
          fetchWithAuth(API_ENDPOINTS.messages.fetchActiveSmsTemplates({ templateType: 'daylist' })),
        ]);
        if (eRes.ok) setEmailTemplates(asTmplList(await eRes.json()));
        if (sRes.ok) setSmsTemplates(asTmplList(await sRes.json()));
      } catch {
        /* ignore */
      }
    })();
  }, [fetchWithAuth]);
```

- [ ] **Step 4: Include `template_id` in the save payload**

In the `save` function, the `body: JSON.stringify({...})` block (currently lines 241-250) — add `template_id`:

```ts
        body: JSON.stringify({
          name: editing.name.trim(),
          is_active: editing.is_active,
          run_every_minutes: editing.run_every_minutes,
          days_ahead: editing.days_ahead,
          risk_tiers: editing.risk_tiers,
          channel: editing.channel,
          template_id: editing.template_id,
          message_subject: editing.message_subject,
          message_body: editing.message_body,
        }),
```

- [ ] **Step 5: Relax the client-side "message required" validation**

The current `save()` guard (line 229) — `if (!editing.message_body.trim()) return toast.error('Add a message');` — blocks saving a template-only automation. Replace lines 228-230:

```ts
    if (!editing.name.trim()) return toast.error('Name the automation');
    if (!editing.message_body.trim() && !editing.template_id) return toast.error('Add a message or select a template');
    if (editing.channel === 'email' && !editing.message_body.trim() && !editing.template_id && !editing.message_subject.trim()) return toast.error('Add a subject');
```

- [ ] **Step 6: Manually verify the fetch wiring**

Run the frontend dev server and confirm no TypeScript errors:

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit
```

Expected: no new type errors from `DayListAutomationSection.tsx` (pre-existing unrelated errors elsewhere, if any, are not this task's concern).

- [ ] **Step 7: Commit**

```bash
git add perfect-pixel-playground-project/src/pages/day-list/components/administration/DayListAutomationSection.tsx
git commit -m "feat(daylist): add template_id to automation form state and fetch daylist templates"
```

---

### Task 5: Frontend UI — template picker + relabeled custom copy

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/day-list/components/administration/DayListAutomationSection.tsx`

- [ ] **Step 1: Add a template `<Select>` above the message fields**

Insert a new block right before the `{editing.channel === 'email' && (` subject block (i.e. right after the "Send via" channel `<Select>` block that ends at line 400, before line 402):

```tsx
        <div className="space-y-1 max-w-md">
          <Label className="text-xs text-gray-500">Template</Label>
          <Select
            value={editing.template_id ? String(editing.template_id) : ''}
            onValueChange={(v) => setEditing({ ...editing, template_id: v ? Number(v) : null })}
          >
            <SelectTrigger className="bg-white">
              <SelectValue
                placeholder={
                  (editing.channel === 'email' ? emailTemplates : smsTemplates).length
                    ? 'Select a template (optional)'
                    : 'No day-list templates yet'
                }
              />
            </SelectTrigger>
            <SelectContent>
              {(editing.channel === 'email' ? emailTemplates : smsTemplates).map((t) => (
                <SelectItem key={t.id} value={String(t.id)}>
                  {t.name}
                </SelectItem>
              ))}
            </SelectContent>
          </Select>
          <p className="text-xs text-gray-400">
            Custom copy below overrides the template when filled in.
          </p>
        </div>
```

- [ ] **Step 2: Relabel the subject field's section as an override**

Replace the current subject block (lines 402-418):

```tsx
        {editing.channel === 'email' && (
          <div className="space-y-2">
            <Label className="text-xs text-gray-500">Custom subject (optional — overrides the template)</Label>
            <div className="relative max-w-md">
              <Input
                ref={subjectRef}
                value={editing.message_subject}
                onChange={(e) => setEditing({ ...editing, message_subject: e.target.value })}
                placeholder="Your appointment on {{ appointment_date }}"
                className="bg-white pr-10"
              />
              <div className="absolute inset-y-0 right-1 flex items-center">
                <PlaceholderPicker onInsert={insertSubjectPlaceholder} />
              </div>
            </div>
          </div>
        )}
```

- [ ] **Step 3: Relabel the message body section as an override**

Replace the current message block (lines 420-434):

```tsx
        <div className="space-y-2">
          <Label className="text-xs text-gray-500">
            Custom copy ({editing.channel === 'email' ? 'Email' : 'SMS'}, optional — overrides the template)
          </Label>
          <div className="relative">
            <Textarea
              ref={messageRef}
              value={editing.message_body}
              onChange={(e) => setEditing({ ...editing, message_body: e.target.value })}
              placeholder="Hi {{ patient_name }}, this is a reminder about your appointment on {{ appointment_date }} at {{ appointment_time }}."
              className="bg-white min-h-[100px] pr-10"
            />
            <div className="absolute bottom-2 right-2">
              <PlaceholderPicker onInsert={insertPlaceholder} />
            </div>
          </div>
        </div>
```

- [ ] **Step 4: Manually verify in the browser**

Start the frontend dev server (or use whatever `run`-skill/dev-server convention this project uses) and navigate to the Day List administration page → Automation tab:

1. Create a new automation, leave "Custom copy" blank, pick a template from the "Template" dropdown (if none exist yet, first go to Settings > Templates and create one with type "Day List" for SMS or Email).
2. Save — should succeed (previously blocked by "Add a message").
3. Click "Send test" with a destination — the response toast should reflect a successful send whose `rendered_message` comes from the template content.
4. Type something into "Custom copy" on the same automation, save, then "Send test" again — confirm the custom text is what gets sent (overriding the template).

Expected: all four steps behave as described, no console errors.

- [ ] **Step 5: Commit**

```bash
git add perfect-pixel-playground-project/src/pages/day-list/components/administration/DayListAutomationSection.tsx
git commit -m "feat(daylist): template picker UI with custom-copy override in automation form"
```

---

## Summary of spec coverage

- Backend `template_id` field + migration → Task 1
- `_resolve_daylist_template` / `_resolve_content` resolution rule (custom body wins, else template) → Task 2
- Wiring into `run_daylist_automation` and `test_send_daylist_automation`, unchanged placeholder context → Task 2
- Serializer accepts `template_id`, validation requires body-or-template → Task 3
- Frontend types + template fetch (`templateType='daylist'`) → Task 4
- Frontend template `<Select>` + relabeled custom-copy override, matching Recall's UX → Task 5
- Out of scope (per spec): no changes to template CRUD/Settings UI, no change to `render_message`'s placeholder scheme, no multi-step sequence support — none of the tasks above touch those areas.

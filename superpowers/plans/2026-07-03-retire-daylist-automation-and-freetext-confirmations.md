# Retire DayListAutomation + Free-Text/Placeholder Override for ConfirmationSequence — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Retire the `DayListAutomation` model/UI entirely (pre-launch, no production data), and bring its one missing capability — free-text message override with a placeholder picker — forward into `ConfirmationSequence`, matching Recall's existing `AutomationSection.tsx` pattern exactly.

**Architecture:** `ConfirmationSequence.steps[]` entries gain optional `subject`/`body` keys (body-truthy wins over `template_id`, at both validation and send time — highest precedence, above even the existing sub-cohort `template_override_id`). A new `template_type` query param on the existing `available_placeholders` endpoint returns confirmation-accurate placeholders instead of the generic set. `DayListAutomation`'s model, serializer, viewset, URLs, Celery beat entry, and the whole `daylist_automation.py` module are deleted, along with the frontend "Automation" tab and its section component.

**Tech Stack:** Django REST Framework, React/TypeScript, shadcn/ui.

---

### Task 1: Remove `DayListAutomation` (backend)

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/models.py`
- Modify: `TreatmentPath/dentallyIntegration/serializers.py`
- Delete: `TreatmentPath/dentallyIntegration/views/daylist_automation_views.py`
- Delete: `TreatmentPath/dentallyIntegration/daylist_automation.py`
- Modify: `TreatmentPath/dentallyIntegration/urls.py`
- Modify: `TreatmentPath/TreatmentPath/settings.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Confirm no tests reference `DayListAutomation` before deleting it**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && grep -n "DayListAutomation" dentallyIntegration/tests.py
```

If this returns any test classes/imports, read them and delete them as part of this task (a `DayListAutomation`-only test class has nothing left to test once the model is gone) — do NOT leave dangling references. If it returns nothing, proceed.

- [ ] **Step 2: Delete the model class**

In `dentallyIntegration/models.py`, find and delete the entire `class DayListAutomation(models.Model):` class (currently lines 3295–3362, ending right before `class RecallSequenceEnrollment(models.Model):` — verify the exact boundaries by reading the file, since other work may have shifted line numbers).

- [ ] **Step 3: Delete the serializer class**

In `dentallyIntegration/serializers.py`, find and delete the entire `class DayListAutomationSerializer(serializers.ModelSerializer):` class. Also remove `DayListAutomation` from whatever `from .models import (...)` line currently imports it (merge-edit the import list, don't leave a dangling unused import).

- [ ] **Step 4: Delete the views file**

```bash
rm dentallyIntegration/views/daylist_automation_views.py
```

- [ ] **Step 5: Delete the automation engine module**

```bash
rm dentallyIntegration/daylist_automation.py
```

- [ ] **Step 6: Remove URL registration**

In `dentallyIntegration/urls.py`, remove:
- The `DayListAutomationViewSet` import line (in the `from .views import (...)` block).
- The router registration block:
```python
_router.register(
    r"daylist-automations", DayListAutomationViewSet, basename="daylist-automation"
)
```
(and its preceding comment lines, if any, describing what it registered).

- [ ] **Step 7: Remove the Celery beat schedule entry**

In `TreatmentPath/settings.py`, find `CELERY_BEAT_SCHEDULE` and remove the entire `"process-daylist-automations"` entry:
```python
    "process-daylist-automations": {
        "task": "dentallyIntegration.tasks.process_daylist_automations",
        "schedule": crontab(minute="*/5"),
        "options": {"queue": "default"},
    },
```
Leave the `"process-confirmation-sequences"` entry immediately above it completely untouched.

- [ ] **Step 8: Check for the Celery task wrapper**

```bash
grep -rn "process_daylist_automations" dentallyIntegration/tasks.py
```

If a Celery task wrapper function (e.g. `@shared_task def process_daylist_automations(): ...`) exists in `dentallyIntegration/tasks.py` calling into the now-deleted `daylist_automation.py` module, delete that wrapper function too (it would otherwise raise `ImportError` the moment Celery Beat fires it).

- [ ] **Step 9: Search for any other remaining references**

```bash
grep -rln "DayListAutomation\|daylist_automation" --include="*.py" . | grep -v migrations
```

Read and clean up any remaining hits this turns up that aren't already handled above (e.g. an admin.py registration, a stray import elsewhere) — do not leave dangling references to deleted code.

- [ ] **Step 10: Attempt to run the full dentallyIntegration test suite to confirm nothing references the deleted code**

```bash
python manage.py test dentallyIntegration --keepdb -v 2 2>&1 | tail -80
```

Expected: no `ImportError`/`AttributeError` referencing `DayListAutomation`. Some pre-existing, unrelated test failures in other test classes (`Recall*`) are expected and NOT your concern — only verify nothing NEW fails due to this deletion. If a `makemigrations`/`migrate` step is needed to actually drop the DB table, you will not be able to run it yourself (hard-blocked) — STOP after this step and report back what migration you expect to be needed (a single `RemoveField`-style `DeleteModel` migration for `DayListAutomation`); the coordinator will run `makemigrations`/`migrate` separately.

**Do NOT commit.**

---

### Task 2: Remove `DayListAutomation` (frontend)

**Files:**
- Delete: `perfect-pixel-playground-project/src/pages/day-list/components/administration/DayListAutomationSection.tsx`
- Modify: `perfect-pixel-playground-project/src/pages/day-list/components/DayListAdministration.tsx`
- Modify: `perfect-pixel-playground-project/src/config/api.ts`

- [ ] **Step 1: Delete the section component**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
rm src/pages/day-list/components/administration/DayListAutomationSection.tsx
```

- [ ] **Step 2: Remove the tab from `DayListAdministration.tsx`**

Read the current file in full first (it's short, ~147 lines) to confirm exact current line numbers, since other work may have shifted them slightly. Make these exact changes:

Remove the import:
```tsx
import { DayListAutomationSection } from './administration/DayListAutomationSection';
```

Remove the tab entry from `OPP_TABS`:
```tsx
  { id: 'automation', label: 'Automation' },
```
(leaving `rules`, `terminology`, `confirmations` — in that order, i.e. removing this one line from the middle of the array).

Remove `isAutomation` and simplify the two ternaries that reference it. Change:
```tsx
  const isAutomation = activeTab === 'automation';
  const isConfirmations = activeTab === 'confirmations';
  // The Automation and Confirmations tabs manage their own per-item saves
  // (create/delete/toggle each hit the API immediately, like Recall's
  // AutomationSection) — they have no SectionSaveHandle, so the header Save
  // button is simply inert (disabled) while either is active, exactly
  // mirroring RecallsAdministration's automation tab.
  const activeRef = isAutomation || isConfirmations ? null : isRules ? rulesRef : termRef;
  const activeDirty = isAutomation || isConfirmations ? false : isRules ? rulesDirty : termDirty;
```
to:
```tsx
  const isConfirmations = activeTab === 'confirmations';
  // The Confirmations tab manages its own per-item saves (create/delete/
  // toggle each hit the API immediately, like Recall's AutomationSection) —
  // it has no SectionSaveHandle, so the header Save button is simply inert
  // (disabled) while it's active, exactly mirroring RecallsAdministration's
  // automation tab.
  const activeRef = isConfirmations ? null : isRules ? rulesRef : termRef;
  const activeDirty = isConfirmations ? false : isRules ? rulesDirty : termDirty;
```

Remove the render line:
```tsx
          {activeTab === 'automation' && <DayListAutomationSection />}
```

- [ ] **Step 3: Remove the API endpoint entries**

In `src/config/api.ts`, remove these three lines from the `DENTALLY` object (find their exact current location — should be adjacent to each other, near `CONFIRMATION_SEQUENCES`):
```ts
    DAYLIST_AUTOMATIONS: getApiUrl('/dentally/daylist-automations/'),
    DAYLIST_AUTOMATION_DETAIL: (id: number | string) => getApiUrl(`/dentally/daylist-automations/${id}/`),
    DAYLIST_AUTOMATION_TEST: (id: number | string) => getApiUrl(`/dentally/daylist-automations/${id}/test/`),
```

- [ ] **Step 4: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit --project tsconfig.app.json --ignoreDeprecations "5.0" 2>&1 | grep -c "error TS"
```

Measure the baseline BEFORE your change too (run the same command on a fresh read before deleting anything) and confirm the count doesn't INCREASE (it may decrease slightly if `DayListAutomationSection.tsx` itself had pre-existing errors being removed — that's fine, just confirm no new errors appear elsewhere from the deletion, e.g. a stray import of the deleted component somewhere else).

**Do NOT commit.**

---

### Task 3: `ConfirmationSequence` steps gain `subject`/`body` + validation (backend)

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/serializers.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

No model/migration change needed — `steps` is already a `JSONField`, so new keys inside its dicts need no schema change, only serializer-level validation.

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class ConfirmationSequenceStepsValidationTests(TestCase):
    """validate_steps: each step needs template_id OR non-blank body (not
    neither); channel must be sms/email (no task channel, unlike Recall's
    steps); offset_days must be >= 0; send_time (if present) must be HH:MM."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Confirmation Steps Validation Dental")

    def test_step_with_template_id_only_is_valid(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Template-only sequence",
            "steps": [{"channel": "sms", "template_id": 1, "offset_days": 3}],
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertTrue(serializer.is_valid(), serializer.errors)

    def test_step_with_body_only_is_valid(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Custom-copy sequence",
            "steps": [{"channel": "sms", "offset_days": 3, "body": "Please confirm your visit."}],
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertTrue(serializer.is_valid(), serializer.errors)

    def test_step_with_neither_template_id_nor_body_fails(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Empty step sequence",
            "steps": [{"channel": "sms", "offset_days": 3}],
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn("steps", serializer.errors)

    def test_task_channel_is_rejected(self):
        """Unlike RecallSequence, ConfirmationSequence steps have no task
        channel concept."""
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Task channel sequence",
            "steps": [{"channel": "task", "title": "Follow up", "offset_days": 3}],
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn("steps", serializer.errors)

    def test_negative_offset_days_fails(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Negative offset sequence",
            "steps": [{"channel": "sms", "template_id": 1, "offset_days": -1}],
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn("steps", serializer.errors)

    def test_malformed_send_time_fails(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Bad send_time sequence",
            "steps": [{"channel": "sms", "template_id": 1, "offset_days": 3, "send_time": "not-a-time"}],
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn("steps", serializer.errors)

    def test_empty_steps_list_fails(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {"name": "No steps sequence", "steps": []}
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn("steps", serializer.errors)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.ConfirmationSequenceStepsValidationTests --keepdb -v 2
```

Expected: `test_step_with_body_only_is_valid` and `test_task_channel_is_rejected` FAIL (no `validate_steps` exists yet, so a body-only step currently fails serializer-level model validation, and a `task`-channel step currently passes with no rejection since nothing validates `channel` at all today). Other tests may pass by accident (e.g. empty steps might already be rejected by the model field itself) — that's fine, just confirm the two specifically-targeted ones fail for the right reason.

- [ ] **Step 3: Add `validate_steps` to `ConfirmationSequenceSerializer`**

In `dentallyIntegration/serializers.py`, find `ConfirmationSequenceSerializer` (currently ending with its `validate_targeting` method) and add this new method, mirroring `RecallSequenceSerializer.validate_steps` but with the `task`/`trigger` branches removed (confirmation steps are sms/email only):

```python
    def validate_steps(self, value):
        if not isinstance(value, list) or not value:
            raise serializers.ValidationError("A sequence needs at least one step.")
        for i, step in enumerate(value):
            if not isinstance(step, dict):
                raise serializers.ValidationError(f"Step {i + 1} is malformed.")
            channel = step.get("channel")
            if channel not in ("email", "sms"):
                raise serializers.ValidationError(
                    f"Step {i + 1}: channel must be 'email' or 'sms'."
                )
            # A step needs a template UNLESS custom copy is supplied — body
            # (with an optional subject for email) overrides template_id
            # entirely at send time (see confirmation_automation.py's
            # advance_confirmation_enrollments), matching
            # RecallSequenceSerializer's identical "template or custom copy"
            # rule for its own steps.
            if not step.get("template_id") and not str(step.get("body") or "").strip():
                raise serializers.ValidationError(
                    f"Step {i + 1}: choose a template or write custom copy."
                )
            try:
                if int(step.get("offset_days", 0)) < 0:
                    raise ValueError
            except (TypeError, ValueError):
                raise serializers.ValidationError(
                    f"Step {i + 1}: offset_days must be 0 or more."
                )
            # Optional per-step send time ("HH:MM"); falls back to the
            # sequence default when absent.
            send_time = step.get("send_time")
            if send_time:
                try:
                    parts = str(send_time).split(":")
                    hh, mm = int(parts[0]), int(parts[1])
                    if not (0 <= hh <= 23 and 0 <= mm <= 59):
                        raise ValueError
                except (TypeError, ValueError, IndexError):
                    raise serializers.ValidationError(
                        f"Step {i + 1}: send_time must be HH:MM (24-hour)."
                    )
        return value
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationSequenceStepsValidationTests --keepdb -v 2
```

Expected: PASS (7 tests).

- [ ] **Step 5: Run the full serializer test suite to confirm no regressions**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationSequenceSerializerTests dentallyIntegration.tests.ConfirmationSequenceViewSetTests --keepdb -v 2
```

Expected: PASS, all green — existing tests that create sequences with `template_id`-only steps must be completely unaffected.

**Do NOT commit.**

---

### Task 4: Send-time precedence: custom body > sub-cohort override > template_id (backend)

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/confirmation_automation.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing test**

Append to `dentallyIntegration/tests.py`:

```python
class CustomBodyOverridesEverythingTests(TestCase):
    """A step's own custom `body` wins outright — even over a sub-cohort's
    template_override_id — since custom text is the most explicit, deliberate
    choice a staff member made for that specific step."""

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Custom Body Override Dental", twilio_phone_number="+447700900000"
        )
        self.opt_in_template = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="VIP opt-in confirmation SMS",
            content="VIP opt-in: {{ confirmation_link }}",
            template_type="appointment_confirmation",
            is_active=True,
        )
        self.sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="Sequence with custom body and override",
            status="active",
            days_ahead=7,
            steps=[
                {
                    "channel": "sms",
                    "offset_days": 0,
                    "body": "Custom copy: {{ confirmation_link }}",
                }
            ],
            targeting={
                "base_audience": {"bucket_keys": []},
                "sub_cohorts": [
                    {
                        "name": "VIP opt-in",
                        "priority": 1,
                        "bucket_keys": [],
                        "confirm_dental_opt_in": True,
                        "template_override_id": self.opt_in_template.id,
                    }
                ],
            },
        )
        self.appointment = DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=50200,
            dentally_patient_id=1,
            patient_name="Custom Body Test Patient",
            patient_phone="+447700900222",
            state="pending",
            duration=30,
            start_time=timezone.now() + timedelta(hours=1),
        )

    @override_settings(TWILIO_ACCOUNT_SID="ACtest", TWILIO_AUTH_TOKEN="tok")
    def test_custom_body_wins_over_sub_cohort_template_override(self):
        now = timezone.now()
        ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=self.appointment,
            enrolled_at=now,
            enrolled_appointment_start=self.appointment.start_time,
            next_due_at=now,
            sub_cohort_name="VIP opt-in",
        )
        with patch("twilio.rest.Client") as MockTwilio:
            MockTwilio.return_value.messages.create.return_value.sid = "SMfake"
            advance_confirmation_enrollments(now=now)

        message = SMSMessage.objects.filter(phone_number="+447700900222").latest("created_at")
        self.assertIn("Custom copy", message.content)
        self.assertNotIn("VIP opt-in:", message.content)
```

Check the top of `dentallyIntegration/tests.py` for the exact existing import list (`Practice`, `SMSMessageTemplate`, `ConfirmationSequence`, `DentallyAppointment`, `ConfirmationSequenceEnrollment`, `SMSMessage`, `timezone`, `timedelta`, `patch`, `override_settings`, `advance_confirmation_enrollments` — these should already be imported/available from prior tasks in this same file; if any need a local import inside the test method to match the file's established convention for a given symbol, do so).

- [ ] **Step 2: Run test to verify it fails**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.CustomBodyOverridesEverythingTests --keepdb -v 2
```

Expected: FAIL — the message sent uses the sub-cohort's `template_override_id` template ("VIP opt-in: ...") instead of the step's own custom `body`, since custom-body handling doesn't exist yet.

- [ ] **Step 3: Wire the precedence into `advance_confirmation_enrollments`**

In `dentallyIntegration/confirmation_automation.py`, find this exact block (added by a prior task, currently around lines 596-609 — verify exact location, it may have shifted):

```python
        # FR8a: if this enrollment matched a sub-cohort with a
        # template_override_id, use THAT template instead of the step's
        # default — staff author a distinct template containing whatever
        # opt-in copy/CTA they want (using the same {{ confirmation_link }}
        # placeholder), so the "opt-in treatment" differentiation lives
        # entirely in which template gets sent, not a different URL.
        effective_template_id = step.get("template_id")
        if enrollment.sub_cohort_name:
            sub_cohort = next(
                (
                    sc
                    for sc in (sequence.targeting or {}).get("sub_cohorts") or []
                    if sc.get("name") == enrollment.sub_cohort_name
                ),
                None,
            )
            if sub_cohort and sub_cohort.get("template_override_id"):
                effective_template_id = sub_cohort["template_override_id"]

        subject, body = _resolve_confirmation_template(practice=sequence.practice, channel=channel, template_id=effective_template_id)
        if body is None:
            stats["failed"] += 1
            enrollment.save(update_fields=["enrolled_appointment_start", "updated_at"])
            continue
```

Replace it with:

```python
        # Precedence (highest to lowest): 1) the step's own custom `body`
        # text — wins outright, even over a sub-cohort's template_override_id,
        # since custom text is the most explicit, deliberate choice a staff
        # member made for this specific step; 2) FR8a's sub-cohort
        # template_override_id (staff author a distinct template containing
        # whatever opt-in copy/CTA they want); 3) the step's own template_id
        # (the original default).
        custom_body = step.get("body")
        if custom_body:
            subject, body = step.get("subject") or "", custom_body
        else:
            effective_template_id = step.get("template_id")
            if enrollment.sub_cohort_name:
                sub_cohort = next(
                    (
                        sc
                        for sc in (sequence.targeting or {}).get("sub_cohorts") or []
                        if sc.get("name") == enrollment.sub_cohort_name
                    ),
                    None,
                )
                if sub_cohort and sub_cohort.get("template_override_id"):
                    effective_template_id = sub_cohort["template_override_id"]

            subject, body = _resolve_confirmation_template(practice=sequence.practice, channel=channel, template_id=effective_template_id)
            if body is None:
                stats["failed"] += 1
                enrollment.save(update_fields=["enrolled_appointment_start", "updated_at"])
                continue
```

- [ ] **Step 4: Run test to verify it passes**

```bash
python manage.py test dentallyIntegration.tests.CustomBodyOverridesEverythingTests --keepdb -v 2
```

Expected: PASS.

- [ ] **Step 5: Run the full confirmation-automation regression suite**

```bash
python manage.py test dentallyIntegration.tests.SendTimeTemplateOverrideTests dentallyIntegration.tests.CustomBodyOverridesEverythingTests dentallyIntegration.tests.ConfirmationAutomationSendingTests dentallyIntegration.tests.AdvanceConfirmationEnrollmentsTests dentallyIntegration.tests.EnrollEligibleAppointmentsTests dentallyIntegration.tests.EnrollEligibleAppointmentsTargetingTests dentallyIntegration.tests.ProcessConfirmationEnrollmentsTests --keepdb -v 2
```

Expected: PASS, all green — `SendTimeTemplateOverrideTests` (the prior task's sub-cohort-override-without-custom-body scenario) must still pass unchanged, since `custom_body` is falsy in those tests and the code falls through to the exact same logic as before.

**Do NOT commit.**

---

### Task 5: Confirmation-specific placeholders endpoint (backend)

**Files:**
- Modify: `TreatmentPath/messaging/views/template_views.py`
- Test: `TreatmentPath/messaging/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `messaging/tests.py`:

```python
class AvailablePlaceholdersConfirmationTypeTests(TestCase):
    """?template_type=appointment_confirmation returns the 7 real keys
    confirmation_automation.render_message actually supports (confirmed via
    reading render_message directly — confirmation_link, patient_name,
    appointment_date, appointment_time, practitioner_name, practice_name,
    practice_phone), instead of the generic patient/clinic/dentist/etc. set.
    Absent or any other template_type value returns the original generic
    dict unchanged (regression-proofing Recall's existing, unparameterized
    usage of this same endpoint)."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Placeholder Endpoint Dental")
        self.user = UserAuthentication.objects.create_user(
            email="placeholders@confirmation.test", password="testpass123", practice=self.practice,
        )
        self.client.force_login(self.user)

    def test_confirmation_template_type_returns_confirmation_keys(self):
        response = self.client.get(
            "/api/backend/messaging/email-templates/placeholders/?template_type=appointment_confirmation"
        )
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertIn("confirmation", data)
        confirmation_keys = set(data["confirmation"].keys())
        self.assertEqual(
            confirmation_keys,
            {
                "confirmation_link",
                "patient_name",
                "appointment_date",
                "appointment_time",
                "practitioner_name",
                "practice_name",
                "practice_phone",
            },
        )

    def test_no_template_type_returns_generic_dict_unchanged(self):
        response = self.client.get("/api/backend/messaging/email-templates/placeholders/")
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertIn("patient", data)
        self.assertIn("clinic", data)
        self.assertNotIn("confirmation", data)

    def test_other_template_type_returns_generic_dict_unchanged(self):
        response = self.client.get(
            "/api/backend/messaging/email-templates/placeholders/?template_type=recall"
        )
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertIn("patient", data)
        self.assertNotIn("confirmation", data)
```

Check the top of `messaging/tests.py` for the exact existing test-user-creation pattern already established in that file (a prior task in this same feature discovered `messaging/tests.py`'s own convention differs from a generic `create_user`/`force_login` — match whatever's ACTUALLY used elsewhere in this file for hitting an authenticated endpoint, e.g. `InboxRecallExclusionTests`'s JWT-bearer-auth pattern via `APITestCase`/`AccessToken`, rather than assuming `force_login` works here).

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test messaging.tests.AvailablePlaceholdersConfirmationTypeTests --keepdb -v 2
```

Expected: `test_confirmation_template_type_returns_confirmation_keys` FAILS (no `"confirmation"` key in the response yet — the endpoint doesn't branch on `template_type` at all today). The other two tests should already PASS (today's unconditional generic-dict behavior is what they assert), confirming your baseline understanding is correct before you change anything.

- [ ] **Step 3: Add the `template_type` branch**

In `messaging/views/template_views.py`, find `EmailMessageTemplateViewSet.available_placeholders` (currently `@action(detail=False, methods=["GET"])`) and modify its signature/body to branch on the query param. Replace:

```python
    @action(detail=False, methods=["GET"])
    def available_placeholders(self, request):
        """
        Get all available placeholders for email templates with their direct mapping syntax.
        Includes both nested object format and flat format for backwards compatibility.
        """
        return Response(
            {
```

with:

```python
    @action(detail=False, methods=["GET"])
    def available_placeholders(self, request):
        """
        Get all available placeholders for email templates with their direct mapping syntax.
        Includes both nested object format and flat format for backwards compatibility.

        ?template_type=appointment_confirmation returns a dedicated
        "confirmation" category matching confirmation_automation.
        render_message's actual 7-key render context exactly (confirmed by
        reading that function directly) — the generic categories below don't
        cover confirmation_link/practice_phone/practitioner_name at all, and
        would silently render as empty strings if used in a confirmation
        template, so this is a separate, accurate category rather than an
        addition to the generic set.
        """
        if request.query_params.get("template_type") == "appointment_confirmation":
            return Response(
                {
                    "confirmation": {
                        "confirmation_link": "{{confirmation_link}}",
                        "patient_name": "{{patient_name}}",
                        "appointment_date": "{{appointment_date}}",
                        "appointment_time": "{{appointment_time}}",
                        "practitioner_name": "{{practitioner_name}}",
                        "practice_name": "{{practice_name}}",
                        "practice_phone": "{{practice_phone}}",
                    }
                }
            )
        return Response(
            {
```

(The rest of the existing method body — the `"patient": {...}` through `"template_specific": {...}` generic dict and its closing `}` / `)` — stays completely unchanged, just now reached only when the new `if` branch above doesn't return early.)

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test messaging.tests.AvailablePlaceholdersConfirmationTypeTests --keepdb -v 2
```

Expected: PASS (3 tests).

- [ ] **Step 5: Run the full template_views test suite to confirm no regressions**

```bash
python manage.py test messaging --keepdb -v 2 2>&1 | tail -60
```

Expected: no NEW failures beyond whatever pre-existing failures already exist in this file — Recall's own usage of this same endpoint (no `template_type` param) must be completely unaffected.

**Do NOT commit.**

---

### Task 6: Frontend types — `subject`/`body` on `ConfirmationSequence.steps`

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/day-list/types.ts`

- [ ] **Step 1: Update the `steps` field type**

In `src/pages/day-list/types.ts`, find the `ConfirmationSequence` interface's `steps` field (currently line 230):

```ts
  steps: Array<{ channel: 'sms' | 'email'; template_id: number; offset_days: number; send_time?: string }>;
```

Replace it with:

```ts
  steps: Array<{
    channel: 'sms' | 'email';
    template_id: number;
    offset_days: number;
    send_time?: string;
    /** Optional custom copy (email subject) — cosmetic for SMS steps. */
    subject?: string;
    /** Optional custom copy — when non-blank, overrides template_id entirely
     *  at both validation and send time (see confirmation_automation.py's
     *  advance_confirmation_enrollments). */
    body?: string;
  }>;
```

- [ ] **Step 2: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit --project tsconfig.app.json --ignoreDeprecations "5.0" 2>&1 | grep -c "error TS"
```

Measure the baseline BEFORE this change too. Confirm the count doesn't increase — `subject`/`body` are both optional, so no existing code constructing a `steps` array literal without them should break.

**Do NOT commit.**

---

### Task 7: Step editor UI — free-text override + placeholder picker (frontend)

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/day-list/components/administration/ConfirmationsSection.tsx`
- Modify: `perfect-pixel-playground-project/src/config/api.ts`

- [ ] **Step 1: Add the confirmation-placeholders API endpoint entry**

In `src/config/api.ts`, find the `messages:` top-level object (the one containing `emailTemplateAvailablePlaceholders`/`smsTemplateAvailablePlaceholders`, currently around line 614) and add a new entry right after `emailTemplateAvailablePlaceholders`:

```ts
    emailTemplateAvailablePlaceholders: () => getApiUrl('/messaging/email-templates/placeholders/'),
    confirmationTemplateAvailablePlaceholders: () => getApiUrl('/messaging/email-templates/placeholders/?template_type=appointment_confirmation'),
```

(Only the second line is new — `emailTemplateAvailablePlaceholders` above it is shown for exact insertion-point context, don't duplicate it.)

- [ ] **Step 2: Read `ConfirmationsSection.tsx` in full first**

Before making any change, read the whole file to confirm the exact current state of: the top-of-file imports (currently `useCallback`/`useEffect`/`useState` from react, `Plus`/`Trash2`/`Pencil`/`CalendarCheck` from `lucide-react`, `toast` from `sonner`, `Button`/`Input`/`Label` from shadcn, `useFetchWithAuth`, `API_ENDPOINTS`, `ConfirmationSequence` type, `ConfirmationTargetingEditor`), the `blankStep()`/`updateStep`/`save()` functions, and the exact step-editor JSX block (a `flex flex-wrap items-end gap-3` row with Channel/Template/Offset/Send-time/Delete-button children) — since other work may have shifted exact line numbers slightly from what's described below.

- [ ] **Step 3: Add new imports**

Add to the top of `ConfirmationsSection.tsx`:

```tsx
import { useCallback, useEffect, useRef, useState } from 'react';
import { Plus, Trash2, Pencil, CalendarCheck, Hash } from 'lucide-react';
import { Textarea } from '@/components/ui/textarea';
import { Popover, PopoverContent, PopoverTrigger } from '@/components/ui/popover';
import { ScrollArea } from '@/components/ui/scroll-area';
import { Accordion, AccordionContent, AccordionItem, AccordionTrigger } from '@/components/ui/accordion';
import { Badge } from '@/components/ui/badge';
```

(Merge `useRef` into the existing `react` import line rather than duplicating it; merge `Hash` into the existing `lucide-react` import line rather than duplicating it. Check `@/components/ui/badge-pill` or similar — Recall's `AutomationSection.tsx` uses a `BadgePill` component for the placeholder tokens themselves; find its exact import path by checking how `AutomationSection.tsx` imports it, and add the identical import here.)

- [ ] **Step 4: Add the `PlaceholderPicker` component**

Add this new component definition near the top of the file (module scope, outside the main exported component), mirroring Recall's `AutomationSection.tsx` pattern but with a single flat category (confirmations only have one placeholder category, not four like Recall):

```tsx
type ConfirmationPlaceholders = Record<string, string>;

/** The categorised placeholder picker — same accordion UI as Recall's
 *  AutomationSection.tsx, but confirmations only have one category
 *  ("confirmation"), unlike Recall's 4 (patient/clinic/current_user/recall). */
function PlaceholderPicker({
  placeholders,
  onInsert,
}: {
  placeholders: ConfirmationPlaceholders;
  onInsert: (token: string) => void;
}) {
  const entries = Object.entries(placeholders);
  if (!entries.length) return null;

  return (
    <Popover>
      <PopoverTrigger asChild>
        <Button
          type="button"
          variant="ghost"
          size="icon"
          className="h-7 w-7 p-0 text-gray-500 hover:bg-gray-100"
          title="Insert a placeholder"
          aria-label="Insert a placeholder"
        >
          <Hash className="h-4 w-4" />
        </Button>
      </PopoverTrigger>
      <PopoverContent align="end" className="w-80 p-0">
        <ScrollArea className="h-[280px]">
          <Accordion type="multiple" defaultValue={['confirmation']} className="w-full">
            <AccordionItem value="confirmation" className="border-b border-gray-100 last:border-b-0">
              <AccordionTrigger className="px-4 py-3 hover:bg-gray-50 hover:no-underline">
                <div className="flex items-center gap-3 w-full">
                  <div className="flex-1 text-left">
                    <div className="flex items-center gap-2">
                      <span className="font-medium text-sm text-gray-900">Confirmation</span>
                      <Badge variant="outline" className="text-[10px] px-1.5 py-0 h-5 bg-gray-50 text-gray-600 border-gray-200">
                        {entries.length}
                      </Badge>
                    </div>
                    <p className="text-xs text-gray-500 mt-0.5">Appointment confirmation details</p>
                  </div>
                </div>
              </AccordionTrigger>
              <AccordionContent className="px-4 pb-4">
                <div className="flex flex-wrap gap-2 pt-1">
                  {entries.map(([key, value]) => (
                    <button
                      key={key}
                      type="button"
                      onClick={() => onInsert(value)}
                      className="inline-flex items-center rounded-full border border-gray-200 px-2.5 py-1 text-xs text-gray-700 hover:bg-gray-50"
                    >
                      {key.replace(/_/g, ' ')}
                    </button>
                  ))}
                </div>
              </AccordionContent>
            </AccordionItem>
          </Accordion>
        </ScrollArea>
      </PopoverContent>
    </Popover>
  );
}
```

(If `AutomationSection.tsx`'s `BadgePill` component is trivial to import and reuse instead of the plain `<button>` shown above, prefer reusing it for visual consistency — check its import path and props signature first; if it requires unrelated setup not worth pulling in for one component, the plain `<button>` above is an acceptable equivalent.)

- [ ] **Step 5: Add placeholder-fetching state and the active-field ref**

Inside the main `ConfirmationsSection` component function, add:

```tsx
  const [placeholders, setPlaceholders] = useState<ConfirmationPlaceholders>({});
  const activeFieldRef = useRef<{ i: number; field: 'subject' | 'body'; el: HTMLInputElement | HTMLTextAreaElement } | null>(null);
```

Find the existing `useEffect` that fetches `emailTemplates`/`smsTemplates` on mount and add a placeholder fetch alongside it (in the same effect or a new one — match whichever the file's existing template-fetch effect structure makes cleaner):

```tsx
  useEffect(() => {
    (async () => {
      try {
        const pRes = await fetchWithAuth(API_ENDPOINTS.messages.confirmationTemplateAvailablePlaceholders());
        if (pRes.ok) {
          const data = await pRes.json();
          if (data && typeof data === 'object' && data.confirmation) {
            setPlaceholders(data.confirmation as ConfirmationPlaceholders);
          }
        }
      } catch {
        /* placeholders are best-effort */
      }
    })();
  }, [fetchWithAuth]);
```

(Confirmed: `confirmationTemplateAvailablePlaceholders`, added in Task 7 Step 1, lives under the top-level `messages:` object — the same one containing `emailTemplateAvailablePlaceholders`/`fetchActiveEmailTemplates`/`fetchActiveSmsTemplates` — so the call site above, `API_ENDPOINTS.messages.confirmationTemplateAvailablePlaceholders()`, is correct as written.)

- [ ] **Step 6: Add the `insertPlaceholder` handler**

```tsx
  const insertPlaceholder = (token: string, fallbackStep: number) => {
    const target = activeFieldRef.current;
    if (target && target.i === fallbackStep) {
      const { i, field, el } = target;
      const cur = (field === 'subject' ? editing?.steps[i]?.subject : editing?.steps[i]?.body) ?? '';
      const start = el.selectionStart ?? cur.length;
      const end = el.selectionEnd ?? cur.length;
      updateStep(i, { [field]: cur.slice(0, start) + token + cur.slice(end) });
      setTimeout(() => {
        el.focus();
        const pos = start + token.length;
        el.setSelectionRange(pos, pos);
      }, 0);
      return;
    }
    const cur = editing?.steps[fallbackStep]?.body ?? '';
    updateStep(fallbackStep, { body: cur + token });
  };
```

(Uses `updateStep`, this file's existing per-step patch function — Recall's equivalent is called `patchStep`; use whichever name `ConfirmationsSection.tsx` actually has, confirmed as `updateStep` per prior research, but double check against the real file.)

- [ ] **Step 7: Add the custom-copy JSX section to the step editor**

Find the step-editor JSX (a `<div key={i} className="flex flex-wrap items-end gap-3 ...">` row containing Channel/Template/Offset/Send-time/Delete-button). Add this new block as an additional sibling `<div>` immediately after that row's closing `</div>` (i.e., a second row underneath each step, matching Recall's two-row layout):

```tsx
                  <div className="w-full space-y-2 border-t border-gray-100 pt-3">
                    <Label className="text-[11px] text-gray-500">Custom copy (optional — overrides the template)</Label>
                    {step.channel === 'email' && (
                      <div className="relative">
                        <Input
                          value={step.subject ?? ''}
                          onChange={(e) => updateStep(i, { subject: e.target.value })}
                          onFocus={(e) => { activeFieldRef.current = { i, field: 'subject', el: e.currentTarget }; }}
                          placeholder="Custom subject (optional)"
                          className="bg-white pr-10 h-9"
                        />
                        <div className="absolute right-1.5 top-1/2 -translate-y-1/2">
                          <PlaceholderPicker placeholders={placeholders} onInsert={(token) => insertPlaceholder(token, i)} />
                        </div>
                      </div>
                    )}
                    <div className="relative">
                      <Textarea
                        value={step.body ?? ''}
                        onChange={(e) => updateStep(i, { body: e.target.value })}
                        onFocus={(e) => { activeFieldRef.current = { i, field: 'body', el: e.currentTarget }; }}
                        placeholder={`Custom ${step.channel === 'email' ? 'email' : 'SMS'} copy — leave blank to use the template`}
                        className="bg-white min-h-[72px] pr-10"
                      />
                      <div className="absolute bottom-2 right-2">
                        <PlaceholderPicker placeholders={placeholders} onInsert={(token) => insertPlaceholder(token, i)} />
                      </div>
                    </div>
                  </div>
```

Note this block has no `channel !== 'task'` guard (unlike Recall's, since confirmation steps never have a `task` channel at all — every step gets this section).

- [ ] **Step 8: Update save-time validation**

In `save()` (currently line 123), change:

```ts
    if (editing.steps.some((s) => !s.template_id)) return toast.error('Select a template for every step');
```

to:

```ts
    if (editing.steps.some((s) => !s.template_id && !(s.body || '').trim())) return toast.error('Choose a template or write custom copy for every step');
```

- [ ] **Step 9: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit --project tsconfig.app.json --ignoreDeprecations "5.0" 2>&1 | grep -c "error TS"
```

Measure the baseline before this change too, confirm no new errors.

- [ ] **Step 10: Do NOT attempt manual browser verification** — the user has asked that visual/browser tooling not be used in this session. Just confirm the code compiles correctly.

**Do NOT commit.**

---

## Summary of spec coverage

- Remove `DayListAutomation` (backend: model/serializer/views/urls/beat entry/module) → Task 1.
- Remove `DayListAutomation` (frontend: section component/tab/api entries) → Task 2.
- `steps[].subject`/`body` + validation (backend) → Task 3.
- Send-time precedence (custom body > sub-cohort override > template_id) → Task 4.
- Confirmation-specific placeholder endpoint → Task 5.
- Frontend types for `subject`/`body` → Task 6.
- Step editor UI (custom-copy section + PlaceholderPicker) → Task 7.

## Out of scope (per the approved design)

- `DayListAutomation`'s per-row `run_every_minutes` and test-send action — not being carried forward into `ConfirmationSequence`.
- Any `trigger_type`/daily-schedule unification — explicitly decided against.
- Extracting a shared `PlaceholderPicker` component used by both Recall and Confirmations — stays locally-defined in each file, matching existing precedent.

# Django Import-Issue Writer Cutover — Implementation Plan (Phase 2 of 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Repoint the Django Celery migration task's failure-recording write from the
legacy `DentallyImportFailure`/`DentallyImportError` models to the new
`DataQualityIssue` model (Phase 1), so import-time issues start landing in the unified
table.

**Architecture:** Extract the inline failure-recording block in
`migrate_all_dentally_patients` (currently ~47 lines of inline code,
`dentallyIntegration/tasks.py:1167-1213`) into a standalone, independently-testable
helper function that writes `DataQualityIssue` rows using the same
`(practice, dentally_patient_id)` upsert key the legacy code used. No other behavior in
the migration task changes.

**Tech Stack:** Django 5.2, Celery, PostgreSQL.

**Spec:** `docs/superpowers/specs/2026-08-13-unified-data-quality-design.md` (Section 2,
"Import-time" writer)

**Prerequisite:** Phase 1 (`docs/superpowers/plans/2026-08-13-dataquality-app-and-model.md`)
must be applied — `DataQualityIssue` must exist and be migrated.

**⚠️ Deployment sequencing note:** This plan does NOT touch the read side —
`DentallyImportFailureViewSet`/`DentallyImportErrorViewSet` (`dentallyIntegration/views/
import_views.py`) keep reading the legacy tables, which stop receiving new rows once
this ships. Between this deploy and Phase 5 (frontend rewire, which repoints the read
side too), the existing Duplicates/Non-Duplicates tabs will show only pre-cutover
historical data — no NEW import failures will appear there. This was an explicit
accepted tradeoff (direct cutover, no dual-write window, agreed as "internal tooling,
low risk") — flagging it here so Phases 2 and 5 are deployed close together in practice,
not left indefinitely far apart.

---

### Task 1: Extract and redirect the failure-recording helper

**Files:**
- Test: `dentallyIntegration/test_record_import_issue.py` (new)
- Modify: `dentallyIntegration/tasks.py:1167-1213` (extract into new function)

- [ ] **Step 1: Write the failing test for the new helper, in isolation**

```python
# dentallyIntegration/test_record_import_issue.py
from django.test import TestCase, override_settings

from UserAuthentication.models import Practice
from dataQuality.models import DataQualityIssue
from dentallyIntegration.tasks import _record_import_issue


@override_settings(SECURE_SSL_REDIRECT=False)
class RecordImportIssueTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(name="Import Issue Test Practice")

    def _dentally_patient(self, dentally_id=88001):
        return {
            "id": dentally_id,
            "first_name": "Jane",
            "last_name": "Doe",
            "email_address": "jane.doe@example.com",
        }

    def test_duplicate_failure_type_writes_duplicate_import_classifier(self):
        _record_import_issue(
            practice=self.practice,
            task_id="test-task-1",
            dentally_patient=self._dentally_patient(88001),
            failure_type="duplicate",
            error_message="duplicate key value violates unique constraint",
            conflict_details={"conflict_type": "phone_only", "existing_patient_id": 42},
            existing_patient_id=42,
            primary_phone="+447700900123",
        )

        issue = DataQualityIssue.objects.get(practice=self.practice, dentally_patient_id=88001)
        self.assertEqual(issue.classifier, "duplicate_import")
        self.assertEqual(issue.source, "django_sync")
        self.assertEqual(issue.status, "open")
        self.assertEqual(issue.record_type, "dentally_patient")
        self.assertEqual(issue.record_id, "88001")
        self.assertEqual(issue.detail["conflict_details"]["existing_patient_id"], 42)
        self.assertEqual(issue.detail["first_name"], "Jane")
        self.assertEqual(issue.detail["phone_number"], "+447700900123")

    def test_validation_failure_type_writes_validation_error_classifier(self):
        _record_import_issue(
            practice=self.practice,
            task_id="test-task-2",
            dentally_patient=self._dentally_patient(88002),
            failure_type="validation",
            error_message="Validation error: invalid date format",
            conflict_details={"validation_errors": "invalid date format"},
            existing_patient_id=None,
            primary_phone=None,
        )

        issue = DataQualityIssue.objects.get(practice=self.practice, dentally_patient_id=88002)
        self.assertEqual(issue.classifier, "validation_error")

    def test_missing_data_failure_type_writes_missing_data_import_classifier(self):
        _record_import_issue(
            practice=self.practice,
            task_id="test-task-3",
            dentally_patient=self._dentally_patient(88003),
            failure_type="missing_data",
            error_message="Missing required field: last_name",
            conflict_details={},
            existing_patient_id=None,
            primary_phone=None,
        )

        issue = DataQualityIssue.objects.get(practice=self.practice, dentally_patient_id=88003)
        self.assertEqual(issue.classifier, "missing_data_import")

    def test_invalid_phone_failure_type_writes_invalid_phone_import_classifier(self):
        _record_import_issue(
            practice=self.practice,
            task_id="test-task-4",
            dentally_patient=self._dentally_patient(88004),
            failure_type="invalid_phone",
            error_message="Invalid phone",
            conflict_details={"rejected_phones": "not-a-phone"},
            existing_patient_id=None,
            primary_phone=None,
        )

        issue = DataQualityIssue.objects.get(practice=self.practice, dentally_patient_id=88004)
        self.assertEqual(issue.classifier, "invalid_phone_import")

    def test_other_failure_type_writes_other_import_error_classifier(self):
        _record_import_issue(
            practice=self.practice,
            task_id="test-task-5",
            dentally_patient=self._dentally_patient(88005),
            failure_type="other",
            error_message="IntegrityError: some other constraint",
            conflict_details={},
            existing_patient_id=None,
            primary_phone=None,
        )

        issue = DataQualityIssue.objects.get(practice=self.practice, dentally_patient_id=88005)
        self.assertEqual(issue.classifier, "other_import_error")

    def test_calling_twice_with_same_dentally_id_updates_in_place(self):
        # Matches the legacy update_or_create behavior — prevents duplicate rows
        # when a patient fails again on the next sync.
        for error_message in ("first error message", "second error message"):
            _record_import_issue(
                practice=self.practice,
                task_id="test-task-6",
                dentally_patient=self._dentally_patient(88006),
                failure_type="validation",
                error_message=error_message,
                conflict_details={},
                existing_patient_id=None,
                primary_phone=None,
            )

        matching = DataQualityIssue.objects.filter(practice=self.practice, dentally_patient_id=88006)
        self.assertEqual(matching.count(), 1)
        self.assertEqual(matching.first().detail["error_message"], "second error message")
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `source venv/bin/activate && python manage.py test dentallyIntegration.test_record_import_issue --keepdb -v 2`
Expected: FAIL — `ImportError: cannot import name '_record_import_issue' from 'dentallyIntegration.tasks'`.

- [ ] **Step 3: Write the helper function**

Add this function to `dentallyIntegration/tasks.py`, near the top of the file (after
imports, before `migrate_all_dentally_patients` — check the existing import block at
the top of the file for where `logger`/`shared_task` etc. are already imported, and add
`from dataQuality.models import DataQualityIssue` alongside the other model imports at
module level, since this helper is called on every failed row and a per-call import
would be wasteful):

```python
# Add near the top of dentallyIntegration/tasks.py, with the other model imports:
from dataQuality.models import DataQualityIssue

# Add this function before migrate_all_dentally_patients:

_FAILURE_TYPE_TO_CLASSIFIER = {
    "duplicate": "duplicate_import",
    "validation": "validation_error",
    "missing_data": "missing_data_import",
    "invalid_phone": "invalid_phone_import",
    "other": "other_import_error",
}


def _record_import_issue(
    practice,
    task_id,
    dentally_patient,
    failure_type,
    error_message,
    conflict_details,
    existing_patient_id,
    primary_phone,
):
    """
    Record a failed Dentally patient import as a DataQualityIssue. Replaces the old
    DentallyImportFailure/DentallyImportError split — every failure_type now maps to
    one classifier on one shared model (see dataQuality/models.py).

    Upserts on (practice, dentally_patient_id) — same key the legacy
    update_or_create used — so a patient that fails again on the next sync updates
    the existing row instead of creating a duplicate.
    """
    classifier = _FAILURE_TYPE_TO_CLASSIFIER.get(failure_type, "other_import_error")
    dentally_id = dentally_patient.get("id")

    detail = {
        "migration_task_id": task_id,
        "first_name": dentally_patient.get("first_name", ""),
        "last_name": dentally_patient.get("last_name", ""),
        "email": dentally_patient.get("email_address"),
        "phone_number": primary_phone,
        "existing_patient_id": existing_patient_id,
        "error_message": error_message,
        "conflict_details": conflict_details,
        "dentally_data": dentally_patient,
    }

    try:
        DataQualityIssue.objects.update_or_create(
            practice=practice,
            dentally_patient_id=dentally_id,
            defaults={
                "classifier": classifier,
                "source": "django_sync",
                "record_type": "dentally_patient",
                "record_id": str(dentally_id),
                "detail": detail,
            },
        )
    except Exception as db_error:
        logger.warning(
            f"[Migration {task_id}] Could not save DataQualityIssue for patient {dentally_id}: {db_error}"
        )
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `python manage.py test dentallyIntegration.test_record_import_issue --keepdb -v 2`
Expected: `Ran 6 tests in ...s` / `OK`

- [ ] **Step 5: Commit**

```bash
git add dentallyIntegration/tasks.py dentallyIntegration/test_record_import_issue.py
git commit -m "feat(dataQuality): add _record_import_issue helper writing to DataQualityIssue"
```

(Leave the actual commit to the user per project convention — do not run `git commit`
yourself unless explicitly asked.)

---

### Task 2: Wire the helper into the migration task, remove the legacy write

**Files:**
- Test: `dentallyIntegration/test_migration_writes_dataquality_issue.py` (new)
- Modify: `dentallyIntegration/tasks.py:1167-1213`

- [ ] **Step 1: Write the failing integration test**

This reuses the exact test harness already proven in
`dentallyIntegration/test_migration_no_fallback_merge.py` (mock
`DentallyAPIService.get_patients`, mock `celery.result.AsyncResult`, run
`migrate_all_dentally_patients.apply(...)`).

**Execution-time correction:** the plan as originally written tried to trigger a
duplicate-phone DB constraint violation (two Dentally patients sharing a phone number).
That does not reproduce — empirically verified both via the task's own `progress` dict
(`created_count: 2, failed_count: 0`, no failure at all) and by reading migration 0068:
the `unique_contact_phone_per_practice`/`unique_contact_email_per_practice` constraints
that code's string-matching (`"duplicate key" in error_message.lower()`) was written
against belong to the `ContactIdentity` model, which has since been retired in favor of
Person/PersonChannel/ContactChannel — two Persons sharing a channel is now normal
household behavior, not a DB error. `_record_import_issue`'s classifier mapping for
`failure_type="duplicate"` is already covered directly (and correctly, since it doesn't
depend on how the failure was triggered) by Task 1's unit tests — this integration test
only needs to prove the WIRING, so it uses a still-live failure path instead: an empty
`last_name` fails `PatientSerializer.is_valid()` (`CharField(required=True)`,
`allow_blank=False` by default — confirmed via `manage.py shell`), raising
`"Validation error: ..."`, which is the exact same except-block the duplicate path would
have gone through.

```python
# dentallyIntegration/test_migration_writes_dataquality_issue.py
from unittest.mock import MagicMock, patch

from django.test import TestCase, override_settings

from UserAuthentication.models import Practice
from dataQuality.models import DataQualityIssue

from .models import DentallyIntegration, DentallyImportFailure, DentallyImportError
from .tasks import migrate_all_dentally_patients


def _fake_dentally_patient(dentally_id, first_name, last_name, phone, email=None):
    return {
        "id": dentally_id,
        "first_name": first_name,
        "last_name": last_name,
        "email_address": email,
        "mobile_phone": phone,
        "mobile_phone_country": "GB",
        "mobile_phone_normalized": None,
        "home_phone": None,
        "home_phone_country": "GB",
        "work_phone": None,
        "work_phone_country": "GB",
        "marketing": False,
        "family_id": None,
        "created_at": "2026-01-01T00:00:00.000+00:00",
    }


def _fake_patients_page(patients):
    return {"patients": patients, "meta": {"total": len(patients), "current_page": 1}}


@override_settings(SECURE_SSL_REDIRECT=False)
class MigrationWritesDataQualityIssueTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(name="Writer Cutover Test Practice")
        self.integration = DentallyIntegration.objects.create(
            practice=self.practice, environment="uk_eu", is_active=True
        )
        self.integration.api_key = "fake-key-for-tests-only-not-real-1234567890"
        self.integration.save()

    def _run_migration(self, dentally_patients):
        fake_response = _fake_patients_page(dentally_patients)
        with patch(
            "dentallyIntegration.services.DentallyAPIService.get_patients",
            return_value=fake_response,
        ), patch("celery.result.AsyncResult") as mock_async_result:
            mock_async_result.return_value = MagicMock(state="SUCCESS")
            result = migrate_all_dentally_patients.apply(
                args=[self.practice.id],
                kwargs={"filters": {"patient_type": "all"}},
            )
        return result.get()

    def test_invalid_patient_writes_to_dataqualityissue_not_legacy_tables(self):
        # PatientSerializer requires a non-blank last_name (CharField(required=True),
        # allow_blank=False by default) — an empty last_name reliably fails
        # serializer.is_valid() and raises "Validation error: ...", landing in the
        # failure-recording block with failure_type="validation".
        invalid = _fake_dentally_patient(
            dentally_id=88102, first_name="NoLastName", last_name="", phone="07700900789",
        )

        self._run_migration([invalid])

        issue = DataQualityIssue.objects.filter(
            practice=self.practice, dentally_patient_id=88102
        ).first()
        self.assertIsNotNone(issue, "expected a DataQualityIssue row for the invalid patient")
        self.assertEqual(issue.source, "django_sync")

        # Legacy tables must receive NOTHING new from this run.
        self.assertEqual(
            DentallyImportFailure.objects.filter(practice=self.practice).count(), 0
        )
        self.assertEqual(
            DentallyImportError.objects.filter(practice=self.practice).count(), 0
        )
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `python manage.py test dentallyIntegration.test_migration_writes_dataquality_issue --keepdb -v 2`
Expected: FAIL — the `DataQualityIssue` row won't exist yet (the task still writes to
the legacy tables at this point), so
`self.assertIsNotNone(issue, ...)` fails.

**Local environment note, unrelated to this plan's correctness:** running Django tests
forces `settings.DEBUG = False`, and `dentallyIntegration/encryption.py` requires
`DENTALLY_ENCRYPTION_KEY` to be set whenever `DEBUG` is off — this is a pre-existing gap
(the already-existing `test_migration_no_fallback_merge.py` fails identically without
it, confirmed empirically) not introduced by this plan. Export one before running any
test in this file: `export DENTALLY_ENCRYPTION_KEY=$(python -c 'from cryptography.fernet
import Fernet; print(Fernet.generate_key().decode())')`.

If the test instead errors on `DentallyImportFailure`/`DentallyImportError` collisions
with existing rows from a prior test run, that means `--keepdb` carried over stale data
— scope every assertion to `self.practice` (already done above) rather than global
counts, per the project's known `--keepdb` cross-test-data trap.

- [ ] **Step 3: Replace the legacy write block with a call to the new helper**

In `dentallyIntegration/tasks.py`, replace the `# Save failure to database for
analysis` block (the `try:` through the matching `except Exception as db_error:
logger.warning(...)`) with the call below. Line numbers in this plan (1167-1213) were
the pre-Task-1 positions; Task 1's helper addition shifted this block down by ~62 lines
(to ~1229-1276 as actually executed) — locate the block by its comment/content, not by
line number.

```python
                        # Save failure to database for analysis — unified
                        # DataQualityIssue model (see dataQuality/models.py).
                        _record_import_issue(
                            practice=practice,
                            task_id=task_id,
                            dentally_patient=dentally_patient,
                            failure_type=failure_type,
                            error_message=error_message,
                            conflict_details=conflict_details,
                            existing_patient_id=existing_patient_id,
                            primary_phone=primary_phone,
                        )
```

Do not delete the `DentallyImportFailure`/`DentallyImportError` model definitions or
their migrations in this task — those models still back the (temporarily frozen) legacy
read-side ViewSets until Phase 5 retires them. This task only stops NEW writes.

- [ ] **Step 4: Run the test to verify it passes**

Run: `python manage.py test dentallyIntegration.test_migration_writes_dataquality_issue --keepdb -v 2`
Expected: `OK`

- [ ] **Step 5: Run the full existing migration test suite to confirm no regression**

Run: `python manage.py test dentallyIntegration.test_migration_no_fallback_merge dentallyIntegration.test_record_import_issue dentallyIntegration.test_migration_writes_dataquality_issue --keepdb -v 2`
Expected: all tests `OK` — the pre-existing fallback-merge behavior (a completely
separate code path, "possible duplicate" flagging, not the DB-constraint duplicate path)
is untouched by this change, and both new test files pass.

- [ ] **Step 6: Commit**

```bash
git add dentallyIntegration/tasks.py dentallyIntegration/test_migration_writes_dataquality_issue.py
git commit -m "feat(dataQuality): migration task writes import failures to DataQualityIssue"
```

(Leave the actual commit to the user per project convention.)

---

## Self-review notes

- **Spec coverage**: implements the Django half of Section 2 ("Import-time" writer) in
  full. The Go half is Phase 3, a separate plan.
- **No placeholders**: every step has complete, runnable code.
- **Type consistency**: `_record_import_issue`'s parameter names
  (`practice`, `task_id`, `dentally_patient`, `failure_type`, `error_message`,
  `conflict_details`, `existing_patient_id`, `primary_phone`) match exactly between the
  Task 1 test, the Task 1 implementation, and the Task 2 call site — verified by
  re-reading the real call site in `tasks.py` (lines 1167-1213) before writing the
  replacement code, so the argument list matches variables that actually exist in scope
  at that point in the function (`practice`, `task_id`, `dentally_patient`,
  `failure_type`, `error_message`, `conflict_details`, `existing_patient_id`,
  `primary_phone` are all already local variables at that point in
  `migrate_all_dentally_patients` per the code read during design).
- **Deployment-gap tradeoff flagged explicitly** at the top of this plan (accepted
  per the spec's direct-cutover decision, but must not be silently forgotten between
  Phase 2 and Phase 5 deploys).
- **Deliberately NOT in this plan**: `retry_dentally_import_errors` (tasks.py:1277) and
  `backfill_dentally_failed_imports` (a management command) still read/mutate the legacy
  `DentallyImportError` table. They keep working against historical (pre-cutover) rows
  until Phase 5 repoints them — not broken by this plan, just increasingly stale, which
  is fine since Phase 5 follows shortly after in the rollout order.

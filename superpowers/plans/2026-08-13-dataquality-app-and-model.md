# Data Quality App & Model — Implementation Plan (Phase 1 of 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the new `dataQuality` Django app with the `DataQualityIssue` model —
the single source of truth that Phases 2-5 (Django writer cutover, Go writer cutover,
Go nightly sweep, frontend rewire) all build on.

**Architecture:** One new Django app (`dataQuality`), one model (`DataQualityIssue`)
with a `classifier`/`status`/`source` triad replacing the four legacy systems described
in the design spec. No views, no URLs, no writers yet — this phase only stands up the
schema so later phases have something to repoint at.

**Tech Stack:** Django 5.2, PostgreSQL (shared DB with Go service).

**Spec:** `docs/superpowers/specs/2026-08-13-unified-data-quality-design.md` (Section 1)

---

### Task 1: App scaffold

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/__init__.py`
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/apps.py`
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/models.py`
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/migrations/__init__.py`
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/tests/__init__.py`
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py:108` (INSTALLED_APPS)

- [ ] **Step 1: Create the app directory skeleton**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
mkdir -p dataQuality/migrations dataQuality/tests
touch dataQuality/__init__.py dataQuality/migrations/__init__.py dataQuality/tests/__init__.py
```

- [ ] **Step 2: Write `apps.py`**

```python
# dataQuality/apps.py
from django.apps import AppConfig


class DataqualityConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "dataQuality"
```

- [ ] **Step 3: Register the app in `settings.py`**

In `TreatmentPath/settings.py`, add `"dataQuality",` to `INSTALLED_APPS` immediately
after the `"dentallyIntegration"` line (~line 108), since this app's model references
`dentallyIntegration`-originated data conceptually (though not via FK):

```python
    "dentallyIntegration",  # Dentally API integration for practice management systems
    "dataQuality",  # Unified data-quality issue tracking (duplicates, errors, missing info)
```

- [ ] **Step 4: Verify the app is recognized**

Run: `source venv/bin/activate && python manage.py check`
Expected: `System check identified no issues (0 silenced).` — confirms `dataQuality` is
importable and registered with no errors, even though it has no models yet.

---

### Task 2: `DataQualityIssue` model — write the failing test first

**Files:**
- Test: `TreatmentPathBackend/TreatmentPath/dataQuality/tests/test_models.py`
- Modify: `TreatmentPathBackend/TreatmentPath/dataQuality/models.py`
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/migrations/0001_initial.py`

- [ ] **Step 1: Write the failing test**

```python
# dataQuality/tests/test_models.py
from django.test import TestCase
from django.utils import timezone

from UserAuthentication.models import Practice, User
from dataQuality.models import DataQualityIssue


class DataQualityIssueModelTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(name="Test Practice")

    def test_create_import_time_issue_with_dentally_patient_id(self):
        issue = DataQualityIssue.objects.create(
            practice=self.practice,
            classifier="duplicate_import",
            source="django_sync",
            record_type="patient",
            record_id="123",
            dentally_patient_id=987654,
        )
        self.assertEqual(issue.status, "open")  # default
        self.assertIsNone(issue.resolved_at)
        self.assertEqual(str(issue.dentally_patient_id), "987654")

    def test_create_scan_based_issue_without_dentally_patient_id(self):
        # duplicate_contact / missing_info are not Dentally-import-specific —
        # dentally_patient_id must be nullable, not required.
        issue = DataQualityIssue.objects.create(
            practice=self.practice,
            classifier="duplicate_contact",
            source="go_sweep",
            record_type="person_cluster",
            record_id="channel-4521",
            detail={"member_person_ids": [111, 222]},
        )
        self.assertIsNone(issue.dentally_patient_id)
        self.assertEqual(issue.detail["member_person_ids"], [111, 222])

    def test_status_transition_to_resolved(self):
        # User has no direct `practice` field — practice membership goes through
        # `current_practice` (FK) / `practices` (M2M via UserPracticeRelationship).
        # Neither is needed here: `resolved_by` is a plain FK to User.
        user = User.objects.create(email="staff@example.com")
        issue = DataQualityIssue.objects.create(
            practice=self.practice,
            classifier="missing_info",
            source="go_sweep",
            record_type="patient",
            record_id="55",
        )
        issue.status = "resolved"
        issue.resolved_by = user
        issue.resolved_at = timezone.now()
        issue.save()

        issue.refresh_from_db()
        self.assertEqual(issue.status, "resolved")
        self.assertEqual(issue.resolved_by, user)
        self.assertIsNotNone(issue.resolved_at)

    def test_classifier_choices_reject_invalid_value(self):
        issue = DataQualityIssue(
            practice=self.practice,
            classifier="not_a_real_classifier",
            source="django_sync",
            record_type="patient",
            record_id="1",
        )
        with self.assertRaises(Exception):
            issue.full_clean()

    def test_practice_and_classifier_and_status_index_query(self):
        DataQualityIssue.objects.create(
            practice=self.practice, classifier="missing_info", source="go_sweep",
            record_type="patient", record_id="1",
        )
        DataQualityIssue.objects.create(
            practice=self.practice, classifier="missing_info", source="go_sweep",
            record_type="patient", record_id="2", status="dismissed",
        )
        open_issues = DataQualityIssue.objects.filter(
            practice=self.practice, classifier="missing_info", status="open"
        )
        self.assertEqual(open_issues.count(), 1)
```

Note the test file imports `Practice`/`User` from `UserAuthentication.models` and
constructs a `Practice` with just `name=` — check
`TreatmentPathBackend/TreatmentPath/UserAuthentication/models.py` for the actual
required fields on `Practice`/`User` before running this; if `Practice.objects.create`
needs more required fields, add them to `setUp` (do not guess — read the model).

- [ ] **Step 2: Run the test to verify it fails**

Run: `source venv/bin/activate && python manage.py test dataQuality.tests.test_models --keepdb`
Expected: FAIL / ERROR — `ModuleNotFoundError: No module named 'dataQuality.models'`
or `ImportError: cannot import name 'DataQualityIssue'` (model doesn't exist yet).

**Do NOT use `--noinput`** — the project's test DB build is fragile from fresh
(see project memory `project_test_db_fresh_build_broken`); always run with `--keepdb`.

- [ ] **Step 3: Write the model**

```python
# dataQuality/models.py
from django.conf import settings
from django.db import models


class DataQualityIssue(models.Model):
    """
    One row per detected data-quality problem for a practice — duplicate/invalid
    records surfaced during a Dentally import, or duplicate-contact clusters and
    missing-info gaps found by the nightly scan. Replaces the legacy
    DentallyImportFailure / DentallyImportError / DuplicateClusterDismissal models
    and the ad hoc per-tab dismissal mechanisms they each had.

    See docs/superpowers/specs/2026-08-13-unified-data-quality-design.md, Section 1.
    """

    CLASSIFIER_CHOICES = [
        # import-time — event-driven, written by Django's Celery sync task AND Go's
        # migration importer (EmailServiceGo/internal/dentally/migration/service.go)
        ("duplicate_import", "Duplicate (import)"),
        ("validation_error", "Validation error"),
        ("missing_data_import", "Missing data (import)"),
        ("invalid_phone_import", "Invalid phone (import)"),
        ("other_import_error", "Other import error"),
        # scan-based — written nightly by Go's cron sweep only
        ("duplicate_contact", "Duplicate contact"),
        ("missing_info", "Missing info"),
    ]
    STATUS_CHOICES = [
        ("open", "Open"),
        ("dismissed", "Dismissed"),
        ("resolved", "Resolved"),
    ]
    SOURCE_CHOICES = [
        ("django_sync", "Django sync"),
        ("go_migration", "Go migration"),
        ("go_sweep", "Go sweep"),
    ]

    # Classifiers that originate from a specific Dentally import attempt and carry
    # a dentally_patient_id. Scan-based classifiers (duplicate_contact, missing_info)
    # are deliberately excluded — they scan every Patient/Intake regardless of
    # Dentally origin, including manually-created contacts.
    IMPORT_TIME_CLASSIFIERS = {
        "duplicate_import",
        "validation_error",
        "missing_data_import",
        "invalid_phone_import",
        "other_import_error",
    }

    practice = models.ForeignKey(
        "UserAuthentication.Practice",
        on_delete=models.CASCADE,
        related_name="data_quality_issues",
    )

    classifier = models.CharField(max_length=32, choices=CLASSIFIER_CHOICES)
    # db_default (not just Python default=) so a Go INSERT that omits this column
    # (e.g. the nightly sweep writer, which always sets it explicitly anyway, or any
    # future writer that forgets to) still lands as "open" at the DB level instead of
    # failing NOT NULL — mirrors this codebase's established db_default pattern for
    # every other Go-written table's naturally-defaultable columns (see
    # EmailServiceGo/internal/db/schemaguard.go header comment).
    status = models.CharField(
        max_length=16, choices=STATUS_CHOICES, default="open", db_default="open"
    )
    source = models.CharField(max_length=16, choices=SOURCE_CHOICES)

    # Generic subject reference. "patient"/"intake" for import-issue and missing_info
    # rows (record_id = that row's pk, as a string). "person_cluster" for
    # duplicate_contact rows (record_id = a synthetic cluster key — the shared
    # channel id — since an N-member cluster has no single natural FK; member
    # Person ids live in `detail`).
    record_type = models.CharField(max_length=16)
    record_id = models.CharField(max_length=64)

    detail = models.JSONField(default=dict, blank=True)

    # Only meaningful for IMPORT_TIME_CLASSIFIERS — null for duplicate_contact and
    # missing_info. Doubles as the upsert key
    # (practice_id, dentally_patient_id) the Celery task and Go migration importer
    # already use today to update-in-place rather than duplicate a row on retry.
    dentally_patient_id = models.BigIntegerField(null=True, blank=True)

    resolved_by = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name="resolved_data_quality_issues",
    )
    resolved_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        # Explicit all-lowercase table name, matching the existing convention for
        # every other Go-written table in this codebase (e.g.
        # dentallyIntegration.DentallyImportFailure uses
        # db_table="dentallyintegration_dentallyimportfailure", NOT Django's
        # mixed-case default) — confirmed live via \dt against treatmentpath_db.
        # Go's raw SQL/GORM .Table() calls (Phase 3) reference this table by exact
        # string, so it must be fixed and lowercase, not left to Django's default.
        db_table = "dataquality_dataqualityissue"
        indexes = [
            models.Index(fields=["practice", "classifier", "status"]),
            models.Index(fields=["practice", "dentally_patient_id"]),
        ]
        ordering = ["-created_at"]

    def __str__(self):
        return f"{self.classifier} ({self.status}) — practice {self.practice_id}"
```

- [ ] **Step 4: Generate the migration**

```bash
source venv/bin/activate
python manage.py makemigrations dataQuality
```

Expected output: `Migrations for 'dataQuality': dataQuality/migrations/0001_initial.py`.
Open the generated file and confirm its `dependencies` list includes the current
`UserAuthentication` migration head — check with:

```bash
ls UserAuthentication/migrations/ | tail -3
```

and confirm the generated migration's dependency matches the latest one listed (do not
hand-edit unless `makemigrations` picked a stale dependency — it shouldn't, but this
project has a known history of migration-numbering gaps, so verify rather than assume).

- [ ] **Step 5: Apply the migration locally**

```bash
python manage.py migrate dataQuality
```

Expected: `Applying dataQuality.0001_initial... OK`

- [ ] **Step 6: Run the test to verify it passes**

Run: `python manage.py test dataQuality.tests.test_models --keepdb -v 2`
Expected: `Ran 5 tests in ...s` / `OK`

`Practice.objects.create(name=...)` and `User.objects.create(email=...)` are confirmed
against the live `UserAuthentication/models.py` (`Practice.save()` auto-generates `slug`
from `name`; `User` has no required fields beyond `email` — `USERNAME_FIELD = "email"`,
`REQUIRED_FIELDS = []`), so this should pass on the first run. If it doesn't, something
changed in `UserAuthentication.models` since this plan was written — re-read that file
before adjusting the test.

- [ ] **Step 7: Commit**

```bash
git add dataQuality/ TreatmentPath/settings.py
git commit -m "feat(dataQuality): add DataQualityIssue model and app scaffold"
```

(Per project convention, this repo's VCS is user-managed — leave this step for the user
to run rather than executing `git commit` yourself, unless explicitly asked to commit.)

---

### Task 3: Admin registration (operational visibility)

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/admin.py`

- [ ] **Step 1: Write the failing test**

```python
# dataQuality/tests/test_admin.py
from django.contrib import admin
from django.test import TestCase

from dataQuality.models import DataQualityIssue


class DataQualityAdminTests(TestCase):
    def test_model_is_registered(self):
        self.assertIn(DataQualityIssue, admin.site._registry)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python manage.py test dataQuality.tests.test_admin --keepdb -v 2`
Expected: FAIL — `AssertionError` (not registered yet).

- [ ] **Step 3: Register the model**

```python
# dataQuality/admin.py
from django.contrib import admin

from .models import DataQualityIssue


@admin.register(DataQualityIssue)
class DataQualityIssueAdmin(admin.ModelAdmin):
    list_display = ("id", "practice", "classifier", "status", "source", "created_at")
    list_filter = ("classifier", "status", "source", "practice")
    search_fields = ("record_id", "dentally_patient_id")
    readonly_fields = ("created_at", "updated_at")
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python manage.py test dataQuality.tests.test_admin --keepdb -v 2`
Expected: `OK`

- [ ] **Step 5: Commit**

```bash
git add dataQuality/admin.py dataQuality/tests/test_admin.py
git commit -m "feat(dataQuality): register DataQualityIssue in admin"
```

(Same note as Task 2 Step 7 — leave the actual commit to the user.)

---

## Self-review notes

- **Spec coverage**: this plan implements Section 1 (Data model) of the spec in full,
  including the nullable `dentally_patient_id` clarification from the design
  conversation. It deliberately does NOT implement Sections 2-5 (writers, schema guard,
  actions, frontend) — those are Phases 2-5, separate plan documents per the user's
  choice to split by rollout phase.
- **No placeholders**: every step has complete, runnable code — no "add appropriate
  fields" or "similar to Task 2" shorthand.
- **Type consistency**: `classifier`, `status`, `source`, `record_type`, `record_id`,
  `detail`, `dentally_patient_id`, `resolved_by`, `resolved_at` are named identically
  between the model definition (Task 2 Step 3) and every test that references them
  (Task 2 Step 1) — verified by re-reading both before finalizing this plan.
- **Added during Phase 3 planning**: `status` got `db_default="open"` alongside its
  existing `default="open"`, so it's drift-safe per this codebase's established
  schema-guard pattern (see `schemaguard.go` header) and doesn't need registering in
  Go's `requiredColumns` snapshot.
- **Added during Phase 3 planning**: `Meta.db_table = "dataquality_dataqualityissue"`
  (explicit, lowercase) was added after discovering — while writing Phase 3 — that
  every other Go-written table in this app-family uses an explicit lowercase
  `db_table` override rather than Django's mixed-case default. Confirmed live against
  `treatmentpath_db` (`dentallyintegration_dentallyimportfailure` exists lowercase;
  Django's un-overridden default would have been mixed-case `dentallyIntegration_...`).
  Without this override, Phase 3's Go raw SQL would target a table name that doesn't
  exist.
- **Verified against live code**: `Practice`/`User` field usage in Task 2's test was
  checked against the real `UserAuthentication/models.py` (not guessed) — `Practice`
  auto-slugs from `name` on save, `User` requires only `email`. The initial draft
  incorrectly used a nonexistent `User(practice=...)` kwarg (the real field is
  `current_practice`); fixed to `User.objects.create(email=...)` since `resolved_by`
  doesn't need any practice relationship for this test.

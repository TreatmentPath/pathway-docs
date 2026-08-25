# Data Quality Actions & Frontend Rewire — Implementation Plan (Phase 5 of 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `dataQuality` app's read/action API against `DataQualityIssue`
(replacing `import_views.py`'s two ViewSets and `data_quality_views.py`'s five views),
then rewire the frontend's four sub-tabs into one classifier-driven screen, then retire
the legacy models.

**Architecture:** One DRF ViewSet (`DataQualityIssueViewSet`) handles list/filter/dismiss
generically across all seven classifiers. Classifier-specific actions (merge, edit,
delete, force-import, replace-existing, retry) are separate `@action` methods on the
same ViewSet, each doing the SAME business logic the legacy views already do (patient
creation, phone parsing, `Person.merge`) — only the read source (`issue.detail` JSON
instead of model attributes) and the completion side-effect (`status="resolved"`
instead of `.delete()`) change. A shared helper module extracts the patient-field-mapping
logic duplicated three times in the legacy code (`force_import`, `replace_existing`,
`create_patient_from_error` are ~90% identical) into one function, ported once.

**Tech Stack:** Django REST Framework, React, TypeScript.

**Spec:** `docs/superpowers/specs/2026-08-13-unified-data-quality-design.md` (Sections
4-5, Section 6 step 5)

**Prerequisites:** Phases 1-4 applied. This plan reads `DataQualityIssue` rows that
Phases 2-4's writers have been populating.

---

### Task 1: Shared patient-field-extraction helper

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/patient_import_utils.py`
- Test: `TreatmentPathBackend/TreatmentPath/dataQuality/tests/test_patient_import_utils.py`

- [ ] **Step 1: Write the failing test**

```python
# dataQuality/tests/test_patient_import_utils.py
from django.test import SimpleTestCase

from dataQuality.patient_import_utils import build_patient_data_from_detail


class BuildPatientDataFromDetailTests(SimpleTestCase):
    def test_prefers_normalized_phone_over_raw(self):
        # Execution-time correction #1: parse_phone_number (TreatmentPlan/utils/
        # phones.py) returns country_code and phone_number SEPARATELY —
        # phone_number is always local digits with leading zeros stripped, never
        # a combined E.164 string. The plan as originally written assumed a
        # combined "+447700900111" result; the real, documented behavior (see the
        # function's own docstring) returns {"country_code": "+44",
        # "phone_number": "7911123111"}. Patient stores them as separate fields
        # anyway, so patient_data already carries both correctly.
        #
        # Execution-time correction #2: uses 07911xxxxxx (the function's own
        # docstring example range), NOT 07700900xxx — Ofcom reserves 07700 900xxx
        # for fiction and libphonenumber's is_valid_number() rejects it, silently
        # routing through the "invalid number" fallback branch instead of real
        # parsing (documented project trap — hit here despite being a known,
        # previously-documented gotcha in this codebase).
        detail = {
            "first_name": "Sam",
            "last_name": "Reed",
            "dentally_data": {
                "mobile_phone_normalized": "+447911123111",
                "mobile_phone": "07911123111",
            },
        }
        data, country_code = build_patient_data_from_detail(detail)
        self.assertEqual(data["phone_number"], "7911123111")
        self.assertEqual(country_code, "+44")

    def test_falls_back_to_raw_mobile_when_no_normalized(self):
        detail = {
            "first_name": "Sam", "last_name": "Reed",
            "dentally_data": {"mobile_phone": "07911123222"},
        }
        data, country_code = build_patient_data_from_detail(detail)
        self.assertEqual(data["phone_number"], "7911123222")

    def test_second_phone_becomes_secondary(self):
        detail = {
            "first_name": "Sam", "last_name": "Reed",
            "dentally_data": {
                "mobile_phone": "07911123333",
                "home_phone": "07911123444",
            },
        }
        data, country_code = build_patient_data_from_detail(detail)
        self.assertEqual(data["phone_number"], "7911123333")
        self.assertEqual(data["secondary_phone_number"], "7911123444")

    def test_defaults_country_code_to_gb_when_no_phone(self):
        detail = {"first_name": "Sam", "last_name": "Reed", "dentally_data": {}}
        data, country_code = build_patient_data_from_detail(detail)
        self.assertEqual(country_code, "GB")
        self.assertIsNone(data["phone_number"])

    def test_email_prefers_detail_email_over_dentally_data(self):
        detail = {
            "first_name": "Sam", "last_name": "Reed", "email": "top-level@example.com",
            "dentally_data": {"email_address": "nested@example.com"},
        }
        data, country_code = build_patient_data_from_detail(detail)
        self.assertEqual(data["email"], "top-level@example.com")

    def test_string_none_email_treated_as_none(self):
        detail = {
            "first_name": "Sam", "last_name": "Reed", "email": "None",
            "dentally_data": {},
        }
        data, country_code = build_patient_data_from_detail(detail)
        self.assertIsNone(data["email"])

    def test_meta_data_carries_full_dentally_payload(self):
        dentally_data = {"id": 12345, "first_name": "Sam"}
        detail = {"first_name": "Sam", "last_name": "Reed", "dentally_data": dentally_data}
        data, _ = build_patient_data_from_detail(detail)
        self.assertEqual(data["meta_data"], dentally_data)
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `python manage.py test dataQuality.tests.test_patient_import_utils --keepdb -v 2`
Expected: FAIL — `ModuleNotFoundError: No module named 'dataQuality.patient_import_utils'`.

- [ ] **Step 3: Write the helper**

Ports the identical phone/email-extraction block duplicated across `force_import`,
`replace_existing`, and `create_patient_from_error`
(`dentallyIntegration/views/import_views.py`) into one function, reading from
`DataQualityIssue.detail` instead of model attributes.

```python
# dataQuality/patient_import_utils.py
from TreatmentPlan.utils import parse_phone_number


def _clean_phone(value):
    """Convert 'None' string, None, and empty strings to None."""
    if value is None or value == "" or value == "None":
        return None
    return value


def build_patient_data_from_detail(detail):
    """
    Build the patient_data dict + resolved country_code from a DataQualityIssue's
    `detail` JSON, exactly matching the phone-preference and country-code logic
    duplicated across the legacy force_import/replace_existing/create_patient_from_error
    actions (dentallyIntegration/views/import_views.py) — normalized phone preferred
    over raw, mobile > home > work, first available phone becomes primary, second
    becomes secondary, default country code GB when nothing parses.

    `detail` is the DataQualityIssue.detail JSON written by _record_import_issue
    (Django) / persistImportIssue (Go) — has "first_name", "last_name", "email",
    "phone_number", "dentally_data" (the raw Dentally payload) among its keys.

    Returns (patient_data: dict, country_code: str).
    """
    dentally_data = detail.get("dentally_data") or {}

    mobile_phone = _clean_phone(dentally_data.get("mobile_phone_normalized")) or _clean_phone(
        dentally_data.get("mobile_phone")
    )
    home_phone = _clean_phone(dentally_data.get("home_phone_normalized")) or _clean_phone(
        dentally_data.get("home_phone")
    )
    work_phone = _clean_phone(dentally_data.get("work_phone_normalized")) or _clean_phone(
        dentally_data.get("work_phone")
    )

    parsed_phones = []
    country_code = None
    for raw in (mobile_phone, home_phone, work_phone):
        if not raw:
            continue
        parsed = parse_phone_number(raw, default_country_code="GB")
        if parsed and parsed.get("phone_number"):
            parsed_phones.append(parsed)
            if not country_code and parsed.get("country_code"):
                country_code = parsed["country_code"]

    primary_phone = parsed_phones[0]["phone_number"] if parsed_phones else None
    secondary_phone = parsed_phones[1]["phone_number"] if len(parsed_phones) > 1 else None
    if not country_code:
        country_code = "GB"

    email_address = detail.get("email") or dentally_data.get("email_address")
    if email_address in (None, "None", ""):
        email_address = None

    patient_data = {
        "first_name": detail.get("first_name") or dentally_data.get("first_name", ""),
        "last_name": detail.get("last_name") or dentally_data.get("last_name", ""),
        "email": email_address,
        "phone_number": primary_phone,
        "secondary_phone_number": secondary_phone,
        "country_code": country_code,
        "meta_data": dentally_data,
    }
    return patient_data, country_code
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `python manage.py test dataQuality.tests.test_patient_import_utils --keepdb -v 2`
Expected: `OK` (7 tests).

- [ ] **Step 5: Commit**

```bash
git add dataQuality/patient_import_utils.py dataQuality/tests/test_patient_import_utils.py
git commit -m "feat(dataQuality): shared patient-field extraction from DataQualityIssue detail"
```

(Leave the actual commit to the user per project convention — repeat this note applies
to every "Commit" step in this plan; not restated per-task below.)

---

### Task 2: `DataQualityIssueViewSet` — list, filter, dismiss

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/serializers.py`
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/views.py`
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/urls.py`
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPath/urls.py` (include the new app)
- Test: `TreatmentPathBackend/TreatmentPath/dataQuality/tests/test_views.py`

- [ ] **Step 1: Write the failing test**

```python
# dataQuality/tests/test_views.py
from django.test import TestCase
from rest_framework.test import APIClient

from UserAuthentication.models import Practice, User
from dataQuality.models import DataQualityIssue


class DataQualityIssueViewSetTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(name="DQ Views Test Practice")
        self.user = User.objects.create(email="staff@example.com", current_practice=self.practice)
        self.user.practices.add(self.practice)
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)

    def test_list_returns_open_issues_for_practice(self):
        DataQualityIssue.objects.create(
            practice=self.practice, classifier="missing_info", source="go_sweep",
            record_type="patient", record_id="1",
        )
        DataQualityIssue.objects.create(
            practice=self.practice, classifier="missing_info", source="go_sweep",
            record_type="patient", record_id="2", status="dismissed",
        )
        response = self.client.get("/api/backend/data-quality/issues/")
        self.assertEqual(response.status_code, 200)
        results = response.json()["results"]
        self.assertEqual(len(results), 1)
        self.assertEqual(results[0]["record_id"], "1")

    def test_filter_by_classifier(self):
        DataQualityIssue.objects.create(
            practice=self.practice, classifier="missing_info", source="go_sweep",
            record_type="patient", record_id="1",
        )
        DataQualityIssue.objects.create(
            practice=self.practice, classifier="duplicate_contact", source="go_sweep",
            record_type="person_cluster", record_id="channel-1",
        )
        response = self.client.get("/api/backend/data-quality/issues/?classifier=duplicate_contact")
        results = response.json()["results"]
        self.assertEqual(len(results), 1)
        self.assertEqual(results[0]["classifier"], "duplicate_contact")

    def test_dismiss_sets_status_dismissed(self):
        issue = DataQualityIssue.objects.create(
            practice=self.practice, classifier="missing_info", source="go_sweep",
            record_type="patient", record_id="1",
        )
        response = self.client.post(f"/api/backend/data-quality/issues/{issue.id}/dismiss/")
        self.assertEqual(response.status_code, 200)
        issue.refresh_from_db()
        self.assertEqual(issue.status, "dismissed")
        self.assertEqual(issue.resolved_by, self.user)
        self.assertIsNotNone(issue.resolved_at)

    def test_list_scoped_to_practice(self):
        other_practice = Practice.objects.create(name="Other Practice")
        DataQualityIssue.objects.create(
            practice=other_practice, classifier="missing_info", source="go_sweep",
            record_type="patient", record_id="99",
        )
        response = self.client.get("/api/backend/data-quality/issues/")
        self.assertEqual(response.json()["results"], [])
```

Check `UserAuthentication.models.User`/`Practice` for the exact
`current_practice`/`practices` relationship pattern other test files in this codebase
already use to authenticate a practice-scoped request (e.g. grep an existing
`APIClient` + `force_authenticate` test in `TreatmentPlan/tests/` for the established
pattern) before finalizing `setUp` — the shape above matches the fields confirmed in
Phase 1's design work, but the exact practice-resolution mechanism your `PracticeAccessMixin.get_user_practice_or_none()` uses (current_practice vs a request header vs
session) should be double-checked against `utils/practice_mixins.py` since this test's
`setUp` assumes `current_practice` is what it reads.

- [ ] **Step 2: Run the test to verify it fails**

Run: `python manage.py test dataQuality.tests.test_views --keepdb -v 2`
Expected: FAIL — `404` (no URL registered yet) or import errors.

- [ ] **Step 3: Write the serializer**

```python
# dataQuality/serializers.py
from rest_framework import serializers

from .models import DataQualityIssue


class DataQualityIssueSerializer(serializers.ModelSerializer):
    class Meta:
        model = DataQualityIssue
        fields = [
            "id", "classifier", "status", "source", "record_type", "record_id",
            "detail", "dentally_patient_id", "resolved_by", "resolved_at",
            "created_at", "updated_at",
        ]
        read_only_fields = fields
```

- [ ] **Step 4: Write the ViewSet**

```python
# dataQuality/views.py
from django.utils import timezone
from rest_framework import status, viewsets
from rest_framework.decorators import action
from rest_framework.pagination import PageNumberPagination
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response

from utils.practice_mixins import PracticeAccessMixin

from .models import DataQualityIssue
from .serializers import DataQualityIssueSerializer


class DataQualityIssuePagination(PageNumberPagination):
    page_size = 50
    page_size_query_param = "page_size"
    max_page_size = 100


class DataQualityIssueViewSet(PracticeAccessMixin, viewsets.ReadOnlyModelViewSet):
    """
    GET  /api/backend/data-quality/issues/                 — list, filterable by ?classifier=
                                                       and ?status= (defaults to open)
    GET  /api/backend/data-quality/issues/{id}/
    POST /api/backend/data-quality/issues/{id}/dismiss/     — generic dismiss for any classifier
    """

    serializer_class = DataQualityIssueSerializer
    pagination_class = DataQualityIssuePagination
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        practice = self.get_user_practice_or_none()
        if not practice:
            return DataQualityIssue.objects.none()

        qs = DataQualityIssue.objects.filter(practice=practice)

        classifier = self.request.query_params.get("classifier")
        if classifier:
            qs = qs.filter(classifier=classifier)

        status_param = self.request.query_params.get("status", "open")
        if status_param != "all":
            qs = qs.filter(status=status_param)

        return qs

    @action(detail=True, methods=["post"], url_path="dismiss")
    def dismiss(self, request, pk=None):
        issue = self.get_object()
        issue.status = "dismissed"
        issue.resolved_by = request.user if request.user.is_authenticated else None
        issue.resolved_at = timezone.now()
        issue.save(update_fields=["status", "resolved_by", "resolved_at", "updated_at"])
        return Response(DataQualityIssueSerializer(issue).data)
```

- [ ] **Step 5: Wire the URLs**

```python
# dataQuality/urls.py
from django.urls import include, path
from rest_framework.routers import DefaultRouter

from .views import DataQualityIssueViewSet

router = DefaultRouter()
router.register("issues", DataQualityIssueViewSet, basename="data-quality-issue")

urlpatterns = [path("", include(router.urls))]
```

In `TreatmentPath/urls.py`, find where other app URLs are included (e.g. the line
including `dentallyIntegration.urls` or `TreatmentPlan.urls`) and add, following the
same prefix convention already used for other apps:

```python
    path("api/backend/data-quality/", include("dataQuality.urls")),
```

- [ ] **Step 6: Run the test to verify it passes**

Run: `python manage.py test dataQuality.tests.test_views --keepdb -v 2`
Expected: `OK` (4 tests). If `test_list_scoped_to_practice` or any practice-resolution
assertion fails, re-check `PracticeAccessMixin.get_user_practice_or_none()`'s actual
implementation (`utils/practice_mixins.py`) against this test's `setUp` per the note in
Step 1 — fix the test to match the real mechanism, not the other way around.

- [ ] **Step 7: Commit**

---

### Task 3: `duplicate_contact` — merge and dismiss actions

**Files:**
- Modify: `dataQuality/views.py`
- Test: `dataQuality/tests/test_duplicate_contact_actions.py`

- [ ] **Step 1: Write the failing test**

```python
# dataQuality/tests/test_duplicate_contact_actions.py
from django.test import TestCase
from rest_framework.test import APIClient

from TreatmentPlan.models import Patient, Person
from UserAuthentication.models import Practice, User
from dataQuality.models import DataQualityIssue


class MergeDuplicateContactActionTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(name="DQ Merge Test Practice")
        self.user = User.objects.create(email="staff2@example.com", current_practice=self.practice)
        self.user.practices.add(self.practice)
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)

        self.winner_person = Person.objects.create(practice=self.practice, first_name="Jon", last_name="Smith")
        self.loser_person = Person.objects.create(practice=self.practice, first_name="John", last_name="Smith")
        self.winner_patient = Patient.objects.create(
            practice=self.practice, person=self.winner_person, first_name="Jon", last_name="Smith",
        )
        self.loser_patient = Patient.objects.create(
            practice=self.practice, person=self.loser_person, first_name="John", last_name="Smith",
        )

        self.issue = DataQualityIssue.objects.create(
            practice=self.practice, classifier="duplicate_contact", source="go_sweep",
            record_type="person_cluster", record_id="channel-1",
            detail={"member_person_ids": [self.winner_person.id, self.loser_person.id]},
        )

    def test_merge_action_merges_persons_and_resolves_issue(self):
        response = self.client.post(
            f"/api/backend/data-quality/issues/{self.issue.id}/merge/",
            {"winner_person_id": self.winner_person.id, "loser_person_id": self.loser_person.id},
            format="json",
        )
        self.assertEqual(response.status_code, 200)

        self.loser_person.refresh_from_db()
        self.assertEqual(self.loser_person.merged_into_id, self.winner_person.id)

        self.issue.refresh_from_db()
        self.assertEqual(self.issue.status, "resolved")

    def test_merge_rejects_person_ids_not_in_cluster(self):
        outsider = Person.objects.create(practice=self.practice, first_name="Someone", last_name="Else")
        response = self.client.post(
            f"/api/backend/data-quality/issues/{self.issue.id}/merge/",
            {"winner_person_id": self.winner_person.id, "loser_person_id": outsider.id},
            format="json",
        )
        self.assertEqual(response.status_code, 400)
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `python manage.py test dataQuality.tests.test_duplicate_contact_actions --keepdb -v 2`
Expected: FAIL — `404` (no `merge` action registered).

- [ ] **Step 3: Add the action**

```python
# add to dataQuality/views.py, inside DataQualityIssueViewSet

    @action(detail=True, methods=["post"], url_path="merge")
    def merge(self, request, pk=None):
        """
        Merge two Persons from a duplicate_contact cluster. Body:
        {winner_person_id, loser_person_id}. Both ids must be members of this
        issue's cluster (issue.detail['member_person_ids']) — refuses arbitrary
        Person ids to prevent merging unrelated records via a stale/tampered request.
        """
        from TreatmentPlan.models import Person

        issue = self.get_object()
        if issue.classifier != "duplicate_contact":
            return Response(
                {"detail": "merge is only valid for duplicate_contact issues."},
                status=status.HTTP_400_BAD_REQUEST,
            )

        winner_id = request.data.get("winner_person_id")
        loser_id = request.data.get("loser_person_id")
        member_ids = set(issue.detail.get("member_person_ids", []))
        if winner_id not in member_ids or loser_id not in member_ids:
            return Response(
                {"detail": "Both persons must belong to this issue's cluster."},
                status=status.HTTP_400_BAD_REQUEST,
            )

        practice = self.get_user_practice_or_none()
        try:
            winner = Person.objects.get(pk=winner_id, practice=practice)
            loser = Person.objects.get(pk=loser_id, practice=practice)
        except Person.DoesNotExist:
            return Response({"detail": "Person not found."}, status=status.HTTP_404_NOT_FOUND)

        performed_by = request.user if request.user.is_authenticated else None
        log = Person.merge(winner, loser, performed_by=performed_by)

        issue.status = "resolved"
        issue.resolved_by = performed_by
        issue.resolved_at = timezone.now()
        issue.save(update_fields=["status", "resolved_by", "resolved_at", "updated_at"])

        return Response({
            "merged": bool(log),
            "merge_log_id": log.id if log else None,
            "person_id": winner.id,
        })
```

`Person.merge(winner, loser, performed_by=...)` is the existing, already-correct
non-destructive merge (confirmed working in the earlier merge investigation — see
`[[project-merge-never-completes]]`) — reused as-is, not reimplemented.

- [ ] **Step 4: Run the test to verify it passes**

Run: `python manage.py test dataQuality.tests.test_duplicate_contact_actions --keepdb -v 2`
Expected: `OK` (2 tests).

- [ ] **Step 5: Commit**

---

### Task 4: `missing_info` — edit and delete actions

**Files:**
- Modify: `dataQuality/views.py`
- Test: `dataQuality/tests/test_missing_info_actions.py`

- [ ] **Step 1: Write the failing test**

```python
# dataQuality/tests/test_missing_info_actions.py
from django.test import TestCase
from rest_framework.test import APIClient

from TreatmentPlan.models import Patient, Person
from UserAuthentication.models import Practice, User
from dataQuality.models import DataQualityIssue


class MissingInfoActionTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(name="DQ Missing Info Test Practice")
        self.user = User.objects.create(email="staff3@example.com", current_practice=self.practice)
        self.user.practices.add(self.practice)
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)

        person = Person.objects.create(practice=self.practice)
        self.patient = Patient.objects.create(
            practice=self.practice, person=person, first_name="No", last_name="Contact",
        )
        self.issue = DataQualityIssue.objects.create(
            practice=self.practice, classifier="missing_info", source="go_sweep",
            record_type="patient", record_id=str(self.patient.id),
        )

    def test_update_contact_sets_email_and_resolves_issue(self):
        response = self.client.post(
            f"/api/backend/data-quality/issues/{self.issue.id}/update-contact/",
            {"email": "now-reachable@example.com"},
            format="json",
        )
        self.assertEqual(response.status_code, 200)

        self.patient.refresh_from_db()
        self.assertEqual(self.patient.email, "now-reachable@example.com")

        self.issue.refresh_from_db()
        self.assertEqual(self.issue.status, "resolved")

    def test_delete_contact_deletes_record_and_resolves_issue(self):
        response = self.client.delete(f"/api/backend/data-quality/issues/{self.issue.id}/delete-contact/")
        self.assertEqual(response.status_code, 204)

        self.assertFalse(Patient.objects.filter(id=self.patient.id).exists())
        self.issue.refresh_from_db()
        self.assertEqual(self.issue.status, "resolved")
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `python manage.py test dataQuality.tests.test_missing_info_actions --keepdb -v 2`
Expected: FAIL — `404` (actions don't exist yet).

- [ ] **Step 3: Add the actions**

```python
# add to dataQuality/views.py, inside DataQualityIssueViewSet

    def _get_record_for_issue(self, issue, practice):
        from TreatmentPlan.models import Intake, Patient

        model = {"patient": Patient, "intake": Intake}.get(issue.record_type)
        if not model:
            return None
        return model.objects.filter(pk=issue.record_id, practice=practice).first()

    @action(detail=True, methods=["post"], url_path="update-contact")
    def update_contact(self, request, pk=None):
        """Edit a missing_info record's email/phone, resolving the issue if the
        edit actually clears the missing-info condition."""
        issue = self.get_object()
        if issue.classifier != "missing_info":
            return Response(
                {"detail": "update-contact is only valid for missing_info issues."},
                status=status.HTTP_400_BAD_REQUEST,
            )

        practice = self.get_user_practice_or_none()
        record = self._get_record_for_issue(issue, practice)
        if not record:
            return Response({"detail": "Record not found."}, status=status.HTTP_404_NOT_FOUND)

        email = request.data.get("email")
        phone = request.data.get("phone_number")
        update_fields = []
        if email is not None:
            record.email = email
            update_fields.append("email")
        if phone is not None:
            record.phone_number = phone
            update_fields.append("phone_number")
        if update_fields:
            record.save(update_fields=update_fields)

        if (record.email or "").strip() or (record.phone_number or "").strip():
            issue.status = "resolved"
            issue.resolved_by = request.user if request.user.is_authenticated else None
            issue.resolved_at = timezone.now()
            issue.save(update_fields=["status", "resolved_by", "resolved_at", "updated_at"])

        return Response({"detail": "Contact updated.", "status": issue.status})

    @action(detail=True, methods=["delete"], url_path="delete-contact")
    def delete_contact(self, request, pk=None):
        """Delete the underlying Patient/Intake record — destructive by design
        (mirrors DataQualityDeleteView), and resolves the issue."""
        issue = self.get_object()
        practice = self.get_user_practice_or_none()
        record = self._get_record_for_issue(issue, practice)
        if record:
            record.delete()

        issue.status = "resolved"
        issue.resolved_by = request.user if request.user.is_authenticated else None
        issue.resolved_at = timezone.now()
        issue.save(update_fields=["status", "resolved_by", "resolved_at", "updated_at"])

        return Response(status=status.HTTP_204_NO_CONTENT)
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `python manage.py test dataQuality.tests.test_missing_info_actions --keepdb -v 2`
Expected: `OK` (2 tests).

- [ ] **Step 5: Commit**

---

### Task 5: `duplicate_import` — force-import, family-import, replace-existing, override, discard

**Files:**
- Modify: `dataQuality/views.py`
- Test: `dataQuality/tests/test_duplicate_import_actions.py`

This ports `force_import`, `replace_existing`, and `discard`
(`dentallyIntegration/views/import_views.py` lines 272-700) exactly, using Task 1's
`build_patient_data_from_detail` for the field-mapping block that was duplicated across
all three, and reading `existing_patient_id` / `dentally_patient_id` from
`issue.detail`/`issue.dentally_patient_id` instead of model attributes.

- [ ] **Step 1: Write the failing test**

```python
# dataQuality/tests/test_duplicate_import_actions.py
from django.test import TestCase
from rest_framework.test import APIClient

from TreatmentPlan.models import Patient, Person
from UserAuthentication.models import Practice, User
from dataQuality.models import DataQualityIssue


class DuplicateImportActionTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(name="DQ Import Test Practice")
        self.user = User.objects.create(email="staff4@example.com", current_practice=self.practice)
        self.user.practices.add(self.practice)
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)

        # No phone/email on this fixture on purpose: TreatmentPlan.contact.signals
        # .set_patient_person (a pre_save signal on Patient) auto-resolves/
        # OVERRIDES the person= passed to Patient.objects.create()/
        # PatientSerializer.save() whenever contact info is present — it only
        # preserves an explicit person on UPDATE, never CREATE (confirmed by
        # reading _preserve_person_and_link_channels). With no phone/email at all,
        # the signal has nothing to resolve and leaves the explicit person alone,
        # which is what these non-family tests need. See
        # _make_family_existing_patient below for the family-link test, which
        # needs the opposite (a real shared channel).
        person = Person.objects.create(practice=self.practice)
        self.existing_patient = Patient.objects.create(
            practice=self.practice, person=person, first_name="Existing", last_name="Patient",
            meta_data={"id": 55001, "created_at": "2020-01-01T00:00:00.000+00:00"},
        )

        self.issue = DataQualityIssue.objects.create(
            practice=self.practice, classifier="duplicate_import", source="django_sync",
            record_type="dentally_patient", record_id="55002", dentally_patient_id=55002,
            detail={
                "first_name": "Incoming", "last_name": "Patient",
                "existing_patient_id": self.existing_patient.id,
                "dentally_data": {
                    # 07911xxxxxx, not 07700900xxx — Ofcom-reserved fictional
                    # numbers are rejected by libphonenumber's is_valid_number(),
                    # silently changing which parse_phone_number branch runs
                    # (documented project trap, hit repeatedly this session).
                    "id": 55002, "first_name": "Incoming", "last_name": "Patient",
                    "mobile_phone": "07911123555",
                    "created_at": "2026-01-01T00:00:00.000+00:00",
                },
            },
        )

    def _make_family_existing_patient(self):
        """A second existing-patient fixture WITH a phone matching the incoming
        record — only for the family-link test."""
        family_person = Person.objects.create(practice=self.practice)
        return Patient.objects.create(
            practice=self.practice, person=family_person, first_name="Existing", last_name="Family",
            phone_number="7911123555", country_code="+44",
            meta_data={"id": 55003, "created_at": "2020-01-01T00:00:00.000+00:00"},
        )

    def test_force_import_creates_patient_and_resolves_issue(self):
        response = self.client.post(f"/api/backend/data-quality/issues/{self.issue.id}/force-import/")
        self.assertEqual(response.status_code, 201)
        self.assertTrue(
            Patient.objects.filter(practice=self.practice, first_name="Incoming").exists()
        )
        self.issue.refresh_from_db()
        self.assertEqual(self.issue.status, "resolved")

    def test_force_import_as_family_links_to_existing_person(self):
        # Execution-time correction: the plan as originally written expected
        # new_patient.person_id == self.existing_patient.person_id — asserting
        # the SAME Person object gets reused. That is not what actually happens,
        # and not what SHOULD happen: "Incoming Patient" and "Existing Family"
        # have different names, so PersonModel.resolve's own logic (see its
        # docstring — a shared channel with a DIFFERING name means distinct
        # people, not a merge) creates a NEW, separate Person in the SAME
        # Household. That's the correct data model for real family members —
        # distinct people sharing a household, not one shared Person record. The
        # explicit person= override in force_import only matters when the
        # pre_save signal has nothing to resolve on its own (no phone/email at
        # all, as in the non-family tests above); it does not and should not
        # force-merge two named individuals into one Person once a real shared
        # channel exists for the signal to resolve against.
        family_existing_patient = self._make_family_existing_patient()
        self.issue.detail["existing_patient_id"] = family_existing_patient.id
        self.issue.save(update_fields=["detail"])

        response = self.client.post(
            f"/api/backend/data-quality/issues/{self.issue.id}/force-import/",
            {"family": True}, format="json",
        )
        self.assertEqual(response.status_code, 201)
        new_patient = Patient.objects.get(practice=self.practice, first_name="Incoming")
        family_existing_patient.person.refresh_from_db()
        self.assertIsNotNone(new_patient.person.household_id)
        self.assertEqual(new_patient.person.household_id, family_existing_patient.person.household_id)

    def test_replace_existing_requires_override_when_incoming_older(self):
        self.issue.detail["dentally_data"]["created_at"] = "2019-01-01T00:00:00.000+00:00"
        self.issue.save(update_fields=["detail"])

        response = self.client.post(f"/api/backend/data-quality/issues/{self.issue.id}/replace-existing/")
        self.assertEqual(response.status_code, 409)
        self.assertTrue(response.json()["requires_override"])

    def test_replace_existing_with_override_replaces_and_resolves(self):
        self.issue.detail["dentally_data"]["created_at"] = "2019-01-01T00:00:00.000+00:00"
        self.issue.save(update_fields=["detail"])

        response = self.client.post(
            f"/api/backend/data-quality/issues/{self.issue.id}/replace-existing/",
            {"override": True}, format="json",
        )
        self.assertEqual(response.status_code, 200)
        self.existing_patient.refresh_from_db()
        self.assertEqual(self.existing_patient.first_name, "Incoming")
        self.issue.refresh_from_db()
        self.assertEqual(self.issue.status, "resolved")

    def test_discard_resolves_without_creating_a_patient(self):
        response = self.client.delete(f"/api/backend/data-quality/issues/{self.issue.id}/discard/")
        self.assertEqual(response.status_code, 204)
        self.assertFalse(Patient.objects.filter(practice=self.practice, first_name="Incoming").exists())
        self.issue.refresh_from_db()
        self.assertEqual(self.issue.status, "dismissed")
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `python manage.py test dataQuality.tests.test_duplicate_import_actions --keepdb -v 2`
Expected: FAIL — `404` on all five actions.

- [ ] **Step 3: Add the actions**

```python
# add to dataQuality/views.py, inside DataQualityIssueViewSet
# add "from .patient_import_utils import build_patient_data_from_detail" to the
# module's imports.

    def _resolve_issue(self, issue, request):
        issue.status = "resolved"
        issue.resolved_by = request.user if request.user.is_authenticated else None
        issue.resolved_at = timezone.now()
        issue.save(update_fields=["status", "resolved_by", "resolved_at", "updated_at"])

    @action(detail=True, methods=["post"], url_path="force-import")
    def force_import(self, request, pk=None):
        from TreatmentPlan.models import ContactChannel, Household
        from TreatmentPlan.models import Person as PersonModel
        from TreatmentPlan.models import Patient
        from TreatmentPlan.serializers import PatientSerializer

        issue = self.get_object()
        if issue.classifier != "duplicate_import":
            return Response(
                {"detail": "force-import is only valid for duplicate_import issues."},
                status=status.HTTP_400_BAD_REQUEST,
            )
        practice = self.get_user_practice_or_none()
        if not practice:
            return Response({"error": "User must be associated with a practice"}, status=400)

        is_family = request.data.get("family", False)
        patient_data, country_code = build_patient_data_from_detail(issue.detail)

        person = None
        existing_patient_id = issue.detail.get("existing_patient_id")
        if is_family and existing_patient_id:
            try:
                existing_patient = Patient.objects.get(id=existing_patient_id, practice=practice)
                if existing_patient.person:
                    person = existing_patient.person
                    if not person.household:
                        household = Household.objects.create(
                            practice=practice,
                            label=f"{person.first_name} {person.last_name}".strip()[:255] or "Family",
                        )
                        person.household = household
                        person.save(update_fields=["household"])
            except Patient.DoesNotExist:
                pass

        if not person:
            channel_ids = []
            if patient_data["phone_number"]:
                ch = ContactChannel.get_or_create_channel(
                    practice, ContactChannel.PHONE, patient_data["phone_number"], country_code
                )
                if ch:
                    channel_ids.append(ch.id)
            if patient_data["email"]:
                ch = ContactChannel.get_or_create_channel(practice, ContactChannel.EMAIL, patient_data["email"])
                if ch:
                    channel_ids.append(ch.id)
            person, _created = PersonModel.resolve(
                practice=practice,
                first_name=patient_data["first_name"],
                last_name=patient_data["last_name"],
                channel_ids=channel_ids,
            )

        serializer = PatientSerializer(data=patient_data)
        if not serializer.is_valid():
            return Response(
                {"error": "Patient validation failed", "details": serializer.errors},
                status=status.HTTP_400_BAD_REQUEST,
            )
        patient = serializer.save(practice=practice, created_by=request.user, person=person)

        from dentallyIntegration.tasks import _apply_dentally_family_grouping
        _apply_dentally_family_grouping(patient, issue.detail.get("dentally_data") or {}, practice)

        self._resolve_issue(issue, request)

        return Response(
            {
                "message": "Patient created successfully" + (" (linked as family)" if is_family else ""),
                "patient": {
                    "id": patient.id, "first_name": patient.first_name, "last_name": patient.last_name,
                    "email": patient.email, "phone_number": patient.phone_number,
                    "secondary_phone_number": patient.secondary_phone_number, "country_code": patient.country_code,
                },
                "family": is_family, "person_id": person.id if person else None,
            },
            status=status.HTTP_201_CREATED,
        )

    @action(detail=True, methods=["post"], url_path="replace-existing")
    def replace_existing(self, request, pk=None):
        from TreatmentPlan.models import Patient
        from TreatmentPlan.serializers import PatientSerializer

        issue = self.get_object()
        practice = self.get_user_practice_or_none()
        if not practice:
            return Response({"error": "User must be associated with a practice"}, status=400)

        existing_patient_id = issue.detail.get("existing_patient_id")
        if not existing_patient_id:
            return Response({"error": "No existing patient is linked to this duplicate issue."}, status=400)

        override_raw = request.data.get("override", False)
        override = (
            override_raw.strip().lower() in ("1", "true", "yes", "on")
            if isinstance(override_raw, str) else bool(override_raw)
        )

        try:
            existing_patient = Patient.objects.get(id=existing_patient_id, practice=practice)
        except Patient.DoesNotExist:
            return Response({"error": "Existing patient not found"}, status=404)

        dentally_data = issue.detail.get("dentally_data") or {}
        incoming_created_at = dentally_data.get("created_at")
        existing_created_at = (existing_patient.meta_data or {}).get("created_at")

        if (
            incoming_created_at and existing_created_at
            and incoming_created_at <= existing_created_at and not override
        ):
            return Response(
                {
                    "requires_override": True,
                    "message": "Incoming Dentally record is older than or equal to existing record. Pass override=true to force replacement.",
                    "comparison": {
                        "incoming_created_at": incoming_created_at,
                        "existing_created_at": existing_created_at,
                        "incoming_is_newer": False,
                    },
                },
                status=status.HTTP_409_CONFLICT,
            )

        patient_data, _ = build_patient_data_from_detail(issue.detail)
        serializer = PatientSerializer(existing_patient, data=patient_data, partial=True)
        if not serializer.is_valid():
            return Response({"error": "Patient validation failed", "details": serializer.errors}, status=400)
        updated_patient = serializer.save()

        self._resolve_issue(issue, request)

        return Response({
            "message": "Existing patient replaced successfully",
            "override_used": override,
            "comparison": {
                "incoming_created_at": incoming_created_at, "existing_created_at": existing_created_at,
                "incoming_is_newer": (
                    (incoming_created_at > existing_created_at)
                    if incoming_created_at and existing_created_at else None
                ),
            },
            "patient": {
                "id": updated_patient.id, "first_name": updated_patient.first_name,
                "last_name": updated_patient.last_name, "email": updated_patient.email,
                "phone_number": updated_patient.phone_number,
                "secondary_phone_number": updated_patient.secondary_phone_number,
                "country_code": updated_patient.country_code,
            },
        })

    @action(detail=True, methods=["delete"], url_path="discard")
    def discard(self, request, pk=None):
        """Discard a duplicate_import issue without creating/replacing anything —
        dismissed, not resolved, since no fix was applied (matches the legacy
        DELETE-and-forget semantics, just non-destructive to the audit trail now)."""
        issue = self.get_object()
        issue.status = "dismissed"
        issue.resolved_by = request.user if request.user.is_authenticated else None
        issue.resolved_at = timezone.now()
        issue.save(update_fields=["status", "resolved_by", "resolved_at", "updated_at"])
        return Response(status=status.HTTP_204_NO_CONTENT)
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `python manage.py test dataQuality.tests.test_duplicate_import_actions --keepdb -v 2`
Expected: `OK` (5 tests). If `PersonModel.resolve` / `Person.merge` / `ContactChannel.get_or_create_channel` signatures don't match what's called here, re-check
`TreatmentPlan/models.py` — this plan was written against the versions read during
Phase 5 planning; if the codebase has since changed these signatures, fix the call
sites to match the live code.

- [ ] **Step 5: Commit**

---

### Task 6: Other import-error classifiers — create-patient and bulk retry

**Files:**
- Modify: `dataQuality/views.py`
- Test: `dataQuality/tests/test_import_error_actions.py`

Ports `create_patient_from_error` (near-identical to Task 5's `force_import` minus the
family branch — reuses the same `build_patient_data_from_detail` helper) and
`retry_dentally_import_errors` (bulk retry, `other_import_error` classifier only —
`validation_error`/`missing_data_import`/`invalid_phone_import` require a manual edit
first, matching the legacy `failure_type="other"` restriction).

- [ ] **Step 1: Write the failing test**

```python
# dataQuality/tests/test_import_error_actions.py
from django.test import TestCase
from rest_framework.test import APIClient

from TreatmentPlan.models import Patient
from UserAuthentication.models import Practice, User
from dataQuality.models import DataQualityIssue


class ImportErrorActionTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(name="DQ Error Actions Test Practice")
        self.user = User.objects.create(email="staff5@example.com", current_practice=self.practice)
        self.user.practices.add(self.practice)
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)

    def _make_issue(self, classifier, dentally_id):
        return DataQualityIssue.objects.create(
            practice=self.practice, classifier=classifier, source="django_sync",
            record_type="dentally_patient", record_id=str(dentally_id), dentally_patient_id=dentally_id,
            detail={
                "first_name": "Retry", "last_name": "Case",
                "dentally_data": {"id": dentally_id, "first_name": "Retry", "last_name": "Case", "mobile_phone": "07911123666"},
            },
        )

    def test_create_patient_resolves_other_import_error(self):
        issue = self._make_issue("other_import_error", 66001)
        response = self.client.post(f"/api/backend/data-quality/issues/{issue.id}/create-patient/")
        self.assertEqual(response.status_code, 201)
        self.assertTrue(Patient.objects.filter(practice=self.practice, first_name="Retry").exists())
        issue.refresh_from_db()
        self.assertEqual(issue.status, "resolved")

    def test_bulk_retry_only_processes_other_import_error(self):
        other_issue = self._make_issue("other_import_error", 66002)
        validation_issue = self._make_issue("validation_error", 66003)

        response = self.client.post("/api/backend/data-quality/issues/retry-errors/")
        self.assertEqual(response.status_code, 200)

        other_issue.refresh_from_db()
        validation_issue.refresh_from_db()
        self.assertEqual(other_issue.status, "resolved")
        self.assertEqual(validation_issue.status, "open")  # untouched — needs manual edit first
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `python manage.py test dataQuality.tests.test_import_error_actions --keepdb -v 2`
Expected: FAIL — `404` on both actions.

- [ ] **Step 3: Add the actions**

```python
# add to dataQuality/views.py, inside DataQualityIssueViewSet

    @action(detail=True, methods=["post"], url_path="create-patient")
    def create_patient(self, request, pk=None):
        from TreatmentPlan.models import ContactChannel
        from TreatmentPlan.models import Person as PersonModel
        from TreatmentPlan.serializers import PatientSerializer

        issue = self.get_object()
        practice = self.get_user_practice_or_none()
        if not practice:
            return Response({"error": "User must be associated with a practice"}, status=400)

        patient_data, country_code = build_patient_data_from_detail(issue.detail)

        channel_ids = []
        if patient_data["phone_number"]:
            ch = ContactChannel.get_or_create_channel(
                practice, ContactChannel.PHONE, patient_data["phone_number"], country_code
            )
            if ch:
                channel_ids.append(ch.id)
        if patient_data["email"]:
            ch = ContactChannel.get_or_create_channel(practice, ContactChannel.EMAIL, patient_data["email"])
            if ch:
                channel_ids.append(ch.id)
        person, _created = PersonModel.resolve(
            practice=practice, first_name=patient_data["first_name"],
            last_name=patient_data["last_name"], channel_ids=channel_ids,
        )

        serializer = PatientSerializer(data=patient_data)
        if not serializer.is_valid():
            return Response({"error": "Patient validation failed", "details": serializer.errors}, status=400)
        patient = serializer.save(practice=practice, created_by=request.user, person=person)

        self._resolve_issue(issue, request)

        return Response(
            {
                "message": "Patient created successfully",
                "patient": {
                    "id": patient.id, "first_name": patient.first_name, "last_name": patient.last_name,
                    "email": patient.email, "phone_number": patient.phone_number,
                    "secondary_phone_number": patient.secondary_phone_number, "country_code": patient.country_code,
                },
            },
            status=status.HTTP_201_CREATED,
        )

    @action(detail=False, methods=["post"], url_path="retry-errors")
    def retry_errors(self, request):
        """Bulk-retry every open other_import_error issue for the practice — the
        only classifier safe to auto-retry (validation/missing_data/invalid_phone
        represent genuine data problems needing a manual edit first, matching the
        legacy failure_type='other' restriction)."""
        from TreatmentPlan.models import Patient
        from TreatmentPlan.serializers import PatientSerializer

        practice = self.get_user_practice_or_none()
        if not practice:
            return Response({"error": "User must be associated with a practice"}, status=400)

        qs = DataQualityIssue.objects.filter(
            practice=practice, classifier="other_import_error", status="open",
        )
        succeeded = failed = skipped = 0

        for issue in qs.iterator(chunk_size=200):
            patient_data, _ = build_patient_data_from_detail(issue.detail)
            if not patient_data["email"] and not patient_data["phone_number"]:
                skipped += 1
                continue

            existing = (
                Patient.objects.filter(practice=practice, meta_data__id=issue.dentally_patient_id).first()
                if issue.dentally_patient_id else None
            )
            serializer = (
                PatientSerializer(existing, data=patient_data, partial=True)
                if existing else PatientSerializer(data=patient_data)
            )
            if not serializer.is_valid():
                issue.detail["error_message"] = f"Validation error: {serializer.errors}"
                issue.save(update_fields=["detail"])
                failed += 1
                continue

            if existing:
                serializer.save()
            else:
                serializer.save(practice=practice, created_by=request.user)

            self._resolve_issue(issue, request)
            succeeded += 1

        return Response({"succeeded": succeeded, "failed": failed, "skipped": skipped})
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `python manage.py test dataQuality.tests.test_import_error_actions --keepdb -v 2`
Expected: `OK` (2 tests).

- [ ] **Step 5: Run the FULL `dataQuality` test suite to confirm no cross-task regressions**

Run: `python manage.py test dataQuality --keepdb -v 2`
Expected: all tests from Tasks 1-6 pass together.

- [ ] **Step 6: Commit**

---

### Task 7: Frontend — unified hook and list screen

**Files:**
- Create: `perfect-pixel-playground-project/src/hooks/useDataQualityIssues.ts`
- Create: `perfect-pixel-playground-project/src/components/data-quality/DataQualityIssuesScreen.tsx`
- Modify: `perfect-pixel-playground-project/src/components/data-quality/DataQualityContent.tsx`
- Modify: `perfect-pixel-playground-project/src/config/api.ts`
- Delete (end of task): `DuplicatesTab.tsx`, `NonDuplicatesTab.tsx`, `DuplicateContactsTab.tsx`, `MissingInfoTab.tsx`, old `useDataQuality.ts`

This task is UI-heavy (list rendering, per-classifier action buttons, dialogs) rather
than algorithm-heavy — write it test-first at the hook/API-contract level (the part with
real logic to break), and treat the visual layout as an iterative build-and-look step
per this project's UI convention (start the dev server, verify in browser) rather than
asserting pixel-level detail in a plan step.

- [ ] **Step 1: Add the new endpoints to `API_ENDPOINTS`**

In `src/config/api.ts`, add a new `dataQualityIssues` group alongside the existing
`dataQuality` group (do not remove the old one yet — Task 8 does that once the old
components are deleted):

```typescript
  dataQualityIssues: {
    list: () => getApiUrl('/data-quality/issues/'),
    dismiss: (id: number) => getApiUrl(`/data-quality/issues/${id}/dismiss/`),
    merge: (id: number) => getApiUrl(`/data-quality/issues/${id}/merge/`),
    updateContact: (id: number) => getApiUrl(`/data-quality/issues/${id}/update-contact/`),
    deleteContact: (id: number) => getApiUrl(`/data-quality/issues/${id}/delete-contact/`),
    forceImport: (id: number) => getApiUrl(`/data-quality/issues/${id}/force-import/`),
    replaceExisting: (id: number) => getApiUrl(`/data-quality/issues/${id}/replace-existing/`),
    discard: (id: number) => getApiUrl(`/data-quality/issues/${id}/discard/`),
    createPatient: (id: number) => getApiUrl(`/data-quality/issues/${id}/create-patient/`),
    retryErrors: () => getApiUrl('/data-quality/issues/retry-errors/'),
  },
```

- [ ] **Step 2: Write the failing hook test**

```typescript
// src/hooks/useDataQualityIssues.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { renderHook, act, waitFor } from "@testing-library/react";
import { useDataQualityIssues } from "./useDataQualityIssues";

vi.mock("@/lib/helpers", () => ({
  useFetchWithAuth: () => vi.fn(),
}));

describe("useDataQualityIssues", () => {
  it("fetches issues filtered by classifier", async () => {
    const fetchMock = vi.fn().mockResolvedValue({
      ok: true,
      json: async () => ({ count: 1, results: [{ id: 1, classifier: "missing_info", status: "open" }] }),
    });
    vi.doMock("@/lib/helpers", () => ({ useFetchWithAuth: () => fetchMock }));

    const { result } = renderHook(() => useDataQualityIssues());
    await act(async () => {
      await result.current.fetchIssues({ classifier: "missing_info" });
    });

    await waitFor(() => expect(result.current.issues).toHaveLength(1));
    expect(result.current.issues[0].classifier).toBe("missing_info");
  });
});
```

Check this project's actual test setup (`vitest.config.ts` / existing hook tests under
`src/hooks/*.test.ts`, e.g. `useJourneyConvert.test.tsx` referenced earlier in this
session) for the real mocking convention for `useFetchWithAuth` before finalizing this
test — the `vi.mock`/`vi.doMock` shape above is illustrative; match whatever pattern
`useJourneyConvert.test.tsx` or a similar existing hook test already uses so this test
is consistent with the rest of the suite.

- [ ] **Step 3: Run the test to verify it fails**

Run: `cd perfect-pixel-playground-project && npx vitest run src/hooks/useDataQualityIssues.test.ts`
Expected: FAIL — module doesn't exist.

- [ ] **Step 4: Write the unified hook**

```typescript
// src/hooks/useDataQualityIssues.ts
import { useState, useCallback } from "react";
import { useFetchWithAuth } from "@/lib/helpers";
import { API_ENDPOINTS } from "@/config/api";
import { toast } from "sonner";

export type DataQualityClassifier =
  | "duplicate_import" | "validation_error" | "missing_data_import"
  | "invalid_phone_import" | "other_import_error"
  | "duplicate_contact" | "missing_info";

export interface DataQualityIssue {
  id: number;
  classifier: DataQualityClassifier;
  status: "open" | "dismissed" | "resolved";
  source: string;
  record_type: string;
  record_id: string;
  detail: Record<string, any>;
  dentally_patient_id: number | null;
  created_at: string;
}

interface FetchParams {
  classifier?: DataQualityClassifier;
  status?: "open" | "dismissed" | "resolved" | "all";
}

export function useDataQualityIssues() {
  const fetchWithAuth = useFetchWithAuth();
  const [issues, setIssues] = useState<DataQualityIssue[]>([]);
  const [count, setCount] = useState(0);
  const [loading, setLoading] = useState(false);

  const fetchIssues = useCallback(async (params: FetchParams = {}) => {
    setLoading(true);
    try {
      const query = new URLSearchParams();
      if (params.classifier) query.set("classifier", params.classifier);
      if (params.status) query.set("status", params.status);

      const response = await fetchWithAuth(`${API_ENDPOINTS.dataQualityIssues.list()}?${query.toString()}`);
      if (!response.ok) throw new Error("Failed to fetch data quality issues");
      const data = await response.json();
      setIssues(data.results || []);
      setCount(data.count || 0);
    } catch (error) {
      console.error("Error fetching data quality issues:", error);
      toast.error("Failed to load data quality issues");
      setIssues([]);
      setCount(0);
    } finally {
      setLoading(false);
    }
  }, [fetchWithAuth]);

  const dismissIssue = useCallback(async (id: number) => {
    const response = await fetchWithAuth(API_ENDPOINTS.dataQualityIssues.dismiss(id), { method: "POST" });
    if (!response.ok) {
      toast.error("Failed to dismiss issue");
      return false;
    }
    setIssues((prev) => prev.filter((i) => i.id !== id));
    return true;
  }, [fetchWithAuth]);

  const resolveIssue = useCallback(async (
    id: number,
    action: "merge" | "update-contact" | "delete-contact" | "force-import" | "replace-existing" | "discard" | "create-patient",
    payload?: Record<string, any>,
  ) => {
    const endpointMap: Record<typeof action, (id: number) => string> = {
      "merge": API_ENDPOINTS.dataQualityIssues.merge,
      "update-contact": API_ENDPOINTS.dataQualityIssues.updateContact,
      "delete-contact": API_ENDPOINTS.dataQualityIssues.deleteContact,
      "force-import": API_ENDPOINTS.dataQualityIssues.forceImport,
      "replace-existing": API_ENDPOINTS.dataQualityIssues.replaceExisting,
      "discard": API_ENDPOINTS.dataQualityIssues.discard,
      "create-patient": API_ENDPOINTS.dataQualityIssues.createPatient,
    };
    const method = action === "delete-contact" || action === "discard" ? "DELETE" : "POST";
    const response = await fetchWithAuth(endpointMap[action](id), {
      method,
      ...(payload ? { headers: { "Content-Type": "application/json" }, body: JSON.stringify(payload) } : {}),
    });
    if (!response.ok) {
      const data = await response.json().catch(() => ({}));
      return { ok: false as const, data };
    }
    setIssues((prev) => prev.filter((i) => i.id !== id));
    const data = await response.json().catch(() => ({}));
    return { ok: true as const, data };
  }, [fetchWithAuth]);

  return { issues, count, loading, fetchIssues, dismissIssue, resolveIssue };
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `npx vitest run src/hooks/useDataQualityIssues.test.ts`
Expected: pass.

- [ ] **Step 6: Build the unified list screen**

Create `DataQualityIssuesScreen.tsx` — a classifier-filter chip row (All / Duplicate
Contacts / Import Duplicates / Validation Errors / Missing Data / Invalid Phone /
Missing Info / Other) plus one list where each row renders classifier-appropriate
action buttons (Merge+Dismiss for `duplicate_contact`; Force Import+Import as
Family+Update Existing+Discard for `duplicate_import`; Edit+Delete for `missing_info`;
Retry+Delete for the remaining import-error classifiers), calling `resolveIssue`/
`dismissIssue` from Task 4's hook. Reuse `DuplicateGroupCard`'s expand/collapse visual
pattern as a starting template for the row component rather than building from scratch.

Since this is presentational/layout work without algorithmic risk, build it iteratively
against the running dev server rather than writing exhaustive assertions here — **run
`npm run dev` (or this project's existing dev-server invocation) and click through All
seven classifiers with real seeded `DataQualityIssue` rows (from Tasks 2-6's tests, or
manually via Django admin — Phase 1 registered `DataQualityIssueAdmin`) to confirm each
action actually fires and the row disappears/updates on success before considering this
step done.**

- [ ] **Step 7: Wire `DataQualityContent.tsx` to render the new screen**

Replace the four `<Route>` entries and their imports with a single route rendering
`<DataQualityIssuesScreen />`, and replace the `SUBTAB_LABEL`/tab-bar with classifier
filter chips driven by `DataQualityClassifier` instead of the four `DataQualitySubTab`
route strings — this is what fixes the "conflicts" vs "duplicates" label confusion
flagged during design, since every chip now names its real classifier directly.

- [ ] **Step 8: Delete the retired components**

```bash
git rm src/components/data-quality/DuplicatesTab.tsx \
       src/components/data-quality/NonDuplicatesTab.tsx \
       src/components/data-quality/DuplicateContactsTab.tsx \
       src/components/data-quality/MissingInfoTab.tsx \
       src/hooks/useDataQuality.ts \
       src/hooks/useDataQuality.test.ts
```

Grep for any remaining import of these files before finalizing:
`grep -rn "DuplicatesTab\|NonDuplicatesTab\|DuplicateContactsTab\|MissingInfoTab\|useDataQuality[^I]" src/` (the `[^I]` excludes matches on the new
`useDataQualityIssues`) — fix any straggler import before this step is complete.

- [ ] **Step 9: Commit**

---

### Task 8: Retire the legacy backend

**Files:**
- Modify: `dentallyIntegration/models.py` (remove `DentallyImportFailure`, `DentallyImportError`)
- Modify: `TreatmentPlan/models.py` (remove `DuplicateClusterDismissal`)
- Delete: `dentallyIntegration/views/import_views.py`'s two ViewSets + their URL entries
- Delete: `TreatmentPlan/views/data_quality_views.py`, its URL entries
- Create: migrations dropping the three tables

- [x] **Step 1: Confirm nothing still references the legacy models**

```bash
grep -rln "DentallyImportFailure\|DentallyImportError\|DuplicateClusterDismissal" \
  --include="*.py" TreatmentPathBackend/TreatmentPath/ | grep -v migrations
```

Expected, at this point in the rollout: zero results outside `migrations/` (Phase 2
already stopped writing; Task 7 stopped reading via the frontend). If anything remains
(a management command, an admin registration, an unexpected import), resolve it before
proceeding — do not force-delete a model something still depends on.

**Execution-time correction:** found several live readers the original plan didn't
enumerate: `dentallyIntegration/views/dentally_views.py`'s `get_import_failures` action
(dead — unreachable, no URL wired to it, but still imported `DentallyImportFailure`),
`dentallyIntegration/tasks.py`'s `retry_dentally_import_errors` celery task (only called
from the now-deleted `DentallyImportErrorViewSet.retry_errors`), and
`dentallyIntegration/management/commands/backfill_dentally_failed_imports.py` (a
standalone batch tool, no live callers). All three removed. Also found and removed 3
orphaned frontend files (`ImportFailureCard.tsx`/`ImportErrorCard.tsx`/
`DuplicateCountBadge.tsx` — already pre-existing dead code per project memory, but they
called the legacy `dataQuality` endpoint group being retired here) and that legacy
`API_ENDPOINTS.dataQuality` group in `api.ts`. Also deleted the orphaned
`dataQuality/tests/test_name_similarity_fixture.py` + its JSON fixture — the Python
`_name_similar` it tested lived only in the now-deleted `data_quality_views.py`; the Go
nightly sweep (Phase 4) already has its own byte-identical fixture test and is the sole
live implementation now.

**Critical discovery — Go nightly sweep depended on `DuplicateClusterDismissal`:**
`sweepDuplicateContacts`'s raw SQL query excluded channels found in
`TreatmentPlan_duplicateclusterdismissal`; dropping that table would have broken the
sweep with "relation does not exist" on the next 2am run for every practice. Worse,
independent of the table drop: both `sweepMissingInfo` and `sweepDuplicateContacts`
unconditionally wrote `status: "open"` on every upsert, meaning **any issue a user
dismissed or that got auto-resolved would be silently reopened by the next nightly
sweep** — a real, already-shipped correctness bug, not something introduced by this
retirement. Fixed both: removed the dismissal-table exclusion from the query (dismissal
now lives on `DataQualityIssue.status` itself), and replaced the plain `Updates(map)`
upsert with a raw `UPDATE ... SET status = CASE WHEN status = 'dismissed' THEN status
ELSE 'open' END` so a dismissed row's `detail` still refreshes but its status is never
clobbered back open — matching the retired table's permanent-suppression semantics.
Added `TestSweepMissingInfo_PreservesDismissedStatus` and
`TestSweepDuplicateContacts_PreservesDismissedStatus` (both pass).

- [x] **Step 2: Remove the ViewSets, views, and URL entries**

Delete `DentallyImportFailureViewSet`/`DentallyImportErrorViewSet` from
`dentallyIntegration/views/import_views.py` and their router registrations; delete
`DataQualityDuplicatesView`/`DataQualityMergeView`/`DataQualityDismissDuplicateView`/
`DataQualityDeleteView`/`DataQualityMissingInfoView` from
`TreatmentPlan/views/data_quality_views.py` and their URL entries in
`TreatmentPlan/urls.py`. Also remove `retry_dentally_import_errors` and
`backfill_dentally_failed_imports` (superseded by Task 6's `retry_errors` action and no
longer has a live table to read from).

- [x] **Step 3: Remove the models and generate the drop migration**

Delete the `DentallyImportFailure`/`DentallyImportError` class definitions from
`dentallyIntegration/models.py` and `DuplicateClusterDismissal` from
`TreatmentPlan/models.py`, then:

```bash
python manage.py makemigrations dentallyIntegration TreatmentPlan
```

Expected: two migrations, each a `DeleteModel` (or `RemoveField`/`DeleteModel` pair
depending on FKs) for the retired models.

Produced `TreatmentPlan/migrations/0141_delete_duplicateclusterdismissal.py` and
`dentallyIntegration/migrations/0163_remove_dentallyimportfailure_practice_and_more.py`,
matching the expected shape exactly.

**Execution-time correction (2026-08-14, caught by user question the next morning):**
the plan as originally written and the "Step 4" note below both said "no data backfill
needed," reasoning the legacy tables only held stale, already-superseded rows. That was
wrong — still-present `DentallyImportFailure`/`DentallyImportError` rows are genuine
unresolved historical import failures (an append-only event log, not derived state —
dropping the table loses them permanently, they can't be regenerated by any sweep), and
still-present `DuplicateClusterDismissal` rows are explicit staff decisions ("not a
duplicate") that, if not carried over, would cause the Go nightly sweep to silently
reopen a previously-dismissed cluster as a fresh "open" issue on its very first run.
Hand-edited both migrations to add a `RunPython` backfill operation before their
`RemoveField`/`DeleteModel` operations (with an added `("dataQuality", "0001_initial")`
dependency so `DataQualityIssue` exists first): 0163 backfills every import-failure/error
row into an "open" `DataQualityIssue` via `get_or_create` keyed on
`(practice, dentally_patient_id)` — never overwrites a row the live writer already
recreated post-cutover; 0141 backfills every dismissal into a "dismissed"
`duplicate_contact` issue via `update_or_create` keyed on `(practice, channel)` —
always wins over a same-channel "open" row, since an explicit historical dismissal must
take precedence over auto-detection. Both preserve the original `created_at` (working
around `auto_now_add` forcing "now" on `.create()`/`get_or_create()` via a follow-up
`.filter(pk=...).update(created_at=...)`). Pure field-mapping logic extracted to
`dataQuality/legacy_backfill.py` and unit tested (8 tests, `dataQuality/tests/
test_legacy_backfill.py`) independent of the migration machinery. The migration wiring
itself was verified against a REAL forward-migration run (not just unit tests): reverted
both migrations locally (`migrate ... --fake` to unapply without touching schema),
manually recreated the exact original table schemas via raw SQL, inserted legacy-shaped
fixture rows, ran `migrate` for real, and confirmed the resulting `DataQualityIssue`
rows had correct classifiers, preserved timestamps, and correct detail — see
`to-run-inprod/2026-08-14-dataquality-unification-deploy.txt` for the corrected NOTES
section (the "no backfill needed" line there was also wrong and has been corrected).

- [x] **Step 4: Apply locally and run the full test suite**

```bash
python manage.py migrate --keepdb
python manage.py test --keepdb -v 2
```

Expected: migrations apply cleanly; full suite passes (any pre-existing unrelated
failures — see project memory on Recall test suite decay — are not this plan's
responsibility to fix, but nothing NEW should fail here).

**Execution-time correction:** `--keepdb` is a `test`-only flag, not accepted by
`migrate` — ran `python manage.py migrate` (no flag) instead; applied cleanly. Full
suite run surfaced 8 failures + 13 errors; traced every one and confirmed none trace
back to this retirement: 5 errors were `DENTALLY_ENCRYPTION_KEY` missing from the shell
env (not code — re-ran with a throwaway key exported and the 38 dataQuality/migration/
api-key tests all passed clean); 2 errors were `TreatmentPlan/tests/
test_dedupe_patients_patientdocuments.py` predating migration 0140's
`patient_practice_dentally_uniq` constraint (unrelated duplicates workstream, added
before this session); the rest (recall/messaging/journey-filter/journey-search tests,
one Go `scheduler` test keyed on local practice-4 seed data, one Go `migration`
contact-key test) are pre-existing environment/data drift or belong to the separate
in-flight contact-identity-remodel work per project memory. `dataQuality`'s own 28
tests pass clean in isolation.

- [x] **Step 5: Add this to `/to-run-in-prod`**

This plan's Task 3 (Django writer), Task 2/8 here (model removal + migration), and
Phase 3's schema-guard registration all need a coordinated prod deploy — use the
`to-run-in-prod` skill to record the exact ordering (migrate before Go deploy, per the
schema-guard's own rule) as its own tracked entry rather than leaving it implicit in
this plan.

Recorded at `TreatmentPathBackend/to-run-inprod/2026-08-14-dataquality-unification-deploy.txt`.

- [ ] **Step 6: Commit**

**Skipped per standing user rule** (project memory `feedback_no_commits.md`) — the user
handles all git add/commit/push themselves. All Task 8 changes are left staged-ready but
uncommitted for the user to review and commit.

```bash
git add -A
git commit -m "chore(dataQuality): retire legacy import-failure/duplicate-cluster models"
```

---

## Self-review notes

- **Spec coverage**: implements Section 4 (Resolution actions) and Section 5 (Frontend)
  in full, plus the Section 6 step 5 retirement. Verified every action in the spec's
  Section 4 table has a corresponding task: `duplicate_contact` → Task 3,
  `duplicate_import` → Task 5, `validation_error`/`other_import_error` → Task 6,
  `missing_data_import`/`invalid_phone_import` → covered by Task 6's `create-patient`
  (same action, different classifier — no separate code needed, matching how the legacy
  `create_patient_from_error` action already handles all of
  `DentallyImportError`'s failure types uniformly), `missing_info` → Task 4.
- **No placeholders in code steps**: every action handler is complete, ported from code
  actually read in full during planning (`import_views.py` lines 272-700, 917-1101,
  `data_quality_views.py` in full, `tasks.py` lines 1277-1419). Two steps (Task 2 Step 1,
  Task 7 Step 2) explicitly ask the implementer to verify a test assumption against live
  code before trusting it — flagged as verification steps, not code gaps, since the
  exact mechanism (practice resolution, hook test mocking convention) wasn't read during
  this planning session and shouldn't be guessed at with false confidence.
- **Type consistency**: `DataQualityIssue`/`DataQualityClassifier` field names match
  between Phase 1's model, this plan's serializer, and the frontend hook's TypeScript
  interface — re-checked after writing all three.
- **DRY applied deliberately**: `build_patient_data_from_detail` (Task 1) collapses
  three near-identical 40-line blocks from the legacy code into one function, ported
  once and reused by Tasks 5 and 6 — a genuine simplification made possible BY this
  migration, not scope creep (the duplication existed in the original code purely
  because three separate DRF actions each needed the same extraction logic inline).

# Day List Outstanding-Items N+1 Batching Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Eliminate the Day List's per-patient N+1 API storm (up to 4 requests per unique patient/contact: `outstanding-items?contact_id=`, `outstanding-items?patient_id=`, `patients/<id>/journeys/?type=Open+Plans`, and a name-search fallback) by embedding the same data directly in the Day List's own response, following the `attach_consent_summaries`/`build_daylist_maps` batching pattern already used in this codebase.

**Architecture:** Add a new batched selector `TreatmentPlan/outstanding_summary.py` (mirrors `Documents/consent_summary.py`) that computes, in a small constant number of queries for the whole page, the same `{journeys, tasks, labs, counts}` shape the per-patient `PatientOutstandingItemsView` returns today — including the harder "Open Plans" fuzzy match against `NonRegisteredPatient` records, inverted into one practice-wide index pass instead of a per-patient scan. Wire it into `DayListAppointmentSerializer.build_daylist_maps()` as a new `outstanding_items` field on every Day List row. On the frontend, `useDayListOutstandingItems` reads `appointment.outstanding_items` directly when present and only falls back to the old per-patient fetch chain for appointments the batched map couldn't resolve (i.e. appointments with no linked `patient_id`/`contact_id` at all) — so `DayListPage.tsx` needs zero changes.

**Tech Stack:** Django REST Framework (DRF `SerializerMethodField` + `.context`), Postgres (`__in` batched queries), React/TypeScript hook.

## Global Constraints

- Response shape for `outstanding_items` on a Day List row MUST be byte-compatible with today's merged client-side `OutstandingResponse` (`{journeys: OutstandingJourney[], tasks: OutstandingTask[], labs: OutstandingLab[], counts: {journeys, tasks, labs}}`, from `src/components/patients/patient-panel/PatientOutstandingItems.tsx:27-54`) — the frontend must be able to drop it in with no reshaping.
- `OutstandingJourney.kind` stays restricted to `"intake" | "nurture" | "treatment_plan"` (no new kinds).
- Do not touch `PatientOutstandingItemsView` or `PatientJourneysView` — the patient-panel Clinical tab still calls these directly for a single patient and must keep working unchanged.
- Every batched query uses `__in=` over already-collected ids — no per-patient query inside a loop anywhere in the new code.
- Preserve the existing `useDayListOutstandingItems(appointments): Record<string, OutstandingResponse>` return type and the existing `getOutstandingKey()` lookup contract, so `DayListPage.tsx:359,609` needs no changes.

---

## Task 1: Batched "Person-linked" outstanding items (Intake/Nurture/Task/Lab/TreatmentPlan)

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/outstanding_summary.py`
- Test: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_outstanding_summary.py`

**Interfaces:**
- Produces: `build_outstanding_items_map(practice_id, patients) -> dict[int, dict]` — `patients` is an iterable of already-fetched `Patient` instances (e.g. `patient_lookup.values()` from `dentallyIntegration/serializers.py`). Returns `{patient.id: {"journeys": [...], "tasks": [...], "labs": [...], "counts": {...}}}`, one entry for EVERY input patient (patients with no open items get an empty-but-present entry — Task 1 covers this; Task 2 later adds matches that don't require a resolved `person_id` at all, e.g. via `treatment_plan_ids`, so "no person" is not the same as "no entry").
- Later tasks (2, 3) import and extend this module.

This task replicates `PatientOutstandingItemsView._journeys()` (Intake/Nurture/TreatmentPlan via `person=identity`), `_tasks()`, and `_labs()` from `TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/patient_outstanding_views.py:73-163`, batched by `person_id__in=`.

**Important semantic to preserve:** the original code resolves ONE `Person` (`identity`) and queries `Task.objects.filter(patient__person=identity, ...)` — i.e. it matches ANY `Patient` row sharing that person (not just the one `Patient` the request started from), because a person can have duplicate `Patient` rows. The batched version must group by `person_id`, not by `patient.id` from the join, then attribute the person's full result list to **every** `Patient` in the input batch that shares that `person_id`.

- [ ] **Step 1: Write the failing test — Intake/Nurture/TreatmentPlan (person-linked) parity**

```python
# TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_outstanding_summary.py
"""
Regression tests for TreatmentPlan/outstanding_summary.py — the batched
version of PatientOutstandingItemsView's per-patient journeys/tasks/labs
lookup, used to embed `outstanding_items` on Day List rows instead of
firing one /outstanding-items/ request per patient.

Each test proves the batched map is byte-for-byte equivalent to calling
the existing (unbatched) per-patient view for the same patients.
"""

from datetime import timedelta

from django.contrib.auth import get_user_model
from django.test import TestCase, override_settings
from django.utils import timezone

from Labs.models import LabCase
from Tasks.models import Task
from TreatmentPlan.models import Intake, JourneyStage, Nurture, Patient, TreatmentPlan
from TreatmentPlan.outstanding_summary import build_outstanding_items_map
from TreatmentPlan.views.patient_outstanding_views import PatientOutstandingItemsView
from UserAuthentication.models import Practice


User = get_user_model()


@override_settings(SECURE_SSL_REDIRECT=False)
class BuildOutstandingItemsMapTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(
            name="Outstanding Batch Practice", slug="outstanding-batch"
        )
        self.intake_stage = JourneyStage.objects.get(
            practice=self.practice, slug="intake"
        )
        self.nurture_stage = JourneyStage.objects.get(
            practice=self.practice, slug="nurture"
        )
        self.open_plan_stage = JourneyStage.objects.get(
            practice=self.practice, slug="open-plan"
        )

    def _make_patient_with_person(self, suffix):
        # Creating a Patient auto-creates/links a Person via the model's
        # save() signal wiring in this codebase (mirrors test_move_views.py's
        # _make_patient) — assert it exists so the test fails loudly instead
        # of silently passing with person_id=None if that assumption breaks.
        patient = Patient.objects.create(
            practice=self.practice,
            first_name=f"P-{suffix}",
            last_name="Last",
            phone_number=f"0803000{suffix.zfill(4)}",
        )
        patient.refresh_from_db()
        self.assertIsNotNone(
            patient.person_id, "test setup assumption broke: Patient has no Person"
        )
        return patient

    def _call_legacy_view(self, patient):
        """Invoke the existing per-patient view's private helpers directly
        (same technique as calling it over HTTP, without needing a request)."""
        view = PatientOutstandingItemsView()
        identity = patient.person
        journeys = view._journeys(self.practice, identity)
        tasks = view._tasks(identity)
        labs = view._labs(identity)
        return {
            "journeys": journeys,
            "tasks": tasks,
            "labs": labs,
            "counts": {
                "journeys": len(journeys),
                "tasks": len(tasks),
                "labs": len(labs),
            },
        }

    def test_batched_map_matches_legacy_per_patient_view_for_intake_nurture_treatment_plan(self):
        patient_a = self._make_patient_with_person("01")
        patient_b = self._make_patient_with_person("02")

        Intake.objects.create(
            practice=self.practice,
            person=patient_a.person,
            first_name="A",
            last_name="Intake",
            phone_number="08030000101",
            status="active",
            journey_stage=self.intake_stage,
        )
        Nurture.objects.create(
            practice=self.practice,
            person=patient_a.person,
            first_name="A",
            last_name="Nurture",
            phone_number="08030000102",
            status="active",
            journey_stage=self.nurture_stage,
        )
        TreatmentPlan.objects.create(
            practice=self.practice,
            patient=patient_b,
            status="pending",
            journey_stage=self.open_plan_stage,
        )

        batched = build_outstanding_items_map(self.practice.id, [patient_a, patient_b])

        self.assertEqual(
            batched[patient_a.id]["journeys"],
            self._call_legacy_view(patient_a)["journeys"],
        )
        self.assertEqual(
            batched[patient_b.id]["journeys"],
            self._call_legacy_view(patient_b)["journeys"],
        )
        self.assertEqual(batched[patient_a.id]["counts"]["journeys"], 2)
        self.assertEqual(batched[patient_b.id]["counts"]["journeys"], 1)

    def test_batched_map_matches_legacy_per_patient_view_for_tasks(self):
        patient = self._make_patient_with_person("03")
        Task.objects.create(
            practice=self.practice,
            patient=patient,
            title="Call back",
            status="pending",
            priority="low",
            due_date=timezone.now().date() + timedelta(days=1),
        )
        Task.objects.create(
            practice=self.practice,
            patient=patient,
            title="Already done",
            status="completed",
            priority="low",
            due_date=timezone.now().date(),
        )

        batched = build_outstanding_items_map(self.practice.id, [patient])

        self.assertEqual(
            batched[patient.id]["tasks"], self._call_legacy_view(patient)["tasks"]
        )
        self.assertEqual(batched[patient.id]["counts"]["tasks"], 1)

    def test_patient_with_no_open_items_gets_empty_but_present_entry(self):
        patient = self._make_patient_with_person("04")

        batched = build_outstanding_items_map(self.practice.id, [patient])

        self.assertIn(patient.id, batched)
        self.assertEqual(
            batched[patient.id],
            {
                "journeys": [],
                "tasks": [],
                "labs": [],
                "counts": {"journeys": 0, "tasks": 0, "labs": 0},
            },
        )

    def test_two_patients_sharing_a_person_both_see_the_shared_persons_items(self):
        # Mirrors the legacy view's own semantics: Task/Lab/TreatmentPlan are
        # joined via patient__person, so a person with 2 duplicate Patient
        # rows sees the SAME outstanding items under either Patient id.
        patient_a = self._make_patient_with_person("05")
        patient_dup = Patient.objects.create(
            practice=self.practice,
            person=patient_a.person,
            first_name="Dup",
            last_name="Of A",
            phone_number="08030000199",
        )
        Task.objects.create(
            practice=self.practice,
            patient=patient_a,
            title="Shared task",
            status="pending",
            priority="low",
            due_date=timezone.now().date(),
        )

        batched = build_outstanding_items_map(
            self.practice, [patient_a, patient_dup]
        )

        self.assertEqual(batched[patient_a.id]["tasks"], batched[patient_dup.id]["tasks"])
        self.assertEqual(len(batched[patient_a.id]["tasks"]), 1)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd TreatmentPathBackend/TreatmentPath && source ../venv/bin/activate && python manage.py test TreatmentPlan.tests.test_outstanding_summary --keepdb -v 2`
Expected: FAIL / ERROR with `ModuleNotFoundError: No module named 'TreatmentPlan.outstanding_summary'` (the module doesn't exist yet).

- [ ] **Step 3: Write minimal implementation**

```python
# TreatmentPathBackend/TreatmentPath/TreatmentPlan/outstanding_summary.py
"""
Batched patient outstanding-items summaries for embedding in the Day List
response.

``PatientOutstandingItemsView`` (TreatmentPlan/views/patient_outstanding_views.py)
answers "what's still open for this ONE patient's identity" via 5 queries
(Intake, Nurture, TreatmentPlan, Task, LabCase). The Day List page was
calling that endpoint (plus a broader Open Plans fuzzy-match endpoint, see
open_plans_summary.py) once per unique patient/contact on every load — up
to 4 HTTP requests per row.

``build_outstanding_items_map`` produces the SAME per-patient shape for many
patients in a small constant number of queries, keyed by ``Patient.id``, so
the Day List serializer can embed ``outstanding_items`` per row and the
frontend hook never has to fetch.
"""

from Labs.models import LabCase
from Tasks.models import Task

from .models import Intake, Nurture, TreatmentPlan
from .views.patient_outstanding_views import (
    LAB_TERMINAL_STATUS,
    OPEN_TASK_STATUSES,
    OPEN_TREATMENT_PLAN_STATUSES,
    _active_journey_qs,
)


def _empty_entry():
    return {
        "journeys": [],
        "tasks": [],
        "labs": [],
        "counts": {"journeys": 0, "tasks": 0, "labs": 0},
    }


def build_outstanding_items_map(practice_id, patients):
    """Return ``{patient_id: outstanding_items_dict}`` for the given patients.

    ``patients`` must be already-fetched ``Patient`` instances (e.g.
    ``patient_lookup.values()``). Patients with no ``person_id`` are simply
    absent journeys/tasks/labs-wise (their entry is still present, just
    empty) — mirrors the legacy view's "no resolvable person -> nothing to
    surface" branch.
    """
    patients = [p for p in patients if p is not None]
    if not practice_id or not patients:
        return {}

    person_ids = {p.person_id for p in patients if p.person_id}
    by_person = {pid: _empty_entry() for pid in person_ids}

    if person_ids:
        intakes = _active_journey_qs(
            Intake.objects.filter(practice_id=practice_id, person_id__in=person_ids)
        ).select_related("journey_stage")
        for j in intakes:
            by_person[j.person_id]["journeys"].append(
                {
                    "kind": "intake",
                    "id": str(j.id),
                    "stage": j.journey_stage.name if j.journey_stage_id else None,
                    "status": j.status,
                    "updated_at": getattr(j, "updated_at", None),
                }
            )

        nurtures = _active_journey_qs(
            Nurture.objects.filter(practice_id=practice_id, person_id__in=person_ids)
        ).select_related("journey_stage")
        for j in nurtures:
            by_person[j.person_id]["journeys"].append(
                {
                    "kind": "nurture",
                    "id": str(j.id),
                    "stage": j.journey_stage.name if j.journey_stage_id else None,
                    "status": j.status,
                    "updated_at": getattr(j, "updated_at", None),
                }
            )

        plans = (
            TreatmentPlan.objects.filter(
                practice_id=practice_id,
                patient__person_id__in=person_ids,
                status__in=OPEN_TREATMENT_PLAN_STATUSES,
            )
            .exclude(journey_stage__slug="archive")
            .select_related("journey_stage", "patient")
        )
        for j in plans:
            by_person[j.patient.person_id]["journeys"].append(
                {
                    "kind": "treatment_plan",
                    "id": str(j.id),
                    "stage": j.journey_stage.name if j.journey_stage_id else None,
                    "status": j.status,
                    "updated_at": getattr(j, "updated_at", None),
                }
            )

        tasks = Task.objects.filter(
            patient__person_id__in=person_ids, status__in=OPEN_TASK_STATUSES
        ).select_related("patient").order_by("due_date")
        for t in tasks:
            by_person[t.patient.person_id]["tasks"].append(
                {
                    "id": t.id,
                    "title": t.title,
                    "description": (t.description or "")[:200],
                    "status": t.status,
                    "priority": t.priority,
                    "due_date": t.due_date,
                }
            )

        labs = (
            LabCase.objects.filter(patient__person_id__in=person_ids)
            .exclude(status=LAB_TERMINAL_STATUS)
            .select_related("catalogue_item", "patient")
            .order_by("appointment_date")
        )
        for lab in labs:
            item_name = getattr(getattr(lab, "catalogue_item", None), "item_name", None)
            by_person[lab.patient.person_id]["labs"].append(
                {
                    "id": lab.id,
                    "description": item_name
                    or getattr(lab, "patient_name", None)
                    or f"Case {lab.id}",
                    "status": lab.status,
                    "is_overdue": bool(getattr(lab, "is_overdue", False)),
                    "appointment_date": getattr(lab, "appointment_date", None),
                }
            )

        for entry in by_person.values():
            entry["counts"] = {
                "journeys": len(entry["journeys"]),
                "tasks": len(entry["tasks"]),
                "labs": len(entry["labs"]),
            }

    result = {}
    for patient in patients:
        entry = by_person.get(patient.person_id, _empty_entry())
        result[patient.id] = entry
    return result
```

`_active_journey_qs`, `LAB_TERMINAL_STATUS`, `OPEN_TASK_STATUSES`, `OPEN_TREATMENT_PLAN_STATUSES` are already defined at module level in `patient_outstanding_views.py:41-48` — import them rather than redefining, so the two implementations can never silently drift apart on which statuses count as "open".

- [ ] **Step 4: Run test to verify it passes**

Run: `cd TreatmentPathBackend/TreatmentPath && source ../venv/bin/activate && python manage.py test TreatmentPlan.tests.test_outstanding_summary --keepdb -v 2`
Expected: PASS (4 tests)

- [ ] **Step 5: Commit**

```bash
git add TreatmentPath/TreatmentPlan/outstanding_summary.py TreatmentPath/TreatmentPlan/tests/test_outstanding_summary.py
git commit -m "Add batched outstanding-items selector for Day List embedding"
```

---

## Task 2: Batch the "Open Plans" fuzzy (NonRegisteredPatient) match

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/outstanding_summary.py`
- Test: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_outstanding_summary.py` (extend)

**Interfaces:**
- Consumes: `build_outstanding_items_map` from Task 1 (this task modifies it in place).
- Produces: same `build_outstanding_items_map(practice, patients)` signature — now ALSO includes `treatment_plan`-kind journeys matched via `Patient.treatment_plan_ids` and via fuzzy `NonRegisteredPatient` phone/email matching, status `"pending"` only (mirrors `PatientJourneysView`'s "Open Plans" type). Deduped against the person-linked treatment plans already added in Task 1 (a plan matched both ways must not appear twice).

This replicates `PatientJourneysView._build_tp_filter()` / `_get_matching_non_registered_patient_ids()` (`TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/patient_journey_views.py:105-172`), but inverts the per-patient `NonRegisteredPatient` phone scan into ONE practice-wide pass instead of re-scanning `NonRegisteredPatient` once per patient.

- [ ] **Step 1: Write the failing test — treatment_plan_ids and fuzzy NonRegisteredPatient match**

Append to `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_outstanding_summary.py`:

```python
from TreatmentPlan.models import NonRegisteredPatient


@override_settings(SECURE_SSL_REDIRECT=False)
class BuildOutstandingItemsMapOpenPlansTests(TestCase):
    """Covers the harder half: Open Plans matched via treatment_plan_ids or
    a fuzzy NonRegisteredPatient email/phone match, NOT via patient__person.
    """

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Outstanding Fuzzy Practice", slug="outstanding-fuzzy"
        )
        self.open_plan_stage = JourneyStage.objects.get(
            practice=self.practice, slug="open-plan"
        )

    def _make_patient(self, suffix, email=None, phone=None):
        patient = Patient.objects.create(
            practice=self.practice,
            first_name=f"P-{suffix}",
            last_name="Last",
            email=email,
            phone_number=phone or f"0803000{suffix.zfill(4)}",
        )
        patient.refresh_from_db()
        return patient

    def test_matches_plan_via_treatment_plan_ids_list_field(self):
        patient = self._make_patient("10")
        # A TreatmentPlan the patient's own person link does NOT cover
        # (different patient FK) but that patient.treatment_plan_ids lists —
        # the exact scenario _build_tp_filter's `Q(id__in=plan_ids)` exists for.
        other_patient = self._make_patient("11")
        plan = TreatmentPlan.objects.create(
            practice=self.practice,
            patient=other_patient,
            status="pending",
            journey_stage=self.open_plan_stage,
        )
        patient.treatment_plan_ids = [plan.id]
        patient.save(update_fields=["treatment_plan_ids"])

        from TreatmentPlan.outstanding_summary import build_outstanding_items_map

        batched = build_outstanding_items_map(self.practice.id, [patient])

        journey_ids = {j["id"] for j in batched[patient.id]["journeys"]}
        self.assertIn(str(plan.id), journey_ids)

    def test_matches_plan_via_fuzzy_email_on_non_registered_patient(self):
        patient = self._make_patient("12", email="shared@example.com")
        nrp = NonRegisteredPatient.objects.create(
            practice=self.practice,
            first_name="Non",
            last_name="Registered",
            email="shared@example.com",
        )
        plan = TreatmentPlan.objects.create(
            practice=self.practice,
            non_registered_patient=nrp,
            status="pending",
            journey_stage=self.open_plan_stage,
        )

        from TreatmentPlan.outstanding_summary import build_outstanding_items_map

        batched = build_outstanding_items_map(self.practice.id, [patient])

        journey_ids = {j["id"] for j in batched[patient.id]["journeys"]}
        self.assertIn(str(plan.id), journey_ids)

    def test_matches_plan_via_fuzzy_normalized_phone_on_non_registered_patient(self):
        patient = self._make_patient("13", phone="08033330013")
        nrp = NonRegisteredPatient.objects.create(
            practice=self.practice,
            first_name="Non",
            last_name="Registered2",
            phone_number="+2348033330013",  # same number, different formatting
        )
        plan = TreatmentPlan.objects.create(
            practice=self.practice,
            non_registered_patient=nrp,
            status="pending",
            journey_stage=self.open_plan_stage,
        )

        from TreatmentPlan.outstanding_summary import build_outstanding_items_map

        batched = build_outstanding_items_map(self.practice.id, [patient])

        journey_ids = {j["id"] for j in batched[patient.id]["journeys"]}
        self.assertIn(str(plan.id), journey_ids)

    def test_active_status_plans_are_not_matched_by_the_fuzzy_path(self):
        # Open Plans == status "pending" only; the person-linked path in
        # Task 1 already covers "active" via OPEN_TREATMENT_PLAN_STATUSES,
        # so the fuzzy path must not ALSO pull in active plans (would
        # duplicate/diverge from PatientJourneysView's own type scoping).
        patient = self._make_patient("14", email="active@example.com")
        nrp = NonRegisteredPatient.objects.create(
            practice=self.practice,
            first_name="Active",
            last_name="Plan",
            email="active@example.com",
        )
        plan = TreatmentPlan.objects.create(
            practice=self.practice,
            non_registered_patient=nrp,
            status="active",
            journey_stage=self.open_plan_stage,
        )

        from TreatmentPlan.outstanding_summary import build_outstanding_items_map

        batched = build_outstanding_items_map(self.practice.id, [patient])

        journey_ids = {j["id"] for j in batched[patient.id]["journeys"]}
        self.assertNotIn(str(plan.id), journey_ids)

    def test_plan_matched_both_person_linked_and_fuzzy_appears_once(self):
        patient = self._make_patient("15", email="dup@example.com")
        # Person-linked path (Task 1): plan directly on this patient.
        plan = TreatmentPlan.objects.create(
            practice=self.practice,
            patient=patient,
            status="pending",
            journey_stage=self.open_plan_stage,
        )
        # ALSO put its id in treatment_plan_ids, so the fuzzy path in this
        # task would independently re-discover the same plan.
        patient.treatment_plan_ids = [plan.id]
        patient.save(update_fields=["treatment_plan_ids"])

        from TreatmentPlan.outstanding_summary import build_outstanding_items_map

        batched = build_outstanding_items_map(self.practice.id, [patient])

        matching = [j for j in batched[patient.id]["journeys"] if j["id"] == str(plan.id)]
        self.assertEqual(len(matching), 1)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd TreatmentPathBackend/TreatmentPath && source ../venv/bin/activate && python manage.py test TreatmentPlan.tests.test_outstanding_summary.BuildOutstandingItemsMapOpenPlansTests --keepdb -v 2`
Expected: FAIL — `test_matches_plan_via_treatment_plan_ids_list_field`, `..._fuzzy_email...`, `..._fuzzy_normalized_phone...` all fail because the current implementation from Task 1 only looks at `patient__person_id__in=`, so none of these plans (linked via `treatment_plan_ids` or `non_registered_patient`) show up.

- [ ] **Step 3: Write minimal implementation**

Replace the whole file with Task 1's content plus this addition (import `normalize_phone_for_comparison` and `NonRegisteredPatient`, and extend `build_outstanding_items_map`):

```python
# Add to imports at the top of outstanding_summary.py
from .models import NonRegisteredPatient
from .utils.phones import normalize_phone_for_comparison
```

Add this new private helper below `_empty_entry()`:

```python
def _build_non_registered_patient_index(practice_id):
    """One practice-wide pass building {normalized_email/phone: {nrp_id, ...}}.

    Mirrors PatientJourneysView._get_matching_non_registered_patient_ids,
    but that function re-scans EVERY NonRegisteredPatient with a phone number
    once per patient (O(patients * non_registered_patients)). This builds the
    index ONCE for the whole batch and matches every patient against it in a
    single pass — same matching semantics, one query instead of N.
    """
    by_email = {}
    by_phone = {}
    for nrp in NonRegisteredPatient.objects.filter(practice_id=practice_id).only(
        "id", "email", "phone_number"
    ):
        if nrp.email:
            by_email.setdefault(nrp.email.strip().lower(), set()).add(nrp.id)
        if nrp.phone_number:
            # No single default_country_code here (this is practice-wide,
            # not per-patient) — normalize_phone_for_comparison degrades to
            # its no-country-code behavior, which is exactly what the
            # legacy per-patient loop did for candidates it iterates without
            # re-deriving their own country code either (it only applies
            # the PATIENT's default_country_code, using the nrp's raw
            # number as-is on the other side of the comparison).
            normalized = normalize_phone_for_comparison(nrp.phone_number)
            if normalized:
                by_phone.setdefault(normalized, set()).add(nrp.id)
    return by_email, by_phone


def _matching_non_registered_patient_ids(patient, by_email, by_phone):
    ids = set()
    if patient.email:
        ids |= by_email.get(patient.email.strip().lower(), set())

    default_country_code = patient.get_country_code()
    for phone in (patient.phone_number, patient.secondary_phone_number):
        normalized = normalize_phone_for_comparison(phone, default_country_code)
        if normalized:
            ids |= by_phone.get(normalized, set())
    return ids
```

Then, inside `build_outstanding_items_map`, after the existing `if person_ids:` block's `tasks`/`labs` loops but still inside the function (before the final `result = {}` assembly), add the fuzzy Open-Plans pass — this runs for ALL patients, not just ones with a `person_id`, since `treatment_plan_ids`/email/phone matching doesn't require a resolved Person:

```python
    # ── Open Plans: treatment_plan_ids + fuzzy NonRegisteredPatient match ──
    # Runs for every patient (not just person-linked ones) — mirrors
    # PatientJourneysView._build_tp_filter, which works off Patient fields
    # directly, independent of person resolution.
    plan_ids_by_patient = {}
    all_plan_ids = set()
    for p in patients:
        ids = set(p.treatment_plan_ids or [])
        plan_ids_by_patient[p.id] = ids
        all_plan_ids |= ids

    by_email, by_phone = _build_non_registered_patient_index(practice_id)
    nrp_ids_by_patient = {
        p.id: _matching_non_registered_patient_ids(p, by_email, by_phone)
        for p in patients
    }
    all_nrp_ids = set().union(*nrp_ids_by_patient.values()) if patients else set()

    if all_plan_ids or all_nrp_ids:
        from django.db.models import Q

        fuzzy_q = Q()
        if all_plan_ids:
            fuzzy_q |= Q(id__in=all_plan_ids)
        if all_nrp_ids:
            fuzzy_q |= Q(non_registered_patient_id__in=all_nrp_ids)

        fuzzy_plans = list(
            TreatmentPlan.objects.filter(
                fuzzy_q, practice_id=practice_id, status="pending"
            ).select_related("journey_stage")
        )
        fuzzy_by_id = {p.id: p for p in fuzzy_plans}

        for patient in patients:
            matched_plan_ids = (
                plan_ids_by_patient[patient.id]
                | {
                    p.id
                    for p in fuzzy_plans
                    if p.non_registered_patient_id in nrp_ids_by_patient[patient.id]
                }
            )
            if not matched_plan_ids:
                continue

            entry = by_person.setdefault(patient.person_id, _empty_entry())
            existing_ids = {j["id"] for j in entry["journeys"]}
            for plan_id in matched_plan_ids:
                plan = fuzzy_by_id.get(plan_id)
                if plan is None or str(plan.id) in existing_ids:
                    continue
                entry["journeys"].append(
                    {
                        "kind": "treatment_plan",
                        "id": str(plan.id),
                        "stage": plan.journey_stage.name
                        if plan.journey_stage_id
                        else None,
                        "status": plan.status,
                        "updated_at": getattr(plan, "updated_at", None),
                    }
                )
                existing_ids.add(str(plan.id))
            entry["counts"]["journeys"] = len(entry["journeys"])
```

This must run AFTER the existing `for entry in by_person.values(): entry["counts"] = ...` line from Task 1 is moved — reorder so counts are computed once, at the very end, after both the person-linked and fuzzy passes have mutated `by_person`. Move the Task 1 counts-recompute loop (`for entry in by_person.values(): entry["counts"] = {...}`) to run right before `result = {}` at the bottom of the function, after this new block, instead of where Task 1 originally placed it.

Note: `by_person.setdefault(patient.person_id, ...)` — when `patient.person_id` is `None`, this creates one shared `None` key entry that could accidentally merge unrelated patients-without-a-person. Guard against it: skip patients with no `person_id` from the `by_person`-keyed fuzzy attribution and instead accumulate their fuzzy matches in a separate `by_patient_id_no_person = {}` dict, and in the final `result` assembly, merge that into the direct per-patient result rather than going through `by_person`. Use this corrected final assembly:

```python
    for entry in by_person.values():
        entry["counts"] = {
            "journeys": len(entry["journeys"]),
            "tasks": len(entry["tasks"]),
            "labs": len(entry["labs"]),
        }

    result = {}
    for patient in patients:
        if patient.person_id and patient.person_id in by_person:
            result[patient.id] = by_person[patient.person_id]
        else:
            entry = by_patient_id_no_person.get(patient.id, _empty_entry())
            entry["counts"] = {
                "journeys": len(entry["journeys"]),
                "tasks": len(entry["tasks"]),
                "labs": len(entry["labs"]),
            }
            result[patient.id] = entry
    return result
```

...and change the fuzzy-match loop above to write into `by_patient_id_no_person` (keyed by `patient.id`, initialized as `by_patient_id_no_person = {}` alongside `plan_ids_by_patient`) whenever `patient.person_id` is falsy, using the same append-with-dedup logic against `by_patient_id_no_person.setdefault(patient.id, _empty_entry())` instead of `by_person.setdefault(patient.person_id, ...)`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd TreatmentPathBackend/TreatmentPath && source ../venv/bin/activate && python manage.py test TreatmentPlan.tests.test_outstanding_summary --keepdb -v 2`
Expected: PASS (all tests from Task 1 + Task 2, 9 total)

- [ ] **Step 5: Commit**

```bash
git add TreatmentPath/TreatmentPlan/outstanding_summary.py TreatmentPath/TreatmentPlan/tests/test_outstanding_summary.py
git commit -m "Batch the Open Plans fuzzy NonRegisteredPatient match into one practice-wide pass"
```

---

## Task 3: Wire into the Day List serializer

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/serializers.py:642-782` (`DayListAppointmentSerializer`)
- Modify: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/views/dentally_views.py:2017-2030` (no change needed here — `build_daylist_maps` is already spread into `context`, this task only changes what that method returns)
- Test: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/test_daylist_outstanding_items.py` (create — this app keeps tests as flat files directly under `dentallyIntegration/`, e.g. `test_marketing_consent_mapping.py`, NOT in a `tests/` subpackage; follow that convention)

**Interfaces:**
- Consumes: `build_outstanding_items_map(practice_id, patients)` from Task 1+2.
- Produces: every Day List row gains `"outstanding_items"` — same shape as Task 1/2's map values.

- [ ] **Step 1: Write the failing test**

```python
# TreatmentPathBackend/TreatmentPath/dentallyIntegration/test_daylist_outstanding_items.py
"""
Regression test: GET /api/backend/dentally/day-list/ must embed
`outstanding_items` per row (TreatmentPlan/outstanding_summary.py), so the
frontend Day List no longer fires one /outstanding-items/ (+ /journeys/)
request per patient on every load.

setUp mirrors TreatmentPlan/tests/test_active_plans_consent_summary.py's
practice+subscription+auth pattern.
"""

from datetime import timedelta

from django.contrib.auth import get_user_model
from django.test import override_settings
from django.urls import reverse
from django.utils import timezone
from rest_framework import status
from rest_framework.test import APITestCase

from dentallyIntegration.models import DentallyAppointment
from payments.models import CustomerProfile, Subscription, SubscriptionPlan
from Tasks.models import Task
from TreatmentPlan.models import Patient
from UserAuthentication.models import (
    Practice,
    PracticeModuleSubscription,
    UserPracticeRelationship,
)
from UserAuthentication.serializers import CustomTokenObtainPairSerializer


User = get_user_model()


def _auth_as(client, user):
    token = CustomTokenObtainPairSerializer.get_token(user)
    client.credentials(HTTP_AUTHORIZATION=f"Bearer {token.access_token}")


@override_settings(SECURE_SSL_REDIRECT=False)
class DayListOutstandingItemsTests(APITestCase):
    @classmethod
    def _ensure_subscription_plan(cls):
        plan, _ = SubscriptionPlan.objects.get_or_create(
            plan_type="practice",
            billing_period="monthly",
            defaults={
                "name": "Test Practice Plan",
                "price": "0.00",
                "stripe_price_id": "price_test_daylist_outstanding",
                "stripe_product_id": "prod_test_daylist_outstanding",
            },
        )
        return plan

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Daylist Outstanding Practice", slug="daylist-outstanding"
        )
        PracticeModuleSubscription.objects.create(
            practice=self.practice, enabled_modules=["patient_journeys"]
        )
        self.admin = User.objects.create_user(
            email="admin-daylistoutstanding@example.com",
            password="testpass123",
            current_practice=self.practice,
            user_type="admin",
        )
        UserPracticeRelationship.objects.create(
            user=self.admin,
            practice=self.practice,
            role="admin",
            is_practice_admin=True,
            practice_verified=True,
        )
        plan = self._ensure_subscription_plan()
        profile, _ = CustomerProfile.objects.get_or_create(
            user=self.admin,
            practice=self.practice,
            defaults={"stripe_customer_id": "cus_test_daylistoutstanding"},
        )
        now = timezone.now()
        Subscription.objects.create(
            customer_profile=profile,
            subscription_plan=plan,
            stripe_subscription_id="sub_test_daylistoutstanding",
            status="active",
            current_period_start=now,
            current_period_end=now + timedelta(days=30),
        )
        self.today = timezone.now().date()
        _auth_as(self.client, self.admin)

    def _make_appointment_for_patient(self, dentally_patient_id, dentally_id, patient_name):
        return DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=dentally_id,
            dentally_patient_id=dentally_patient_id,
            patient_name=patient_name,
            start_time=timezone.now(),
            state="confirmed",
            dentally_practitioner_id=1,
        )

    def test_day_list_response_embeds_outstanding_items_for_patient_with_open_task(self):
        patient = Patient.objects.create(
            practice=self.practice,
            first_name="Jane",
            last_name="Doe",
            phone_number="08030000901",
            meta_data={"id": 9001},
        )
        self._make_appointment_for_patient(
            dentally_patient_id=9001, dentally_id=1, patient_name="Jane Doe"
        )
        Task.objects.create(
            practice=self.practice,
            patient=patient,
            title="Call back",
            status="pending",
            priority="low",
            due_date=self.today,
        )

        url = reverse("dentally-day-list")
        response = self.client.get(url, {"date": str(self.today)})

        self.assertEqual(response.status_code, status.HTTP_200_OK)
        row = response.data["appointments"][0]
        self.assertIn("outstanding_items", row)
        self.assertEqual(len(row["outstanding_items"]["tasks"]), 1)
        self.assertEqual(row["outstanding_items"]["counts"]["tasks"], 1)

    def test_day_list_response_embeds_empty_outstanding_items_when_nothing_open(self):
        Patient.objects.create(
            practice=self.practice,
            first_name="John",
            last_name="Smith",
            phone_number="08030000902",
            meta_data={"id": 9002},
        )
        self._make_appointment_for_patient(
            dentally_patient_id=9002, dentally_id=2, patient_name="John Smith"
        )

        url = reverse("dentally-day-list")
        response = self.client.get(url, {"date": str(self.today)})

        row = response.data["appointments"][0]
        self.assertIn("outstanding_items", row)
        self.assertEqual(
            row["outstanding_items"],
            {"journeys": [], "tasks": [], "labs": [], "counts": {"journeys": 0, "tasks": 0, "labs": 0}},
        )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd TreatmentPathBackend/TreatmentPath && source ../venv/bin/activate && python manage.py test dentallyIntegration.test_daylist_outstanding_items --keepdb -v 2`
Expected: FAIL — `KeyError: 'outstanding_items'` (field doesn't exist on the response yet).

- [ ] **Step 3: Write minimal implementation**

In `dentallyIntegration/serializers.py`, add the field declaration right after `consent_summary` (line 487):

```python
    # Consent summary — from the Documents app SigningRequest
    consent_summary = serializers.SerializerMethodField()

    # Outstanding journeys/tasks/labs for the patient — batched via
    # build_daylist_maps' outstanding_by_patient context map (was N+1: one
    # /outstanding-items/ + one /journeys/?type=Open+Plans request per
    # patient on every Day List load). See TreatmentPlan/outstanding_summary.py.
    outstanding_items = serializers.SerializerMethodField()
```

Add `"outstanding_items"` to `Meta.fields`, right after `"consent_summary"` (near line 573):

```python
            "consent_summary",
            "outstanding_items",
```

Add the getter method near `get_consent_summary` (same style — reads from `self.context`, falls back to a live per-object lookup only when the context map wasn't provided, matching every other batched field in this serializer):

```python
    def get_outstanding_items(self, obj):
        patient = self._get_patient(obj)
        if not patient:
            return None

        outstanding_map = self.context.get("outstanding_by_patient")
        if outstanding_map is not None:
            return outstanding_map.get(patient.id)

        from TreatmentPlan.outstanding_summary import build_outstanding_items_map

        return build_outstanding_items_map(obj.practice, [patient]).get(patient.id)
```

Extend `build_daylist_maps()` (line 689) to compute the new map and include it in its return dict. `practice_id = appointments[0].practice_id` is already resolved a few lines into that method (line ~712) — reuse it directly, no extra `Practice` query needed since `build_outstanding_items_map` takes `practice_id` (an int), not a `Practice` instance. Add right before the `return {` at the end of that method:

```python
        # Outstanding journeys/tasks/labs, batched across every patient on
        # the page — was 2-4 HTTP requests per patient from the frontend.
        from TreatmentPlan.outstanding_summary import build_outstanding_items_map

        outstanding_by_patient = build_outstanding_items_map(
            practice_id, patient_lookup.values()
        )
```

and add `"outstanding_by_patient": outstanding_by_patient,` to that method's final returned dict (alongside `"signing_by_patient"` etc).

- [ ] **Step 4: Run test to verify it passes**

Run: `cd TreatmentPathBackend/TreatmentPath && source ../venv/bin/activate && python manage.py test dentallyIntegration.test_daylist_outstanding_items TreatmentPlan.tests.test_outstanding_summary --keepdb -v 2`
Expected: PASS

- [ ] **Step 5: Run the broader dentallyIntegration test suite to check for regressions**

Run: `python manage.py test dentallyIntegration --keepdb -v 1`
Expected: PASS, same pass count as before this task (no new failures)

- [ ] **Step 6: Commit**

```bash
git add TreatmentPath/dentallyIntegration/serializers.py TreatmentPath/dentallyIntegration/test_daylist_outstanding_items.py TreatmentPath/TreatmentPlan/outstanding_summary.py
git commit -m "Embed outstanding_items in Day List response via batched selector"
```

---

## Task 4: Frontend — carry `outstanding_items` onto the `Appointment` type

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/day-list/types.ts`
- Modify: `perfect-pixel-playground-project/src/pages/day-list/lib/fetchDayListAppointments.ts`
- Test: `perfect-pixel-playground-project/src/pages/day-list/lib/fetchDayListAppointments.test.ts` (extend if it exists — check with `ls perfect-pixel-playground-project/src/pages/day-list/lib/*.test.ts` first; create alongside the file if it doesn't)

**Interfaces:**
- Consumes: `OutstandingResponse` type from `src/components/patients/patient-panel/PatientOutstandingItems.tsx:49-54` (reuse, don't redeclare — same precedent as reusing `RawConsentSummary` for `consentSummary` in the JourneyBoard fix).
- Produces: `Appointment.outstanding_items?: OutstandingResponse | null` — consumed by Task 5.

- [ ] **Step 1: Write the failing test**

```typescript
// Add to perfect-pixel-playground-project/src/pages/day-list/lib/fetchDayListAppointments.test.ts
// (create this file if it doesn't exist yet, following the describe/it
// pattern used by src/components/journey-board/adapters.test.ts)
import { describe, expect, it } from "vitest";
import { mapDayListAppointment } from "./fetchDayListAppointments";

describe("mapDayListAppointment — outstanding_items", () => {
  it("carries an embedded outstanding_items through onto the appointment", () => {
    const appointment = mapDayListAppointment({
      id: 1,
      patient_name: "Jane Doe",
      appointment_time: "09:00",
      duration: 30,
      appointment_type: "Checkup",
      provider: "Dr Smith",
      status: "pending",
      outstanding_items: {
        journeys: [{ kind: "intake", id: "5", stage: "Intake", status: "active", updated_at: null }],
        tasks: [],
        labs: [],
        counts: { journeys: 1, tasks: 0, labs: 0 },
      },
    } as any);
    expect(appointment.outstanding_items).toEqual({
      journeys: [{ kind: "intake", id: "5", stage: "Intake", status: "active", updated_at: null }],
      tasks: [],
      labs: [],
      counts: { journeys: 1, tasks: 0, labs: 0 },
    });
  });

  it("leaves outstanding_items undefined when the field is absent (must trigger the old fetch fallback, not read as empty)", () => {
    const appointment = mapDayListAppointment({
      id: 2,
      patient_name: "John Smith",
      appointment_time: "09:00",
      duration: 30,
      appointment_type: "Checkup",
      provider: "Dr Smith",
      status: "pending",
    } as any);
    expect(appointment.outstanding_items).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd perfect-pixel-playground-project && npx vitest run src/pages/day-list/lib/fetchDayListAppointments.test.ts`
Expected: FAIL — `appointment.outstanding_items` is `undefined` in the FIRST test (should be the embedded object) because the field doesn't exist on `Appointment` or get mapped yet.

- [ ] **Step 3: Write minimal implementation**

In `perfect-pixel-playground-project/src/pages/day-list/types.ts`, add the import at the top of the file (near other type imports) and the field on `Appointment` right after `consent_signing_request_id` (line 140):

```typescript
import type { OutstandingResponse } from "@/components/patients/patient-panel/PatientOutstandingItems";
```

```typescript
  consent_signing_request_id?: string | number | null;
  // Embedded outstanding journeys/tasks/labs (TreatmentPlan/outstanding_summary.py).
  // undefined means the Day List endpoint didn't resolve a linked patient for
  // this row — useDayListOutstandingItems falls back to its old fetch chain
  // ONLY in that case. null/a real object means "resolved, use as-is".
  outstanding_items?: OutstandingResponse | null;
}
```

In `perfect-pixel-playground-project/src/pages/day-list/lib/fetchDayListAppointments.ts`, add `outstanding_items` to the `DayListApiAppointment` interface (near `consent_summary` at line 92-98):

```typescript
  outstanding_items?: import("@/components/patients/patient-panel/PatientOutstandingItems").OutstandingResponse | null;
```

and in `mapDayListAppointment` (after line 167, alongside the other `consent_*` mappings):

```typescript
    outstanding_items: apt.outstanding_items,
```

(Deliberately NOT `apt.outstanding_items ?? null` — preserve the `undefined` vs `null` distinction exactly like the earlier `consentSummary` fix, since Task 5 branches on `!== undefined`.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd perfect-pixel-playground-project && npx vitest run src/pages/day-list/lib/fetchDayListAppointments.test.ts`
Expected: PASS (2 tests)

- [ ] **Step 5: Typecheck**

Run: `cd perfect-pixel-playground-project && npx tsc --noEmit -p tsconfig.app.json 2>&1 | grep -i "day-list\|fetchDayListAppointments"`
Expected: no new errors (compare against a run before this task's changes if any pre-existing errors show up, same technique as the JourneyBoard fix — `git stash` this task's files and re-run to confirm any remaining errors are pre-existing)

- [ ] **Step 6: Commit**

```bash
git add src/pages/day-list/types.ts src/pages/day-list/lib/fetchDayListAppointments.ts src/pages/day-list/lib/fetchDayListAppointments.test.ts
git commit -m "Thread outstanding_items through the Day List appointment mapper"
```

---

## Task 5: Frontend — read embedded data first, fetch only for unresolved rows

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/day-list/hooks/useDayListOutstandingItems.ts`
- Test: `perfect-pixel-playground-project/src/pages/day-list/hooks/useDayListOutstandingItems.test.ts` (create — no existing test file for this hook; follow the `renderHook` pattern used elsewhere in this codebase for hooks with `useFetchWithAuth`, e.g. check `src/hooks/*.test.ts` for a close analog to copy the mock setup from)

**Interfaces:**
- Consumes: `Appointment.outstanding_items` from Task 4.
- Produces: same exported signature as today — `useDayListOutstandingItems(appointments: Appointment[]): Record<string, OutstandingResponse>` and `getOutstandingKey(appointment: Appointment): string | null` — **unchanged**, so `DayListPage.tsx` requires no edits.

- [ ] **Step 1: Write the failing test**

```typescript
// perfect-pixel-playground-project/src/pages/day-list/hooks/useDayListOutstandingItems.test.ts
import { describe, expect, it, vi, beforeEach } from "vitest";
import { renderHook, waitFor } from "@testing-library/react";
import { useDayListOutstandingItems, getOutstandingKey } from "./useDayListOutstandingItems";
import type { Appointment } from "../types";

const mockFetch = vi.fn();
vi.mock("@/lib/helpers", () => ({
  useFetchWithAuth: () => mockFetch,
}));

function makeAppointment(overrides: Partial<Appointment>): Appointment {
  return {
    id: "1",
    patientName: "Jane Doe",
    age: null,
    patientId: 1,
    time: "09:00",
    duration: 30,
    type: "Checkup",
    description: "Checkup",
    provider: "Dr Smith",
    practitioner_id: null,
    quick_note: "",
    appointmentsToday: null,
    dueItems: [],
    opportunities: [],
    treatments: [],
    lastTaken: null,
    status: "pending",
    ...overrides,
  } as Appointment;
}

describe("useDayListOutstandingItems", () => {
  beforeEach(() => {
    mockFetch.mockReset();
  });

  it("uses the embedded outstanding_items and does NOT fetch when every appointment already has one", async () => {
    const embedded = {
      journeys: [{ kind: "intake" as const, id: "9", stage: "Intake", status: "active", updated_at: null }],
      tasks: [],
      labs: [],
      counts: { journeys: 1, tasks: 0, labs: 0 },
    };
    const appointment = makeAppointment({ patient_id: 42, outstanding_items: embedded });

    const { result } = renderHook(() => useDayListOutstandingItems([appointment]));

    await waitFor(() => {
      const key = getOutstandingKey(appointment)!;
      expect(result.current[key]).toEqual(embedded);
    });
    expect(mockFetch).not.toHaveBeenCalled();
  });

  it("falls back to fetching for an appointment with no embedded outstanding_items (unlinked patient)", async () => {
    mockFetch.mockResolvedValue({
      ok: true,
      json: async () => ({ journeys: [], tasks: [], labs: [], counts: { journeys: 0, tasks: 0, labs: 0 } }),
    });
    const appointment = makeAppointment({ patient_id: null, contact_id: null, patient_email: "x@example.com" });

    renderHook(() => useDayListOutstandingItems([appointment]));

    await waitFor(() => {
      expect(mockFetch).toHaveBeenCalled();
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd perfect-pixel-playground-project && npx vitest run src/pages/day-list/hooks/useDayListOutstandingItems.test.ts`
Expected: FAIL — first test fails because `mockFetch` IS called today (the hook always fetches, regardless of embedded data).

- [ ] **Step 3: Write minimal implementation**

In `useDayListOutstandingItems.ts`, change the `useEffect` (lines 278-364) to split appointments into "already resolved via embedded data" vs "needs the old fetch chain", before the existing cache-check logic. Replace the loop that currently starts `for (const lookup of lookups) {` (line 288) with:

```typescript
    const embeddedByKey = new Map<string, OutstandingResponse>();
    for (const appointment of appointments) {
      if (appointment.outstanding_items === undefined) continue;
      const key = getOutstandingKey(appointment);
      if (!key) continue;
      // null means "resolved, nothing outstanding" — still counts as resolved.
      embeddedByKey.set(key, appointment.outstanding_items ?? emptyOutstandingResponse());
    }

    for (const lookup of lookups) {
      if (embeddedByKey.has(lookup.key)) {
        nextInitialState[lookup.key] = embeddedByKey.get(lookup.key)!;
        continue;
      }
      const cached = getCachedOutstanding(lookup.key);
      if (cached) {
        nextInitialState[lookup.key] = cached;
      } else {
        missingKeys.push(lookup.key);
      }
    }
```

This must be inserted right after `const missingKeys: string[] = [];` (line 286) and BEFORE the existing `setItemsByKey(nextInitialState);` call (line 297) — i.e. it replaces the body of the original `for (const lookup of lookups) { const cached = ...; }` loop with the version above that checks `embeddedByKey` first. Everything else in the effect (the `loadMissingKeys` async function, chunking, caching of fetched results) stays unchanged — `missingKeys` will now correctly only contain appointments whose `outstanding_items` was `undefined`.

Also add a dependency on the appointments' embedded data to the effect's dependency array so a later data refresh (e.g. date change bringing in freshly-embedded rows) re-evaluates correctly — since `appointments` is already a dependency of `lookups` (which the effect depends on via `lookupsSignature`), and `lookupsSignature` doesn't currently encode `outstanding_items`, add one more signature term:

```typescript
  const embeddedSignature = appointments
    .map((a) => (a.outstanding_items === undefined ? "0" : "1"))
    .join("");
```

and add `embeddedSignature` to the `useEffect`'s dependency array (alongside `fetchWithAuth, lookups, lookupsSignature`).

- [ ] **Step 4: Run test to verify it passes**

Run: `cd perfect-pixel-playground-project && npx vitest run src/pages/day-list/hooks/useDayListOutstandingItems.test.ts`
Expected: PASS (2 tests)

- [ ] **Step 5: Run the full day-list test suite for regressions**

Run: `cd perfect-pixel-playground-project && npx vitest run src/pages/day-list/`
Expected: PASS, same or better pass count than before this task

- [ ] **Step 6: Commit**

```bash
git add src/pages/day-list/hooks/useDayListOutstandingItems.ts src/pages/day-list/hooks/useDayListOutstandingItems.test.ts
git commit -m "Read outstanding_items off the Day List row before falling back to per-patient fetches"
```

---

## Task 6: End-to-end verification

**Files:** none (verification only)

- [ ] **Step 1:** Start both dev servers (Django `manage.py runserver`, Vite `npm run dev`) if not already running.
- [ ] **Step 2:** Log into the app, navigate to `/daylist` for a date with several appointments.
- [ ] **Step 3:** Open browser DevTools → Network tab, filter for `outstanding-items` and `journeys`.
- [ ] **Step 4:** Confirm: for appointments with a resolved `patient_id`/`contact_id`, **zero** `outstanding-items` or `patients/<id>/journeys` requests fire. Only appointments genuinely lacking a linked patient (if any exist in the test data) should still trigger the old fallback fetch.
- [ ] **Step 5:** Spot-check the Day List UI: outstanding badges/counts on patient cards match what they showed before this change (compare a couple of patients against the patient-panel Clinical tab's own outstanding-items display for the same patient, which still calls the untouched `PatientOutstandingItemsView` directly — the two should agree).
- [ ] **Step 6:** Run the full backend test suite for touched apps to confirm no regressions: `cd TreatmentPathBackend/TreatmentPath && python manage.py test TreatmentPlan dentallyIntegration --keepdb -v 1`.
- [ ] **Step 7:** Run the full frontend test suite for touched areas: `cd perfect-pixel-playground-project && npx vitest run src/pages/day-list/`.
- [ ] **Step 8:** Run `git status` in both repos to confirm only the intended files changed, no scratch files left behind.

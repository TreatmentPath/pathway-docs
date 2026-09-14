# Draft Plans and Guide Preview — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let staff preview a patient guide exactly as the patient will see it before issuing it, backed by a new `draft` plan state that survives leaving the screen — and stop issued plans being edited through the create flow.

**Architecture:** Pressing Preview saves a `draft` treatment plan, then renders it through **the same payload builder and the same React component** the patient-facing guide uses, so the preview cannot drift from the real thing. The payload assembly currently lives inline inside the public view's `retrieve`; it gets extracted once and called by both. Confirming creation flips `draft` → `pending`, which is also the moment the plan stops being editable.

**Tech Stack:** Django 4 + DRF, Celery + django-celery-beat (schedules registered via migrations), React 18 + TypeScript + Vite, vitest + @testing-library/react.

**Spec:** `docs/superpowers/specs/2026-09-12-patient-guide-preview-and-journeys-access-design.md` (§3, §5)

**Covers:** FR1, FR2, FR3 (as amended), FR4, FR5, FR12, FR13, FR14, FR15. FR6–FR11 belong to the companion plan (`2026-09-12-journeys-plan-access-and-qr.md`).

---

## Global Constraints

- **Branch:** frontend work happens on `mannieJuly` in `perfect-pixel-playground-project`. Verify with `git rev-parse --abbrev-ref HEAD` before starting.
- **NEVER run `git add`, `git commit`, or `git push`.** The user performs all VCS operations. Tasks end with verification, not commits.
- **NEVER run `makemigrations` or `migrate` against DEV or PROD.** Generate migration files locally; the user runs them. Flag any migration you create so it reaches the prod to-run list.
- **Do not create new functions by default.** Find the existing helper and reuse it. Where a new one is genuinely required it must be pure where possible, defined **once** in a shared module, and imported — never re-implemented per call site. Task 1 exists purely to honour this.
- **Backend tests:** always `--keepdb`, **NEVER `--noinput`** (it destroys the persistent test DB). Run one suite at a time — two concurrent runs corrupt the shared test DB, so re-run alone before believing a failure.
  ```bash
  source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
  cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
  python manage.py test <dotted.path> --keepdb
  ```
- **Frontend typecheck:** `NODE_OPTIONS=--max-old-space-size=8192 npx tsc --noEmit -p tsconfig.app.json`.
  Two traps: the bare root form checks nothing and exits 0; and WITHOUT the raised heap the
  `-p` form dies of OOM (`exit=134`, `Aborted (core dumped)`) after printing an ~880-line
  stack dump that `wc -l` happily reports as if it were an error count. **Check the exit code**
  and count with `grep -c "error TS"`, never `wc -l`. Baseline: **501 errors** — judge by
  **delta**, and grep the output for your own file paths.
- **Frontend tests:** `npx vitest run <path>`.
- **Exact status value:** `"draft"`. Exact display label: `"Draft"`.
- **Draft lifetime:** 30 days.
- The working tree already contains an applied-but-uncommitted cherry-pick of `41558f946`. Do not revert it.

---

## File Structure

**Created (backend):**
- `TreatmentPath/TreatmentPlan/guide_payload.py` — the single patient-guide payload builder, extracted from `public_treatment_views.retrieve`. One responsibility: turn a `TreatmentPlan` into the dict the guide renders from.
- `TreatmentPath/TreatmentPlan/tests/test_guide_payload.py`
- `TreatmentPath/TreatmentPlan/tests/test_guide_preview_endpoint.py`
- `TreatmentPath/TreatmentPlan/tests/test_draft_plan_status.py`
- `TreatmentPath/TreatmentPlan/tests/test_draft_cleanup.py`
- `TreatmentPath/TreatmentPlan/migrations/XXXX_treatmentplan_draft_status.py`
- `TreatmentPath/TreatmentPlan/migrations/XXXX_register_draft_cleanup_schedule.py`

**Created (frontend):**
- `src/pages/create/GuidePreview.tsx` — the full-screen preview shell (Back to edit / Continue to create).
- `src/pages/create/GuidePreview.test.tsx`

**Modified (backend):**
- `TreatmentPath/TreatmentPlan/models.py:3787-3798` — add the `draft` choice.
- `TreatmentPath/TreatmentPlan/views/public_treatment_views.py:356-455` — call the extracted builder.
- `TreatmentPath/TreatmentPlan/views/treatment_plan_views.py` — preview action + the non-draft edit gate.
- `TreatmentPath/TreatmentPlan/urls.py` — the preview route.
- `TreatmentPath/TreatmentPlan/tasks.py` — the 30-day cleanup task.

**Modified (frontend):**
- `src/pages/treatment-verification/Services.tsx` — accept an authenticated data source.
- `src/pages/Conpact3.tsx` — the Preview action and draft wiring.
- `src/hooks/useTreatmentPlanCreation.ts` — draft save; draft → pending on confirm.
- `src/components/compact/OpenTable.tsx`, `src/components/compact/ActiveTable.tsx` — remove the edit chooser.
- `src/pages/DocsPage.tsx` — list drafts.

---

## Task 1: Extract the guide payload builder

`PublicTreatmentPlanViewSet.retrieve` (`public_treatment_views.py:356-455`) assembles the guide payload inline: serializer, enhanced procedures, practice background, financing terms, interest rate, branding, the practitioner-bio suppression, and the `view_count` strip. The preview endpoint needs **the same dict**. Copying it would guarantee the two drift, so it is extracted once before anything else is built.

Note what stays behind: the `view_count += 1` increment at `:385` is engagement tracking, which FR3 forbids in preview. It must remain in the public view and **must not** move into the builder.

**Files:**
- Create: `TreatmentPath/TreatmentPlan/guide_payload.py`
- Create: `TreatmentPath/TreatmentPlan/tests/test_guide_payload.py`
- Modify: `TreatmentPath/TreatmentPlan/views/public_treatment_views.py:356-455`

**Interfaces:**
- Consumes: `TreatmentPlanSerializer`, `_build_enhanced_category_data`, `practice_branding_payload` (all already in `public_treatment_views.py`).
- Produces: `build_guide_payload(treatment_plan, request) -> dict`. Task 4 calls it.

- [ ] **Step 1: Write the failing test**

```python
# TreatmentPath/TreatmentPlan/tests/test_guide_payload.py
from django.test import RequestFactory, TestCase

from TreatmentPlan.guide_payload import build_guide_payload


class BuildGuidePayloadTests(GuidePlanTestCase):
    """The guide payload must be built in exactly one place.

    Both the public patient view and the staff preview render from this dict;
    if they ever build it separately, the preview stops predicting what the
    patient sees, which is the entire point of the feature.
    """

    def setUp(self):
        super().setUp()
        self.plan = self.make_plan_with_procedure()

    def test_includes_enhanced_procedures(self):
        payload = build_guide_payload(self.plan, RequestFactory().get("/"))
        self.assertIn("procedures", payload)
        self.assertTrue(payload["procedures"], "procedures must not be empty")

    def test_includes_practice_branding_and_financing(self):
        payload = build_guide_payload(self.plan, RequestFactory().get("/"))
        for key in (
            "practice_service_background_image",
            "practice_financing_terms",
            "practice_interest_rate",
        ):
            self.assertIn(key, payload)

    def test_strips_view_count(self):
        payload = build_guide_payload(self.plan, RequestFactory().get("/"))
        self.assertNotIn("view_count", payload)

    def test_does_not_increment_view_count(self):
        before = self.plan.view_count
        build_guide_payload(self.plan, RequestFactory().get("/"))
        self.plan.refresh_from_db()
        self.assertEqual(
            self.plan.view_count,
            before,
            "the builder must not track engagement — FR3 forbids it in preview",
        )

    def test_suppresses_practitioner_bio_when_the_practice_disables_it(self):
        self.practice.show_dentist_bio = False
        self.practice.save(update_fields=["show_dentist_bio"])

        payload = build_guide_payload(self.plan, RequestFactory().get("/"))

        self.assertIsNone(payload.get("practitioner"))
        self.assertIsNone(payload.get("practitioner_profile"))
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python manage.py test TreatmentPlan.tests.test_guide_payload --keepdb`
Expected: FAIL — no module `TreatmentPlan.guide_payload`.

- [ ] **Step 3: Write minimal implementation**

```python
# TreatmentPath/TreatmentPlan/guide_payload.py
"""The one place the patient-guide payload is assembled.

Both the public patient view (code-authorised) and the staff preview endpoint
(session-authorised) render from `build_guide_payload`. They differ only in how
the caller proves they may see the plan — never in what the guide contains.

Engagement tracking deliberately lives OUTSIDE this function: the public view
increments `view_count` itself, because FR3 forbids preview from recording a
view.
"""

from .serializers import TreatmentPlanSerializer
from .views.public_treatment_views import (
    _build_enhanced_category_data,
    practice_branding_payload,
)


def resolve_guide_practice(treatment_plan):
    """The practice whose branding a guide renders with.

    A plan may hang off a registered patient, a non-registered patient, or only
    the plan's own practice FK — all three occur in production.
    """
    if treatment_plan.patient:
        return treatment_plan.patient.practice
    if treatment_plan.non_registered_patient:
        return treatment_plan.non_registered_patient.practice
    return treatment_plan.practice


def build_guide_payload(treatment_plan, request):
    """Serialize `treatment_plan` into the dict the patient guide renders from."""
    data = TreatmentPlanSerializer(treatment_plan).data

    data["procedures"] = [
        _build_enhanced_category_data(plan_category, request)
        for plan_category in treatment_plan.plan_categories.all()
    ]

    practice = resolve_guide_practice(treatment_plan)

    if practice and practice.service_background_image:
        data["practice_service_background_image"] = request.build_absolute_uri(
            practice.service_background_image.url
        )
    else:
        data["practice_service_background_image"] = None

    data["practice_financing_terms"] = (
        practice.financing_terms if practice and practice.financing_terms else []
    )
    data["practice_interest_rate"] = (
        float(practice.interest_rate) if practice and practice.interest_rate else None
    )

    data.update(practice_branding_payload(practice, request))

    if practice and hasattr(practice, "show_dentist_bio") and not practice.show_dentist_bio:
        if "practitioner" in data:
            data["practitioner"] = None
        if "practitioner_profile" in data:
            data["practitioner_profile"] = None

    data.pop("view_count", None)

    return data
```

If importing from `views.public_treatment_views` creates a circular import, move `_build_enhanced_category_data` and `practice_branding_payload` into `guide_payload.py` and have the view import them from there instead — one direction only, and still one definition each.

- [ ] **Step 4: Run test to verify it passes**

Run: `python manage.py test TreatmentPlan.tests.test_guide_payload --keepdb`
Expected: PASS, 5 tests.

- [ ] **Step 5: Rewrite the public view to use the builder**

Replace the body of `retrieve` from `# Get the standard serialized data` through `data.pop("view_count")` with:

```python
            # Engagement tracking stays HERE, not in the builder: the staff
            # preview shares the payload but must not record a patient view.
            treatment_plan.view_count += 1
            treatment_plan.save(update_fields=["view_count"])

            data = build_guide_payload(treatment_plan, request)

            return Response(data)
```

Add `from ..guide_payload import build_guide_payload` at the top.

- [ ] **Step 6: Prove the public payload did not change**

Run the existing public-view tests:
```bash
python manage.py test TreatmentPlan.tests --keepdb -k public
```
Expected: PASS. If no public-view test exists, write one first that snapshots the payload keys before the refactor and re-run it after — a silent payload change here breaks live patient guides.

---

## Task 2: Add the draft status

**Files:**
- Modify: `TreatmentPath/TreatmentPlan/models.py:3787-3798`
- Create: `TreatmentPath/TreatmentPlan/migrations/XXXX_treatmentplan_draft_status.py` (generated)
- Create: `TreatmentPath/TreatmentPlan/tests/test_draft_plan_status.py`

**Interfaces:**
- Consumes: nothing.
- Produces: `TreatmentPlan.status` accepts `"draft"`. Tasks 3–6 depend on it.

- [ ] **Step 1: Write the failing test**

```python
# TreatmentPath/TreatmentPlan/tests/test_draft_plan_status.py
from django.test import TestCase

from TreatmentPlan.models import TreatmentPlan


class DraftStatusTests(GuidePlanTestCase):
    def test_draft_is_an_accepted_status(self):
        statuses = dict(TreatmentPlan._meta.get_field("status").choices)
        self.assertIn("draft", statuses)
        self.assertEqual(statuses["draft"], "Draft")

    def test_a_draft_plan_saves_and_reads_back(self):
        plan = self.make_plan(status="draft")
        plan.refresh_from_db()
        self.assertEqual(plan.status, "draft")

    def test_default_status_is_still_pending(self):
        plan = TreatmentPlan.objects.create(practice=self.practice, patient=self.patient)
        self.assertEqual(
            plan.status,
            "pending",
            "adding draft must not change what an unspecified plan becomes",
        )

    def test_drafts_are_excluded_from_open_plan_queries(self):
        self.make_plan(status="draft")
        self.make_plan(status="pending")

        self.assertEqual(
            TreatmentPlan.objects.filter(practice=self.practice, status="pending").count(),
            1,
            "an explicit pending filter must not pick up drafts",
        )
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python manage.py test TreatmentPlan.tests.test_draft_plan_status --keepdb`
Expected: FAIL — `"draft"` is not in choices.

- [ ] **Step 3: Write minimal implementation**

In `models.py`, add `draft` as the **first** choice (it precedes `pending` in the lifecycle):

```python
    status = models.CharField(
        max_length=20,
        choices=[
            # An unfinished guide the patient has never seen. Excluded from every
            # plan listing except Docs and the patient's own record. Cleaned up
            # after 30 days.
            ("draft", "Draft"),
            ("pending", "Pending"),
            ("active", "Active"),
            ("processing", "Processing"),
            ("processed", "Processed"),
            ("completed", "Completed"),
            ("cancelled", "Cancelled"),
        ],
        default="pending",
    )
```

Leave `default="pending"` alone. A plan created without an explicit status is still an open plan; only the guide-create flow asks for a draft.

- [ ] **Step 4: Generate the migration**

Run: `python manage.py makemigrations TreatmentPlan`
Expected: one `AlterField` migration on `treatmentplan.status`.

**Do not run `migrate`.** Record the migration filename for the user's prod to-run list. `choices` is Python-only — this adds no DB constraint, so the Go service needs no change to write or read these rows.

- [ ] **Step 5: Run test to verify it passes**

Run: `python manage.py test TreatmentPlan.tests.test_draft_plan_status --keepdb`
Expected: PASS, 4 tests.

---

## Task 3: The leakage sweep

Queries that filter `status="pending"` exclude drafts for free. The danger is queries with **no** status filter, which will now start returning half-finished guides. This task finds and fixes them.

**Files:**
- Modify: whichever files the enumeration below identifies.
- Test: one regression test per fixed query, added to the suite that already covers that view.

**Interfaces:**
- Consumes: the `draft` status (Task 2).
- Produces: nothing for later tasks.

- [ ] **Step 1: Enumerate the candidates**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend
grep -rn "TreatmentPlan.objects" TreatmentPath --include=*.py \
  | grep -v "/tests/" | grep -v migrations | grep -v "status"
```
This returns ~88 lines. Write them to a scratch file and triage each.

- [ ] **Step 2: Triage each call site**

Classify by what the query does:

- **`.create(...)`** — safe, ignore. It writes one row.
- **`.get(pk=…)` / `.filter(pk=…)` / `.filter(id__in=…)`** — safe, ignore. Addressing a known plan by id is legitimate for drafts too.
- **Anything that lists, counts, aggregates, or feeds a serializer with many rows** — **must exclude drafts.** These are the leaks: dashboards, reporting, patient workspace, journey boards, outstanding summaries.

Pay particular attention to:
- `TreatmentPath/TreatmentPlan/views/patient_views.py:1015, 1901, 2249`
- `TreatmentPath/TreatmentPlan/views/intake_views.py:1618, 1872`
- `TreatmentPath/TreatmentPlan/outstanding_summary.py:133`
- `TreatmentPath/Tasks/serializers.py:90`

- [ ] **Step 3: Check the Go side**

Run:
```bash
grep -rn "treatment_plan\|TreatmentPlan" /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo --include=*.go \
  | grep -i "select\|find\|where" | head -40
```
Go shares the database. Any Go query listing plans without a status filter leaks drafts into Go-driven features. Report what you find — do not change Go code as part of this task without flagging it.

- [ ] **Step 4: Fix each leaking query**

For each, add the exclusion. Prefer `.exclude(status="draft")` over enumerating statuses, so future statuses don't need revisiting:

```python
TreatmentPlan.objects.filter(practice=practice).exclude(status="draft")
```

- [ ] **Step 5: Write a regression test per fixed query**

For each fixed site, add a test to the suite already covering that view. Template — adapt the endpoint and assertion per site:

```python
    def test_drafts_do_not_appear_in_the_listing(self):
        draft = self.make_plan(status="draft")
        visible = self.make_plan(status="pending")

        response = self.client.get(self.url)

        self.assertEqual(response.status_code, 200)
        returned = {row["id"] for row in response.data["results"]}
        self.assertNotIn(
            draft.id,
            returned,
            "a draft guide must never surface in a plan listing",
        )
        self.assertIn(
            visible.id,
            returned,
            "guard against the exclusion being so broad it empties the list",
        )
```

- [ ] **Step 6: Run the full backend suite**

Run: `python manage.py test TreatmentPlan --keepdb`
Expected: PASS, or only failures that pre-date this work. Record the pre-existing failure list before you start — the Recall suite in particular has ~22 known date-sensitive failures unrelated to this.

- [ ] **Step 7: Report the sweep**

Write the triage table (call site → classification → action taken) into the task hand-off. This is the plan's highest-risk task; the user needs to see what was examined, not just what changed.

---

## Task 4: The staff preview endpoint

**Files:**
- Modify: `TreatmentPath/TreatmentPlan/views/treatment_plan_views.py` (new action on `TreatmentPlanViewSet`)
- Modify: `TreatmentPath/TreatmentPlan/urls.py`
- Create: `TreatmentPath/TreatmentPlan/tests/test_guide_preview_endpoint.py`

**Interfaces:**
- Consumes: `build_guide_payload` (Task 1).
- Produces: `GET /treatment-plan/treatment-plans/<pk>/guide-preview/`, route name `treatment-plan-guide-preview`, returning the guide payload. Task 8 fetches it.

- [ ] **Step 1: Write the failing test**

```python
# TreatmentPath/TreatmentPlan/tests/test_guide_preview_endpoint.py
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase

from TreatmentPlan.models import TreatmentPlan


class GuidePreviewEndpointTests(GuidePlanTestCase):
    def url_for(self, plan):
        return reverse("treatment-plan-guide-preview", kwargs={"pk": plan.pk})

    def test_returns_the_guide_payload_for_a_draft(self):
        draft = self.make_plan(status="draft")

        response = self.client.get(self.url_for(draft))

        self.assertEqual(response.status_code, status.HTTP_200_OK)
        self.assertIn("procedures", response.data)
        self.assertIn("practice_financing_terms", response.data)

    def test_requires_authentication(self):
        draft = self.make_plan(status="draft")
        # setUp authenticates the shared client; strip it for this case only.
        self.client.credentials()

        response = self.client.get(self.url_for(draft))

        self.assertIn(
            response.status_code,
            (status.HTTP_401_UNAUTHORIZED, status.HTTP_403_FORBIDDEN),
        )

    def test_refuses_a_plan_from_another_practice(self):
        foreign = self.make_plan(
            status="draft", practice=self.other_practice, patient=self.other_patient
        )

        response = self.client.get(self.url_for(foreign))

        self.assertEqual(response.status_code, status.HTTP_404_NOT_FOUND)

    def test_does_not_record_a_view(self):
        draft = self.make_plan(status="draft")
        before = draft.view_count

        self.client.get(self.url_for(draft))

        draft.refresh_from_db()
        self.assertEqual(
            draft.view_count, before, "preview must not track engagement (FR3)"
        )

    def test_does_not_mint_a_verification_code(self):
        from TreatmentPlan.models import SixDigitVerificationCode

        draft = self.make_plan(status="draft")

        self.client.get(self.url_for(draft))

        self.assertFalse(
            SixDigitVerificationCode.objects.filter(treatment_plan=draft).exists(),
            "preview must not create a public access credential (FR3)",
        )

    def test_works_for_an_issued_plan_too(self):
        issued = self.make_plan(status="pending")

        response = self.client.get(self.url_for(issued))

        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python manage.py test TreatmentPlan.tests.test_guide_preview_endpoint --keepdb`
Expected: FAIL — `Reverse for 'treatment-plan-guide-preview' not found`.

- [ ] **Step 3: Write minimal implementation**

On `TreatmentPlanViewSet` in `treatment_plan_views.py`:

```python
    @action(detail=True, methods=["get"], url_path="guide-preview")
    def guide_preview(self, request, pk=None):
        """The patient guide as staff, without issuing anything (FR1, FR3).

        Deliberately NOT the public route: previewing through that would require
        minting a verification code, which is a public credential for an
        unfinished guide. Authorisation here is the session plus the viewset's
        practice scoping, so `get_object()` already 404s a foreign plan.

        Records nothing: no view count, no code, no QR, no touchpoint.
        """
        treatment_plan = self.get_object()
        return Response(build_guide_payload(treatment_plan, request))
```

Add `from ..guide_payload import build_guide_payload` at the top.

In `urls.py`, beside the other `treatment-plans/<int:pk>/…` routes:

```python
    path(
        "treatment-plans/<int:pk>/guide-preview/",
        TreatmentPlanViewSet.as_view({"get": "guide_preview"}),
        name="treatment-plan-guide-preview",
    ),
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python manage.py test TreatmentPlan.tests.test_guide_preview_endpoint --keepdb`
Expected: PASS, 6 tests.

- [ ] **Step 5: Confirm the practice scoping is real**

Verify `TreatmentPlanViewSet.get_queryset` filters by practice. If it does not, the cross-practice test will have failed at Step 4 — fix the queryset, do not weaken the test. The `test_refuses_a_plan_from_another_practice` case is the security boundary for this endpoint.

---

## Task 5: The non-draft edit gate

FR12/FR13 remove the edit buttons. A hidden button is not access control, so the server must refuse too.

**Files:**
- Modify: `TreatmentPath/TreatmentPlan/views/treatment_plan_views.py` (the viewset's `update` / `partial_update`)
- Test: `TreatmentPath/TreatmentPlan/tests/test_draft_edit_gate.py`

**Interfaces:**
- Consumes: the `draft` status (Task 2).
- Produces: content edits to a non-draft plan return 403.

- [ ] **Step 1: Write the failing test**

```python
# TreatmentPath/TreatmentPlan/tests/test_draft_edit_gate.py
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase

from TreatmentPlan.models import TreatmentPlan

# Fields that define what the patient is shown. Operational fields (priority,
# attending, notes) stay editable at every status — FR14 preserves those.
CONTENT_FIELDS = {"categories_data", "final_price_set", "welcome_message"}


class DraftEditGateTests(GuidePlanTestCase):
    def url_for(self, plan):
        return reverse("treatment-plan-detail", kwargs={"pk": plan.pk})

    def test_a_draft_accepts_content_edits(self):
        draft = self.make_plan(status="draft")

        response = self.client.patch(
            self.url_for(draft), {"welcome_message": "Hello"}, format="json"
        )

        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_an_issued_plan_refuses_content_edits(self):
        issued = self.make_plan(status="pending")

        response = self.client.patch(
            self.url_for(issued), {"welcome_message": "Changed"}, format="json"
        )

        self.assertEqual(response.status_code, status.HTTP_403_FORBIDDEN)
        issued.refresh_from_db()
        self.assertNotEqual(issued.welcome_message, "Changed")

    def test_an_issued_plan_still_accepts_operational_edits(self):
        issued = self.make_plan(status="pending")

        response = self.client.patch(
            self.url_for(issued), {"priority": "high"}, format="json"
        )

        self.assertEqual(
            response.status_code,
            status.HTTP_200_OK,
            "FR14 keeps assigning and prioritising working at every status",
        )

    def test_the_status_transition_itself_is_allowed(self):
        draft = self.make_plan(status="draft")

        response = self.client.patch(
            self.url_for(draft), {"status": "pending"}, format="json"
        )

        self.assertEqual(response.status_code, status.HTTP_200_OK)
        draft.refresh_from_db()
        self.assertEqual(draft.status, "pending")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python manage.py test TreatmentPlan.tests.test_draft_edit_gate --keepdb`
Expected: FAIL — the issued plan is edited successfully and returns 200.

- [ ] **Step 3: Write minimal implementation**

On `TreatmentPlanViewSet`:

```python
    # What the patient is shown. Editable only while the plan is a draft
    # (FR12, FR13). Operational fields are deliberately absent — FR14 keeps
    # those editable at every status.
    GUIDE_CONTENT_FIELDS = frozenset(
        {
            "categories_data",
            "final_price_set",
            "discount_percentage",
            "discount_amount",
            "welcome_message",
            "additional_treatments",
            "discount_valid_until",
            "assets_data",
            "review_ids",
        }
    )

    def _reject_content_edit_to_issued_plan(self, request, instance):
        """403 when a caller edits guide content on a plan that has left draft.

        Hiding the buttons is not access control; the create flow's update path
        is still reachable by anyone with the endpoint.
        """
        if instance.status == "draft":
            return None
        touched = self.GUIDE_CONTENT_FIELDS & set(request.data.keys())
        if not touched:
            return None
        return Response(
            {
                "error": "This treatment plan has already been issued and can no "
                "longer be edited. Create a new patient guide instead.",
                "fields": sorted(touched),
            },
            status=status.HTTP_403_FORBIDDEN,
        )

    def update(self, request, *args, **kwargs):
        denied = self._reject_content_edit_to_issued_plan(request, self.get_object())
        if denied is not None:
            return denied
        return super().update(request, *args, **kwargs)

    def partial_update(self, request, *args, **kwargs):
        denied = self._reject_content_edit_to_issued_plan(request, self.get_object())
        if denied is not None:
            return denied
        return super().partial_update(request, *args, **kwargs)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python manage.py test TreatmentPlan.tests.test_draft_edit_gate --keepdb`
Expected: PASS, 4 tests.

- [ ] **Step 5: Confirm nothing legitimate broke**

Run: `python manage.py test TreatmentPlan --keepdb`
Expected: PASS aside from pre-existing failures. If a conversion or journey flow now 403s, it is PATCHing a content field on an issued plan — decide per case whether to exempt it (add to the hand-off) rather than widening the gate silently.

---

## Task 6: Draft cleanup after 30 days

**Files:**
- Modify: `TreatmentPath/TreatmentPlan/tasks.py`
- Create: `TreatmentPath/TreatmentPlan/migrations/XXXX_register_draft_cleanup_schedule.py`
- Create: `TreatmentPath/TreatmentPlan/tests/test_draft_cleanup.py`

**Interfaces:**
- Consumes: the `draft` status (Task 2).
- Produces: task `TreatmentPlan.tasks.delete_expired_draft_plans`, periodic task named `treatment-plan-delete-expired-drafts`.

- [ ] **Step 1: Write the failing test**

```python
# TreatmentPath/TreatmentPlan/tests/test_draft_cleanup.py
from datetime import timedelta

from django.test import TestCase
from django.utils import timezone

from TreatmentPlan.models import TreatmentPlan
from TreatmentPlan.tasks import delete_expired_draft_plans

DRAFT_TTL_DAYS = 30


class DeleteExpiredDraftPlansTests(GuidePlanTestCase):
    def make_plan(self, status_value, age_days):
        plan = TreatmentPlan.objects.create(
            practice=self.practice, patient=self.patient, status=status_value
        )
        # created_at is auto_now_add, so rewrite it directly.
        TreatmentPlan.objects.filter(pk=plan.pk).update(
            created_at=timezone.now() - timedelta(days=age_days)
        )
        return plan

    def test_deletes_a_draft_past_the_threshold(self):
        stale = self.make_plan("draft", DRAFT_TTL_DAYS + 1)
        delete_expired_draft_plans()
        self.assertFalse(TreatmentPlan.objects.filter(pk=stale.pk).exists())

    def test_keeps_a_draft_inside_the_threshold(self):
        fresh = self.make_plan("draft", DRAFT_TTL_DAYS - 1)
        delete_expired_draft_plans()
        self.assertTrue(TreatmentPlan.objects.filter(pk=fresh.pk).exists())

    def test_never_touches_an_issued_plan_however_old(self):
        ancient = self.make_plan("pending", DRAFT_TTL_DAYS * 10)
        delete_expired_draft_plans()
        self.assertTrue(
            TreatmentPlan.objects.filter(pk=ancient.pk).exists(),
            "only drafts expire — an old open plan is still real work",
        )

    def test_reports_how_many_it_removed(self):
        self.make_plan("draft", DRAFT_TTL_DAYS + 1)
        self.make_plan("draft", DRAFT_TTL_DAYS + 5)
        self.assertEqual(delete_expired_draft_plans(), 2)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python manage.py test TreatmentPlan.tests.test_draft_cleanup --keepdb`
Expected: FAIL — cannot import `delete_expired_draft_plans`.

- [ ] **Step 3: Write minimal implementation**

Append to `tasks.py`:

```python
DRAFT_TTL_DAYS = 30


@shared_task
def delete_expired_draft_plans():
    """Remove unfinished guides nobody came back to (30 days).

    Drafts only. An old open plan is real work someone is still chasing; an old
    draft is an abandoned edit. Returns the count so the beat log says what it did.
    """
    from django.utils import timezone
    from datetime import timedelta

    from .models import TreatmentPlan

    cutoff = timezone.now() - timedelta(days=DRAFT_TTL_DAYS)
    deleted, _ = TreatmentPlan.objects.filter(
        status="draft", created_at__lt=cutoff
    ).delete()
    return deleted
```

`.delete()` returns `(total_rows, per_model_dict)`, and cascades will inflate `total_rows` beyond the plan count. If the count assertion fails, count first and delete second:

```python
    queryset = TreatmentPlan.objects.filter(status="draft", created_at__lt=cutoff)
    count = queryset.count()
    queryset.delete()
    return count
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python manage.py test TreatmentPlan.tests.test_draft_cleanup --keepdb`
Expected: PASS, 4 tests.

- [ ] **Step 5: Register the schedule via migration**

This project registers beat schedules as migrations, not settings — follow `marketingBroadcast/migrations/0029_register_requeue_beat_schedule.py` exactly:

```python
from django.db import migrations

CLEANUP_TASK = "treatment-plan-delete-expired-drafts"


def create_schedule(apps, schema_editor):
    CrontabSchedule = apps.get_model("django_celery_beat", "CrontabSchedule")
    PeriodicTask = apps.get_model("django_celery_beat", "PeriodicTask")

    # Once daily, off-peak. Drafts expire on a 30-day boundary, so the exact
    # hour is immaterial — only that it is not competing with the 2am sweep.
    daily, _ = CrontabSchedule.objects.get_or_create(
        minute="30",
        hour="3",
        day_of_week="*",
        day_of_month="*",
        month_of_year="*",
        timezone="UTC",
    )
    PeriodicTask.objects.update_or_create(
        name=CLEANUP_TASK,
        defaults={
            "task": "TreatmentPlan.tasks.delete_expired_draft_plans",
            "crontab": daily,
            "enabled": False,
            "description": "Delete unfinished patient-guide drafts older than 30 days.",
        },
    )


def remove_schedule(apps, schema_editor):
    PeriodicTask = apps.get_model("django_celery_beat", "PeriodicTask")
    PeriodicTask.objects.filter(name=CLEANUP_TASK).delete()


class Migration(migrations.Migration):
    dependencies = [
        ("TreatmentPlan", "XXXX_treatmentplan_draft_status"),
        ("django_celery_beat", "0019_alter_periodictasks_options"),
    ]

    operations = [migrations.RunPython(create_schedule, remove_schedule)]
```

Replace `XXXX_treatmentplan_draft_status` with Task 2's real migration name. Note `enabled: False` — matching the project's convention that a new schedule is switched on deliberately, not by deploying.

- [ ] **Step 6: Flag both migrations for the user**

Two migrations now need running in DEV and PROD, plus enabling the periodic task. Add them to the prod to-run list and say so in the hand-off. Do not run `migrate` yourself.

---

## Task 7: Services.tsx accepts an authenticated source

**Files:**
- Modify: `src/pages/treatment-verification/Services.tsx:230-290`
- Test: `src/pages/treatment-verification/Services.preview.test.tsx`

**Interfaces:**
- Consumes: `GET /treatment-plan/treatment-plans/<pk>/guide-preview/` (Task 4).
- Produces: `Services` accepts an optional `previewPlanId?: string` prop. When set, it fetches the preview endpoint with auth instead of resolving a code from the URL. Task 8 passes it.

- [ ] **Step 1: Add the endpoint to the API config**

In `src/config/api.ts`, inside `patientTreatmentPlanView`:

```ts
    // Staff-authenticated preview of the guide. Same payload as the public
    // view, no verification code, no engagement tracking (FR1, FR3).
    guidePreview: (treatmentPlanId: string) =>
      getApiUrl(`/treatment-plan/treatment-plans/${treatmentPlanId}/guide-preview/`),
```

- [ ] **Step 2: Write the failing test**

```tsx
// src/pages/treatment-verification/Services.preview.test.tsx
import { render, screen, waitFor } from "@testing-library/react";
import { MemoryRouter } from "react-router-dom";
import { beforeEach, describe, expect, it, vi } from "vitest";

const { mockFetchWithAuth } = vi.hoisted(() => ({ mockFetchWithAuth: vi.fn() }));

vi.mock("@/lib/helpers", () => ({ useFetchWithAuth: () => mockFetchWithAuth }));

import Services from "@/pages/treatment-verification/Services";

const payload = {
  id: 42,
  welcome_message: "Welcome to your plan",
  procedures: [
    { id: 1, category: "Restorative", procedures: [{ id: 7, name: "Crown", price: 500, description: "A crown" }] },
  ],
  assets: [],
  reviews: [],
  practice_financing_terms: [],
  practice_interest_rate: null,
  practice_service_background_image: null,
};

describe("Services in preview mode", () => {
  beforeEach(() => {
    mockFetchWithAuth.mockReset();
    global.fetch = vi.fn();
  });

  it("loads from the authenticated preview endpoint", async () => {
    mockFetchWithAuth.mockResolvedValue({ ok: true, json: async () => payload });

    render(
      <MemoryRouter>
        <Services previewPlanId="42" />
      </MemoryRouter>,
    );

    await waitFor(() => expect(mockFetchWithAuth).toHaveBeenCalled());
    expect(String(mockFetchWithAuth.mock.calls[0][0])).toContain("guide-preview");
  });

  it("never calls the public code-based endpoint in preview mode", async () => {
    mockFetchWithAuth.mockResolvedValue({ ok: true, json: async () => payload });

    render(
      <MemoryRouter>
        <Services previewPlanId="42" />
      </MemoryRouter>,
    );

    await waitFor(() => expect(mockFetchWithAuth).toHaveBeenCalled());
    expect(global.fetch).not.toHaveBeenCalled();
  });

  it("renders the guide content from the preview payload", async () => {
    mockFetchWithAuth.mockResolvedValue({ ok: true, json: async () => payload });

    render(
      <MemoryRouter>
        <Services previewPlanId="42" />
      </MemoryRouter>,
    );

    await waitFor(() => expect(screen.getByText(/Crown/i)).toBeInTheDocument());
  });

  it("shows an error when the preview fails to load", async () => {
    mockFetchWithAuth.mockResolvedValue({ ok: false, status: 404, text: async () => "gone" });

    render(
      <MemoryRouter>
        <Services previewPlanId="42" />
      </MemoryRouter>,
    );

    await waitFor(() => expect(screen.getByText(/failed to load|couldn't load/i)).toBeInTheDocument());
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `npx vitest run src/pages/treatment-verification/Services.preview.test.tsx`
Expected: FAIL — `Services` takes no `previewPlanId`.

- [ ] **Step 4: Write minimal implementation**

Give `Services` the prop, and branch **only** where the request is made — the rendering path below must stay untouched, because sharing it is the whole point:

```tsx
interface ServicesProps {
  /** When set, load this plan through the authenticated staff preview endpoint
   *  instead of resolving a verification code from the URL (FR1). */
  previewPlanId?: string;
}
```

Inside `fetchTreatmentPlan`, before the existing endpoint selection:

```tsx
      if (previewPlanId) {
        const response = await fetchWithAuth(
          API_ENDPOINTS.patientTreatmentPlanView.guidePreview(previewPlanId),
        );
        if (!response.ok) {
          setError("Couldn't load the preview.");
          setLoading(false);
          return;
        }
        const data = await response.json();
        setTreatmentPlan(data);
        if (data.procedures?.length) setServices(convertToServices(data.procedures));
        if (data.reviews?.length) setReviews(convertToPatientReviews(data.reviews));
        setError(null);
        setLoading(false);
        return;
      }
```

Also update the effect's guard so preview mode does not require `verificationCode` or `qrToken`, and add `previewPlanId` to its dependency array.

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run src/pages/treatment-verification/Services.preview.test.tsx`
Expected: PASS, 4 tests.

- [ ] **Step 6: Confirm the patient path is unharmed**

Run: `npx vitest run src/pages/treatment-verification src/components/treatment-verification`
Expected: PASS. The public code-based path must behave exactly as before — this task adds a branch, it does not change the existing one.

---

## Task 8: Preview from the create flow

**Files:**
- Create: `src/pages/create/GuidePreview.tsx`
- Create: `src/pages/create/GuidePreview.test.tsx`
- Modify: `src/hooks/useTreatmentPlanCreation.ts`
- Modify: `src/pages/Conpact3.tsx`

**Interfaces:**
- Consumes: `Services` with `previewPlanId` (Task 7); the `draft` status (Task 2).
- Produces: `<GuidePreview planId onBackToEdit onContinue patientName />`.

- [ ] **Step 1: Add draft saving to the creation hook**

In `useTreatmentPlanCreation.ts`, add a `saveDraft` callback beside `handleConfirmTreatmentPlanCreation`. The payload assembly is identical, so **extract the existing payload build into one function** used by both rather than copying it:

```ts
  /** Build the create/update payload. Used by BOTH draft save and confirmed
   *  creation — they differ only in the status they send. */
  const buildPayload = useCallback((): Record<string, unknown> => {
    // Move the existing body of handleConfirmTreatmentPlanCreation's payload
    // assembly here verbatim (categories_data through review_ids) and return it.
    // Do not duplicate it.
  }, [/* the same dependency list the confirm callback already uses */]);
```

Then:

```ts
  /** Persist the in-progress guide as a draft so it can be previewed and
   *  resumed. Returns the plan id. */
  const saveDraft = useCallback(async (): Promise<string | null> => {
    const payload = { ...buildPayload(), status: "draft" };
    if (draftPlanId) {
      await updateTreatmentPlan(draftPlanId, payload);
      return draftPlanId;
    }
    const created = await createTreatmentPlan(payload);
    const id = String(
      Number(created?.id ?? created?.plan_id ?? created?.treatment_plan_id ?? 0),
    ) || extractTreatmentPlanId(created);
    setDraftPlanId(id);
    return id;
  }, [buildPayload, createTreatmentPlan, draftPlanId, updateTreatmentPlan]);
```

In `handleConfirmTreatmentPlanCreation`, when a `draftPlanId` exists, send `{ ...buildPayload(), status: "pending" }` to `updateTreatmentPlan(draftPlanId, …)` instead of creating a second plan. That is the draft → pending transition.

- [ ] **Step 2: Write the failing test for the preview shell**

```tsx
// src/pages/create/GuidePreview.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MemoryRouter } from "react-router-dom";
import { describe, expect, it, vi } from "vitest";

vi.mock("@/pages/treatment-verification/Services", () => ({
  default: ({ previewPlanId }: { previewPlanId?: string }) => (
    <div data-testid="guide">preview of {previewPlanId}</div>
  ),
}));

import GuidePreview from "@/pages/create/GuidePreview";

describe("GuidePreview", () => {
  it("renders the guide for the draft", () => {
    render(
      <MemoryRouter>
        <GuidePreview planId="42" patientName="Ada" onBackToEdit={vi.fn()} onContinue={vi.fn()} />
      </MemoryRouter>,
    );

    expect(screen.getByTestId("guide")).toHaveTextContent("preview of 42");
  });

  it("offers Back to edit and Continue", async () => {
    const onBackToEdit = vi.fn();
    const onContinue = vi.fn();

    render(
      <MemoryRouter>
        <GuidePreview planId="42" patientName="Ada" onBackToEdit={onBackToEdit} onContinue={onContinue} />
      </MemoryRouter>,
    );

    await userEvent.click(screen.getByRole("button", { name: /back to edit/i }));
    expect(onBackToEdit).toHaveBeenCalled();

    await userEvent.click(screen.getByRole("button", { name: /continue/i }));
    expect(onContinue).toHaveBeenCalled();
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `npx vitest run src/pages/create/GuidePreview.test.tsx`
Expected: FAIL — module not found.

- [ ] **Step 4: Write minimal implementation**

```tsx
// src/pages/create/GuidePreview.tsx
import React from "react";
import { ArrowLeft } from "lucide-react";

import { Button } from "@/components/ui/button";
import Services from "@/pages/treatment-verification/Services";

interface GuidePreviewProps {
  planId: string;
  patientName: string;
  onBackToEdit: () => void;
  onContinue: () => void;
}

/**
 * The guide as the patient will see it, wrapped in staff chrome (FR1, FR2).
 *
 * The guide itself is the very same `Services` component the patient loads —
 * nothing here re-implements it, so the preview cannot drift from reality.
 */
const GuidePreview: React.FC<GuidePreviewProps> = ({
  planId,
  patientName,
  onBackToEdit,
  onContinue,
}) => (
  <div className="fixed inset-0 z-50 flex flex-col bg-white">
    <div className="flex items-center justify-between border-b px-4 py-3">
      <Button variant="ghost" onClick={onBackToEdit}>
        <ArrowLeft className="mr-2 h-4 w-4" />
        Back to edit
      </Button>
      <p className="text-sm text-gray-600">
        Preview of {patientName}'s guide — nothing has been sent
      </p>
      <Button onClick={onContinue}>Continue to create</Button>
    </div>
    <div className="flex-1 overflow-y-auto">
      <Services previewPlanId={planId} />
    </div>
  </div>
);

export default GuidePreview;
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run src/pages/create/GuidePreview.test.tsx`
Expected: PASS, 2 tests.

- [ ] **Step 6: Wire the Preview action into Conpact3**

In `Conpact3.tsx`, add `previewPlanId` state and a `Preview patient guide` button that:
1. runs **the same validation the create action already uses** — do not restate the rules (FR4);
2. calls `saveDraft()`;
3. stores the returned id and renders `<GuidePreview>`.

`onBackToEdit` clears `previewPlanId`; `onContinue` clears it and opens the existing confirmation modal (FR2, FR5).

- [ ] **Step 7: Verify the side-effect ban by hand (FR3)**

With the network tab open, click `Preview patient guide` and confirm:
- exactly one write, the draft save;
- **no** call to `verification-code`, `generate-qr`, `patient-guide-documents`, or any send endpoint.

If a document upload fires, the staged-attachment flow is being triggered early — fix that before continuing. This is the requirement most easily broken by accident.

---

## Task 9: Drafts in Docs and on the patient record

**Files:**
- Modify: `src/pages/DocsPage.tsx`
- Modify: the patient record's plans list (find it: `grep -rn "treatment_plan" src/components/patients/workspace --include=*.tsx | head`)
- Test: `src/pages/DocsPage.drafts.test.tsx`

**Interfaces:**
- Consumes: plans with `status: "draft"`.
- Produces: nothing for later tasks.

- [ ] **Step 1: Confirm how drafts are fetched**

The list endpoints exclude drafts after Task 3. Drafts therefore need an explicit opt-in — confirm whether the plans endpoint accepts a `status` query parameter:

Run: `grep -n "status" TreatmentPath/TreatmentPlan/views/treatment_plan_views.py | grep -i "query_param\|filterset" | head`

If it does not, add `?status=draft` support to the viewset's `get_queryset` in this task, with a test, before touching the frontend.

- [ ] **Step 2: Write the failing test**

```tsx
// src/pages/DocsPage.drafts.test.tsx
import { render, screen, waitFor } from "@testing-library/react";
import { MemoryRouter } from "react-router-dom";
import { beforeEach, describe, expect, it, vi } from "vitest";

const { mockFetchWithAuth } = vi.hoisted(() => ({ mockFetchWithAuth: vi.fn() }));
vi.mock("@/lib/helpers", () => ({ useFetchWithAuth: () => mockFetchWithAuth }));

import DocsPage from "@/pages/DocsPage";

describe("DocsPage drafts", () => {
  beforeEach(() => mockFetchWithAuth.mockReset());

  it("lists draft guides marked as drafts", async () => {
    mockFetchWithAuth.mockResolvedValue({
      ok: true,
      json: async () => ({
        results: [{ id: 42, status: "draft", patient_name: "Ada Lovelace" }],
      }),
    });

    render(<MemoryRouter><DocsPage /></MemoryRouter>);

    await waitFor(() => expect(screen.getByText(/Ada Lovelace/)).toBeInTheDocument());
    expect(screen.getByText(/draft/i)).toBeInTheDocument();
  });

  it("links a draft back into the create flow to finish it", async () => {
    mockFetchWithAuth.mockResolvedValue({
      ok: true,
      json: async () => ({
        results: [{ id: 42, status: "draft", patient_name: "Ada Lovelace" }],
      }),
    });

    render(<MemoryRouter><DocsPage /></MemoryRouter>);

    await waitFor(() => expect(screen.getByRole("link", { name: /continue/i })).toBeInTheDocument());
    expect(screen.getByRole("link", { name: /continue/i })).toHaveAttribute(
      "href",
      expect.stringContaining("planId=42"),
    );
  });
});
```

- [ ] **Step 3: Run test to verify it fails, then implement**

Run: `npx vitest run src/pages/DocsPage.drafts.test.tsx` — expect FAIL, then add the draft section to `DocsPage` and re-run to PASS.

- [ ] **Step 4: Mirror it on the patient record**

Add the same draft section to the patient's plans list, marked as drafts, with the same resume link.

---

## Task 10: Remove the edit choosers (FR12, FR13, FR15)

**Files:**
- Modify: `src/components/compact/OpenTable.tsx:1880-1921` (the `Edit Open Plan` dialog), `:580-601` (`openQuickEditModal`, `openEditModal`, `openComprehensiveEdit`), `:603-634` (`handleEditSubmit`)
- Modify: `src/components/compact/ActiveTable.tsx:1687-1725`, `:641-660`
- Modify: `src/hooks/useTreatmentPlanCreation.ts` (copy)
- Test: `src/components/compact/OpenTable.createOnly.test.tsx`

**Interfaces:**
- Consumes: the draft carve-out from Task 8.
- Produces: nothing for later tasks.

- [ ] **Step 1: Write the failing test**

```tsx
// src/components/compact/OpenTable.createOnly.test.tsx
import { describe, expect, it } from "vitest";
import { readFileSync } from "node:fs";

/**
 * These are source-level assertions on purpose: the requirement is that the
 * entry points cease to EXIST, which a rendering test cannot prove — it can
 * only show they were not reached in one particular state.
 */
describe("create-only treatment plans", () => {
  it("OpenTable no longer offers Quick Edit or Comprehensive", () => {
    const source = readFileSync("src/components/compact/OpenTable.tsx", "utf8");
    expect(source).not.toMatch(/Quick Edit/);
    expect(source).not.toMatch(/Comprehensive/);
    expect(source).not.toMatch(/editMode/);
  });

  it("ActiveTable no longer offers Quick Edit or Comprehensive", () => {
    const source = readFileSync("src/components/compact/ActiveTable.tsx", "utf8");
    expect(source).not.toMatch(/Quick Edit/);
    expect(source).not.toMatch(/Comprehensive/);
    expect(source).not.toMatch(/editMode/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/components/compact/OpenTable.createOnly.test.tsx`
Expected: FAIL — all four strings are present today.

- [ ] **Step 3: Delete the edit paths**

In both tables remove: the `Edit Open Plan` / equivalent dialog, `openQuickEditModal`, `openEditModal`, `openComprehensiveEdit`, `handleEditSubmit`, the `Edit` popover entry that opens the chooser, and the `isEditTypeModalOpen` / `pendingEditLead` / `editingLead` state. Remove the now-unused edit-modal component import if nothing else uses it.

Note this deletes the only route that edits an open plan's treatments in one place. Priority and assignee remain available through the separate operational controls FR14 preserves.

- [ ] **Step 4: Gate the create route on draft status**

In `Conpact3.tsx`, when the URL carries `planId`, fetch that plan and branch:
- `status === "draft"` → load it and resume (Task 8's flow);
- anything else → clear the params, start a fresh guide, and show: `This plan has already been created. Start a new patient guide to make changes.`

Treat any `editMode` parameter as legacy: ignore it and start fresh.

- [ ] **Step 5: Remove update-specific copy (FR15)**

In `useTreatmentPlanCreation.ts`, the `existingPlanId` branch currently toasts `Patient guide updated`. After Task 8 that branch only ever finalises a draft, so it becomes `Patient guide created`. Keep "draft" wording only inside the draft flow itself. Sweep the confirmation and success modals for remaining update copy:

Run: `grep -rn "updated\|Update" src/components/modals/treatmentPlan/`

- [ ] **Step 6: Run test to verify it passes**

Run: `npx vitest run src/components/compact/OpenTable.createOnly.test.tsx`
Expected: PASS, 2 tests.

- [ ] **Step 7: Full verification**

Run: `NODE_OPTIONS=--max-old-space-size=8192 npx tsc --noEmit -p tsconfig.app.json 2>&1 | grep -c "error TS"` — no increase over the 501-error baseline.
Run: `npx vitest run` — PASS or pre-existing failures only.
Run: `git status --porcelain` and `git rev-parse --abbrev-ref HEAD` — confirm `mannieJuly`, summarise, and hand off. **Do not commit.**

---

## Manual verification

- [ ] Preview with no patient selected: blocked, with the same message the create action gives (FR4).
- [ ] Preview a complete guide: it matches what the patient sees, documents included, images and PDFs inline, unsupported files falling back to detail/download (FR1).
- [ ] Network tab during preview: one draft write, nothing else (FR3).
- [ ] Back to edit: every selection intact (FR2).
- [ ] Hard-refresh mid-preview, then reopen the draft from Docs: the work is still there.
- [ ] Continue to create, confirm: **one** plan exists, status `pending`, not two rows (FR5).
- [ ] The finished plan no longer appears under drafts.
- [ ] Open Plans, Journeys boards, reporting and the patient workspace show no drafts (Task 3).
- [ ] No `Quick Edit` or `Comprehensive` anywhere (FR12).
- [ ] A legacy `/create?planId=<issued>&editMode=comprehensive` URL lands on a fresh guide with the create-only message (FR13).
- [ ] Moving, assigning, journey notes and tasks all still work (FR14).

---

## Risks

- **Task 3 is the one that can break production quietly.** A missed unfiltered query puts drafts into counts or reporting with no error. The triage table is the deliverable, not just the diff.
- **Two migrations plus a periodic task to enable** — none applied by deploying. They must reach the prod to-run list.
- **`Services.tsx` is 722 lines** and gains a second data source. If the data-loading concern wants extracting, that is in scope; a broader rewrite is not.
- **`view_count` must stay out of `build_guide_payload`.** If it migrates in during Task 1, every preview silently inflates the patient's engagement figures.

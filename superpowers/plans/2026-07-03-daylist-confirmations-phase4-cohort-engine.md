# Daylist Confirmations Phase 4: Cohort/Bucket Targeting Engine — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let staff control which appointments get enrolled in a confirmation sequence (via 7 named cohort "buckets" + a base-audience/sub-cohort opt-in overlay), preview the effect before saving, and stop confirmation/daylist-automation messages from cluttering the staff Inbox (moving them into a new dedicated activity panel instead, mirroring how Recall already works).

**Architecture:** A small, curated bucket-predicate registry (not a generic rule builder) drives a new `targeting` JSON config on `ConfirmationSequence`, which `enroll_eligible_appointments` applies as an extra filter before its existing eligibility checks. A new `risk_tier` column (synced by Go, mirroring existing risk-enrichment fields) makes risk-based buckets possible. A brand-new `ConfirmationSequenceViewSet` (none exists today) exposes CRUD + a preview-count action. Separately, the Inbox's message-visibility rule flips from an exclude-list to an allow-list, and a new `DaylistActivityPanel`/`DaylistReportingViewSet` (mirroring Recall's own reporting panel) gives staff a place to see the now-hidden automation messages.

**Tech Stack:** Django REST Framework (`TreatmentPathBackend`), React/TypeScript (`perfect-pixel-playground-project`), Go (`EmailServiceGo`).

---

### Task 1: `risk_tier` column + Go sync

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/models.py` (DentallyAppointment)
- Create: `TreatmentPath/dentallyIntegration/migrations/0148_dentallyappointment_risk_tier.py`
- Modify: `EmailServiceGo/internal/dentally/daylist/appointment/store.go`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing test**

Append to `dentallyIntegration/tests.py`:

```python
class DentallyAppointmentRiskTierFieldTests(TestCase):
    """risk_tier is a plain, nullable, Go-synced string column — no Django
    default computation, just storage for whatever Go writes."""

    def test_risk_tier_defaults_to_blank_and_is_queryable(self):
        practice = Practice.objects.create(name="Risk Tier Field Dental")
        appointment = DentallyAppointment.objects.create(
            practice=practice,
            dentally_id=20100,
            dentally_patient_id=2001,
            patient_name="Risk Tier Test Patient",
            state="pending",
            duration=30,
        )
        self.assertEqual(appointment.risk_tier, "")

        appointment.risk_tier = "High"
        appointment.save(update_fields=["risk_tier"])
        appointment.refresh_from_db()
        self.assertEqual(appointment.risk_tier, "High")

        self.assertEqual(
            DentallyAppointment.objects.filter(risk_tier="High").count(), 1
        )
```

- [ ] **Step 2: Run test to verify it fails**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.DentallyAppointmentRiskTierFieldTests --keepdb -v 2
```

Expected: FAIL — `AttributeError: 'DentallyAppointment' object has no attribute 'risk_tier'`.

- [ ] **Step 3: Add the field**

In `dentallyIntegration/models.py`, `DentallyAppointment`, add this field right after `patient_deposit_paid` (matching the exact `default=`/`db_default=` pairing convention used by every other Go-synced field on this model, per the project's Go↔Django parity rule — Go writes this column via raw SQL with an explicit column list, so a NOT-NULL column with no `db_default` would break that INSERT on a fresh row):

```python
    risk_tier = models.CharField(
        max_length=20,
        blank=True,
        default="",
        db_default="",
        help_text="No-show risk tier ('High'/'Elevated'/'Low') computed and "
        "synced by the Go service (EmailServiceGo/internal/dentally/daylist/"
        "noshow/risk.go) — previously computed only on-request and never "
        "persisted; this column lets the cohort/bucket engine query it "
        "directly instead of calling out to Go per appointment.",
    )
```

- [ ] **Step 4: Generate and apply the migration**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py makemigrations dentallyIntegration
```

Expected: a new migration file `dentallyIntegration/migrations/0148_dentallyappointment_risk_tier.py` (number may differ if other migrations have landed since — use whatever number Django assigns) with a single `AddField` operation for `risk_tier`, `default=""`, `db_default=""`.

Then apply it:

```bash
python manage.py migrate dentallyIntegration
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.DentallyAppointmentRiskTierFieldTests --keepdb -v 2
```

Expected: PASS.

- [ ] **Step 6: Go side — compute and write `risk_tier` during the appointment sync**

In `EmailServiceGo/internal/dentally/daylist/appointment/store.go`, the function containing the `INSERT INTO dentally_appointment (...)` upsert (around lines 469-679) already extracts `patientOutstandingBalance`, `patientNoShowCount`, `bookedAt`, `patientDOB`, etc. before building the query. Add one more computed value right before the `query := ...` string, using the package's public risk-scoring entrypoint:

```go
riskScore := noshow.ComputeRiskFull(bookedAt, patientDOB, patientOutstandingBalance, patientNoShowCount, patientLatePaymentCount, patientLatePaymentRatio, patientAvgDaysLate, patientMaxOverdueDays)
riskTier := ""
if riskScore != nil {
	riskTier = riskScore.Label
}
```

(Match `ComputeRiskFull`'s actual parameter list exactly as declared in `EmailServiceGo/internal/dentally/daylist/noshow/risk.go` — read that function's signature first and adjust the call above to match; the parameters shown here are inferred from the fields already available in this same function and may need reordering/renaming to match the real signature.)

Add `risk_tier` to the INSERT column list (right after `bucket`):

```go
			procedure_words, bucket, risk_tier,
			last_synced_at, created_at, updated_at
```

Add one more placeholder to the `VALUES` list (renumber every placeholder after it by +1 — i.e. if `bucket` was `$44`, `risk_tier` becomes `$45` and everything from `last_synced_at`'s implicit `NOW()` onward is unaffected since those are literal `NOW()` calls, not placeholders):

```go
			$32, $33,
			$34,
			$35, $36, $37, $38, $39,
			$40, $41, $42, $43, $44, $45,
```

Add to the `ON CONFLICT ... DO UPDATE SET` clause (right after `bucket`):

```go
			bucket                             = EXCLUDED.bucket,
			risk_tier                          = EXCLUDED.risk_tier,
```

Add `riskTier` to the `Exec(query, ...)` argument list (right after `apptBkt`):

```go
	apptProcWords, apptBkt, riskTier,
```

- [ ] **Step 7: Do not commit.**

---

### Task 2: Bucket registry

**Files:**
- Create: `TreatmentPath/dentallyIntegration/cohort_buckets.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class CohortBucketRegistryTests(TestCase):
    """The 7 FR5 buckets, each a pure predicate over DentallyAppointment.
    apply_bucket(qs, key) filters a queryset; apply_bucket_group(qs, keys)
    combines same-group keys with OR, cross-group with AND."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Cohort Bucket Dental")

    def _appt(self, **overrides):
        defaults = dict(
            practice=self.practice,
            dentally_id=30000 + CohortBucketRegistryTests._next_id(),
            dentally_patient_id=1,
            patient_name="Bucket Test Patient",
            state="pending",
            duration=30,
            risk_tier="Low",
            patient_outstanding_balance=0,
            booked_at=timezone.now(),
        )
        defaults.update(overrides)
        return DentallyAppointment.objects.create(**defaults)

    _counter = [0]

    @classmethod
    def _next_id(cls):
        cls._counter[0] += 1
        return cls._counter[0]

    def test_high_risk_bucket(self):
        from .cohort_buckets import apply_bucket

        high = self._appt(risk_tier="High")
        low = self._appt(risk_tier="Low")
        qs = apply_bucket(DentallyAppointment.objects.all(), "high_risk")
        ids = set(qs.values_list("id", flat=True))
        self.assertIn(high.id, ids)
        self.assertNotIn(low.id, ids)

    def test_medium_risk_bucket(self):
        from .cohort_buckets import apply_bucket

        elevated = self._appt(risk_tier="Elevated")
        low = self._appt(risk_tier="Low")
        qs = apply_bucket(DentallyAppointment.objects.all(), "medium_risk")
        ids = set(qs.values_list("id", flat=True))
        self.assertIn(elevated.id, ids)
        self.assertNotIn(low.id, ids)

    def test_booking_age_buckets(self):
        from .cohort_buckets import apply_bucket

        now = timezone.now()
        recent = self._appt(booked_at=now - timedelta(days=5))
        mid = self._appt(booked_at=now - timedelta(days=45))
        old = self._appt(booked_at=now - timedelta(days=90))

        under_30 = set(
            apply_bucket(DentallyAppointment.objects.all(), "booked_under_30d")
            .values_list("id", flat=True)
        )
        over_30 = set(
            apply_bucket(DentallyAppointment.objects.all(), "booked_over_30d")
            .values_list("id", flat=True)
        )
        over_60 = set(
            apply_bucket(DentallyAppointment.objects.all(), "booked_over_60d")
            .values_list("id", flat=True)
        )

        self.assertIn(recent.id, under_30)
        self.assertNotIn(mid.id, under_30)
        self.assertNotIn(old.id, under_30)

        self.assertIn(mid.id, over_30)
        self.assertIn(old.id, over_30)
        self.assertNotIn(recent.id, over_30)

        self.assertIn(old.id, over_60)
        self.assertNotIn(mid.id, over_60)
        self.assertNotIn(recent.id, over_60)

    def test_long_and_old_booking_compound_bucket(self):
        from .cohort_buckets import apply_bucket

        now = timezone.now()
        matches = self._appt(duration=60, booked_at=now - timedelta(days=10))
        too_short = self._appt(duration=20, booked_at=now - timedelta(days=10))
        too_recent = self._appt(duration=60, booked_at=now - timedelta(days=2))

        ids = set(
            apply_bucket(DentallyAppointment.objects.all(), "long_and_old_booking")
            .values_list("id", flat=True)
        )
        self.assertIn(matches.id, ids)
        self.assertNotIn(too_short.id, ids)
        self.assertNotIn(too_recent.id, ids)

    def test_zero_balance_bucket(self):
        from .cohort_buckets import apply_bucket

        zero = self._appt(patient_outstanding_balance=0)
        owes = self._appt(patient_outstanding_balance=45.50)

        ids = set(
            apply_bucket(DentallyAppointment.objects.all(), "zero_balance")
            .values_list("id", flat=True)
        )
        self.assertIn(zero.id, ids)
        self.assertNotIn(owes.id, ids)

    def test_unknown_bucket_key_raises(self):
        from .cohort_buckets import apply_bucket

        with self.assertRaises(KeyError):
            apply_bucket(DentallyAppointment.objects.all(), "not_a_real_bucket")

    def test_apply_bucket_group_same_group_is_or(self):
        """high_risk OR medium_risk — both in the 'risk' group."""
        from .cohort_buckets import apply_bucket_group

        high = self._appt(risk_tier="High")
        elevated = self._appt(risk_tier="Elevated")
        low = self._appt(risk_tier="Low")

        ids = set(
            apply_bucket_group(
                DentallyAppointment.objects.all(), ["high_risk", "medium_risk"]
            ).values_list("id", flat=True)
        )
        self.assertIn(high.id, ids)
        self.assertIn(elevated.id, ids)
        self.assertNotIn(low.id, ids)

    def test_apply_bucket_group_cross_group_is_and(self):
        """high_risk AND zero_balance — different groups, both must match."""
        from .cohort_buckets import apply_bucket_group

        now = timezone.now()
        both = self._appt(
            risk_tier="High", patient_outstanding_balance=0, booked_at=now
        )
        risk_only = self._appt(
            risk_tier="High", patient_outstanding_balance=99, booked_at=now
        )
        balance_only = self._appt(
            risk_tier="Low", patient_outstanding_balance=0, booked_at=now
        )

        ids = set(
            apply_bucket_group(
                DentallyAppointment.objects.all(), ["high_risk", "zero_balance"]
            ).values_list("id", flat=True)
        )
        self.assertIn(both.id, ids)
        self.assertNotIn(risk_only.id, ids)
        self.assertNotIn(balance_only.id, ids)

    def test_apply_bucket_group_empty_list_is_noop(self):
        from .cohort_buckets import apply_bucket_group

        appt = self._appt()
        ids = set(
            apply_bucket_group(DentallyAppointment.objects.all(), [])
            .values_list("id", flat=True)
        )
        self.assertIn(appt.id, ids)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.CohortBucketRegistryTests --keepdb -v 2
```

Expected: FAIL — `ModuleNotFoundError: No module named 'dentallyIntegration.cohort_buckets'`.

- [ ] **Step 3: Create the bucket registry**

Create `dentallyIntegration/cohort_buckets.py`:

```python
"""Cohort/bucket targeting engine (Daylist Confirmations Phase 4, FR4/FR5).

A curated, non-generic catalog of the exact 7 named buckets FR5 requires —
not a generic field/operator/value rule builder. Each entry is a small, pure,
independently-testable predicate over a DentallyAppointment queryset.

Buckets in the same `group` combine with OR when multiple are selected
together (FR4a's "High and Elevated risk" example); buckets from different
groups combine with AND. `booking_age` has 3 buckets that overlap by design
(e.g. an appointment booked 90 days ago matches BOTH booked_over_30d and
booked_over_60d) — staff pick whichever range(s) they actually want.
"""
from datetime import timedelta

from django.db.models import DurationField, ExpressionWrapper, F
from django.db.models.functions import Now


def _with_booking_age(qs):
    """Annotate `booking_age` = now - booked_at. Appointments with a NULL
    booked_at get a NULL annotation and will never match any booking_age
    bucket filter (excluded, not errored) — acceptable: it just means an
    appointment with no known booking date can't be targeted by booking-age
    buckets, which is the only sane behavior for missing data here."""
    return qs.annotate(
        booking_age=ExpressionWrapper(
            Now() - F("booked_at"), output_field=DurationField()
        )
    )


BUCKET_REGISTRY = {
    "high_risk": {
        "label": "High risk",
        "group": "risk",
        "apply": lambda qs: qs.filter(risk_tier="High"),
    },
    "medium_risk": {
        "label": "Medium/Elevated risk",
        "group": "risk",
        "apply": lambda qs: qs.filter(risk_tier="Elevated"),
    },
    "booked_under_30d": {
        "label": "Booked less than 30 days ago",
        "group": "booking_age",
        "apply": lambda qs: _with_booking_age(qs).filter(
            booking_age__lt=timedelta(days=30)
        ),
    },
    "booked_over_30d": {
        "label": "Booked over 30 days ago",
        "group": "booking_age",
        "apply": lambda qs: _with_booking_age(qs).filter(
            booking_age__gte=timedelta(days=30)
        ),
    },
    "booked_over_60d": {
        "label": "Booked over 60 days ago",
        "group": "booking_age",
        "apply": lambda qs: _with_booking_age(qs).filter(
            booking_age__gte=timedelta(days=60)
        ),
    },
    "long_and_old_booking": {
        "label": "Over 45 minutes and booked over 7 days ago",
        "group": "compound",
        "apply": lambda qs: _with_booking_age(qs).filter(
            duration__gt=45, booking_age__gt=timedelta(days=7)
        ),
    },
    "zero_balance": {
        "label": "GBP 0 balance",
        "group": "balance",
        "apply": lambda qs: qs.filter(patient_outstanding_balance__lte=0),
    },
}


def apply_bucket(queryset, key):
    """Apply a single named bucket's filter. Raises KeyError for an unknown
    key — callers (serializer validation, enrollment) should validate keys
    against BUCKET_REGISTRY before calling this, so KeyError here signals a
    real bug (a key that slipped past validation), not a shrug-worthy typo."""
    return BUCKET_REGISTRY[key]["apply"](queryset)


def apply_bucket_group(queryset, keys):
    """Apply a list of bucket keys: keys sharing the same `group` combine
    with OR (matches ANY of them); the resulting per-group sets combine with
    AND (must match every group that has at least one selected key). Empty
    `keys` is a no-op (returns queryset unchanged) — the "no filter selected"
    / "all eligible appointments" case."""
    if not keys:
        return queryset

    groups = {}
    for key in keys:
        group = BUCKET_REGISTRY[key]["group"]
        groups.setdefault(group, []).append(key)

    ids_per_group = []
    for group_keys in groups.values():
        # OR within a group: union the id sets each key's filter matches,
        # rather than trying to compose Q objects across the different
        # annotate() calls some buckets use (booking_age vs plain filters) —
        # simplest correct approach, and appointment counts here are small
        # enough (per-practice daily windows) that this is not a perf concern.
        group_ids = set()
        for key in group_keys:
            group_ids |= set(
                apply_bucket(queryset, key).values_list("id", flat=True)
            )
        ids_per_group.append(group_ids)

    matched_ids = ids_per_group[0]
    for ids in ids_per_group[1:]:
        matched_ids &= ids

    return queryset.filter(id__in=matched_ids)
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.CohortBucketRegistryTests --keepdb -v 2
```

Expected: PASS (9 tests).

- [ ] **Step 5: Do not commit.**

---

### Task 3: `targeting` field on `ConfirmationSequence` + `sub_cohort_name` on enrollment

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/models.py`
- Create: `TreatmentPath/dentallyIntegration/migrations/0149_confirmationsequence_targeting_and_more.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing test**

Append to `dentallyIntegration/tests.py`:

```python
class ConfirmationSequenceTargetingFieldTests(TestCase):
    """targeting is a JSON config: base_audience.bucket_keys + sub_cohorts
    list (each with bucket_keys/priority/confirm_dental_opt_in/
    template_override_id). Defaults to {} (= unconditional enrollment,
    today's behavior, unchanged)."""

    def test_targeting_defaults_to_empty_dict(self):
        practice = Practice.objects.create(name="Targeting Field Dental")
        sequence = ConfirmationSequence.objects.create(
            practice=practice, name="Untargeted sequence"
        )
        self.assertEqual(sequence.targeting, {})

    def test_targeting_round_trips(self):
        practice = Practice.objects.create(name="Targeting Round Trip Dental")
        sequence = ConfirmationSequence.objects.create(
            practice=practice,
            name="Targeted sequence",
            targeting={
                "base_audience": {"bucket_keys": ["booked_over_30d"]},
                "sub_cohorts": [
                    {
                        "name": "VIP opt-in",
                        "priority": 1,
                        "bucket_keys": ["high_risk"],
                        "confirm_dental_opt_in": True,
                        "template_override_id": None,
                    }
                ],
            },
        )
        sequence.refresh_from_db()
        self.assertEqual(
            sequence.targeting["base_audience"]["bucket_keys"], ["booked_over_30d"]
        )
        self.assertEqual(len(sequence.targeting["sub_cohorts"]), 1)
        self.assertTrue(sequence.targeting["sub_cohorts"][0]["confirm_dental_opt_in"])


class ConfirmationSequenceEnrollmentSubCohortFieldTests(TestCase):
    """sub_cohort_name: nullable, records which sub-cohort (if any) an
    enrollment matched at enrollment time."""

    def test_sub_cohort_name_defaults_to_blank(self):
        practice = Practice.objects.create(name="Sub Cohort Field Dental")
        sequence = ConfirmationSequence.objects.create(
            practice=practice, name="Test sequence", status="active"
        )
        appointment = DentallyAppointment.objects.create(
            practice=practice,
            dentally_id=30200,
            dentally_patient_id=3001,
            patient_name="Sub Cohort Test Patient",
            state="pending",
            duration=30,
            start_time=timezone.now() + timedelta(days=1),
        )
        now = timezone.now()
        enrollment = ConfirmationSequenceEnrollment.objects.create(
            practice=practice,
            sequence=sequence,
            appointment=appointment,
            enrolled_at=now,
            enrolled_appointment_start=appointment.start_time,
            next_due_at=now,
        )
        self.assertEqual(enrollment.sub_cohort_name, "")

        enrollment.sub_cohort_name = "VIP opt-in"
        enrollment.save(update_fields=["sub_cohort_name"])
        enrollment.refresh_from_db()
        self.assertEqual(enrollment.sub_cohort_name, "VIP opt-in")
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.ConfirmationSequenceTargetingFieldTests dentallyIntegration.tests.ConfirmationSequenceEnrollmentSubCohortFieldTests --keepdb -v 2
```

Expected: FAIL — `TypeError: ConfirmationSequence() got unexpected keyword arguments: 'targeting'` (and similarly for `sub_cohort_name`).

- [ ] **Step 3: Add the fields**

In `dentallyIntegration/models.py`, `ConfirmationSequence`, add right after `steps`:

```python
    targeting = models.JSONField(
        default=dict,
        blank=True,
        help_text="Cohort/bucket targeting config: "
        "{'base_audience': {'bucket_keys': [...]}, "
        "'sub_cohorts': [{'name', 'priority', 'bucket_keys', "
        "'confirm_dental_opt_in', 'template_override_id'}, ...]}. "
        "Empty dict (default) means unconditional enrollment — every "
        "eligible appointment in the window, today's unchanged behavior.",
    )
```

In `ConfirmationSequenceEnrollment`, add right after `stopped_reason`:

```python
    sub_cohort_name = models.CharField(
        max_length=120,
        blank=True,
        default="",
        help_text="Name of the sub_cohort (from the sequence's targeting "
        "config) this enrollment matched at enrollment time, or blank if it "
        "only matched the base audience. Snapshotted at enrollment time, not "
        "re-resolved later, so changing the sequence's targeting config "
        "after the fact doesn't retroactively change already-enrolled "
        "appointments' treatment.",
    )
```

- [ ] **Step 4: Generate and apply the migration**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py makemigrations dentallyIntegration
python manage.py migrate dentallyIntegration
```

Expected: a new migration with two `AddField` operations (`targeting` on `ConfirmationSequence`, `sub_cohort_name` on `ConfirmationSequenceEnrollment`).

- [ ] **Step 5: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationSequenceTargetingFieldTests dentallyIntegration.tests.ConfirmationSequenceEnrollmentSubCohortFieldTests --keepdb -v 2
```

Expected: PASS (3 tests).

- [ ] **Step 6: Do not commit.**

---

### Task 4: `ConfirmationSequence` serializer + validation

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/serializers.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

No `ConfirmationSequenceSerializer` exists anywhere in this codebase today — this task builds it from scratch, modeled on the existing `RecallSequenceSerializer` (`serializers.py:1612-1701`).

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class ConfirmationSequenceSerializerTests(TestCase):
    """Validates targeting JSON: every bucket_key must exist in
    BUCKET_REGISTRY, sub_cohort priorities must be unique."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Confirmation Serializer Dental")

    def test_valid_targeting_passes(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Valid sequence",
            "steps": [{"channel": "sms", "template_id": 1, "offset_days": 3}],
            "targeting": {
                "base_audience": {"bucket_keys": ["booked_over_30d"]},
                "sub_cohorts": [
                    {
                        "name": "VIP",
                        "priority": 1,
                        "bucket_keys": ["high_risk"],
                        "confirm_dental_opt_in": True,
                        "template_override_id": None,
                    }
                ],
            },
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertTrue(serializer.is_valid(), serializer.errors)

    def test_unknown_bucket_key_in_base_audience_fails(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Bad sequence",
            "steps": [{"channel": "sms", "template_id": 1, "offset_days": 3}],
            "targeting": {"base_audience": {"bucket_keys": ["not_a_real_bucket"]}},
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn("targeting", serializer.errors)

    def test_unknown_bucket_key_in_sub_cohort_fails(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Bad sub-cohort sequence",
            "steps": [{"channel": "sms", "template_id": 1, "offset_days": 3}],
            "targeting": {
                "base_audience": {"bucket_keys": []},
                "sub_cohorts": [
                    {"name": "Bad", "priority": 1, "bucket_keys": ["nope"]}
                ],
            },
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn("targeting", serializer.errors)

    def test_duplicate_sub_cohort_priority_fails(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Duplicate priority sequence",
            "steps": [{"channel": "sms", "template_id": 1, "offset_days": 3}],
            "targeting": {
                "base_audience": {"bucket_keys": []},
                "sub_cohorts": [
                    {"name": "A", "priority": 1, "bucket_keys": ["high_risk"]},
                    {"name": "B", "priority": 1, "bucket_keys": ["zero_balance"]},
                ],
            },
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertFalse(serializer.is_valid())
        self.assertIn("targeting", serializer.errors)

    def test_empty_targeting_is_valid(self):
        from .serializers import ConfirmationSequenceSerializer

        data = {
            "name": "Untargeted sequence",
            "steps": [{"channel": "sms", "template_id": 1, "offset_days": 3}],
        }
        serializer = ConfirmationSequenceSerializer(data=data)
        self.assertTrue(serializer.is_valid(), serializer.errors)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.ConfirmationSequenceSerializerTests --keepdb -v 2
```

Expected: FAIL — `ImportError: cannot import name 'ConfirmationSequenceSerializer'`.

- [ ] **Step 3: Add the serializer**

In `dentallyIntegration/serializers.py`, add (near `RecallSequenceSerializer`, since this is a direct sibling):

```python
class ConfirmationSequenceSerializer(serializers.ModelSerializer):
    """A confirmation sequence + its FR4/FR4a-d cohort/bucket targeting
    config. Unlike RecallSequenceSerializer's steps-only validation, this
    also validates `targeting`'s bucket keys against the live bucket
    registry and sub-cohort priority uniqueness."""

    enrollment_count = serializers.SerializerMethodField()

    class Meta:
        model = ConfirmationSequence
        fields = [
            "id",
            "name",
            "status",
            "days_ahead",
            "send_time",
            "steps",
            "targeting",
            "enrollment_count",
            "created_at",
            "updated_at",
        ]
        read_only_fields = ["id", "created_at", "updated_at"]

    def get_enrollment_count(self, obj):
        return obj.enrollments.filter(status="active").count()

    def validate_targeting(self, value):
        from .cohort_buckets import BUCKET_REGISTRY

        if not value:
            return value
        if not isinstance(value, dict):
            raise serializers.ValidationError("targeting must be an object.")

        base_audience = value.get("base_audience") or {}
        base_keys = base_audience.get("bucket_keys") or []
        if not isinstance(base_keys, list):
            raise serializers.ValidationError(
                "base_audience.bucket_keys must be a list."
            )
        for key in base_keys:
            if key not in BUCKET_REGISTRY:
                raise serializers.ValidationError(
                    f"Unknown bucket key in base_audience: '{key}'."
                )

        sub_cohorts = value.get("sub_cohorts") or []
        if not isinstance(sub_cohorts, list):
            raise serializers.ValidationError("sub_cohorts must be a list.")

        seen_priorities = set()
        for i, sub_cohort in enumerate(sub_cohorts):
            if not isinstance(sub_cohort, dict):
                raise serializers.ValidationError(
                    f"sub_cohorts[{i}] must be an object."
                )
            if not str(sub_cohort.get("name") or "").strip():
                raise serializers.ValidationError(
                    f"sub_cohorts[{i}] needs a name."
                )
            priority = sub_cohort.get("priority")
            if not isinstance(priority, int):
                raise serializers.ValidationError(
                    f"sub_cohorts[{i}].priority must be an integer."
                )
            if priority in seen_priorities:
                raise serializers.ValidationError(
                    f"sub_cohorts[{i}].priority ({priority}) duplicates "
                    "another sub-cohort's priority — priorities must be "
                    "unique so precedence is unambiguous."
                )
            seen_priorities.add(priority)
            sub_keys = sub_cohort.get("bucket_keys") or []
            if not isinstance(sub_keys, list):
                raise serializers.ValidationError(
                    f"sub_cohorts[{i}].bucket_keys must be a list."
                )
            for key in sub_keys:
                if key not in BUCKET_REGISTRY:
                    raise serializers.ValidationError(
                        f"Unknown bucket key in sub_cohorts[{i}]: '{key}'."
                    )

        return value
```

Add `ConfirmationSequence` to whatever model import line at the top of `serializers.py` already imports `RecallSequence` from `.models` (merge into the existing import, don't create a duplicate import line).

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationSequenceSerializerTests --keepdb -v 2
```

Expected: PASS (5 tests).

- [ ] **Step 5: Do not commit.**

---

### Task 5: `ConfirmationSequenceViewSet` (CRUD)

**Files:**
- Create: `TreatmentPath/dentallyIntegration/views/confirmation_sequence_views.py`
- Modify: `TreatmentPath/dentallyIntegration/urls.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

No viewset or URL exists for `ConfirmationSequence` today — this builds the CRUD layer from scratch, modeled on `RecallSequenceViewSet` (`views/recall_sequence_views.py:14-162`). Unlike Recall (which needs a manual `enroll` action for staff to hand-pick patients), confirmations enroll automatically by appointment date window — so this viewset needs no `enroll` action, just CRUD.

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class ConfirmationSequenceViewSetTests(TestCase):
    """CRUD for confirmation sequences, scoped to the requesting user's
    practice — mirrors RecallSequenceViewSetTests' shape."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Confirmation Sequence CRUD Dental")
        self.user = UserAuthentication.objects.create_user(
            email="crud@confirmationsequence.test",
            password="testpass123",
            practice=self.practice,
        )
        self.client.force_login(self.user)

    def test_create_sequence(self):
        response = self.client.post(
            "/api/backend/dentally/confirmation-sequences/",
            {
                "name": "New sequence",
                "steps": [{"channel": "sms", "template_id": 1, "offset_days": 3}],
            },
            content_type="application/json",
        )
        self.assertEqual(response.status_code, 201)
        self.assertEqual(ConfirmationSequence.objects.filter(practice=self.practice).count(), 1)

    def test_list_scoped_to_practice(self):
        other_practice = Practice.objects.create(name="Other Practice")
        ConfirmationSequence.objects.create(practice=self.practice, name="Mine")
        ConfirmationSequence.objects.create(practice=other_practice, name="Not mine")

        response = self.client.get("/api/backend/dentally/confirmation-sequences/")
        self.assertEqual(response.status_code, 200)
        names = {row["name"] for row in response.json()}
        self.assertIn("Mine", names)
        self.assertNotIn("Not mine", names)

    def test_update_targeting(self):
        sequence = ConfirmationSequence.objects.create(
            practice=self.practice, name="Editable sequence"
        )
        response = self.client.patch(
            f"/api/backend/dentally/confirmation-sequences/{sequence.id}/",
            {"targeting": {"base_audience": {"bucket_keys": ["high_risk"]}}},
            content_type="application/json",
        )
        self.assertEqual(response.status_code, 200)
        sequence.refresh_from_db()
        self.assertEqual(
            sequence.targeting["base_audience"]["bucket_keys"], ["high_risk"]
        )

    def test_delete_sequence(self):
        sequence = ConfirmationSequence.objects.create(
            practice=self.practice, name="Deletable sequence"
        )
        response = self.client.delete(
            f"/api/backend/dentally/confirmation-sequences/{sequence.id}/"
        )
        self.assertEqual(response.status_code, 204)
        self.assertFalse(ConfirmationSequence.objects.filter(id=sequence.id).exists())
```

(Check the top of `dentallyIntegration/tests.py` for however `UserAuthentication` test users are created elsewhere in the file — e.g. `AppointmentConfirmActivityLogTests` or similar existing test classes that hit authenticated endpoints — and match that exact pattern if it differs from `create_user`/`force_login` shown here, since this project may have a custom user-creation helper.)

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.ConfirmationSequenceViewSetTests --keepdb -v 2
```

Expected: FAIL — 404s (no such URL exists yet).

- [ ] **Step 3: Create the viewset**

Create `dentallyIntegration/views/confirmation_sequence_views.py`:

```python
"""Confirmation sequence CRUD + cohort/bucket preview-count (Daylist
Confirmations Phase 4). Unlike RecallSequenceViewSet, no manual `enroll`
action exists here — confirmation enrollment is fully automatic, driven by
enroll_eligible_appointments' date-window + targeting-config filtering."""

from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework import status


class ConfirmationSequenceViewSet(viewsets.ModelViewSet):
    """CRUD for confirmation sequences, scoped to the requesting user's
    practice."""

    permission_classes = [IsAuthenticated]

    def get_serializer_class(self):
        from ..serializers import ConfirmationSequenceSerializer

        return ConfirmationSequenceSerializer

    def get_queryset(self):
        from ..models import ConfirmationSequence

        practice = getattr(self.request.user, "practice", None)
        if not practice:
            return ConfirmationSequence.objects.none()
        return ConfirmationSequence.objects.filter(practice=practice).order_by(
            "-updated_at"
        )

    def perform_create(self, serializer):
        serializer.save(practice=self.request.user.practice)

    @action(detail=True, methods=["post"], url_path="preview-count")
    def preview_count(self, request, pk=None):
        """POST confirmation-sequences/{id}/preview-count/ — {"targeting": {...}}
        (candidate, not-yet-saved targeting config). Returns the FR4d/FR6
        breakdown: base-audience count, per-sub-cohort counts, deduped/
        overlap count, final send count, and sendable/excluded/already-
        confirmed/already-cancelled/missing-contact splits — computed against
        the SAME eligibility base enroll_eligible_appointments uses, so the
        preview can't drift from what would actually happen on save. See
        Task 6 for the implementation of the shared helper this calls."""
        from ..confirmation_automation import compute_targeting_preview
        from ..models import ConfirmationSequence

        practice = getattr(request.user, "practice", None)
        if not practice:
            return Response(
                {"error": "No practice associated with this user"},
                status=status.HTTP_400_BAD_REQUEST,
            )
        sequence = ConfirmationSequence.objects.filter(
            id=pk, practice=practice
        ).first()
        if not sequence:
            return Response(
                {"error": "Sequence not found"}, status=status.HTTP_404_NOT_FOUND
            )

        candidate_targeting = request.data.get("targeting") or {}
        return Response(
            compute_targeting_preview(sequence, candidate_targeting)
        )
```

- [ ] **Step 4: Register the URL**

In `dentallyIntegration/urls.py`, add the import (alongside the existing `RecallSequenceViewSet`/`DayListAutomationViewSet` imports):

```python
from .views.confirmation_sequence_views import ConfirmationSequenceViewSet
```

Add the router registration (right after the existing `daylist-automations` registration, before `urlpatterns += _router.urls`):

```python
# Confirmation sequences — CRUD + cohort/bucket preview-count (ModelViewSet via router):
#   GET/POST  confirmation-sequences/
#   GET/PUT/PATCH/DELETE  confirmation-sequences/{id}/
#   POST  confirmation-sequences/{id}/preview-count/
_router.register(
    r"confirmation-sequences",
    ConfirmationSequenceViewSet,
    basename="confirmation-sequence",
)
```

- [ ] **Step 5: Run tests to verify they pass (targeting update/delete/list tests will pass now; the create test needs Task 6's `compute_targeting_preview` stub to not exist yet — it's fine if the preview action itself isn't tested until Task 6)**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationSequenceViewSetTests --keepdb -v 2
```

Expected: PASS (4 tests) — none of these tests hit the `preview-count` action yet, so Task 6 hasn't been reached and that's fine.

- [ ] **Step 6: Do not commit.**

---

### Task 6: Enrollment integration + preview-count logic

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/confirmation_automation.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class EnrollEligibleAppointmentsTargetingTests(TestCase):
    """enroll_eligible_appointments applies targeting.base_audience before
    its existing eligibility checks, and tags sub_cohort_name when an
    appointment matches a sub-cohort."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Targeting Enrollment Dental")

    def _appt(self, dentally_id, **overrides):
        defaults = dict(
            practice=self.practice,
            dentally_id=dentally_id,
            dentally_patient_id=1,
            patient_name="Targeting Enrollment Test Patient",
            patient_phone="+447700900000",
            state="pending",
            duration=30,
            risk_tier="Low",
            patient_outstanding_balance=0,
            start_time=timezone.now() + timedelta(days=2),
        )
        defaults.update(overrides)
        return DentallyAppointment.objects.create(**defaults)

    def test_base_audience_filters_out_non_matching_appointments(self):
        sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="Zero balance only",
            status="active",
            days_ahead=7,
            steps=[{"channel": "sms", "template_id": 1, "offset_days": 1}],
            targeting={"base_audience": {"bucket_keys": ["zero_balance"]}},
        )
        matching = self._appt(40100, patient_outstanding_balance=0)
        non_matching = self._appt(40101, patient_outstanding_balance=50)

        count = enroll_eligible_appointments(sequence)

        self.assertEqual(count, 1)
        self.assertTrue(
            ConfirmationSequenceEnrollment.objects.filter(appointment=matching).exists()
        )
        self.assertFalse(
            ConfirmationSequenceEnrollment.objects.filter(appointment=non_matching).exists()
        )

    def test_empty_base_audience_enrolls_everyone(self):
        """No targeting configured = today's unconditional behavior, unchanged."""
        sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="Untargeted",
            status="active",
            days_ahead=7,
            steps=[{"channel": "sms", "template_id": 1, "offset_days": 1}],
        )
        appt_a = self._appt(40200, patient_outstanding_balance=0)
        appt_b = self._appt(40201, patient_outstanding_balance=500)

        count = enroll_eligible_appointments(sequence)

        self.assertEqual(count, 2)
        self.assertTrue(
            ConfirmationSequenceEnrollment.objects.filter(appointment=appt_a).exists()
        )
        self.assertTrue(
            ConfirmationSequenceEnrollment.objects.filter(appointment=appt_b).exists()
        )

    def test_sub_cohort_tagging(self):
        sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="With VIP sub-cohort",
            status="active",
            days_ahead=7,
            steps=[{"channel": "sms", "template_id": 1, "offset_days": 1}],
            targeting={
                "base_audience": {"bucket_keys": []},
                "sub_cohorts": [
                    {
                        "name": "VIP opt-in",
                        "priority": 1,
                        "bucket_keys": ["high_risk"],
                        "confirm_dental_opt_in": True,
                    }
                ],
            },
        )
        vip = self._appt(40300, risk_tier="High")
        standard = self._appt(40301, risk_tier="Low")

        enroll_eligible_appointments(sequence)

        vip_enrollment = ConfirmationSequenceEnrollment.objects.get(appointment=vip)
        standard_enrollment = ConfirmationSequenceEnrollment.objects.get(
            appointment=standard
        )
        self.assertEqual(vip_enrollment.sub_cohort_name, "VIP opt-in")
        self.assertEqual(standard_enrollment.sub_cohort_name, "")

    def test_sub_cohort_priority_first_match_wins(self):
        sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="Two overlapping sub-cohorts",
            status="active",
            days_ahead=7,
            steps=[{"channel": "sms", "template_id": 1, "offset_days": 1}],
            targeting={
                "base_audience": {"bucket_keys": []},
                "sub_cohorts": [
                    {"name": "Highest priority", "priority": 1, "bucket_keys": ["high_risk"]},
                    {"name": "Lower priority", "priority": 2, "bucket_keys": ["zero_balance"]},
                ],
            },
        )
        # Matches BOTH sub-cohorts — priority 1 should win.
        both = self._appt(40400, risk_tier="High", patient_outstanding_balance=0)

        enroll_eligible_appointments(sequence)

        enrollment = ConfirmationSequenceEnrollment.objects.get(appointment=both)
        self.assertEqual(enrollment.sub_cohort_name, "Highest priority")


class ComputeTargetingPreviewTests(TestCase):
    """compute_targeting_preview returns the FR4d/FR6 count breakdown for a
    CANDIDATE (not-yet-saved) targeting config, against the same eligibility
    base enrollment uses."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Preview Count Dental")
        self.sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="Preview sequence",
            status="active",
            days_ahead=7,
            steps=[{"channel": "sms", "template_id": 1, "offset_days": 1}],
        )

    def test_preview_counts_match_what_enrollment_would_do(self):
        DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=40500,
            dentally_patient_id=1,
            patient_name="Preview Test Patient A",
            patient_phone="+447700900000",
            state="pending",
            duration=30,
            risk_tier="High",
            patient_outstanding_balance=0,
            start_time=timezone.now() + timedelta(days=2),
        )
        DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=40501,
            dentally_patient_id=2,
            patient_name="Preview Test Patient B",
            patient_phone="+447700900001",
            state="pending",
            duration=30,
            risk_tier="Low",
            patient_outstanding_balance=99,
            start_time=timezone.now() + timedelta(days=2),
        )

        preview = compute_targeting_preview(
            self.sequence, {"base_audience": {"bucket_keys": ["high_risk"]}}
        )
        self.assertEqual(preview["base_audience_count"], 1)
        self.assertEqual(preview["final_send_count"], 1)

    def test_preview_breaks_out_sub_cohort_counts(self):
        DentallyAppointment.objects.create(
            practice=self.practice,
            dentally_id=40600,
            dentally_patient_id=3,
            patient_name="Preview VIP Patient",
            patient_phone="+447700900002",
            state="pending",
            duration=30,
            risk_tier="High",
            patient_outstanding_balance=0,
            start_time=timezone.now() + timedelta(days=2),
        )

        preview = compute_targeting_preview(
            self.sequence,
            {
                "base_audience": {"bucket_keys": []},
                "sub_cohorts": [
                    {"name": "VIP", "priority": 1, "bucket_keys": ["high_risk"]}
                ],
            },
        )
        sub_cohort_counts = {sc["name"]: sc["count"] for sc in preview["sub_cohorts"]}
        self.assertEqual(sub_cohort_counts["VIP"], 1)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.EnrollEligibleAppointmentsTargetingTests dentallyIntegration.tests.ComputeTargetingPreviewTests --keepdb -v 2
```

Expected: FAIL — the targeting tests fail because `enroll_eligible_appointments` doesn't apply `targeting` yet (all appointments get enrolled regardless of bucket match); `ComputeTargetingPreviewTests` fails with `NameError: name 'compute_targeting_preview' is not defined`.

- [ ] **Step 3: Add `resolve_sub_cohort` and wire targeting into enrollment**

In `dentallyIntegration/confirmation_automation.py`, add this new function right before `enroll_eligible_appointments`:

```python
def resolve_sub_cohort(appointment, targeting):
    """Return the matching sub_cohort dict (from targeting['sub_cohorts'],
    already sorted here by priority ascending) for this appointment, or None
    if it matches no sub-cohort. First match by priority wins (FR4c)."""
    from .cohort_buckets import apply_bucket_group
    from .models import DentallyAppointment

    sub_cohorts = sorted(
        targeting.get("sub_cohorts") or [], key=lambda sc: sc["priority"]
    )
    single_appointment_qs = DentallyAppointment.objects.filter(id=appointment.id)
    for sub_cohort in sub_cohorts:
        matched = apply_bucket_group(
            single_appointment_qs, sub_cohort.get("bucket_keys") or []
        ).exists()
        if matched:
            return sub_cohort
    return None
```

Modify `enroll_eligible_appointments` — replace the `candidates = ...` line and the body of the `for appointment in candidates:` loop's start with this (the only changes are: applying `targeting.base_audience` to `candidates`, and setting `sub_cohort_name` on the created enrollment):

```python
def enroll_eligible_appointments(sequence, now=None):
    """Pass 1: enroll every eligible future appointment (within
    sequence.days_ahead, valid contact, not already confirmed/cancelled/
    cancellation-requested, no existing active enrollment for ANY sequence,
    AND matching sequence.targeting.base_audience if configured) that
    doesn't have one yet. FR3a: starts current_step at the first step whose
    due date is still >= now; if every step's due date has already passed,
    does not enroll at all (nothing left to catch up to). FR4c: tags the
    enrollment with whichever sub_cohort (if any) the appointment matches,
    by priority — first match wins."""
    from django.utils import timezone as dj_timezone

    from .cohort_buckets import apply_bucket_group
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

    targeting = sequence.targeting or {}
    base_audience_keys = (targeting.get("base_audience") or {}).get("bucket_keys") or []
    candidates = apply_bucket_group(candidates, base_audience_keys)

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

        sub_cohort = resolve_sub_cohort(appointment, targeting)

        ConfirmationSequenceEnrollment.objects.create(
            practice=practice,
            sequence=sequence,
            appointment=appointment,
            current_step=start_index,
            enrolled_at=now,
            enrolled_appointment_start=appointment.start_time,
            next_due_at=due_dates[start_index],
            sub_cohort_name=sub_cohort["name"] if sub_cohort else "",
        )
        enrolled_count += 1

    return enrolled_count
```

- [ ] **Step 4: Add `compute_targeting_preview`**

Add this new function to the same file, after `enroll_eligible_appointments`:

```python
def compute_targeting_preview(sequence, candidate_targeting):
    """FR4d/FR6: given a CANDIDATE (not-yet-saved) targeting config, return
    the count breakdown a save would produce — computed against the exact
    same eligibility base enroll_eligible_appointments uses, so this can't
    drift from what actually happens on save. Read-only, no persistence."""
    from django.utils import timezone as dj_timezone

    from .cohort_buckets import apply_bucket_group
    from .models import DentallyAppointment

    now = dj_timezone.now()
    practice = sequence.practice
    window_end = now + timedelta(days=sequence.days_ahead)

    already_enrolled_ids = set(
        ConfirmationSequenceEnrollment.objects.filter(
            practice=practice, status="active"
        ).values_list("appointment_id", flat=True)
    )
    window_candidates = DentallyAppointment.objects.filter(
        practice=practice,
        start_time__gte=now,
        start_time__lte=window_end,
    ).exclude(id__in=already_enrolled_ids)

    base_keys = (candidate_targeting.get("base_audience") or {}).get("bucket_keys") or []
    base_matched = apply_bucket_group(window_candidates, base_keys)
    base_audience_count = base_matched.count()

    sendable_sms = 0
    sendable_email = 0
    excluded = 0
    already_confirmed = 0
    already_cancelled = 0
    missing_contact = 0
    sub_cohort_ids = {}

    for appointment in base_matched:
        status_display = compute_confirmation_display_status(appointment)
        if status_display == "Confirmed":
            already_confirmed += 1
            continue
        if status_display == "Cancelled":
            already_cancelled += 1
            continue
        if status_display == "No valid contact":
            missing_contact += 1
            continue
        if status_display in NOT_ELIGIBLE_STATUSES:
            excluded += 1
            continue
        if appointment.patient_phone:
            sendable_sms += 1
        if appointment.patient_email:
            sendable_email += 1

        sub_cohort = resolve_sub_cohort(appointment, candidate_targeting)
        if sub_cohort:
            sub_cohort_ids.setdefault(sub_cohort["name"], set()).add(appointment.id)

    final_send_count = (
        base_audience_count - already_confirmed - already_cancelled
        - missing_contact - excluded
    )
    overlap_deduped_count = len(
        set().union(*sub_cohort_ids.values()) if sub_cohort_ids else set()
    )

    return {
        "base_audience_count": base_audience_count,
        "sub_cohorts": [
            {"name": name, "count": len(ids)} for name, ids in sub_cohort_ids.items()
        ],
        "overlap_deduped_count": overlap_deduped_count,
        "final_send_count": max(0, final_send_count),
        "sendable_sms": sendable_sms,
        "sendable_email": sendable_email,
        "excluded": excluded,
        "already_confirmed": already_confirmed,
        "already_cancelled": already_cancelled,
        "missing_contact": missing_contact,
    }
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.EnrollEligibleAppointmentsTargetingTests dentallyIntegration.tests.ComputeTargetingPreviewTests --keepdb -v 2
```

Expected: PASS (6 tests).

- [ ] **Step 6: Run the full confirmation-automation suite to confirm no regressions**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationAutomationSendingTests dentallyIntegration.tests.AdvanceConfirmationEnrollmentsTests dentallyIntegration.tests.EnrollEligibleAppointmentsTests dentallyIntegration.tests.ProcessConfirmationEnrollmentsTests dentallyIntegration.tests.ConfirmationSequenceViewSetTests --keepdb -v 2
```

Expected: PASS, all green — `EnrollEligibleAppointmentsTests` (the pre-existing, pre-targeting test class) must still pass unchanged, since an empty `targeting` dict is a no-op. `ConfirmationSequenceViewSetTests`' `test_create_sequence` should now pass (it doesn't hit `preview-count`, so it was unaffected by this task, but confirming it's still green here).

- [ ] **Step 7: Do not commit.**

---

### Task 7: Inbox allow-list change

**Files:**
- Modify: `TreatmentPath/messaging/views/session_views.py`
- Test: `TreatmentPath/messaging/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `messaging/tests.py`:

```python
class MessageSessionInboxAllowListTests(TestCase):
    """The main Inbox shows a session only if it has at least one
    message_purpose='manual' message — recall/automation-only sessions are
    hidden, replacing the old exclude-list-just-'recall' rule."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Inbox Allow List Dental")
        self.user = UserAuthentication.objects.create_user(
            email="inbox@allowlist.test", password="testpass123", practice=self.practice,
        )
        self.client.force_login(self.user)

    def _session(self, **overrides):
        defaults = dict(practice=self.practice, session_type="sms")
        defaults.update(overrides)
        return MessageSession.objects.create(**defaults)

    def test_session_with_only_recall_message_is_hidden(self):
        session = self._session()
        SMSMessage.objects.create(
            practice=self.practice, session=session, phone_number="+447700900000",
            content="Recall reminder", message_purpose="recall", direction="outgoing",
        )
        response = self.client.get("/api/backend/messaging/sessions/")
        ids = {row["id"] for row in response.json()["results"]}
        self.assertNotIn(str(session.id), {str(i) for i in ids})

    def test_session_with_only_automation_message_is_hidden(self):
        session = self._session()
        SMSMessage.objects.create(
            practice=self.practice, session=session, phone_number="+447700900001",
            content="Confirmation reminder", message_purpose="automation",
            direction="outgoing",
        )
        response = self.client.get("/api/backend/messaging/sessions/")
        ids = {row["id"] for row in response.json()["results"]}
        self.assertNotIn(str(session.id), {str(i) for i in ids})

    def test_session_with_a_manual_message_is_shown(self):
        session = self._session()
        SMSMessage.objects.create(
            practice=self.practice, session=session, phone_number="+447700900002",
            content="Hi there", message_purpose="manual", direction="outgoing",
        )
        response = self.client.get("/api/backend/messaging/sessions/")
        ids = {str(row["id"]) for row in response.json()["results"]}
        self.assertIn(str(session.id), ids)

    def test_session_with_mixed_manual_and_automation_messages_is_shown(self):
        session = self._session()
        SMSMessage.objects.create(
            practice=self.practice, session=session, phone_number="+447700900003",
            content="Reminder", message_purpose="automation", direction="outgoing",
        )
        SMSMessage.objects.create(
            practice=self.practice, session=session, phone_number="+447700900003",
            content="Patient reply", message_purpose="manual", direction="incoming",
        )
        response = self.client.get("/api/backend/messaging/sessions/")
        ids = {str(row["id"]) for row in response.json()["results"]}
        self.assertIn(str(session.id), ids)

    def test_empty_session_is_shown(self):
        """Sessions with zero messages (message_count=0) are kept as-is,
        matching the pre-existing behavior."""
        session = self._session()
        response = self.client.get("/api/backend/messaging/sessions/")
        ids = {str(row["id"]) for row in response.json()["results"]}
        self.assertIn(str(session.id), ids)
```

(Check the top of `messaging/tests.py` for the exact existing import list and any established test-user-creation helper, and match it — this file likely already imports `MessageSession`, `SMSMessage`, `Practice`, `UserAuthentication` from prior tests.)

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test messaging.tests.MessageSessionInboxAllowListTests --keepdb -v 2
```

Expected: FAIL — `test_session_with_only_automation_message_is_hidden` fails (automation sessions currently show up), since today's rule only excludes `"recall"`.

- [ ] **Step 3: Flip the filter to an allow-list**

In `messaging/views/session_views.py`, replace this block (the exclude-list logic inside `get_queryset`, in the `else` branch of the archive/spam check):

```python
            # Hide recall-only conversations from the main inbox. Recall sends
            # (message_purpose="recall" — both bulk_send and recall_automation)
            # belong in recall reporting / sandbox, not the regular inbox. A
            # session re-appears the moment it has ANY non-recall message (e.g. a
            # patient reply, which defaults to "manual"). `.exclude(purpose="recall")`
            # keeps blank/NULL/manual/automation rows, so regular and legacy
            # messages count as non-recall and are unaffected. Empty sessions
            # (message_count=0) are kept as-is.
            non_recall_email = EmailMessages.objects.filter(
                session=OuterRef("pk")
            ).exclude(message_purpose="recall")
            non_recall_sms = SMSMessage.objects.filter(session=OuterRef("pk")).exclude(
                message_purpose="recall"
            )
            non_recall_wa = WhatsAppMessage.objects.filter(
                session=OuterRef("pk")
            ).exclude(message_purpose="recall")
            queryset = queryset.filter(
                Q(message_count=0)
                | Exists(non_recall_email)
                | Exists(non_recall_sms)
                | Exists(non_recall_wa)
            )
```

with:

```python
            # Show a session only if it has at least one message_purpose=
            # "manual" message (an allow-list, not an exclude-list) — this
            # automatically hides EVERY automated-send label (recall,
            # automation, and any future one added later) with no repeated
            # code change needed per new label, unlike the old exclude-just-
            # "recall" rule this replaces. Empty sessions (message_count=0)
            # are kept as-is, matching prior behavior.
            manual_email = EmailMessages.objects.filter(
                session=OuterRef("pk"), message_purpose="manual"
            )
            manual_sms = SMSMessage.objects.filter(
                session=OuterRef("pk"), message_purpose="manual"
            )
            manual_wa = WhatsAppMessage.objects.filter(
                session=OuterRef("pk"), message_purpose="manual"
            )
            queryset = queryset.filter(
                Q(message_count=0)
                | Exists(manual_email)
                | Exists(manual_sms)
                | Exists(manual_wa)
            )
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test messaging.tests.MessageSessionInboxAllowListTests --keepdb -v 2
```

Expected: PASS (5 tests).

- [ ] **Step 5: Run the full messaging test suite to confirm no regressions**

```bash
python manage.py test messaging --keepdb -v 2 2>&1 | tail -40
```

Expected: no NEW failures beyond whatever pre-existing failures already exist in this file (check for any that specifically assert on the OLD exclude-`"recall"`-only behavior — if any exist, they need updating to match the new allow-list rule, since that old behavior is being deliberately replaced).

- [ ] **Step 6: Do not commit.**

---

### Task 8: `DaylistReportingViewSet` + URL + API config

**Files:**
- Create: `TreatmentPath/dentallyIntegration/views/daylist_reporting_views.py`
- Modify: `TreatmentPath/dentallyIntegration/urls.py`
- Modify: `perfect-pixel-playground-project/src/config/api.ts`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `dentallyIntegration/tests.py`:

```python
class DaylistReportingViewSetTests(TestCase):
    """Mirrors RecallReportingViewSet's shape, but scoped to
    source_type in ('daylist_automation', 'confirmation_automation')
    instead of message_purpose='recall' — both currently share the generic
    message_purpose='automation' label, so source_type is the discriminator."""

    def setUp(self):
        self.practice = Practice.objects.create(name="Daylist Reporting Dental")
        self.user = UserAuthentication.objects.create_user(
            email="reporting@daylist.test", password="testpass123", practice=self.practice,
        )
        self.client.force_login(self.user)
        self.contact = ContactIdentity.objects.create(
            practice=self.practice, normalized_phone="7700900555", display_name="Reporting Test Patient",
        )

    def test_count_mode_includes_daylist_and_confirmation_automation(self):
        SMSMessage.objects.create(
            practice=self.practice, contact=self.contact, phone_number="+447700900555",
            content="Daylist reminder", message_purpose="automation",
            source_type="daylist_automation", direction="outgoing",
        )
        EmailMessages.objects.create(
            practice=self.practice, contact=self.contact, direction="outgoing",
            message_purpose="automation", source_type="confirmation_automation",
            to_patient="patient@test.com", subject="Please confirm", body="...",
        )
        response = self.client.get(
            f"/api/backend/dentally/daylist-reporting/?mode=count&contact_id={self.contact.id}&months=12"
        )
        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertEqual(data["by_channel"]["sms"], 1)
        self.assertEqual(data["by_channel"]["email"], 1)

    def test_count_mode_excludes_recall_and_manual(self):
        SMSMessage.objects.create(
            practice=self.practice, contact=self.contact, phone_number="+447700900555",
            content="Recall", message_purpose="recall", source_type="recall_automation",
            direction="outgoing",
        )
        SMSMessage.objects.create(
            practice=self.practice, contact=self.contact, phone_number="+447700900555",
            content="Manual chat", message_purpose="manual", direction="outgoing",
        )
        response = self.client.get(
            f"/api/backend/dentally/daylist-reporting/?mode=count&contact_id={self.contact.id}&months=12"
        )
        data = response.json()
        self.assertEqual(data["by_channel"]["sms"], 0)

    def test_events_mode_returns_message_list(self):
        SMSMessage.objects.create(
            practice=self.practice, contact=self.contact, phone_number="+447700900555",
            content="Confirmation reminder", message_purpose="automation",
            source_type="confirmation_automation", direction="outgoing",
        )
        response = self.client.get(
            f"/api/backend/dentally/daylist-reporting/?mode=events&contact_id={self.contact.id}&months=12"
        )
        self.assertEqual(response.status_code, 200)
        self.assertEqual(len(response.json()["results"]), 1)
```

(Match whatever exact `ContactIdentity`/`SMSMessage`/`EmailMessages` field names are already established elsewhere in `dentallyIntegration/tests.py` — e.g. `ConfirmationAutomationActivityLogTests` from an earlier phase already creates these objects, use that as the reference for exact field names/kwargs.)

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.DaylistReportingViewSetTests --keepdb -v 2
```

Expected: FAIL — 404 (no such URL exists yet).

- [ ] **Step 3: Create the viewset**

Create `dentallyIntegration/views/daylist_reporting_views.py`, modeled directly on `RecallReportingViewSet` (`views/recall_sequence_views.py:165-478`), with two changes: filter by `source_type__in=[...]` instead of `message_purpose="recall"`, and drop the `CallLog` channel entirely (daylist/confirmation automation has no call-logging concept, unlike Recall):

```python
"""Daylist activity reporting (Daylist Confirmations Phase 4, Part B).

Mirrors RecallReportingViewSet's shape exactly, but scoped to
source_type in ('daylist_automation', 'confirmation_automation') rather than
message_purpose='recall' — both currently share the generic
message_purpose='automation' label (see messaging/models.py's
MESSAGE_PURPOSE choices), so source_type is what actually discriminates
these from any other automation-purpose message. No call-log channel here,
unlike Recall — daylist/confirmation automation never logs calls."""

import logging

from rest_framework import status, viewsets
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response


logger = logging.getLogger(__name__)

DAYLIST_SOURCE_TYPES = ["daylist_automation", "confirmation_automation"]


class DaylistReportingViewSet(viewsets.ViewSet):
    """GET /dentally/daylist-reporting/
      ?mode=count|events  (default count)
      &months=12 | &date_from=ISO&date_to=ISO
      &dentally_patient_id= | &contact_id=
      &channel=email|sms  &source_type=  &status=
    """

    permission_classes = [IsAuthenticated]

    @staticmethod
    def _resolve_contact(practice, dentally_patient_id):
        from TreatmentPlan.models import ContactIdentity, Patient

        try:
            patient = Patient.objects.select_related("contact").get(
                practice=practice, meta_data__id=dentally_patient_id
            )
        except (Patient.DoesNotExist, Patient.MultipleObjectsReturned):
            return None
        return patient.contact_id if patient.contact_id else None

    def list(self, request):
        from datetime import datetime as _dt
        from datetime import time as _time
        from datetime import timedelta
        from collections import defaultdict

        from django.db.models import Count
        from django.utils import timezone
        from django.utils.dateparse import parse_date, parse_datetime

        from messaging.models import EmailMessages, SMSMessage

        practice = getattr(request.user, "practice", None)
        if not practice:
            return Response(
                {"error": "No practice associated with this user"},
                status=status.HTTP_400_BAD_REQUEST,
            )

        qp = request.query_params
        now = timezone.now()

        def _parse_dt(v):
            if not v:
                return None
            dt = parse_datetime(v)
            if dt:
                return dt if timezone.is_aware(dt) else timezone.make_aware(dt)
            d = parse_date(v)
            return timezone.make_aware(_dt.combine(d, _time.min)) if d else None

        start = _parse_dt(qp.get("date_from"))
        end = _parse_dt(qp.get("date_to")) or now
        if start is None:
            try:
                m = int(qp.get("months") or 12)
            except (TypeError, ValueError):
                m = 12
            start = now - timedelta(days=max(1, m) * 30)

        channel = qp.get("channel")
        source_type = qp.get("source_type")
        status_f = qp.get("status")
        mode = qp.get("mode", "count")

        contact_id = qp.get("contact_id") or None
        scoped = False
        if qp.get("dentally_patient_id"):
            scoped = True
            try:
                contact_id = self._resolve_contact(
                    practice, int(qp["dentally_patient_id"])
                )
            except (TypeError, ValueError):
                contact_id = None
        elif contact_id:
            scoped = True

        source_types = [source_type] if source_type else DAYLIST_SOURCE_TYPES

        email_qs = EmailMessages.objects.filter(
            practice=practice,
            message_purpose="automation",
            source_type__in=source_types,
            direction="outgoing",
            received_at__gte=start,
            received_at__lte=end,
        )
        sms_qs = SMSMessage.objects.filter(
            practice=practice,
            message_purpose="automation",
            source_type__in=source_types,
            direction="outgoing",
            created_at__gte=start,
            created_at__lte=end,
        )

        if scoped:
            if contact_id:
                email_qs = email_qs.filter(contact_id=contact_id)
                sms_qs = sms_qs.filter(contact_id=contact_id)
            else:
                email_qs = email_qs.none()
                sms_qs = sms_qs.none()

        if status_f:
            sms_qs = sms_qs.filter(status=status_f)
            if status_f != "sent":
                email_qs = email_qs.none()

        if channel:
            email_qs = email_qs if channel == "email" else email_qs.none()
            sms_qs = sms_qs if channel == "sms" else sms_qs.none()

        if mode == "events":
            events = []
            for r in email_qs.values("id", "received_at", "source_type", "subject"):
                events.append(
                    {
                        "channel": "email",
                        "timestamp": r["received_at"],
                        "status": "sent",
                        "source_type": r["source_type"],
                        "preview": r["subject"] or "Daylist email",
                    }
                )
            for r in sms_qs.values("id", "created_at", "source_type", "status", "content"):
                events.append(
                    {
                        "channel": "sms",
                        "timestamp": r["created_at"],
                        "status": r["status"],
                        "source_type": r["source_type"],
                        "preview": (r["content"] or "")[:120],
                    }
                )
            events.sort(key=lambda e: e["timestamp"] or now, reverse=True)
            try:
                limit = min(int(qp.get("limit") or 100), 500)
            except (TypeError, ValueError):
                limit = 100
            return Response(
                {
                    "count": len(events),
                    "date_from": start,
                    "date_to": end,
                    "results": events[:limit],
                }
            )

        e = email_qs.count()
        s = sms_qs.count()

        by_status = defaultdict(int)
        by_status["sent"] += e
        for row in sms_qs.values("status").annotate(n=Count("id")):
            by_status[row["status"] or "unknown"] += row["n"]

        by_source = defaultdict(int)
        for qs in (email_qs, sms_qs):
            for row in qs.values("source_type").annotate(n=Count("id")):
                by_source[row["source_type"] or "unspecified"] += row["n"]

        return Response(
            {
                "date_from": start,
                "date_to": end,
                "total_contact_attempts": e + s,
                "total_messages": e + s,
                "by_channel": {"email": e, "sms": s},
                "by_status": dict(by_status),
                "by_source_type": dict(by_source),
            }
        )
```

- [ ] **Step 4: Register the URL**

In `dentallyIntegration/urls.py`, add the import (alongside `RecallReportingViewSet`):

```python
from .views.daylist_reporting_views import DaylistReportingViewSet
```

Add the path (right after the existing `recall-reporting/` path):

```python
    path(
        "daylist-reporting/",
        DaylistReportingViewSet.as_view({"get": "list"}),
        name="dentally-daylist-reporting",
    ),
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.DaylistReportingViewSetTests --keepdb -v 2
```

Expected: PASS (3 tests).

- [ ] **Step 6: Add the frontend API config entry**

In `perfect-pixel-playground-project/src/config/api.ts`, add right after `RECALL_REPORTING` in the `DENTALLY` object:

```ts
    DAYLIST_REPORTING: (params?: string) => getApiUrl(params ? `/dentally/daylist-reporting/?${params}` : '/dentally/daylist-reporting/'),
```

- [ ] **Step 7: Do not commit.**

---

### Task 9: Frontend types

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/day-list/types.ts`

- [ ] **Step 1: Add the new types**

Add near the existing `ConfirmationStatus`/`ConfirmationEnrollment` types (from Phase 3):

```ts
export interface CohortBucketOption {
  key: string;
  label: string;
  group: string;
}

export const COHORT_BUCKETS: CohortBucketOption[] = [
  { key: 'high_risk', label: 'High risk', group: 'risk' },
  { key: 'medium_risk', label: 'Medium/Elevated risk', group: 'risk' },
  { key: 'booked_under_30d', label: 'Booked less than 30 days ago', group: 'booking_age' },
  { key: 'booked_over_30d', label: 'Booked over 30 days ago', group: 'booking_age' },
  { key: 'booked_over_60d', label: 'Booked over 60 days ago', group: 'booking_age' },
  { key: 'long_and_old_booking', label: 'Over 45 minutes and booked over 7 days ago', group: 'compound' },
  { key: 'zero_balance', label: 'GBP 0 balance', group: 'balance' },
];

export interface ConfirmationSubCohort {
  name: string;
  priority: number;
  bucket_keys: string[];
  confirm_dental_opt_in: boolean;
  template_override_id: number | null;
}

export interface ConfirmationTargeting {
  base_audience: { bucket_keys: string[] };
  sub_cohorts: ConfirmationSubCohort[];
}

export interface ConfirmationSequence {
  id: number;
  name: string;
  status: 'draft' | 'active' | 'paused' | 'archived';
  days_ahead: number;
  send_time: string;
  steps: Array<{ channel: 'sms' | 'email'; template_id: number; offset_days: number; send_time?: string }>;
  targeting: ConfirmationTargeting;
  enrollment_count: number;
  created_at: string;
  updated_at: string;
}

export interface ConfirmationTargetingPreview {
  base_audience_count: number;
  sub_cohorts: Array<{ name: string; count: number }>;
  overlap_deduped_count: number;
  final_send_count: number;
  sendable_sms: number;
  sendable_email: number;
  excluded: number;
  already_confirmed: number;
  already_cancelled: number;
  missing_contact: number;
}
```

- [ ] **Step 2: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit --project tsconfig.app.json --ignoreDeprecations "5.0" 2>&1 | grep -c "error TS"
```

Expected: no new errors vs. the current baseline (get the current baseline count first by running this same command before making the change, since it may have shifted from 447 as other work has landed since Phase 3 — compare before/after, not against a hardcoded historical number).

- [ ] **Step 3: Do not commit.**

---

### Task 10: Bucket/cohort setup UI + live preview

**Files:**
- Create: `perfect-pixel-playground-project/src/pages/day-list/components/administration/ConfirmationTargetingEditor.tsx`
- Modify: `perfect-pixel-playground-project/src/config/api.ts`

- [ ] **Step 1: Add API config entries for confirmation sequences**

In `src/config/api.ts`, add to the `DENTALLY` object, right after `DAYLIST_AUTOMATION_TEST`:

```ts
    CONFIRMATION_SEQUENCES: getApiUrl('/dentally/confirmation-sequences/'),
    CONFIRMATION_SEQUENCE_DETAIL: (id: number | string) => getApiUrl(`/dentally/confirmation-sequences/${id}/`),
    CONFIRMATION_SEQUENCE_PREVIEW_COUNT: (id: number | string) => getApiUrl(`/dentally/confirmation-sequences/${id}/preview-count/`),
```

- [ ] **Step 2: Build the setup editor component**

Create `src/pages/day-list/components/administration/ConfirmationTargetingEditor.tsx`:

```tsx
import { useEffect, useMemo, useState } from 'react';
import { Plus, Trash2 } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Checkbox } from '@/components/ui/checkbox';
import { useFetchWithAuth } from '@/lib/helpers';
import { API_ENDPOINTS } from '@/config/api';
import {
  COHORT_BUCKETS,
  ConfirmationSubCohort,
  ConfirmationTargeting,
  ConfirmationTargetingPreview,
} from '../../types';

const GROUP_LABELS: Record<string, string> = {
  risk: 'Risk',
  booking_age: 'Booking age',
  compound: 'Duration + booking age',
  balance: 'Balance',
};

function BucketChecklist({
  selected,
  onChange,
}: {
  selected: string[];
  onChange: (keys: string[]) => void;
}) {
  const grouped = useMemo(() => {
    const byGroup: Record<string, typeof COHORT_BUCKETS> = {};
    for (const bucket of COHORT_BUCKETS) {
      (byGroup[bucket.group] ??= []).push(bucket);
    }
    return byGroup;
  }, []);

  const toggle = (key: string) => {
    onChange(selected.includes(key) ? selected.filter((k) => k !== key) : [...selected, key]);
  };

  return (
    <div className="space-y-3">
      {Object.entries(grouped).map(([group, buckets]) => (
        <div key={group}>
          <div className="text-xs font-medium text-gray-500 mb-1">{GROUP_LABELS[group] ?? group}</div>
          <div className="space-y-1">
            {buckets.map((bucket) => (
              <label key={bucket.key} className="flex items-center gap-2 text-sm cursor-pointer">
                <Checkbox checked={selected.includes(bucket.key)} onCheckedChange={() => toggle(bucket.key)} />
                {bucket.label}
              </label>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
}

/** Setup UI for a confirmation sequence's cohort/bucket targeting (FR4b-d):
 *  base audience checklist, sub-cohort builder (own buckets + opt-in +
 *  priority), and a live preview-count panel. */
export function ConfirmationTargetingEditor({ sequenceId }: { sequenceId: number }) {
  const fetchWithAuth = useFetchWithAuth();
  const [targeting, setTargeting] = useState<ConfirmationTargeting>({
    base_audience: { bucket_keys: [] },
    sub_cohorts: [],
  });
  const [preview, setPreview] = useState<ConfirmationTargetingPreview | null>(null);
  const [previewLoading, setPreviewLoading] = useState(false);

  useEffect(() => {
    let alive = true;
    setPreviewLoading(true);
    (async () => {
      const res = await fetchWithAuth(API_ENDPOINTS.DENTALLY.CONFIRMATION_SEQUENCE_PREVIEW_COUNT(sequenceId), {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ targeting }),
      });
      if (!alive) return;
      if (res.ok) setPreview(await res.json());
      setPreviewLoading(false);
    })();
    return () => {
      alive = false;
    };
  }, [targeting, sequenceId, fetchWithAuth]);

  const addSubCohort = () => {
    const nextPriority = targeting.sub_cohorts.length
      ? Math.max(...targeting.sub_cohorts.map((sc) => sc.priority)) + 1
      : 1;
    const newSubCohort: ConfirmationSubCohort = {
      name: `Sub-cohort ${nextPriority}`,
      priority: nextPriority,
      bucket_keys: [],
      confirm_dental_opt_in: false,
      template_override_id: null,
    };
    setTargeting((t) => ({ ...t, sub_cohorts: [...t.sub_cohorts, newSubCohort] }));
  };

  const updateSubCohort = (index: number, patch: Partial<ConfirmationSubCohort>) => {
    setTargeting((t) => ({
      ...t,
      sub_cohorts: t.sub_cohorts.map((sc, i) => (i === index ? { ...sc, ...patch } : sc)),
    }));
  };

  const removeSubCohort = (index: number) => {
    setTargeting((t) => ({ ...t, sub_cohorts: t.sub_cohorts.filter((_, i) => i !== index) }));
  };

  return (
    <div className="space-y-6">
      <div>
        <h4 className="text-sm font-semibold mb-2">Base audience</h4>
        <BucketChecklist
          selected={targeting.base_audience.bucket_keys}
          onChange={(keys) => setTargeting((t) => ({ ...t, base_audience: { bucket_keys: keys } }))}
        />
      </div>

      <div>
        <div className="flex items-center justify-between mb-2">
          <h4 className="text-sm font-semibold">Sub-cohorts</h4>
          <Button variant="outline" size="sm" onClick={addSubCohort}>
            <Plus className="h-3.5 w-3.5 mr-1" /> Add sub-cohort
          </Button>
        </div>
        {targeting.sub_cohorts.map((subCohort, index) => (
          <div key={index} className="border border-gray-200 rounded-lg p-3 mb-2 space-y-2">
            <div className="flex items-center gap-2">
              <Input
                value={subCohort.name}
                onChange={(e) => updateSubCohort(index, { name: e.target.value })}
                className="flex-1"
              />
              <Input
                type="number"
                value={subCohort.priority}
                onChange={(e) => updateSubCohort(index, { priority: Number(e.target.value) })}
                className="w-20"
                title="Priority (lower = higher precedence)"
              />
              <Button variant="ghost" size="sm" onClick={() => removeSubCohort(index)}>
                <Trash2 className="h-3.5 w-3.5" />
              </Button>
            </div>
            <BucketChecklist
              selected={subCohort.bucket_keys}
              onChange={(keys) => updateSubCohort(index, { bucket_keys: keys })}
            />
            <label className="flex items-center gap-2 text-sm cursor-pointer">
              <Checkbox
                checked={subCohort.confirm_dental_opt_in}
                onCheckedChange={(checked) => updateSubCohort(index, { confirm_dental_opt_in: Boolean(checked) })}
              />
              confirm.dental opt-in treatment
            </label>
          </div>
        ))}
      </div>

      <div className="border border-gray-200 rounded-lg p-3 bg-gray-50">
        <h4 className="text-sm font-semibold mb-2">Preview</h4>
        {previewLoading || !preview ? (
          <div className="text-xs text-gray-400">Calculating…</div>
        ) : (
          <div className="grid grid-cols-2 gap-x-4 gap-y-1 text-sm">
            <div>Base audience: <span className="font-medium">{preview.base_audience_count}</span></div>
            <div>Final send count: <span className="font-medium">{preview.final_send_count}</span></div>
            <div>Sendable SMS: <span className="font-medium">{preview.sendable_sms}</span></div>
            <div>Sendable email: <span className="font-medium">{preview.sendable_email}</span></div>
            <div>Already confirmed: <span className="font-medium">{preview.already_confirmed}</span></div>
            <div>Already cancelled: <span className="font-medium">{preview.already_cancelled}</span></div>
            <div>Missing contact: <span className="font-medium">{preview.missing_contact}</span></div>
            <div>Excluded: <span className="font-medium">{preview.excluded}</span></div>
            {preview.sub_cohorts.map((sc) => (
              <div key={sc.name}>{sc.name}: <span className="font-medium">{sc.count}</span></div>
            ))}
          </div>
        )}
      </div>
    </div>
  );
}
```

(Check `src/components/ui/` for the exact existing `Checkbox`/`Input`/`Button` component APIs before assuming the props used above — e.g. `onCheckedChange` vs `onChange` for `Checkbox` — and adjust to match whatever this codebase's actual shared UI primitives expect, since these are typically shadcn/ui-style components with a specific prop contract.)

- [ ] **Step 3: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit --project tsconfig.app.json --ignoreDeprecations "5.0" 2>&1 | grep -c "error TS"
```

Expected: no new errors vs. the baseline measured in Task 9.

- [ ] **Step 4: Do not commit.**

---

### Task 11: Opt-in indicator badge (FR12e)

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/serializers.py`
- Modify: `perfect-pixel-playground-project/src/pages/day-list/components/AppointmentCard.tsx`
- Modify: `perfect-pixel-playground-project/src/pages/day-list/types.ts`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

`get_confirmation_enrollment` (added in Phase 3, `dentallyIntegration/serializers.py:694-704`) does not currently expose `sub_cohort_name` in its returned dict — only `sequence_name`/`status`/`next_due_at`/`stopped_reason`/`last_sent_step`. This task adds it.

- [ ] **Step 1: Write the failing backend test**

Append to `dentallyIntegration/tests.py`:

```python
class DayListAppointmentSerializerSubCohortFieldTests(TestCase):
    """confirmation_enrollment includes sub_cohort_name so the frontend can
    show an opt-in-treatment badge (FR12e) without a second API call."""

    def test_confirmation_enrollment_includes_sub_cohort_name(self):
        practice = Practice.objects.create(name="Sub Cohort Serializer Dental")
        sequence = ConfirmationSequence.objects.create(
            practice=practice, name="Test sequence", status="active"
        )
        appointment = DentallyAppointment.objects.create(
            practice=practice,
            dentally_id=50200,
            dentally_patient_id=1,
            patient_name="Sub Cohort Serializer Test Patient",
            state="pending",
            duration=30,
        )
        now = timezone.now()
        ConfirmationSequenceEnrollment.objects.create(
            practice=practice,
            sequence=sequence,
            appointment=appointment,
            enrolled_at=now,
            enrolled_appointment_start=now,
            next_due_at=now,
            sub_cohort_name="VIP opt-in",
        )
        serializer = DayListAppointmentSerializer(appointment)
        self.assertEqual(
            serializer.data["confirmation_enrollment"]["sub_cohort_name"], "VIP opt-in"
        )
```

(Match whatever exact `DayListAppointmentSerializer` import/instantiation pattern the existing `DayListAppointmentSerializerConfirmationFieldsTests` class from Phase 3 already uses in this same file — copy its setup exactly.)

- [ ] **Step 2: Run test to verify it fails**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.DayListAppointmentSerializerSubCohortFieldTests --keepdb -v 2
```

Expected: FAIL — `KeyError: 'sub_cohort_name'`.

- [ ] **Step 3: Add the field to the serializer**

In `dentallyIntegration/serializers.py`, `get_confirmation_enrollment` (lines 694-704), add `sub_cohort_name` to the returned dict:

```python
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
            "sub_cohort_name": enrollment.sub_cohort_name,
        }
```

- [ ] **Step 4: Run test to verify it passes**

```bash
python manage.py test dentallyIntegration.tests.DayListAppointmentSerializerSubCohortFieldTests --keepdb -v 2
```

Expected: PASS.

- [ ] **Step 5: Add `sub_cohort_name` to the frontend `ConfirmationEnrollment` type**

In `perfect-pixel-playground-project/src/pages/day-list/types.ts`, add to the `ConfirmationEnrollment` interface (defined in Phase 3), right after `last_sent_step`:

```ts
  sub_cohort_name: string;
```

- [ ] **Step 6: Add the badge**

In `AppointmentCard.tsx`, near the existing `CONFIRMATION_STATUS_STYLE`/`statusCornerEntry` logic (Phase 3), add:

```tsx
  const isOptInTreatment = Boolean(appointment.confirmation_enrollment?.sub_cohort_name);
```

Render a small secondary badge (not replacing the main confirmation-status corner slot — additive, shown alongside it) wherever makes sense given the card's existing layout — e.g. immediately below the corner slot:

```tsx
              {isOptInTreatment && (
                <span
                  className="mt-1 flex items-center justify-center rounded px-1.5 py-0.5 text-[10px] font-medium bg-purple-50 text-purple-600"
                  title="Sent via confirm.dental opt-in treatment"
                >
                  Opt-in
                </span>
              )}
```

- [ ] **Step 3: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit --project tsconfig.app.json --ignoreDeprecations "5.0" 2>&1 | grep -c "error TS"
```

Expected: no new errors vs. baseline.

- [ ] **Step 4: Do not commit.**

---

### Task 12: `DaylistActivityPanel`

**Files:**
- Create: `perfect-pixel-playground-project/src/components/patients/patient-panel/DaylistActivityPanel.tsx`

- [ ] **Step 1: Build the panel, modeled directly on `RecallActivityPanel.tsx`**

Create `src/components/patients/patient-panel/DaylistActivityPanel.tsx`:

```tsx
import { useEffect, useState } from 'react';
import { Loader2 } from 'lucide-react';
import { useFetchWithAuth } from '@/lib/helpers';
import { API_ENDPOINTS } from '@/config/api';
import { ActivityFeed, ActivityItem, ActivityType } from './PatientActivityFeed';

interface DaylistCount {
  total_contact_attempts: number;
  total_messages: number;
  by_channel: { email: number; sms: number };
}
interface DaylistEvent {
  channel: 'email' | 'sms';
  timestamp: string;
  status: string;
  source_type: string;
  preview: string;
}

const SOURCE_LABEL: Record<string, string> = {
  daylist_automation: 'Daylist reminder',
  confirmation_automation: 'Confirmation reminder',
};

const CHANNEL_TITLE: Record<DaylistEvent['channel'], string> = {
  email: 'Daylist email',
  sms: 'Daylist SMS',
};

const toActivityItem = (e: DaylistEvent, i: number): ActivityItem => {
  const src = SOURCE_LABEL[e.source_type] || '';
  const title = src ? `${CHANNEL_TITLE[e.channel]} · ${src}` : CHANNEL_TITLE[e.channel];
  return {
    id: `daylist-${i}-${e.timestamp}`,
    type: e.channel as ActivityType,
    title,
    timestamp: e.timestamp,
    createdAt: e.timestamp,
    direction: 'outgoing',
    user: { initials: '—', name: '' },
    preview: (e.preview || '').trim() || undefined,
  };
};

/** Daylist-reminder + confirmation-automation activity for a patient — the
 *  same style as RecallActivityPanel, since both message types were moved
 *  out of the main Inbox (Task 7) and need somewhere to still be seen. */
export function DaylistActivityPanel({
  contactId,
  dentallyPatientId,
  refreshKey,
  patientName,
}: {
  contactId?: number | null;
  dentallyPatientId?: number | null;
  refreshKey?: number;
  patientName?: string;
}) {
  const fetchWithAuth = useFetchWithAuth();
  const [count, setCount] = useState<DaylistCount | null>(null);
  const [events, setEvents] = useState<DaylistEvent[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const scope = dentallyPatientId
      ? `dentally_patient_id=${dentallyPatientId}`
      : contactId
        ? `contact_id=${contactId}`
        : '';
    if (!scope) {
      setLoading(false);
      return;
    }
    let alive = true;
    setLoading(true);
    (async () => {
      try {
        const [cRes, eRes] = await Promise.all([
          fetchWithAuth(API_ENDPOINTS.DENTALLY.DAYLIST_REPORTING(`mode=count&months=12&${scope}`)),
          fetchWithAuth(API_ENDPOINTS.DENTALLY.DAYLIST_REPORTING(`mode=events&months=12&${scope}`)),
        ]);
        if (!alive) return;
        if (cRes.ok) setCount(await cRes.json());
        if (eRes.ok) {
          const d = await eRes.json();
          setEvents((d.results ?? []) as DaylistEvent[]);
        }
      } finally {
        if (alive) setLoading(false);
      }
    })();
    return () => {
      alive = false;
    };
  }, [contactId, dentallyPatientId, refreshKey, fetchWithAuth]);

  if (loading) {
    return (
      <div className="flex items-center justify-center py-4">
        <Loader2 className="h-4 w-4 animate-spin text-purple-400" />
        <span className="ml-2 text-xs text-gray-400">Loading daylist activity…</span>
      </div>
    );
  }

  const bc = count?.by_channel;
  return (
    <div className="animate-patient-panel-activity-slide-in">
      <div className="flex gap-2 mb-3 flex-wrap">
        <span className="inline-flex items-center gap-1 rounded-md bg-gray-50 border border-gray-200 px-2 py-1 text-xs text-gray-600">
          <span className="font-medium">{count?.total_messages ?? 0}</span> message
          {(count?.total_messages ?? 0) !== 1 ? 's' : ''}
        </span>
        {!!bc?.email && (
          <span className="inline-flex items-center gap-1 rounded-md bg-[#ede9fc] border border-[#ccc5f5] px-2 py-1 text-xs text-[#5b4fcf]">
            <span className="font-medium">{bc.email}</span> email{bc.email !== 1 ? 's' : ''}
          </span>
        )}
        {!!bc?.sms && (
          <span className="inline-flex items-center gap-1 rounded-md bg-[#ede9fc] border border-[#ccc5f5] px-2 py-1 text-xs text-[#5b4fcf]">
            <span className="font-medium">{bc.sms}</span> SMS
          </span>
        )}
      </div>

      {events.length > 0 ? (
        <ActivityFeed activities={events.map(toActivityItem)} patientName={patientName ?? ''} />
      ) : (
        <div className="text-center py-4 text-gray-400 text-sm">No daylist activity in the last 12 months</div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Wire it into the patient panel**

Find wherever `RecallActivityPanel` is rendered (likely `PatientPanelSheet.tsx` or `PatientPanelAccordion.tsx` — search for `<RecallActivityPanel` to find the exact render site) and add `<DaylistActivityPanel>` alongside it with the same `contactId`/`dentallyPatientId`/`refreshKey`/`patientName` props, in whatever tab/section structure that render site already uses (e.g. if `RecallActivityPanel` lives inside a labeled "Recall" tab/section, add a parallel "Daylist" tab/section for this new panel — match the exact existing pattern rather than inventing a new layout convention).

- [ ] **Step 3: Verify with TypeScript**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit --project tsconfig.app.json --ignoreDeprecations "5.0" 2>&1 | grep -c "error TS"
```

Expected: no new errors vs. baseline.

- [ ] **Step 4: Do not commit.**

---

### Task 13: Send-time template override (FR8a)

**Files:**
- Modify: `TreatmentPath/dentallyIntegration/confirmation_automation.py`
- Test: `TreatmentPath/dentallyIntegration/tests.py`

Task 6 tags each enrollment with `sub_cohort_name` but never actually USES the matching sub-cohort's `template_override_id` at send time — the step's own `template_id` is always used regardless. This task closes that gap: FR8a's "opt-in link or call-to-action" requirement is satisfied by staff authoring a distinct template (selected via `template_override_id`) that contains whatever opt-in copy/CTA they want, using the same existing `{confirmation_link}` placeholder `render_message` already supports (confirmed: `get_confirmation_link`/`confirm_utils.py` generate one plain link with no separate "opt-in variant" URL format — the differentiation is entirely in which template gets used, not a different link). `confirm_dental_opt_in`'s "must record that the appointment was sent through the opt-in treatment" half of FR8a is already satisfied by `sub_cohort_name` being non-blank on the enrollment (Task 6) — nothing further needed for the recording half.

- [ ] **Step 1: Write the failing test**

Append to `dentallyIntegration/tests.py`:

```python
class SendTimeTemplateOverrideTests(TestCase):
    """When an enrollment's sub_cohort_name matches a sub-cohort with a
    template_override_id, the send uses THAT template, not the step's
    default template_id."""

    def setUp(self):
        self.practice = Practice.objects.create(
            name="Template Override Dental", twilio_phone_number="+447700900000"
        )
        self.default_template = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="Default confirmation SMS",
            content="Please confirm: {confirmation_link}",
            template_type="appointment_confirmation",
            is_active=True,
        )
        self.opt_in_template = SMSMessageTemplate.objects.create(
            practice=self.practice,
            name="VIP opt-in confirmation SMS",
            content="VIP opt-in: {confirmation_link}",
            template_type="appointment_confirmation",
            is_active=True,
        )
        self.sequence = ConfirmationSequence.objects.create(
            practice=self.practice,
            name="Sequence with override",
            status="active",
            days_ahead=7,
            steps=[
                {
                    "channel": "sms",
                    "template_id": self.default_template.id,
                    "offset_days": 0,
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
            dentally_id=50100,
            dentally_patient_id=1,
            patient_name="Override Test Patient",
            patient_phone="+447700900111",
            state="pending",
            duration=30,
            start_time=timezone.now() + timedelta(hours=1),
        )

    @override_settings(TWILIO_ACCOUNT_SID="ACtest", TWILIO_AUTH_TOKEN="tok")
    def test_vip_enrollment_uses_template_override(self):
        now = timezone.now()
        enrollment = ConfirmationSequenceEnrollment.objects.create(
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

        message = SMSMessage.objects.filter(phone_number="+447700900111").latest("created_at")
        self.assertIn("VIP opt-in", message.content)

    @override_settings(TWILIO_ACCOUNT_SID="ACtest", TWILIO_AUTH_TOKEN="tok")
    def test_non_matching_enrollment_uses_default_template(self):
        now = timezone.now()
        enrollment = ConfirmationSequenceEnrollment.objects.create(
            practice=self.practice,
            sequence=self.sequence,
            appointment=self.appointment,
            enrolled_at=now,
            enrolled_appointment_start=self.appointment.start_time,
            next_due_at=now,
            sub_cohort_name="",
        )
        with patch("twilio.rest.Client") as MockTwilio:
            MockTwilio.return_value.messages.create.return_value.sid = "SMfake"
            advance_confirmation_enrollments(now=now)

        message = SMSMessage.objects.filter(phone_number="+447700900111").latest("created_at")
        self.assertIn("Please confirm", message.content)
```

(Match whatever exact `SMSMessageTemplate`/`EmailMessageTemplate` field names are already used by the existing `ConfirmationAutomationSendingTests` class in this same file, since that class already exercises `advance_confirmation_enrollments` with templates — copy its exact setUp pattern rather than guessing field names.)

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate && cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath && python manage.py test dentallyIntegration.tests.SendTimeTemplateOverrideTests --keepdb -v 2
```

Expected: FAIL — `test_vip_enrollment_uses_template_override` fails (message content contains "Please confirm" instead of "VIP opt-in") since the override isn't consulted yet.

- [ ] **Step 3: Wire the override into `advance_confirmation_enrollments`**

In `dentallyIntegration/confirmation_automation.py`, find the line (around line 435):

```python
        subject, body = _resolve_confirmation_template(practice=sequence.practice, channel=channel, template_id=step.get("template_id"))
```

Replace it with:

```python
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
```

(Confirmed: the loop variable at this point in `advance_confirmation_enrollments` is named `enrollment` — `for enrollment in due:`, per the function's current code — so `enrollment.sub_cohort_name` above is correct as written.)

- [ ] **Step 4: Run tests to verify they pass**

```bash
python manage.py test dentallyIntegration.tests.SendTimeTemplateOverrideTests --keepdb -v 2
```

Expected: PASS (2 tests).

- [ ] **Step 5: Run the full confirmation-automation suite to confirm no regressions**

```bash
python manage.py test dentallyIntegration.tests.ConfirmationAutomationSendingTests dentallyIntegration.tests.AdvanceConfirmationEnrollmentsTests dentallyIntegration.tests.EnrollEligibleAppointmentsTests dentallyIntegration.tests.EnrollEligibleAppointmentsTargetingTests dentallyIntegration.tests.ProcessConfirmationEnrollmentsTests --keepdb -v 2
```

Expected: PASS, all green — enrollments with a blank `sub_cohort_name` (the pre-existing, non-targeted case) must be completely unaffected, since `effective_template_id` falls back to `step.get("template_id")` exactly as before.

- [ ] **Step 6: Do not commit.**

---

## Summary of spec coverage

- Bucket registry (7 named buckets, group-based OR/AND) → Task 2.
- Risk tier sync (Go + Django) → Task 1.
- `targeting` config + `sub_cohort_name` → Task 3.
- `ConfirmationSequenceSerializer` validation → Task 4.
- `ConfirmationSequenceViewSet` CRUD → Task 5.
- Enrollment integration + preview-count logic → Task 6.
- Inbox allow-list flip → Task 7.
- `DaylistReportingViewSet` + API config → Task 8.
- Frontend types → Task 9.
- Bucket/cohort setup UI + live preview (FR4b-d) → Task 10.
- Opt-in indicator badge (FR12e) → Task 11.
- `DaylistActivityPanel` (replacement visibility for Inbox-hidden messages) → Task 12.
- Send-time template override + opt-in CTA (FR8a) → Task 13.

## Out of scope (per the approved design doc)

- A fully generic rule builder.
- Historical `risk_tier` backfill for past appointments.
- Changing Recall's own Inbox-hiding behavior.

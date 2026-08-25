# Three-State Marketing Consent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop the Dentally sync collapsing an unrecorded marketing preference (`null`) into a hard "no", so Pathway can tell "patient declined" apart from "we never asked".

**Architecture:** Dentally's patient object carries `marketing` as a three-state value: `true` (opted in), `false` (opted out), `null` (never recorded). Six write paths in Pathway currently coerce that to a two-state boolean, defaulting `null` to `false`. This plan makes the two storage columns nullable, updates every write path to preserve `None`, backfills existing rows from the verbatim Dentally payload already stored in `meta_data`, and teaches the consent ledger to treat `None` as "no signal" rather than a refusal.

**Tech Stack:** Django 5 / PostgreSQL (shared DB), Go 1.26 (GORM raw `Updates` maps), Celery.

## Global Constraints

- **Shared table.** `TreatmentPlan_patient` is written by BOTH Django and the Go sync service. Any schema change must be applied before the Go change ships, and the Go service must tolerate the column being nullable.
- **Semantics, exactly:** `true` → consented; `false` → declined; `NULL` → never recorded.
- **POLICY DECISION (user, 2026-08-05): `NULL` IS SENDABLE.** Only an explicit `false` blocks a
  send. Rationale given: "they haven't said yes or no". This is a deliberate override of PRD
  FR10 ("Send only when marketing consent, email permission, valid address, and no opt-out/block
  exist"), recorded here so it reads as a decision and not as drift. The supporting legal basis
  is the soft opt-in (existing patients, own similar services, opt-out in every message) rather
  than consent. Effect: campaign 2 goes from 12 recipients to ~794.
  An explicit `false`, a bounce/complaint block, an invalid address, and under-18 all still block.
- **Run backend tests with `--keepdb`. NEVER pass `--noinput`** — it destroys the persistent test DB.
- **Do not run `migrate` against the dev or prod database.** Hand-author the migration and validate it via `manage.py test <app> --keepdb`, which applies pending migrations to the test DB.
- **Never commit or push.** The user handles all VCS operations. Ignore any "Commit" step wording from the parent skill — stop after tests pass.
- Virtualenv: `source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate`
- manage.py: `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/manage.py`

## Measured starting state (practice 16, verified against the live Dentally API)

| Dentally `marketing` | patients | current `Patient.marketing_consent` |
|---|---|---|
| `null` | 10,809 | `False` (wrong — should be NULL) |
| absent | 283 | `False` (wrong — should be NULL) |
| `false` | 94 | `False` (correct) |
| `true` | 74 | `True` (correct) |
| `true` | 4 | `False` (**drifted** — will be corrected by the backfill) |

---

## File Structure

| File | Responsibility |
|---|---|
| `TreatmentPlan/models.py` | Make both consent columns nullable |
| `TreatmentPlan/migrations/0122_marketing_consent_nullable.py` | Schema change + drop the `DEFAULT false` |
| `dentallyIntegration/tasks.py` | 3 Python sync write paths preserve `None` |
| `dentallyIntegration/management/commands/backfill_dentally_failed_imports.py` | 2 more write paths |
| `EmailServiceGo/internal/dentally/migration/service.go` | Go sync writes NULL, not false |
| `TreatmentPlan/management/commands/backfill_marketing_consent.py` | **New** — restore truth from `meta_data` |
| `marketingBroadcast/consent_ledger.py` | `None` = no signal; drop `use_email` as a consent signal |
| `TreatmentPlan/serializers/patient.py` | Expose the nullable value to the API |

---

### Task 1: Make both consent columns nullable

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/models.py:695` and `:1253`
- Create: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/migrations/0122_marketing_consent_nullable.py`
- Create test: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_marketing_consent_nullable.py`

**Interfaces:**
- Produces: `Patient.marketing_consent` and `PersonChannel.marketing_consent` accept and return `None`.

- [ ] **Step 1: Write the failing test**

`TreatmentPlan` uses a `tests/` package, not a single `tests.py`. Create `TreatmentPlan/tests/test_marketing_consent_nullable.py`:

```python
from django.test import TestCase

from TreatmentPlan.models import Patient
from UserAuthentication.models import Practice


class MarketingConsentNullableTests(TestCase):
    """NULL means "never recorded" — distinct from False, which means "declined"."""

    def setUp(self):
        # `practice` is Patient's only required field — everything else defaults.
        self.practice = Practice.objects.create(name="Consent Test Practice")

    def test_patient_marketing_consent_accepts_null(self):
        p = Patient.objects.create(practice=self.practice, marketing_consent=None)
        p.refresh_from_db()
        self.assertIsNone(p.marketing_consent)

    def test_patient_marketing_consent_defaults_to_null_not_false(self):
        p = Patient.objects.create(practice=self.practice)
        p.refresh_from_db()
        self.assertIsNone(
            p.marketing_consent,
            "A new patient has not declined marketing — they have not been asked.",
        )
```

- [ ] **Step 2: Run it and confirm it fails**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPlan.tests.test_marketing_consent_nullable --keepdb -v 2
```

Expected: FAIL — `IntegrityError: null value in column "marketing_consent" violates not-null constraint`.

- [ ] **Step 3: Change both model fields**

`TreatmentPlan/models.py:1253` (Patient):

```python
    # Three-state, mirroring Dentally's own `marketing` field: True = opted in,
    # False = declined, None = never recorded. None is NOT consent — it means
    # the patient can still be asked, which False does not.
    marketing_consent = models.BooleanField(null=True, blank=True, default=None)
```

`TreatmentPlan/models.py:695` (PersonChannel) — same change, dropping `db_default`:

```python
    marketing_consent = models.BooleanField(null=True, blank=True, default=None)
```

- [ ] **Step 4: Hand-author the migration**

Create `TreatmentPlan/migrations/0122_marketing_consent_nullable.py`:

```python
from django.db import migrations, models


class Migration(migrations.Migration):
    """Marketing consent becomes three-state.

    Migration 0102 set a database-level DEFAULT false on
    TreatmentPlan_patient.marketing_consent so an INSERT omitting the column
    could not violate NOT NULL. With the column nullable that default now does
    active harm: any writer omitting the column would record a refusal the
    patient never gave. It is dropped here, not kept.
    """

    dependencies = [
        ("TreatmentPlan", "0121_contactchannel_channel_kind_value_idx"),
    ]

    operations = [
        migrations.AlterField(
            model_name="patient",
            name="marketing_consent",
            field=models.BooleanField(blank=True, default=None, null=True),
        ),
        migrations.AlterField(
            model_name="personchannel",
            name="marketing_consent",
            field=models.BooleanField(blank=True, default=None, null=True),
        ),
        migrations.RunSQL(
            sql='ALTER TABLE "TreatmentPlan_patient" ALTER COLUMN "marketing_consent" DROP DEFAULT;',
            reverse_sql='ALTER TABLE "TreatmentPlan_patient" ALTER COLUMN "marketing_consent" SET DEFAULT false;',
        ),
        migrations.RunSQL(
            sql='ALTER TABLE "TreatmentPlan_personchannel" ALTER COLUMN "marketing_consent" DROP DEFAULT;',
            reverse_sql='ALTER TABLE "TreatmentPlan_personchannel" ALTER COLUMN "marketing_consent" SET DEFAULT false;',
        ),
    ]
```

- [ ] **Step 5: Run the test and confirm it passes**

```bash
python manage.py test TreatmentPlan.tests.test_marketing_consent_nullable --keepdb -v 2
```

Expected: PASS (2 tests). The migration is proven valid by having been applied to the test DB.

- [ ] **Step 6: Confirm no unexpected model drift**

```bash
python manage.py makemigrations --check --dry-run TreatmentPlan
```

Expected: "No changes detected". If it wants a new migration, the hand-authored field definition does not match the model — fix the migration, not the model.

---

### Task 2: Preserve `None` in the three Python sync paths

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tasks.py:727`, `:1323`, `:1420`
- Test: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests.py`

**Interfaces:**
- Consumes: nullable columns from Task 1.
- Produces: `marketing_consent_from_dentally(raw) -> bool | None`, importable from `dentallyIntegration.tasks`.

- [ ] **Step 1: Write the failing test**

Append to `dentallyIntegration/tests.py`:

```python
class MarketingConsentMappingTests(SimpleTestCase):
    """Dentally sends three states; all three must survive the sync."""

    def test_maps_all_three_dentally_states(self):
        from dentallyIntegration.tasks import marketing_consent_from_dentally

        self.assertIs(marketing_consent_from_dentally(True), True)
        self.assertIs(marketing_consent_from_dentally(False), False)
        self.assertIsNone(marketing_consent_from_dentally(None))

    def test_absent_key_is_unknown_not_declined(self):
        from dentallyIntegration.tasks import marketing_consent_from_dentally

        self.assertIsNone(marketing_consent_from_dentally({}.get("marketing")))

    def test_string_booleans_are_honoured(self):
        # Dentally has been observed serialising booleans as quoted strings.
        from dentallyIntegration.tasks import marketing_consent_from_dentally

        self.assertIs(marketing_consent_from_dentally("true"), True)
        self.assertIs(marketing_consent_from_dentally("false"), False)
```

- [ ] **Step 2: Run it and confirm it fails**

```bash
python manage.py test dentallyIntegration.tests.MarketingConsentMappingTests --keepdb -v 2
```

Expected: FAIL — `ImportError: cannot import name 'marketing_consent_from_dentally'`.

- [ ] **Step 3: Add the helper**

Near the top of `dentallyIntegration/tasks.py`, after the imports:

```python
def marketing_consent_from_dentally(raw):
    """Dentally's `marketing` field as Pathway's three-state consent value.

    `None` means the practice has never recorded a preference — which is NOT
    the same as a refusal. Collapsing it to False (what every write path used
    to do) made ~97% of one practice's patients look like they had opted out,
    and destroyed the only signal showing who could still be asked.

    Booleans sometimes arrive as quoted strings, as they do for `gender`.
    """
    if isinstance(raw, bool):
        return raw
    if isinstance(raw, str):
        lowered = raw.strip().lower()
        if lowered == "true":
            return True
        if lowered == "false":
            return False
    return None
```

- [ ] **Step 4: Replace all three call sites**

`tasks.py:727` and `tasks.py:1323` — replace:

```python
"marketing_consent": bool(dentally_patient.get("marketing") or False),
```

with:

```python
"marketing_consent": marketing_consent_from_dentally(dentally_patient.get("marketing")),
```

`tasks.py:1420` — replace:

```python
("marketing_consent",             "marketing",                  lambda v: bool(v) if v is not None else False),
```

with:

```python
("marketing_consent",             "marketing",                  marketing_consent_from_dentally),
```

- [ ] **Step 5: Run the tests and confirm they pass**

```bash
python manage.py test dentallyIntegration.tests.MarketingConsentMappingTests --keepdb -v 2
```

Expected: PASS (3 tests).

- [ ] **Step 6: Confirm no remaining collapse in this file**

```bash
grep -n 'marketing' dentallyIntegration/tasks.py | grep -i 'bool(\|or False'
```

Expected: no output.

---

### Task 3: Preserve `None` in the failed-import backfill

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/management/commands/backfill_dentally_failed_imports.py:161`, `:188`

**Interfaces:**
- Consumes: `marketing_consent_from_dentally` from Task 2.

- [ ] **Step 1: Replace both call sites**

Add the import at the top of the file:

```python
from dentallyIntegration.tasks import marketing_consent_from_dentally
```

Replace both occurrences of:

```python
"marketing_consent": bool(d.get("marketing") or False),
```

with:

```python
"marketing_consent": marketing_consent_from_dentally(d.get("marketing")),
```

- [ ] **Step 2: Verify both are gone**

```bash
grep -n "marketing" dentallyIntegration/management/commands/backfill_dentally_failed_imports.py
```

Expected: only the two `marketing_consent_from_dentally(...)` lines and the import.

- [ ] **Step 3: Confirm the module still imports**

```bash
python manage.py help backfill_dentally_failed_imports
```

Expected: the command's help text, no traceback.

---

### Task 4: Go sync writes NULL instead of false

**Files:**
- Modify: `EmailServiceGo/internal/dentally/migration/service.go:987`, `:1050`
- Test: `EmailServiceGo/internal/dentally/migration/marketing_consent_test.go` (create)

**Interfaces:**
- Produces: `marketingConsent(v any) *bool` in package `migration`.

- [ ] **Step 1: Write the failing test**

Create `EmailServiceGo/internal/dentally/migration/marketing_consent_test.go`:

```go
package migration

import "testing"

func TestMarketingConsentPreservesUnknown(t *testing.T) {
	cases := []struct {
		name string
		in   any
		want *bool
	}{
		{"opted in", true, boolPtr(true)},
		{"declined", false, boolPtr(false)},
		{"never recorded", nil, nil},
		{"key absent", any(nil), nil},
		{"string true", "true", boolPtr(true)},
		{"string false", "false", boolPtr(false)},
	}
	for _, c := range cases {
		got := marketingConsent(c.in)
		if (got == nil) != (c.want == nil) {
			t.Fatalf("%s: got %v want %v", c.name, got, c.want)
		}
		if got != nil && *got != *c.want {
			t.Fatalf("%s: got %v want %v", c.name, *got, *c.want)
		}
	}
}

func boolPtr(b bool) *bool { return &b }
```

- [ ] **Step 2: Run it and confirm it fails**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
go test ./internal/dentally/migration/ -run TestMarketingConsentPreservesUnknown
```

Expected: FAIL — `undefined: marketingConsent`.

- [ ] **Step 3: Add the helper**

Next to `toCleanBool` in `service.go`:

```go
// marketingConsent maps Dentally's `marketing` field to a three-state value.
//
// A nil result writes SQL NULL, meaning "no preference recorded". toCleanBool
// cannot express this: its fallback turned every unrecorded preference into an
// explicit refusal, which is both wrong and unrecoverable once written.
func marketingConsent(v any) *bool {
	switch t := v.(type) {
	case bool:
		return &t
	case string:
		switch strings.ToLower(strings.TrimSpace(t)) {
		case "true":
			b := true
			return &b
		case "false":
			b := false
			return &b
		}
	}
	return nil
}
```

`strings` is already imported in this file.

- [ ] **Step 4: Replace both write sites**

At `service.go:987` and `service.go:1050`, replace:

```go
"marketing_consent":              toCleanBool(dentally["marketing"], false),
```

with:

```go
"marketing_consent":              marketingConsent(dentally["marketing"]),
```

- [ ] **Step 5: Run the test and the package suite**

```bash
go test ./internal/dentally/migration/ -run TestMarketingConsentPreservesUnknown -v
go build ./... && go test ./internal/dentally/...
```

Expected: the new test PASSes and the package still builds and passes.

- [ ] **Step 6: Confirm the old collapse is gone**

```bash
grep -n 'toCleanBool(dentally\["marketing"\]' internal/dentally/migration/service.go
```

Expected: no output.

---

### Task 5A: Add the `unknown` state to the consent ledger

**Files:**
- Modify: `marketingBroadcast/models.py` — `STATE_CHOICES`, `MarketingConsent.state`, and both audit state columns
- Create: `marketingBroadcast/migrations/0026_consent_unknown_state.py`

> `state` is currently `max_length=3` — sized for "on"/"off". `"unknown"` is 7 characters and
> will not fit; the audit table mirrors the same choices on two more columns.

- [ ] **Step 1: Widen and extend the choices**

```python
    STATE_CHOICES = [
        ("on", "On"),
        ("off", "Off"),
        # Never recorded — neither agreed nor refused. Distinct from "off",
        # which is an explicit refusal. Sendable under the practice's soft
        # opt-in basis; see the policy decision at the top of this plan.
        ("unknown", "Not recorded"),
    ]
```

Change `state` to `models.CharField(max_length=16, choices=STATE_CHOICES, default="unknown")`,
and both `MarketingConsentAudit.previous_state` and `.new_state` to `max_length=16`.

- [ ] **Step 2: Hand-author `marketingBroadcast/migrations/0026_consent_unknown_state.py`**

```python
from django.db import migrations, models

CHOICES = [("on", "On"), ("off", "Off"), ("unknown", "Not recorded")]


class Migration(migrations.Migration):
    dependencies = [("marketingBroadcast", "0025_add_campaign_sender_fields")]

    operations = [
        migrations.AlterField(
            model_name="marketingconsent",
            name="state",
            field=models.CharField(choices=CHOICES, default="unknown", max_length=16),
        ),
        migrations.AlterField(
            model_name="marketingconsentaudit",
            name="previous_state",
            field=models.CharField(blank=True, choices=CHOICES, max_length=16, null=True),
        ),
        migrations.AlterField(
            model_name="marketingconsentaudit",
            name="new_state",
            field=models.CharField(choices=CHOICES, max_length=16),
        ),
    ]
```

- [ ] **Step 3: Prove it applies with no drift**

```bash
python manage.py makemigrations --check --dry-run marketingBroadcast
python manage.py test marketingBroadcast.tests.SeedConsentFromLegacySourcesTests --keepdb
```

Expected: "No changes detected", existing seeder tests still pass.

---

### Task 5B: `unknown` is a real state, and it is sendable

**Files:**
- Modify: `marketingBroadcast/consent_ledger.py` — `get_or_create` defaults, `is_marketing_eligible`, `seed_consent_from_legacy_sources`
- Test: `marketingBroadcast/tests.py`

- [ ] **Step 1: Write the failing tests**

```python
class ConsentSeedThreeStateTests(SeedConsentFromLegacySourcesTests):
    def setUp(self):
        super().setUp()
        self.patient = Patient.objects.create(
            practice=self.practice, person=self.person, meta_data={"id": "12345"}
        )
        self.recall_patient = RecallPatient.objects.create(
            practice=self.practice, dentally_patient_id=12345,
            use_email=True, first_name="Jane", last_name="Doe",
        )

    def test_never_recorded_seeds_unknown_not_off(self):
        self.patient.marketing_consent = None
        self.patient.save(update_fields=["marketing_consent"])
        consent = seed_consent_from_legacy_sources(self.practice, self.person)
        self.assertEqual(consent.state, "unknown")

    def test_unknown_is_eligible_to_send(self):
        # Policy decision: never-asked patients are sendable under soft opt-in.
        self.patient.marketing_consent = None
        self.patient.save(update_fields=["marketing_consent"])
        seed_consent_from_legacy_sources(self.practice, self.person)
        eligible, reason = is_marketing_eligible(self.practice, self.person)
        self.assertTrue(eligible)
        self.assertIsNone(reason)

    def test_explicit_refusal_is_not_eligible(self):
        self.patient.marketing_consent = False
        self.patient.save(update_fields=["marketing_consent"])
        seed_consent_from_legacy_sources(self.practice, self.person)
        eligible, reason = is_marketing_eligible(self.practice, self.person)
        self.assertFalse(eligible)
        self.assertEqual(reason, "unsubscribed")

    def test_missing_row_is_treated_as_unknown_and_eligible(self):
        eligible, _ = is_marketing_eligible(self.practice, self.person)
        self.assertTrue(eligible)

    def test_use_email_alone_does_not_record_an_optin(self):
        # use_email is appointment/recall permission, not marketing consent.
        self.patient.marketing_consent = None
        self.patient.save(update_fields=["marketing_consent"])
        consent = seed_consent_from_legacy_sources(self.practice, self.person)
        self.assertEqual(consent.state, "unknown")

    def test_a_block_still_wins_over_unknown(self):
        auto_block_address(self.practice, self.person, "hard_bounce")
        eligible, reason = is_marketing_eligible(self.practice, self.person)
        self.assertFalse(eligible)
        self.assertEqual(reason, "hard_bounce")
```

Import `is_marketing_eligible` and `auto_block_address` alongside the existing imports.

- [ ] **Step 2: Run and confirm failures**

```bash
python manage.py test marketingBroadcast.tests.ConsentSeedThreeStateTests --keepdb -v 2
```

- [ ] **Step 3: Update `is_marketing_eligible`**

```python
def is_marketing_eligible(practice, person):
    """Used by the segment engine and campaign engine to read consent state — neither
    queries MarketingConsent directly.

    Only an explicit refusal blocks. A patient who has never been asked is
    eligible under the practice's soft opt-in basis (policy decision, 2026-08-05);
    an absent ledger row means exactly that and is treated the same as "unknown".
    """
    try:
        consent = MarketingConsent.objects.get(practice=practice, person=person)
    except MarketingConsent.DoesNotExist:
        return True, None
    if consent.state == "off":
        return False, consent.block_reason or "unsubscribed"
    return True, None
```

- [ ] **Step 4: Update the seeder's signal collection**

Replace the signal-gathering block in `seed_consent_from_legacy_sources` with:

```python
    from TreatmentPlan.models import PersonChannel

    # Only fields recording a MARKETING preference count. Dentally's `use_email`
    # was read here previously, but it is permission for appointment and recall
    # mail — counting it would record an opt-in the patient never gave and erase
    # the unasked/declined distinction this change exists to create.
    signals = [
        value
        for value in PersonChannel.objects.filter(person=person).values_list(
            "marketing_consent", flat=True
        )
        if value is not None
    ]
    patient = person.patients.filter(practice=practice).first()
    if patient is not None and patient.marketing_consent is not None:
        signals.append(patient.marketing_consent)

    if False in signals:
        seeded_state = "off"      # an explicit refusal always wins
    elif True in signals:
        seeded_state = "on"
    else:
        seeded_state = "unknown"  # never recorded — not a refusal
```

Delete the now-unused `RecallPatient` import and the `resolve_dentally_patient_id`/`use_email`
block from this function. Also change the `get_or_create` defaults from
`{"state": "off", "source": "system_seed"}` to `{"state": "unknown", "source": "system_seed"}`.

- [ ] **Step 5: Run the tests**

```bash
python manage.py test marketingBroadcast.tests.ConsentSeedThreeStateTests --keepdb -v 2
python manage.py test marketingBroadcast --keepdb
```

Expected: new tests pass, no NEW failures. Record pre-existing failures before starting.

---

### Task 6: Backfill existing rows from the stored Dentally payload

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/management/commands/backfill_marketing_consent.py`

**Interfaces:**
- Consumes: `marketing_consent_from_dentally` (Task 2), nullable columns (Task 1).

> `meta_data` holds Dentally's reply verbatim and is the trustworthy copy — it is already
> correct for all 11,264 rows in practice 16, including the 4 where the typed column drifted.

- [ ] **Step 1: Write the command**

```python
from django.core.management.base import BaseCommand

from dentallyIntegration.tasks import marketing_consent_from_dentally
from TreatmentPlan.models import Patient


class Command(BaseCommand):
    help = (
        "Recompute Patient.marketing_consent from the verbatim Dentally payload in "
        "meta_data, restoring NULL where no preference was ever recorded."
    )

    def add_arguments(self, parser):
        parser.add_argument("--practice-id", type=int, required=True)
        parser.add_argument(
            "--commit",
            action="store_true",
            help="Write changes. Without it the command only reports what it would do.",
        )

    def handle(self, *args, **options):
        changed, unchanged, updates = 0, 0, []
        qs = Patient.objects.filter(practice_id=options["practice_id"]).only(
            "id", "meta_data", "marketing_consent"
        )
        for patient in qs.iterator():
            meta = patient.meta_data if isinstance(patient.meta_data, dict) else {}
            correct = marketing_consent_from_dentally(meta.get("marketing"))
            if correct is patient.marketing_consent:
                unchanged += 1
                continue
            patient.marketing_consent = correct
            updates.append(patient)
            changed += 1

        if options["commit"] and updates:
            Patient.objects.bulk_update(updates, ["marketing_consent"], batch_size=500)

        verb = "Updated" if options["commit"] else "Would update"
        self.stdout.write(
            self.style.SUCCESS(f"{verb} {changed} patient(s); {unchanged} already correct.")
        )
```

- [ ] **Step 2: Dry-run it against practice 16**

```bash
python manage.py backfill_marketing_consent --practice-id 16
```

Expected: `Would update 11096 patient(s); 168 already correct.`
(10,809 nulls + 283 absent + 4 drifted = 11,096; 94 false + 74 true = 168.)

If the numbers differ materially, STOP and investigate before writing anything.

- [ ] **Step 3: Apply it**

```bash
python manage.py backfill_marketing_consent --practice-id 16 --commit
```

- [ ] **Step 4: Verify the outcome**

```bash
python manage.py shell -c "
from TreatmentPlan.models import Patient
from collections import Counter
c = Counter(Patient.objects.filter(practice_id=16).values_list('marketing_consent', flat=True))
print(dict(c))
"
```

Expected: `{None: 11092, False: 94, True: 78}` — note **78** True, up from 74, because the 4 drifted rows are corrected.

---

### Task 7: Expose the third state through the API

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/serializers/patient.py:149`

- [ ] **Step 1: Confirm the field serialises `None` rather than coercing it**

```bash
python manage.py shell -c "
from TreatmentPlan.serializers.patient import PatientSerializer
f = PatientSerializer().fields['marketing_consent']
print('allow_null =', f.allow_null, '| required =', f.required)
"
```

Expected: `allow_null = True`. A `ModelSerializer` derives this from the model field, so Task 1 should have handled it. If it prints `False`, declare it explicitly on the serializer:

```python
    marketing_consent = serializers.BooleanField(allow_null=True, required=False)
```

- [ ] **Step 2: Run the patient serializer tests**

```bash
python manage.py test TreatmentPlan --keepdb
```

Expected: no new failures.

---

## Deliberately out of scope

Each of these is real and worth doing, but none belongs in this change:

1. **The staff "Marketing consent" toggle does not reach `MarketingConsent`.** It writes `Patient.marketing_consent` only, so ticking it still leaves the campaign reporting "no consent record". Needs the patient PATCH path to call `set_marketing_consent`.
2. **The preferences page is unreachable.** `MarketingPreferenceToken` is only minted when a marketing email is sent, so nobody without consent can ever be given the opt-in link.
3. **The Dentally write-back is likely a no-op.** `tasks.py:37` sends `{"marketing_consent": ...}`, but the live API field is `marketing`, and the payload is not wrapped in the `{"patient": {...}}` envelope Dentally's responses use. Unverified — testing it requires a write.
4. **The frontend toggle renders `null` as "off"** (`p.marketing_consent ?? false`). It should show a distinct "Not recorded" state after this lands.
5. **`PersonChannel.marketing_consent` has no writer at all.** Made nullable here for consistency; still nothing populates it from Dentally.

## What this change does and does not do

**Does:** makes "never asked" (11,092 patients) distinguishable from "declined" (94), corrects 4
drifted rows, stops six write paths manufacturing refusals, and — per the policy decision above —
makes the never-asked group sendable.

**Effect on campaign 2:** 826 audience → ~794 sendable (826 minus 146 under-18 and 45 without a
valid email, which overlap). Previously 0.

**Still blocked:** explicit `false` (94 practice-wide), bounce/complaint/invalid-address blocks,
under-18, and missing or invalid email.

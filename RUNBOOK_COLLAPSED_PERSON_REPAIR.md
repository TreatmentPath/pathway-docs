# Runbook — repairing collapsed Person identities

**Status:** rehearsed on DEV 2026-09-04. NOT yet run on production.

---

## 1. What went wrong

A `Person` is meant to be one human. **817 Person rows each hold records for 3 or
more different humans — 2,728 people in total.**

The worst measured case, PROD Person 152744, named *"John Geary"*:

| patient | name | dob | phone | email |
|---|---|---|---|---|
| 135585 | Micaela Geary | 1984-01-04 | 7368557184 | gearymicaela@gmail.com |
| 131893 | John Geary | 1985-02-25 | 7887705702 | gearymicaela@gmail.com |
| 133908 | Beau Geary | 2012-04-26 | 7368557184 | gearymicaela@gmail.com |
| 133907 | Hallie Geary | 2014-01-10 | 7368557184 | gearymicaela@gmail.com |
| 133906 | Johnny Geary | 2018-06-02 | 7368557184 | gearymicaela@gmail.com |
| 133905 | Gigi Geary | 2021-02-15 | — | gearymicaela@gmail.com |

Two parents and four children, six distinct birth dates, one identity — named
after the youngest child.

## 2. How it happened

**Measured, not inferred.** Collapse date taken from when each Person's records
were *attached*, not when the Person row was created (an earlier reading of
`Person.created_at` gave the wrong answer and was corrected):

| Day the collapse completed | Persons |
|---|---|
| **2026-07-27** | **812** |
| 2026-02-11 | 1 |
| 2026-09-02 | 1 (test fixture — "Alex/Jamie/Rudy Testfamily") |

2026-07-27 was a single bulk import: **24,555 patients in one day** (the three
days either side created 7, 1, 14 and 14).

The resolution in force that day matched on **channel only**. From the Go
source's own description of what it replaced:

> *"SELECT person_id FROM TreatmentPlan_personchannel WHERE channel_id IN (...)
> LIMIT 1 — no name check, no DOB check, no merged_into filter and no ORDER BY.
> Any patient sharing a family phone was absorbed into whichever Person the
> database happened to hand back."*

A family sharing one email therefore became one Person.

## 3. Is the cause fixed? YES — and proven, not assumed

Both sides were fixed *after* that import:

| | Commit | Date |
|---|---|---|
| Go | `4e6b27a` — adds `dobConflict` + exact-name match | 2026-08-17 |
| Django | `e1d172da` — "Stop merging family members into shared Person identities" | 2026-08-18 |

**The proof is a test, not the commit message.**
`TreatmentPlan/tests/test_family_collapse_cannot_recur.py` replays the exact
six-Geary shape through current code and asserts six Persons in one Household.

Mutation-verified: deleting the name/DOB check from `Person.resolve` makes **5
of its 6 tests fail**, including the collapse signature. So the tests are wired
to the behaviour, not passing vacuously.

Also confirmed: no genuine collapse has occurred since the fix. The only
post-fix case is test data with no contact details at all.

**Nothing further is needed to stop new collapses. What remains is repairing the
historical damage.**

## 4. The repair

`TreatmentPlan/management/commands/split_collapsed_persons.py`

### Safety contract

It **only creates Persons and re-points record FKs. It never deletes** a Person,
record, channel or dependent row. The worst case for any row is that it stays
where it is.

- **Anchor** — the name group matching the Person's own name (or the largest
  group) keeps the original Person id, so every person-level attachment stays
  valid: `Note`, `NoteHistory`, `Activity`, `ActivityLog`,
  `MarketingPatientProfile`, `ContactMergeDismissal`, `merged_into` chains.
- **Other name groups** go through `Person.resolve` — the app's own rule — so a
  human who already exists rejoins their real identity instead of gaining a
  duplicate.
- **Blank / "unknown" named records never split.** There is no name to split
  them on and a nameless new Person would just be a fresh orphan.
- All resulting Persons share one **Household**, matching what `Person.resolve`
  does for a shared channel today.

### Known limits — deliberate, and neither loses data

1. **Person-level rows stay on the anchor.** Notes and activity carry no
   per-human marker, so moving them would be guesswork. They are counted in the
   report for human review. On the Alex Cooper case that was 5 Notes,
   7 NoteHistory, 16 Activities, 1 ActivityLog, 1 MarketingProfile.
2. **Stale channel links are not removed.** Splitting the records does not
   unlink the wrong contact channel — after repair, Jacqui Rogan's number is
   registered to *both* her and Alex Cooper. Needs a separate decision.

## 5. Rehearsed result (DEV, Person 120461 "Alex Cooper")

Before: one Person held Alex Cooper (patient), Jacqui Rogan (intake),
Alison Sims (intake) and three "Unknown" call-agent intakes.

```
intake  3067 Jacqui Rogan  -> person 118602   HER EXISTING Person (reunited with patient 30022)
intake  3076 Alison Sims   -> person 121068   new
intake  3064/3074/3075     -> stayed on Alex  unnamed, never split
patient 32120 Alex Cooper  -> stayed on Alex  anchor
```

Verified afterwards:

- Row counts unchanged — 60,506 patients / 3,966 intakes / 619 nurtures /
  143,599 person-channels. **Nothing deleted.**
- **Zero orphaned `person` FKs** across Note, NoteHistory, Activity,
  MarketingPatientProfile, Patient.
- Households assigned to all three resulting Persons.

Two bugs the dry run caught in the command itself before any write:
its first version moved blank-named records into a nameless Person (violating
its own contract), and minted a duplicate Person for Jacqui instead of reusing
hers. Both fixed before `--apply` was ever used.

## 6. Production procedure

```bash
# 0. baseline — read-only, identity of every affected row
python manage.py contact_identity_audit --snapshot prod-before.csv

# 1. review the plan — writes nothing
python manage.py split_collapsed_persons --csv prod-plan.csv

# 2. ONE person first, and inspect it by hand
python manage.py split_collapsed_persons --person <id> --apply

# 3. a small batch, then verify
python manage.py split_collapsed_persons --limit 20 --apply

# 4. the rest
python manage.py split_collapsed_persons --apply

# 5. prove it — FIXED/PERSISTING/NEW per defect, plus a control group
python manage.py contact_identity_audit --snapshot prod-after.csv
python manage.py contact_identity_audit --compare prod-before.csv prod-after.csv
```

**Acceptance:** `D_COLLAPSED_PERSON` falls, **`NEW` is zero on every defect**, and
no control row breaks. A better total with any NEW rows is not a pass.

### Prerequisites

- Deploy the identity fixes first (canonical keys, the contact-correction
  re-resolve gate) or the repair competes with a live defect.
- Take the standard DB backup — the repair is additive, but the anchor choice is
  not reversible without one.
- Run outside sync windows: a Dentally import mid-repair re-resolves records
  while they are being moved.

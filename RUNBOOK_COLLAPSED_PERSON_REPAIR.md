# Runbook — repairing collapsed Person identities

**Status:** DEV fully repaired and verified 2026-09-05 — D 816 -> **0**, D2
1,491 -> **0**, run to a proven fixed point, all 1,500 moved records audited
individually. NOT yet run on production.

Read in order: §5b (D), §5c (how D2 was found), §5d (D2 repaired + a real defect
in the repair command that the per-record audit caught).

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

---

## 5b. FULL DEV RUN — 2026-09-05 (816 Persons, verified)

The complete repair, run end to end on DEV with before/after proof.

### Result

| Defect | Before | After | Fixed | NEW |
|---|---|---|---|---|
| **D — collapsed Persons** | **816** | **1** | **816** | 1 (explained below) |
| C — wrong-person channels | 3,534 | 2,715 | 892 | 73 (explained below) |
| **CONTROL — healthy rows** | 2,000 | 2,000 | — | **0 broken** |

**1,909 name-groups split, 1,911 records re-pointed.**

### The tool's own message is WRONG — read this before trusting it

It prints `created 1909 Persons`. **Only 146 Person rows were actually created.**

The counter (`made` in the command) increments once per non-anchor name group,
whether `Person.resolve` minted a Person or returned an existing one. It is a
*name-groups split* count wearing the word "created". Do not report it as a
creation count on prod.

**Reconciled exactly**, by re-reading where every MOVE row in the plan actually
landed and checking each target Person's `created_at`:

| | |
|---|---|
| MOVE rows in plan | 1,911 |
| distinct target Persons | 1,909 |
| **created by the repair** | **146** |
| **rejoined an EXISTING identity** | **1,763** |

146 + 1,763 = 1,909. Nothing is missing.

**1,763 of 1,909 is the good outcome, not a shortfall.** The typical collapsed
record is a duplicate of a patient who already exists properly elsewhere, so
`Person.resolve` hands back their real Person and the record rejoins its own
identity instead of gaining a second one.

### Every moved record landed on the RIGHT person — checked, all 1,911

For each of the 1,911 moved records, the record's name group was compared against
the canonical name of the Person it ended up on, using the app's own
`canonical_name_key`:

**0 mismatches.** No record was filed under a different human's name — including
all 1,763 reuses, which are the risky half (a wrong reuse would silently merge two
people, the exact failure this repair exists to undo).

Re-run this on prod before accepting the result:

```python
# for every MOVE row in the plan CSV, read the record's CURRENT person_id and
# assert canonical_name_key(record name group) == canonical_name_key(target Person)
```

### Provenance — the verification queried the SAME database the repair wrote to

Verifying against the wrong DB has bitten this project before (Django once
connected to `:5432` while `.env` pointed at a sim container on `:5434`). So the
check was pinned explicitly:

```
DJANGO TARGET: treatmentpath_db @ localhost:5432 user mannie
SERVER SAYS  : ('treatmentpath_db', '127.0.0.1/32', 5432, 'mannie')
```

Cross-checks that agree: patients 60,514 / intakes 3,966 / persons 83,648 — the
same figures the post-repair integrity pass reported. And `persons created today
= 146`, a **whole-table count derived independently** of the per-target join that
also produced 146.

**No silent skips.** All 1,911 MOVE rows resolve to a live record: 1,903 Patient
+ 8 Intake, 0 missing, 0 with a NULL person, and **0 still sitting on the original
collapsed Person**. So the 0-mismatch name result covers every moved row, not a
subset that happened to load.

On prod, run these three before believing any verification output: confirm
`current_database()`, confirm the plan's record ids all resolve, and confirm no
MOVE row is still on its anchor.

### Integrity — measured, not assumed

- Row counts unchanged: 60,514 patients / 3,966 intakes / 619 nurtures
- **Zero orphaned `person` FKs** across Patient, Intake, Nurture, Note, NoteHistory,
  MarketingPatientProfile
- **Zero** split Persons left without a household
- Nothing deleted

### The 1 "new" collapsed Person is NOT new damage

Person 78391 "Jay Welch" appeared in the after-snapshot holding three humans. Traced:

- **Before:** Person 141721 was collapsed, holding *alana lindsay | amelia lindsay |
  jason stevens | jay welch*
- The repair split it; `Person.resolve` correctly returned **78391**, an existing Person
  genuinely named Jay Welch, and reunited him with his own identity
- 78391 was **already** holding Lewis and Tommy Stevens — **two** names, below the
  3-name detection threshold, therefore invisible
- Adding Jay made it three, so it surfaced

The repair did the right thing. It exposed contamination the detector could not see.

**Consequence for the detector:** a 3-name threshold cannot see 2-name collapses.
Expect a small number of these to surface on prod for the same reason. They are
pre-existing, not caused by the repair — verify each by tracing which before-snapshot
Person the record came from, as above.

### The 73 "new" C rows are the same shape

Sampled channels 191554 / 191600 / 191609: families (Elbi/Tarelli, the Truman
brothers, the Memduhoglus) sharing one phone, whose members sit in **different
households** (12762 vs 21729, 12969 vs 21064, 13735 vs 21066). None of those Persons
were created by the repair — all date from 2026-07-12.

Splitting the family made the shared number separately-owned, so the C rule's
"outside the household" test began firing. That is **pre-existing household
fragmentation becoming visible**, not new harm — and C is a known-noisy measure (see
the findings doc: ~97% of C is orphaned Person rows, not misattribution).

### What this run did NOT do

- **Person-level rows stay on the anchor:** 12 Note, 20 NoteHistory, 55 Activity,
  8 ActivityLog, **815 MarketingPatientProfile**. They carry no per-human marker, so
  moving them would be guesswork. The MarketingPatientProfile count is ~1 per collapsed
  Person and is the largest item needing a human decision.
- **Stale channel links are not removed.** Splitting records does not unlink a wrong
  contact channel.

### Verification commands used

```bash
python manage.py contact_identity_audit --snapshot before.csv
python manage.py split_collapsed_persons --csv plan.csv          # writes nothing
python manage.py split_collapsed_persons --limit 20 --apply      # small batch first
# integrity check (orphan FKs + row counts) — see below
python manage.py split_collapsed_persons --apply
python manage.py contact_identity_audit --snapshot after.csv
python manage.py contact_identity_audit --compare before.csv after.csv
```

Orphan-FK check (run after every batch):

```sql
select 'ORPHAN Patient.person', count(*) from "TreatmentPlan_patient" x
 where x.person_id is not null
   and not exists (select 1 from "TreatmentPlan_person" p where p.id=x.person_id);
-- repeat for Intake, Nurture, Note, NoteHistory, marketingBroadcast_marketingpatientprofile
```

**Note:** the plan CSV is generated by a dry run. If you apply in batches afterwards, the
plan no longer matches what was executed — do not use it as the record of what happened.
Use the before/after snapshots for that.

### Acceptance on prod

D falls to ~0; **CONTROL breakage must be 0**; any NEW row must be traced to a
pre-existing condition (as both categories above were) before it is accepted. A better
total with unexplained NEW rows is not a pass.

## 5c. The verification found MORE damage — read before quoting "816 -> 1"

Checking each of the 1,911 moved records individually (rather than trusting the
totals) produced two results: the repair is clean, and the measurement was not.

### The repair itself is clean — 1,911 records checked one by one

Each moved record was checked against evidence the resolver never used: practice,
target not merged away, DOB agreement with the records already on the target, and
whether the record's own canonicalised email/phone actually links it there.

| | |
|---|---|
| contact evidence links record to its target | **1,904** |
| record has no contact data to check | 3 |
| landed on a Person already holding someone else | **4** |
| **misfiled by the repair** | **0** |

All 4 were traced individually: in every case the target was created 2026-07-12,
the other human was **already sitting on it**, and was not moved by the repair.
The record itself went to a Person genuinely bearing its own name. The repair
joined pre-existing contamination; it did not create any.

**A name check alone would have missed this and did.** Comparing each record's
name to its target Person's name returned 0 problems across all 1,911 — because
`Person.resolve` matches on exact canonical name, so agreement is guaranteed by
construction. It is a tautology, not a test. Only DOB and contact evidence,
which the resolver does not consult, can actually falsify a placement.

### The detector was under-counting by ~1,491

`D_COLLAPSED_PERSON` fired on **3 or more** names per Person. That threshold was
a guess to avoid flagging nicknames, and it hid every two-name collapse.

Post-repair, 1,575 Persons still hold two or more distinct names. Splitting them
on name count alone would be **destructive** — some are one human recorded twice:

| Person | Records | Truth |
|---|---|---|
| 74764 | Tricia / **Patricia** De-Heer, no DOB conflict | one human, nickname |
| 77351 | Alyssia Price / Alyssia (blank surname) | one human |
| 85103 | Maureen Reeder **1934** + John Reeder **1933** | **two humans** |
| 100217 | Tatiana **1977** + Sofia Balakina **2014** | **two humans** |
| 100277 | Sumeet **1990** + Veer Nathani **2023** | **two humans** |

**A conflicting date of birth is the evidence; a count of names is not.**

Two changes, each verified against the table above:

1. `identity_defects.py` now loads DOB per name and emits a new
   **`D2_COLLAPSED_PAIR`** — exactly two names with conflicting DOBs.
   **1,491 on DEV.** D was deliberately left alone so existing before/after
   snapshots stay comparable.
2. `split_collapsed_persons --min-names 2` now **refuses** a two-name group
   unless the DOBs conflict. Without this, running `--min-names 2` on prod would
   split Tricia from Patricia and manufacture the duplicates this work exists to
   remove. Default (`--min-names 3`) is unchanged — still selects 1 Person.

Cross-check: the repair selects 1,492 at `--min-names 2`; the detector reports
1 (D) + 1,491 (D2) = 1,492. Two independent implementations agree.

### So the true remaining position on DEV

| | |
|---|---|
| D (3+ names) | 816 -> **1** |
| **D2 (2 names, conflicting DOB)** | **1,491 — not yet repaired** |

`--min-names 2 --apply` has **not** been run. Repairing D2 is a separate,
larger job needing its own before/after proof; expect prod to hold a
proportionally similar population.

## 5d. D2 REPAIRED — 1,491 -> 0 on DEV, 2026-09-05

### Is there a live code cause? NO — and that was checked, not assumed

Every D2 pair was dated by when its records were **attached** (not when the Person
row was created):

| Month the pair completed | Persons |
|---|---|
| **2026-07** | **1,489** |
| 2026-02 | 2 |
| **after the 2026-08-18 Django fix** | **0** |

Same origin as D: the July bulk import under channel-only matching.

**Absence in a static snapshot is not proof**, so the exact D2 shape is now pinned
as a shared-fixture scenario, `two_humans_one_channel_conflicting_dob` — two
humans, one shared phone, conflicting DOBs, must resolve apart. It runs in
**both** languages off `person_resolution_fixtures.json`.

Mutation-proven: revert `Person.resolve` to the historical channel-only match and
it fails with *"['elder', 'younger'] must each be a DIFFERENT Person, got
[32855, 32855]"* — Maureen Reeder (1934) and John Reeder (1933) welded into one.

### The repair, and why it needed several passes

`split_collapsed_persons --min-names 2 --apply`. At two names the DOB conflict is
required (see `_collapsed`), so nicknames like *Tricia*/*Patricia De-Heer* are
never split.

| Pass | Splits | New Persons | Rejoined existing | Links released |
|---|---|---|---|---|
| 1 (staged 20, then full) | 1,493 | 1,279 | 214 | — |
| 2 | 7 | 7 | 0 | 2 |
| **3 and 4** | **0** | **0** | **0** | **0** |

Passes 3 and 4 were complete no-ops — **a proven fixed point**, which is the only
honest way to say "finished".

### Final result

| Defect | before | after |
|---|---|---|
| **D2_COLLAPSED_PAIR** | **1,491** | **0** |
| D_COLLAPSED_PERSON | 1 | **0** |
| C_WRONG_PERSON | 13 | 13 |
| CONTROL broken | — | **0** |
| NEW rows | — | **0** |

Integrity: 0 orphaned person FKs (Patient/Intake/Nurture/Note), 0 split Persons
without a household, record counts unchanged (60,514 / 3,966 / 619), ContactChannel
count unchanged.

**Every one of the 1,500 moved records audited individually**: right practice,
target not merged away, no DOB conflict with the records already on the target,
household assigned, and the record's own canonicalised email/phone actually
linking it there. **1,500 OK, 0 failures.**

### Why pass 2 was needed — a pre-existing defect the detector cannot see

The 7 rows that appeared after pass 1 all had one shape, e.g. Person 145916 named
**"Jamie Guy"** which held only **Daniel Guy's** record. One name, so invisible to
any name-count rule. When the repair correctly returned Jamie to the Person
bearing his own name, the pair became visible and pass 2 split it.

**This is a third blind spot: a Person whose own name matches none of its
records.** Expect the same on prod — run the repair to a fixed point rather than
once. The 2 C rows from pass 2 were the same story: pre-existing stale links that
only became visible when household membership changed.

### A REAL defect in the repair itself, found by the per-record audit

The command built the new Person by splitting the **flattened** name key at the
first space. A patient recorded as `Chon Long` / `Ao` became a Person named
`chon` / `long ao`.

That is not cosmetic. `canonical_name_key` would no longer match the record's own
name, so **that Person could never be resolved to again and the next import would
mint a duplicate** — the repair would have seeded the very defect it exists to
remove.

- Fixed: `_collapsed` now carries the original first/last per name group
  (`self._orig`) and `_split_off` uses it instead of re-splitting the key.
- Repaired in place: **42** Persons had a name key that matched none of their
  records; all corrected. **1,387** more had their original capitalisation
  restored (the key matched, but the display name was lower-cased).
- Verified after: **0** remaining name-key mismatches.

**Run this check on prod after the repair** — it is invisible to the defect audit,
because a mangled name breaks nothing measurable until the next import:

```python
# every Person created by the repair must share a canonical name key with at
# least one of its own records
canonical_name_key(person.first_name, person.last_name) in
    {canonical_name_key(r.first_name, r.last_name) for r in person's records}
```

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

# 4b. THE TWO-NAME COLLAPSES (defect D2). At min-names 2 a DOB conflict is
#     required, so nicknames are never split. Re-run BOTH commands until a pass
#     reports 0 — splitting one pair can expose another (see §5d).
python manage.py split_collapsed_persons --min-names 2 --apply
python manage.py repair_stale_person_channels --apply
#     ... repeat until both report 0

# 5. prove it — FIXED/PERSISTING/NEW per defect, plus a control group
python manage.py contact_identity_audit --snapshot prod-after.csv
python manage.py contact_identity_audit --compare prod-before.csv prod-after.csv
```

**Acceptance:** `D_COLLAPSED_PERSON` falls, **`NEW` is zero on every defect**, and
no control row breaks. A better total with any NEW rows is not a pass.

**Totals are not acceptance.** Before signing off, audit every moved record
individually (§5c): practice match, target not merged away, no DOB conflict with
the records already on the target, and the record's own email/phone actually
linking it there. Do **not** accept a name-match check — `Person.resolve` matches
on name, so it agrees by construction and cannot fail.

Snapshot `D2_COLLAPSED_PAIR` before and after too, and repair it with
`--min-names 2` (§5d). Run to a **fixed point** — a pass that changes nothing —
rather than once.

Then run the name-key check from §5d. A Person the repair created with a mangled
name breaks nothing the defect audit can measure, and silently guarantees a
duplicate at the next import.

### Prerequisites

- Deploy the identity fixes first (canonical keys, the contact-correction
  re-resolve gate) or the repair competes with a live defect.
- Take the standard DB backup — the repair is additive, but the anchor choice is
  not reversible without one.
- Run outside sync windows: a Dentally import mid-repair re-resolves records
  while they are being moved.

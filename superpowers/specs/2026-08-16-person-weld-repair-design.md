# Person weld: stop the bleeding, repair the damage, fix the hub

**Date:** 2026-08-16
**Status:** design approved in chat, implementation starting
**Scope:** Django backend, EmailServiceGo, perfect-pixel frontend

---

## 1. The defect in one sentence

Several ingest paths collapse *different humans* into a single `Person` row,
when they should place those humans in a shared `Household`.

`Person` = one canonical human. `ContactChannel` = one phone or email.
`Household` = a family group of several distinct Persons. Sharing a phone
number is a household signal, never an identity signal. The paths below treat
it as an identity signal.

## 2. Measured damage (production, read-only, 2026-08-16)

| Metric | Value |
|---|---|
| Welded Persons (>1 distinct name among attached patients) | 2,357 |
| Patients involved | 5,802 |
| Patients sitting on the wrong identity ("displaced") | 3,495 |
| Attributable to the Dentally family bug (single `family_id`) | 1,816 (77%) |
| Other cause (multiple or absent `family_id`) | 541 |
| One-member Households (the weld's fingerprint) | 2,360 |
| Abandoned Person shells (no record points at them) | 25,074 |
| PersonChannel rows on welded Persons | 4,807 |
| Intakes / Nurtures on welded Persons | 13 / 5 |

Practices affected: 27 (1,453), 26 (649), 24 (241), 19 (14).

**Repairability** — for each displaced patient, can we find the original Person
shell the weld abandoned?

| Outcome | Displaced patients |
|---|---|
| Exactly one surviving shell → unambiguous restore | 3,049 |
| No shell → mint a new Person | 352 |
| Multiple shells, DOB disambiguates | 56 |
| Ambiguous even with DOB → needs a human | 38 |

Every displaced patient has a `date_of_birth`, which is what makes shell
matching trustworthy. 98.9% resolves deterministically.

## 3. Part 1 — stop the bleeding

Nothing may merge two Persons, or repoint a record onto another human's Person,
without a human approving it. Six sites violate this.

### 3.1 `dentallyIntegration/tasks.py:169` — the primary cause

```python
# Move the sibling's person into the same household
PatientModel.objects.filter(pk=sibling.pk).update(person=primary_person)
```

The comment says household; the code repoints the patient's Person. Result: all
family members become one Person, their original Person rows are abandoned
(not tombstoned), and the Household created at :151-160 ends up with exactly
one member.

**Fix:** move the sibling's *Person* into the household, leave `patient.person`
alone.

```python
if sibling.person.household_id != household.id:
    sibling.person.household = household
    sibling.person.save(update_fields=["household"])
```

Also drop the channel transfer at :180 — channels belong to their own Person.

This matches what Go already does in `applyDentallyFamilyGrouping`
(`service.go:1140-1215`), which only sets `household_id`. Django is the
divergent side.

### 3.2 `EmailServiceGo/internal/dentally/migration/service.go:1299-1302`

```go
m.DB.Table("TreatmentPlan_personchannel").
    Select("person_id AS id").
    Where("channel_id IN ?", channelIDs).
    Limit(1).Scan(&existingPerson)
```

No name check, no DOB check, no `merged_into IS NULL` filter, no `ORDER BY`.
Any patient sharing a family phone is absorbed into whichever Person the
database happened to return — possibly a dead, merged-away one.

**Fix:** implement `Person.resolve` semantics in Go — candidate Persons on those
channels, excluding `merged_into IS NOT NULL`; reuse only on exact
case-insensitive first+last name match with no DOB conflict (conflict = both
DOBs present and differing); otherwise create a new Person and place it in a
shared Household with the candidates. Deterministic `ORDER BY id`.

### 3.3 `messaging/models.py:388, 424-447`

`channel.person_channels.first()` picks an arbitrary Person on a shared channel,
then that Person is stamped onto every unlinked Patient/Intake/Nurture matching
a set of fuzzy phone variants. On a family phone this attaches the wrong human.

The value filter is already load-bearing (a prior port dropped it and stamped
the Person onto every unlinked record in the practice). The remaining defect is
the arbitrary Person choice.

**Fix:** only auto-link a record whose name matches the resolved Person's name
(same rule as `Person.resolve`). Non-matching records stay unlinked and are
surfaced in the hub rather than guessed at.

### 3.4 `TreatmentPlan/journey/mixins.py:142-150`

```python
# Sync this record onto the patient's Person/Household if they're in a
# family — ensures all records share one Person.
if patient.person.household.is_family and ...:
    self.__class__.objects.filter(pk=self.pk).update(person=patient.person)
```

Being in a family household is precisely when records must **not** share one
Person. Compounded by the match at :95-100 preferring `email__iexact`, which
families share.

**Fix:** delete the block. `Person.resolve` already assigns the correct Person
at record creation.

### 3.5 `dentallyIntegration/serializers.py:1032` — identity mutation from a GET

`get_contact` on the recall list serializer calls `get_or_create_channel` and
`Person.resolve` while rendering a GET, so browsing the recall list creates
Persons, Channels and Households as a side effect. It also omits `dob`, so the
father/son split cannot fire.

**Fix:** read-only lookup. Resolve an existing Person via the channel, return
`None` when absent. Never create on a read path.

### 3.6 `dentallyIntegration/tasks.py:937-945` — false duplicate flag

A patient matched by its own Dentally ID, with byte-identical name, email,
phone and country code, raises a `duplicate_import` issue purely because the
`created_at` string differs. This is the same record seen again — exactly the
case that must not be flagged. Go's importer has no such rule, so the two
importers disagree on the same data.

**Fix:** remove the `created_at`-difference duplicate. A Dentally-ID match with
identical fields is an update or a no-op, never a duplicate.

### 3.7 Rule to preserve

The similarity detection in the Go nightly sweep
(`dataquality_sweep.go:213-224`) is correct and stays as-is: shared channel,
**both** first and last name similar (0.82 Ratcliff/Obershelp), and the pair is
excluded when both DOBs are present and differ. That encodes exactly the
intended policy — flag when similar, stay silent when identical or clearly
distinct.

## 4. Part 2 — the repair command

`TreatmentPlan/management/commands/unweld_persons.py`

**Dry-run by default.** `--commit` to write. `--practice N` to stage.
`--report <csv>` for the ambiguous cases.

### Algorithm

For each welded Person (>1 distinct `(first_name, last_name)` among attached
Patients):

1. **Anchor:** the patient whose name matches the Person's own name keeps that
   Person. If no patient matches, the earliest-created patient anchors.
2. **Household:** ensure the anchor Person has a Household (reuse the existing
   one-member household created by the weld).
3. For every other ("displaced") patient:
   - Find abandoned Person shells in the same practice matching
     `(lower(first_name), lower(last_name))`, with `merged_into IS NULL` and no
     Patient/Intake/Nurture pointing at them.
   - One shell → **restore**: repoint the patient at it.
   - Multiple shells → filter by DOB equality. Exactly one → restore.
   - Still ambiguous → **skip, report to CSV, raise a hub issue.** Never guess.
   - No shell → **mint** a new Person from the patient's own name and DOB.
   - Place the restored/minted Person in the anchor's Household.
   - Move that patient's own channels onto its own Person.
4. Write a `ContactMergeLog` per weld so the operation is auditable and
   reversible.

### Safety properties

- Idempotent: a repaired weld no longer matches the welded-Person predicate.
- Never deletes a Person.
- Never merges — it only splits.
- Practice-scoped, batched, wrapped in `transaction.atomic()`.
- Verify gate before commit: assert no patient was left with `person_id` NULL
  and that patient count is unchanged.

### Ordering constraint (hard)

**Part 1 must be deployed before Part 2 runs.** Otherwise the next Dentally
sync re-welds everything the repair just fixed.

## 5. Part 3 — the hub

`Contacts → Data Quality` is the only working merge surface. Defects to fix:

| Defect | Location |
|---|---|
| Every duplicate card titled "Unknown"; search can never match one | `DataQualityIssuesScreen.tsx:73-85` reads `detail.first_name`; sweep writes `detail.contacts` |
| Merge resolves only 2 of N, but marks the whole issue resolved | `DataQualityIssuesScreen.tsx:497-502`, `dataQuality/views.py:97-100` |
| Silent no-op when both rows are the same person | `DataQualityIssuesScreen.tsx:499-501` |
| Confirm says "This can be undone"; no undo exists, `merge_log_id` discarded | `DataQualityIssuesScreen.tsx:407`, `useDataQualityIssues.ts:100` |
| "Not Duplicates" is one click, no confirm, permanent, and channel-keyed — suppresses future genuine people on that number | `DataQualityIssuesScreen.tsx:561`, `dataquality_sweep.go:259-269` |
| A pair matching on both phone and email produces two separate issues | `dataquality_sweep.go:196-269` |
| Failed resolves are silent | `useDataQualityIssues.ts:94-97` |

## 6. Rollout

1. Land Part 1 (Django + Go + tests). Deploy.
2. Restore a fresh prod dump locally; rehearse Part 2 end-to-end on real
   damaged data; confirm counts and reversibility.
3. Run Part 2 on prod dry-run, per practice, review the CSV.
4. Run Part 2 with `--commit`, per practice, backup taken first.
5. Land Part 3.

## 6a. Rehearsal result (2026-08-16, full prod copy)

Restored the 16:00 prod dump into `treatmentpath_weld_sim` — an exact match
(83,051 persons / 60,414 patients / 15,756 households / 135,406 channels) with
all 2,357 welds present.

`unweld_persons` outcome across every practice:

| | |
|---|---|
| welded Persons examined | 2,357 |
| patients repointed | 3,407 |
| restored from a surviving shell | 3,007 |
| restored via DOB tie-break | 50 |
| new Person minted | 350 |
| **ambiguous, skipped** | **37** |
| welds remaining afterwards | 33 (only the ambiguous) |
| patients lost | 0 |
| new person-less patients | 0 |
| audit log rows written | 2,340 |

Second run repoints 0 — idempotent. The 37 refusals are 34 patients with two
identical-name shells and 3 with three, all in practices 26 and 27; DOB cannot
separate them, so they route to the hub for a human.

Worked example — Dentally family in practice 27, previously ONE Person:
John Geary, Micaela Geary (1984), Beau (2012), Hallie (2014), Johnny (2018),
Gigi (2021). Two adults and four children sharing one identity, one set of
notes, one message thread. Now six Persons in one Household.

## 6b. SUPERSEDED by the 2026-08-16 19:30–19:45 UTC deploy

Both blockers below were written against the 16:00 backup and the pre-deploy
state. A release landed at 19:30–19:45 UTC that resolves both. Verified against
LIVE prod at 22:06 WAT:

* **B1 is already fixed and applied.** The deployed `0140` had independently
  been rewritten to discover referencing tables from `pg_constraint`
  (`FK_REFERENCES_SQL`) — the same conclusion reached here, arrived at
  separately. Applied 19:45:18 UTC. Prod is now 60,143 patients, 0 duplicate
  groups, unique index present, **0 orphaned financial rows**, and the
  patient_accounts counts match the rehearsal exactly (charge 10,730 /
  ledger 15,837 / allocation 6,238 / payment 5,091 / account 1,149) — the
  financial rows were RE-POINTED, not deleted. The local copy of this migration
  was stale and has been replaced with the deployed text, which also documents a
  residual UNIQUE-constraint risk on `patient_accounts_patientaccount`.
* **B2 is resolved.** `dataquality_dataqualityissue` now exists on prod; all 11
  previously-unapplied migrations are applied. The Data Quality hub IS live, so
  the Go nightly sweep has somewhere to write and §5's defects are now
  user-visible rather than theoretical.

**Still true and unchanged: none of the Part 1 fixes are deployed.** Verified by
grepping the running container — every welding line is live at the same line
numbers (`tasks.py:169`, `mixins.py:149`, `messaging/models.py:388`,
`serializers.py:1007/1016`, `tasks.py:944`), and `ContactChannel.find_channel`
does not exist there. The welds are still being created on every Dentally sync.

A SEVENTH site was found during this check: `messaging/models.py` `CallLog.save`
picked an arbitrary Person off a shared channel to name the call, AND read a
`parts` variable bound inside a guard it was outside of — a latent `NameError`
whenever a channel's Person had no display name. Both fixed.

Local working tree confirmed in sync with prod (md5 match on untouched files).

## 6c. The blockers as originally found (historical)

**B1 — migration `0140_dedupe_patients_and_unique_dentally_id` fails on prod
data.** It re-points references off the duplicate patients it deletes, but from
a HAND-WRITTEN list of three tables (`Tasks_task`, `Notes_note`,
`Labs_labcase`) — the ones that happened to hold a reference on the environment
where it was written. Production has **nine**, and **68 of the 70 blocking
references are financial** (charges, payments, ledger entries, allocations).
The migration aborted with an IntegrityError on
`patient_accounts_patientledgerentry`. Every FK is `NO ACTION`, so the failure
mode is a hard abort rather than silent data loss — but the deploy is blocked.
FIXED: the table list is now enumerated from `pg_constraint` instead of
hand-maintained. Re-verified on the prod copy: 60,414 → 60,143 patients
(−271 exactly), zero orphaned financial rows.

**B2 — the Data Quality hub is not deployed.** `dataquality_dataqualityissue`
does not exist on production (`dataQuality/0001_initial` unapplied, along with
10 other migrations). So the merge surface described in §5 — the one path
previously believed to work — is not live. **Production currently has no
working way to resolve a duplicate at all.** This also means the Go nightly
sweep has nowhere to write its findings.

## 7. Open question

The 541 welds not attributable to the Dentally family bug (multiple or absent
`family_id`) are most likely the Go wrong-attach path (§3.2). The same split
logic repairs them. Decision pending: include in the same run, or handle
separately.

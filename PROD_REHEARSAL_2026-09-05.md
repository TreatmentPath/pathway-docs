# Production rehearsal — 2026-09-05

The full repair sequence run end to end against a **restored copy of the
2026-09-04 production backup**, with an untouched copy of the same backup kept
alongside it so every difference could be attributed.

| database | what it is |
|---|---|
| `prod_control` | the production backup, restored and never touched |
| `prod_rehearsal` | the same backup, repaired |

Both verified identical before starting: 60,326 patients / 83,700 persons /
136,676 person-channels / 108,615 contact-channels.

---

## 1. Result

| Defect | before | after |
|---|---|---|
| **A** — own email unregistered | 3,596 | **0** |
| **B** — own phone unregistered | 3,823 | **0** |
| **C** — channel on wrong person | 3,528 | **0** |
| **D** — collapsed, 3+ names | 817 | **0** |
| **D2** — collapsed pair, DOB conflict | 1,489 | **0** |
| CONTROL rows broken | — | **0** |

Sequence (each command run until a pass changed nothing):

```
backfill_missing_person_channels --apply      # A, B
split_collapsed_persons --apply               # D  → 1,910 groups, 118 new Persons
split_collapsed_persons --min-names 2 --apply # D2 → 1,493 groups, 225 new Persons
repair_stale_person_channels --apply          # C  → 1,877 links released, 430 primaries restored
# then both of the last two repeated: pass 1 = 7 splits, passes 2-4 = 0
```

Prod needed the same second pass DEV did (7 splits), for the same reason — see
the collapsed-person runbook §5d.

## 2. What the control copy proved

**Only 4 of 546 tables differ**, and all four are the identity tables the repairs
are meant to touch:

| table | control | rehearsed | delta |
|---|---|---|---|
| TreatmentPlan_contactchannel | 108,615 | 114,886 | +6,271 |
| TreatmentPlan_household | 15,816 | 15,890 | +74 |
| TreatmentPlan_person | 83,700 | 84,050 | +350 |
| TreatmentPlan_personchannel | 136,676 | 143,256 | +6,580 |

**542 tables are untouched.** No appointment, note, activity, message, treatment
plan, billing or recall table changed at all.

Row counts can hide edits, so the content was compared too:

- For Patient, Intake and Nurture, a hash of **every column except `person_id`**
  is **identical** between the two databases. Not one other field was modified.
- `person_id` changed on **3,400 patients and 12 intakes**; **0** lost their
  Person (none set to NULL), **0** gained one that had none.
- **0** Person rows and **0** ContactChannel rows were deleted.
- Person-channel links: 1,873 removed, 8,453 added.

That is the strongest available statement that the repairs do what they claim
and nothing else.

## 3. What the rehearsal FOUND — a bug in the measurement

The first comparison reported **"36 healthy rows BROKE"**. They had not.

`healthy_control()` claimed in its own docstring to be *"keyed on record id, so
it does not shift between runs"*. It was not. It recomputed **the lowest 2,000
currently-healthy rows**, so anything that makes more rows healthy shifts the
window. The A/B backfill made ~5,100 records healthy; 36 low-id intakes entered
the window and pushed 36 others off the end, and the diff called that breakage.

**This is worse than a false alarm.** The same instability can *hide* real
breakage, by pulling fresh healthy rows in to replace rows that genuinely failed.
A control group that moves is not a control group — and this is the check the
whole production acceptance rests on.

**Fixed:** `contact_identity_audit --snapshot AFTER.csv --control-from BEFORE.csv`
re-checks exactly the rows the before-snapshot named. Re-run with it: **2,000
control rows, 0 broken.**

**This never fired on DEV** — A and B were already 0 there, so no new rows became
healthy and the window never moved. Only a rehearsal from an unrepaired
production baseline could surface it.

## 4. The last 13 C rows — cleared, not deferred

13 channels survived as `KEEP_MESSAGING`, including channel 190505
(`+447864538288`, Alex Cooper vs Jacqui Rogan).

Inspecting each one showed the guard was too blunt: **in all 18 of 18 links a
Person who genuinely carries the value was still linked**, so releasing the wrong
link could not orphan any conversation. The rule now protects a channel only when
releasing would leave it with **no** owner (`KEEP_MESSAGING_WOULD_ORPHAN`).

All 13 verified individually afterwards — only the flagged links removed, a real
owner still linked on each, message counts unchanged, 0 problems. Channel 190505
now resolves to **Jacqui Rogan**.

Full detail: `docs/RUNBOOK_STALE_CHANNEL_REPAIR.md` §7.

## 5. Production procedure — corrected by this rehearsal

```bash
python manage.py contact_identity_audit --snapshot prod-before.csv

python manage.py backfill_missing_person_channels --apply
python manage.py split_collapsed_persons --apply
python manage.py split_collapsed_persons --min-names 2 --apply
python manage.py repair_stale_person_channels --apply
# repeat the last two until both report 0

# NOTE the --control-from flag. Without it the control group shifts and the
# comparison is meaningless in BOTH directions.
python manage.py contact_identity_audit --snapshot prod-after.csv \
       --control-from prod-before.csv
python manage.py contact_identity_audit --compare prod-before.csv prod-after.csv
```

Then the two checks the defect audit cannot make:

1. **Name keys** — every Person created by the repair must share a canonical name
   key with one of its own records (0 mismatches here, on 350 new Persons).
2. **Primaries** — no person/kind left with channels but no `is_primary`
   (430 were re-promoted automatically during this run).

## 6. Expected production numbers

| | |
|---|---|
| records re-pointed | ~3,412 |
| Persons created | ~350 |
| households created | ~74 |
| channel links released | ~1,877 |
| channel links added (backfill) | ~8,453 |
| primaries re-promoted | ~430 |
| runtime | minutes, not hours |

---

## 8. Cross-reference: did the deletions lose anything? NO

The purge deleted Person rows. This section proves, against the **untouched
control copy**, that nothing of substance went with them.

### Which Persons actually disappeared

Derived by set difference, not by trusting the tool's own count:
Persons in control (83,700) minus Persons in the repaired copy (62,206) =
**21,844** — exactly the number the purge reported.

### What those 21,844 had attached, measured IN THE UNTOUCHED COPY

Checked against **all 18 foreign keys** that reference `TreatmentPlan_person`
anywhere in the schema (enumerated from `information_schema`, not hand-picked):

| Reference | Rows |
|---|---|
| Patient / Intake / Nurture | **0 / 0 / 0** |
| Note / NoteHistory | **0 / 0** |
| Activity / ActivityLog | **0 / 0** |
| MarketingConsent | **0** |
| BroadcastRecipient / FormSubmission / PreferenceToken | **0 / 0 / 0** |
| Journey enrolment | **0** |
| ContactMergeDismissal / MergeLog / merged_into target | **0 / 0 / 0** |
| MarketingPatientProfile | 21,844 (1:1, derived analytics) |
| PersonChannel | 37,855 (links only) |

**Treatment plans:** `TreatmentPlan_treatmentplan` links to `patient_id`, not to a
Person. Since these Persons held zero patients, the reachable treatment-plan count
is **0** — confirmed by query, not by inference.

So the only things deleted alongside were a derived marketing profile (rebuilt by
the Dentally sync; it holds appointment counts, spend and dates, no consent) and
channel *links*. **No ContactChannel row was deleted** — that table went UP.

### No conversation was orphaned — it improved

A channel with messages but no linked Person is an orphaned inbox thread.

| | control | repaired |
|---|---|---|
| ownerless channels carrying messages | 337 | **296** |

All 296 were **already ownerless in the control**; **0 were newly orphaned**. The
repairs gave 41 previously ownerless channels an owner.

### Whole-database diff — 541 of 546 tables identical

| table | control | repaired | delta |
|---|---|---|---|
| TreatmentPlan_contactchannel | 108,615 | 114,886 | **+6,271** |
| TreatmentPlan_household | 15,816 | 15,890 | +74 |
| TreatmentPlan_person | 83,700 | 62,206 | −21,494 |
| TreatmentPlan_personchannel | 136,676 | 105,983 | −30,693 |
| marketingBroadcast_marketingpatientprofile | 83,700 | 61,856 | −21,844 |

**Every other table — all 541 of them — is byte-for-byte identical in row count:**
patients, intakes, nurtures, notes, treatment plans, appointments, invoices,
messages, recalls, day-list, journeys, marketing sends. The work touched only the
five identity tables it was supposed to.

Person is −21,494 rather than −21,844 because the split repairs created 350 new
Persons; ContactChannel rose because the A/B backfill registered contact details
that had never been channels at all.

---

## 9. Stranded clinical history — re-attached

### The defect

70 Persons held Note / NoteHistory / Activity / ActivityLog rows but **no Patient,
Intake or Nurture**. For most of them the same human still existed in the same
practice as a *different* Person holding patients, so the history was **invisible
on the patient's record**. One NoteHistory row read:

> *"Clincheck to be made by Adam and aligners to be ordered ASAP to arrive in time
> for the fit appt on 05/09 - Adam emailed 18/08"*

Same cause as everything else in this workstream: the 2026-07-27 re-import
re-pointed patients at freshly resolved Persons, and history rows — which carry
their own `person_id` — were left behind on the old, now-empty Person.

### The fix

`TreatmentPlan/management/commands/reattach_stranded_history.py`. Re-points
history FKs only — nothing deleted, no Patient/Intake/Nurture touched, no Person
created. Writes use `.update()` so no save signal can re-resolve and undo the move.

History moves only when **all** hold: the source Person holds no records; exactly
**one** live Person in the **same practice** has the same canonical name; and the
two have no conflicting DOB.

| Verdict | Persons |
|---|---|
| **MOVE** | **53** |
| SKIP_NO_LIVE_PERSON | 24 |
| SKIP_NO_NAME | 19 |
| SKIP_AMBIGUOUS (several live people share the name) | 3 |

### Result — verified row by row, not by totals

Applied in two batches (10, then 43). Every one of the 96 history rows was tracked
by id from before to after:

| | |
|---|---|
| moved to the **correct** target | **96 / 96** |
| moved anywhere unexpected | **0** |
| rows that vanished | **0** |
| still on the old Person | 0 |

- Global row counts **unchanged**: Note 4,326 / NoteHistory 8,588 /
  Activity 19,577 / ActivityLog 4,220
- All 39 target Persons still exist and **all 39 hold patients**
- 0 orphaned history FKs
- All five defects still measure **0**
- Whole-database diff still shows **541 of 546 tables identical** — the move
  changed no row counts at all, only which Person the rows point at

**The aligner note now sits on Person 107866 "William Vince", who holds two patient
records — so it is visible on the patient again.**

### What deliberately remains

131 history rows stay stranded on the 46 skipped Persons (19 nameless, 24 with no
live counterpart, 3 ambiguous). Each would require guessing which human they belong
to, and a wrong guess puts one patient's clinical note on another patient's record.
They are listed in `docs/identity-snapshots/prod-stranded-history-plan.csv` for
human review.

---

## 10. Recon: who does each history row actually belong to?

§9 matched stranded history to a live Person **by name**. That was the only
evidence used, and it was not good enough.

### The stronger evidence we had all along

Every Note / NoteHistory / Activity / ActivityLog row carries a generic FK —
`content_type` + `object_id` — pointing at the record it is **about**. That record
carries the authoritative `person_id`. A note about *treatment plan 713* belongs to
whoever owns treatment plan 713, regardless of names.

**Two content_type conventions exist and both must be handled:**

| Model | content_type |
|---|---|
| NoteHistory | a **string** — `treatment_plan`, `nurture`, `intake` |
| Activity / ActivityLog | a **ContentType id** — 24, 79, 14, … |

A resolver handling only one silently resolves nothing for the other. The first
attempt did exactly that and reported "3 usable links" out of 96.

### The name-based pass had misfiled 4 rows — including a clinical one

| Row | Text | I put it on | The record says |
|---|---|---|---|
| NoteHistory 4994 | *"Spark - Rescan needed for new planning"* | **Zuri** Maclean | **Kerrie** Maclean |
| ActivityLog 2120 | *"Treatment Plan created for **Kerrie Maclean**"* | **Zuri** Maclean | **Kerrie** Maclean |
| ActivityLog 1373 | "Note created" | Christian Rule | Lee Vader |
| ActivityLog 192 | "Note created" | Paul Bibby | Adrian Deegan |

ActivityLog 2120 **names Kerrie in its own text** and was attached to a family
member. That is the exact misattribution this workstream exists to remove, created
by the repair itself. Name matching cannot tell two Macleans apart; the record can.

### Scale — and how much of it was ours

Same check run against the **untouched control copy**:

| | rows pointing at the wrong Person |
|---|---|
| control (untouched production) | **529** |
| after our repairs | 542 |
| introduced by our work | 47 |
| fixed by our work | 34 |
| **net added** | **+13** |

So 529 were pre-existing production damage that nobody had ever measured, and our
repairs added 13 net — a real regression, found only because the control copy was
kept.

### The fix

`reattach_stranded_history` now runs **PHASE A first**: every history row with a
resolvable generic FK is re-pointed at the Person of the record it references.
This is authoritative and **overrides** the name-based pass, which now only
handles rows with no usable link.

| | |
|---|---|
| rows re-pointed | **542** (295 ActivityLog, 123 NoteHistory, 114 Activity, 10 Note) |
| **disagreements remaining** | **0** |
| global row counts | **unchanged** (4,326 / 8,588 / 19,577 / 4,220) |
| orphaned history FKs | 0 |
| all five defects | **0** |
| whole-database diff | still **541 of 546 tables identical** |

All four misfiles verified corrected by id — NoteHistory 4994 and ActivityLog 2120
are now on Kerrie Maclean.

### Still unresolved, and honestly so

| Reason | Rows |
|---|---|
| the record it referenced has been deleted | 3,166 |
| SMSMessage / EmailMessages / CallLog carry no person or patient link | 3,255 |
| DentallyAppointment has no person link | 38 |
| unresolvable content_type | 12 |

These cannot be resolved from the data we hold. They are unchanged from the control
copy — pre-existing, not caused here — and guessing an owner would risk putting one
patient's note on another's record.

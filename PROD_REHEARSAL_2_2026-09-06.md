# Production rehearsal 2 — 2026-09-06

The master runbook (`PROD_IDENTITY_REPAIR_MASTER.md`) run **as written, top to
bottom**, on a fresh clone of the 2026-09-04 production dump, with an untouched
copy as control — then every result compared against the control **and against
rehearsal 1** (`PROD_REHEARSAL_2026-09-05.md`, database `prod_rehearsal`).

| database | what it is |
|---|---|
| `prod_pristine` | the dump, never touched — passes the runbook's own "untouched" check (`last_sync` 2026-09-04 12:19, max Person id 158293, 83,700 Persons) |
| `prod_control` | the dump plus ONE contaminated row (`dentallyintegration` id 11, `last_sync` 2026-09-05 18:59). Identical to pristine on every identity table. |
| `prod_rehearsal2` | `CREATE DATABASE … TEMPLATE prod_control`, then the runbook |
| `prod_rehearsal` | rehearsal 1 (2026-09-05) |

Every log, plan CSV and snapshot is in `docs/identity-snapshots/rehearsal2/`,
numbered in execution order.

---

## 1. Verdict

**The runbook fixes what it says it fixes, and rehearsal 2 reproduces rehearsal 1
almost exactly.** All five defects go to 0 with no new breakage and no control row
broken; every one of the 3,412 moved records and 354 created Persons passes the
per-record checks; only the identity tables change.

**It did not fix three things it never measured**, all found by checking the
*shape* of the result rather than the defect counts. All three are now fixed in code
and in the runbook (the third only after its own rehearsal went wrong first — §7):

| Found | Size on the repaired copy | Now |
|---|---|---|
| The split leaves the **anchor named after a human it no longer holds** (Person 100335 "Amaina Amhad" b.1992 holding only Zayd Amhad b.2020). One record, one name — invisible to every audit, and `Person.resolve` matches on name, so Zayd's next record mints a duplicate. | 10 created by the split + 535 pre-existing = **545** Persons | **Fixed.** The split renames its anchor; `split_collapsed_persons --fix-misnamed` repairs the rest. 545 → **4** (the 4 hold two humans and are left for a person to decide). |
| **The primary phone/email is a value the Person's own records do not carry**, while their own value IS linked. The A/B backfill links each record's own number but never touches `is_primary`, so SMS/email keep going to the old — usually a relative's — value. | 105 before the repair, **5,537 after** (the backfill made the right number *available*, not *chosen*) | **Fixed.** `repair_primary_channels --prefer-own`. 5,537 → **0**; also promotes the 6,810 person/kinds with no primary at all and demotes the 238 with several. |
| **252 humans exist twice** (same name, same channel, two Persons — `identity_guard` SPLIT). Pre-existing (265 in the control); no runbook step addressed it. | 252 | **Rehearsed and added (§7).** The first `dedupe_persons` run welded 17 humans into relatives while every defect count stayed 0; rolled back, guarded (own-name key + DOB), re-run: SPLIT → **53**, 0 regressions. |

Plus one thing it cannot do here: **defect E was not rehearsed** — the local `.env`
has no `DENTALLY_ENCRYPTION_KEY`, Dentally answered 401 and the command reported
`83 SKIP_API_ERROR … would update 0 patients` with **exit code 0**. On production
that is a silent no-op that looks like success. Rule added to the runbook.

---

## 2. The run, step by step (numbers are rehearsal 2 / rehearsal 1)

| Step | Result |
|---|---|
| baseline snapshot | A 3,596 · B 3,823 · C 5,411 rows (3,528 records) · D 817 · D2 1,489 · 2,000 control rows — identical to production and to rehearsal 1 |
| A+B `backfill_missing_person_channels` | linked 5,003 patients + 91 intakes + 8 nurtures = **5,102** (r1 ≈5,100) |
| D plan → `--limit 20` → all | 817 Persons / 2,727 named humans; **1,910 groups, 1,912 records**; 119 new Persons, 1,793 rejoined an existing identity (r1: 1,910 / 118) |
| D2 `--min-names 2` | **1,493 groups**, 228 new Persons (r1: 1,493 / 225) |
| C plan → `--limit 50` → all | **1,898 links released** (629 orphan-Person, 1,269 stale), **440 primaries re-promoted**, 0 held back for messaging (r1: 1,877 / 430) |
| fixed-point loop | pass 2 = 7 D2 splits, 0 C; passes 3–4 = nothing — **identical to r1** |
| defect snapshot (`--control-from`) | **all five 0, NEW 0, control broken 0** |
| `reattach_stranded_history` | PHASE A re-pointed **575** rows (124 Activity, 306 ActivityLog, 10 Note, 135 NoteHistory); then 34 stranded Persons moved (25 ActivityLog, 34 NoteHistory); re-run → 0 remaining; 41 skipped (3 ambiguous, 19 no live twin, 19 nameless). Row counts unchanged 4,326 / 8,588 / 19,577 / 4,220. (r1: 542 + 53) |
| `purge_recordless_persons` `--limit 100` → all | **21,906 deleted** (21,563 live-name twin, 193 in Dentally mirror, 50 share a channel with a live patient); kept 41 with history + 106 unexplained. Persons 83,700 → 62,148. (r1: 21,844 — see §4) |
| E `repair_unusable_phones --landlines` | **not rehearsed** — 401 from Dentally without the key |
| **new** `split_collapsed_persons --fix-misnamed` | **541 renamed**, second run 0 |
| **new** `repair_primary_channels --prefer-own` | promote 12,326 (5,537 moved to own value, 6,789 had none), demote 5,783; second run 0 |
| final snapshot | all five 0, NEW 0, control 0 — unchanged by the two new steps |

## 3. Proof it did nothing else

- **Whole-database content checksum** (`_prod_sim/checksum_db.sh`, every row of all
  546 tables, not row counts) vs `prod_pristine`: **12 tables differ** — the 11 the
  runbook predicts (contactchannel +6,271, household +74, person, personchannel,
  marketingpatientprofile, and in-place `person_id` edits on patient / intake / note
  / notehistory / activity / activitylog) plus `dentallyintegration`, which was
  already different **before any command ran** (the control-contamination row the
  clone inherited). The two new steps changed only `person` and `personchannel`.
- `_prod_sim/verify_split_targets.py` on all 3,412 moved records and 354 created
  Persons: **0 problems** (practice, merged-away, household, DOB vs target and vs
  its other records, own email/phone linked, name key, twin).
- `_prod_sim/verify_repair.py` (row-by-row vs the control, 65,179 comparable
  records): A 3,594 on a name-matching Person / 2 differ; B 3,819 / 4 (all four are
  "Unknown" callers); D 2,724 landed on a same-name Person / 10 differ (the misnamed
  anchors, now fixed); **records that were right and became wrong: 0**.
- 0 orphaned `person` FKs on all 10 referencing tables, before and after the purge.
- Ownerless channels carrying messages 337 → 293 (none newly orphaned).
- `identity_guard --strict`: COLLAPSE 817 → 0, MISLINKED 88 → 0, SPLIT 265 → 252,
  NAMELESS 24,980 → 104 (the 147 kept record-less Persons that have channels).

## 4. Rehearsal 1 vs rehearsal 2 (`_prod_sim/compare_rehearsals.py`)

Person ids minted by a repair differ between runs, so identities were compared
structurally: for every record, the set of records sharing its Person.

| | r1 | r2 |
|---|---|---|
| records whose grouping is identical | **65,179 of 65,179** | |
| records moved off their control Person | 3,412 | 3,412 (**the same 3,412**) |
| Persons deleted | 21,844 | 21,906 — 21,841 in common; r2 deletes 65 more because the runbook now re-attaches history **before** the purge, freeing Persons that r1 kept as "has history" |
| Persons created | 350 | 354 |
| created Persons with **no DOB** | **350** | **1** (an Intake-only group — intakes carry no DOB) |

**The 4 extra Persons are r2 being right where r1 was wrong.** In the control,
Persons 82581 Charlotte Edge (dob 2020-04-**05**), 92458 / 141165 Michael Bayfield
(1957-09-**20**) and 100115 Jack Robbins (1991-04-**27**) hold **no records at all**
— stale leftovers of the July re-import. The patients' own rows (created
2026-07-27, DOB from Dentally) say 04-**06**, 09-**24**, 04-**22**. Rehearsal 1
passed no DOB to `Person.resolve`, so the split rejoined each record to the stale
Person and revived it with the wrong birth date. Rehearsal 2 passes the group's
DOB (the uncommitted `group_dob` change in `split_collapsed_persons`), the resolver
refused the conflict, a fresh Person got the right DOB, and the stale one was purged.

23 records differ in which channels their Person is linked to: the phone numbers
the defect-E step recovered in r1 (not run in r2), and the Bayfield / Rains
household numbers — where r1 *released* a shared number that r2 kept because the
split had put both owners in one household. That is the "wrong primary" shape in
§1, and `--prefer-own` resolves it without releasing anything.

## 5. Traps this rehearsal added

- **A 401 from Dentally is not an error to `repair_unusable_phones`.** It is counted
  as `SKIP_API_ERROR`, the command prints `would update 0` and exits 0. Abort if
  that counter is non-zero.
- **`os.environ["DB_NAME"]` lies after `django.setup()`.** `settings.py:561` runs
  `load_dotenv(override=True)` *after* `DATABASES` is built, so a script that prints
  the env var reports the `.env` database while actually querying the one you
  passed. Use `connection.settings_dict["NAME"]`. (Caught when three runs against
  three databases all printed `treatmentpath_db`.)
- **A `TEMPLATE` clone needs the full 8.6 GB up front** and any connection to the
  template blocks it; this machine had 4.7 GB left afterwards. Check `df` first.
- **Four migrations are unapplied on the dump** (`activityLog.0021`,
  `dentallyIntegration.0166/0167`, `debug_toolbar.0001`). No identity command needs
  them; production will have them after the deploy.
- **The 4 Persons left misnamed hold two humans each** (e.g. 81175 "Afaaf Rajbee"
  holding Mohammed Akhter and Mohammed Akhte — one human, two spellings, no DOB
  conflict). Never rename by guess; list them for a person.
- **Intakes carry no DOB, so a lead sitting on a relative's Person is invisible to
  D2** — intake 3097 Sarah Franklin on Person "Simon Franklin" (b.1963), unchanged
  by every step. Shape, not count; no tool addresses it yet.

## 6. Still open after this rehearsal

1. The 252 SPLIT duplicates (§1) — `dedupe_persons` exists, is dry-run by default,
   finds 201 groups; decide whether it belongs in the runbook.
2. The 4 two-human Persons named after neither (§5).
3. Defect E — rehearse once with the production key and **save the API responses**.
4. Everything in `PROD_IDENTITY_REPAIR_MASTER.md` §7, unchanged.

---

## 7. Dedupe rehearsal and the messaging fix (afternoon, same day)

Approved plan: rehearse `dedupe_persons` on the already-repaired `prod_rehearsal2`
(the state production will be in after §2), verify per record against the control,
and fix the messaging blocker in code.

### 7.1 `dedupe_persons` — the first run was wrong, and only the control saw it

| Step | Result |
|---|---|
| exact-name dry run (`--no-partial-names`) | 143 groups; pre-check: **3 pairs with conflicting DOBs**, 22 with a different name split, 0 sharing a Dentally id |
| DOB guard added; dry run | 3 × `SKIP_DOB_CONFLICT` — and those 3 were NOT same-name pairs: *Simon Franklin* ↔ *Sarah Franklin*, *Gary Holt* ↔ *Lynn Holt*, *Riz Tejpar* ↔ *Jehanara Tejpar* |
| apply, 140 groups | 141 merged, every defect **still 0**, control 0 broken, records unchanged, 0 Patient rows deleted — **and `verify_repair.py` reported 17 regressions**: patient *Sian Jeram* now on Person "Nolan Jeram", *Gergo Simon* on "Zsofia Simon", *Connor Burman* on "Lee-anne Corrigan" |

Cause: the command's identity key came from **any** record a Person held. Nolan's
Person held Sian's misfiled intake, so it "matched" Sian's real Person; Nolan had a
Patient, so he was the winner and Sian's identity was merged into her relative. The
DOB guard cannot see it when one side has no DOB. **Every defect count was 0
throughout** — this was visible only in the row-by-row comparison against the
untouched copy.

Rolled back: `Person.unmerge` on all 141 `ContactMergeLog` rows (141/141 reverted,
state verified identical to before: 0 regressions, 0 problems). Fixed: a key now comes
only from records carrying the Person's **own** name (plus the DOB guard). Tests
`DedupeNeverMergesTwoHumans` ×2 — green; red run against HEAD's code (imported from
`git show`, inside a rolled-back transaction) reproduced both merges.

| Corrected run | Result |
|---|---|
| dry run | **133 groups**, 0 DOB skips |
| apply | **134 merged**, 0 Patient rows collapsed, all logged/reversible |
| C repair after | **17 stale links** carried over by the merges, released |
| two-name split after (positive-evidence rule, §7.2) | 6 more records rejoined their own Person; then 0 / 0 |
| audit vs control | A–D2 **0**, NEW 0, control 0 broken |
| `verify_repair.py` (split-tolerant names) | **regressed 0**; 2,734 D records on a same-name Person |
| `verify_split_targets.py` | 0 name problems; 15 merge winners without a household (cosmetic: a merge has no household to assign) |
| `identity_guard --strict` | COLLAPSE 0 · MISLINKED 0 · **SPLIT 246 → 53** · NAMELESS 103 |
| new links to a Person who does not carry the value | 3 (all were on the loser before; benign, listed in the log) |

A second surprise: 10 "regressions" reported after the corrected run were all
`Ken|(Kenneth) Judge` → `Ken (Kenneth)|Judge` — the split-tolerant name key
(audit #1 fix, landed by a parallel session during this run) letting the dedupe
correctly merge the name-split duplicates. Both verifiers compared the `(first,
last)` tuple; they now compare the joined key.

### 7.2 Misfiled relatives — a positive-evidence split

Measured: 84 Persons hold two names with no DOB conflict (so D2 never touches
them); for **10** the second name is the exact name of another live Person on a
shared channel — the record is misfiled and its home is known. `split_collapsed_persons
--min-names 2` now accepts that as evidence; the record rejoins the existing Person
(nothing minted). 5 + 6 records on the rehearsal (intake 3097 Sarah Franklin → Person
119389 "Sarah Franklin"). The 77 with no such twin are left alone.

### 7.3 Messaging — the "blocker" was real but smaller than written

Fixed in `messaging/views/contact_views.py` (5 new tests, red-run proven): a Person
id no longer collapses to one shared channel; a channel with two live owners is its
own row in the contact list; THR-03's `is_family` now counts live owners across the
contact's channels, so `patient_id` scopes a shared phone again. The unfixed code
returned *fewer* messages for a shared channel, not more — a functional regression
and a disabled guard, not a new PHI leak.

### 7.4 Concurrency warning

While this ran, another session edited `contact_keys.py`, `models.py`,
`identity_resolution.py`, `identity_defects.py` and `dedupe_persons.py` (the audit #1
fix, 12:48–12:52). All changes described here were re-checked as intact afterwards and
the full test set (43 + 2) is green on the combined tree — but two sessions in
`dedupe_persons.py` at once is a merge hazard; review that file's diff as a whole.

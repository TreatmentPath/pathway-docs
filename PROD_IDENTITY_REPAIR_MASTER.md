# Contact identity — PRODUCTION execution runbook

**One document to run the whole thing.** Everything below was rehearsed end to end
on a restored copy of the 2026-09-04 production dump and verified record by record.
The detail docs are linked per step; you should not need to re-derive anything.

**Status: RUN ON PRODUCTION 2026-09-07, 04:20–06:10 UTC** — see
`PROD_RUN_2026-09-07.md`. Every step below executed in this order against the live
database (local rehearsed code in a throwaway container on the prod host, deployed
image, prod `.env`), with the acceptance gates in §3 after each phase. Result:
**A 3,595 / B 3,823 / C 3,528 / D 817 / D2 1,489 → all 0, NEW 0, 0 control rows
broken, 0 regressions on 65,221 records**, 354 Persons created, 3,679 records moved,
21,909 obsolete Persons removed, 134 duplicates merged (reversible), 495 misnamed
Persons renamed, 12,318 primaries fixed, 18 phone numbers recovered from Dentally.
Rehearsed twice beforehand on the same dump (`PROD_REHEARSAL_2026-09-05.md`,
`PROD_REHEARSAL_2_2026-09-06.md`).

---

## 0. What is wrong, in one table

| Defect | On production today | After the rehearsal |
|---|---|---|
| **A** — a record's own email is not linked to its Person | 3,596 | 0 |
| **B** — a record's own phone is not linked | 3,823 | 0 |
| **C** — a channel linked to a Person who does not carry it | 3,528 | 0 |
| **D** — one Person holding 3+ different humans | 817 | 0 |
| **D2** — one Person holding 2 humans with conflicting DOBs | 1,489 | 0 |
| **E** — phone numbers that cannot be canonicalised | 774 patients | 63 (unrecoverable at source) |
| **NEW** — a Person named after a human it does not hold (one record, one name) | 556 | **4** (two-human, left for review) |
| **NEW** — primary phone/email is a value none of the Person's records carry, while their own value is linked | 105 (**5,537** after A/B alone) | **0** |
| SPLIT — one human, same name + channel, two Persons (`identity_guard`) | 265 | **53** after the dedupe phase (§2, NEW 2026-09-06 #3); 252 without it |
| **NEW** — a relative's record misfiled on a Person, with the relative's own Person on the same channel | 10 | **0** (`split_collapsed_persons --min-names 2` now splits on that evidence) |

Also cleared in the rehearsal: **21,906 obsolete Persons** removed (21,844 in
rehearsal 1 — history is now re-attached first, which frees 62 more), **575 + 34
Persons' history** re-pointed to the person it actually belongs to.

---

## 1. Prerequisites — do not skip

1. **Deploy the code fixes first.** Repairs run against unfixed code compete with a
   live defect and will re-create damage behind you.

   > ### ✅ RESOLVED 2026-09-06 (was a blocker) — messaging identity on shared channels
   >
   > The shared-channel fix (`sole_person_for_channel` returning `None` for a
   > channel with several owners — correct) left three gaps in
   > `messaging/views/contact_views.py`, all fixed and tested
   > (`messaging/test_shared_channel_attribution.py`,
   > `messaging/test_contact_views.py::FamilyGuardOnSharedChannelTests`, red-run
   > proven against HEAD):
   > 1. a **Person id** was resolved Person → most-active channel → owner-of-channel,
   >    so when that channel was a family phone the identity collapsed to one shared
   >    channel: star/archive/spam hit the family SMS thread and missed the person's
   >    own email thread. `_resolve_identity` now treats a Person id as the identity.
   > 2. `_group_channels_by_identity` still gave a shared channel to
   >    `links[0].person`; it now uses the same one-live-owner rule.
   > 3. THR-03's `is_family` came from the (deliberately `None`) channel Person, so
   >    `patient_id` was discarded on a shared channel. `_is_family` now also counts
   >    live owners across the contact's channels.
   >
   > Accuracy note: on the 09-04 copy the unfixed code returned **fewer** messages
   > for a shared channel (the lone channel's sessions), not more — the family SMS
   > thread was shown either way, and the patient's own threads were dropped. It
   > was a functional regression plus a disabled guard, not a new PHI leak.
   - Django: `TreatmentPlan/contact/signals.py`, `TreatmentPlan/models.py`,
     `TreatmentPlan/utils/contact_keys.py`, `TreatmentPlan/utils/phones.py`,
     `dentallyIntegration/tasks.py`, `dataQuality/patient_import_utils.py`
   - Go: `internal/dentally/migration/service.go`, `pkg/email`, `pkg/personname`
   - The Go identity fix (`dobConflict`) is ALREADY live in production — the
     deployed image was built 2026-08-31 and contains it. The Django side is not.
2. **Take a database backup.** The repairs are additive except the purge in step 6,
   which deletes Person rows and cannot be undone.
3. **Run outside sync windows.** A Dentally import mid-repair re-resolves records
   while they are being moved.
4. **Keep an untouched copy** of the same dump. Every regression in the rehearsal —
   including 13 we introduced ourselves — was found by diffing against it. Without
   it you cannot tell "fixed" from "broke something else".

   **Prove it is untouched before trusting it.** The rehearsal's control copy was
   *not*: `dentallyIntegration_dentallyintegration` id 11 carries
   `last_sync = 2026-09-05 18:59` in the control against `2026-09-04 12:19` in the
   repaired copy. A repair cannot move a timestamp forward in the control, so
   something wrote to it after the restore. Check before starting:

   ```sql
   SELECT max(last_sync) FROM "dentallyIntegration_dentallyintegration";
   SELECT max(created_at) FROM "TreatmentPlan_person";
   ```

   Both must predate the dump. Revoke write access to the control database for the
   duration of the run. (Locally: `prod_pristine` passes both checks; `prod_control`
   fails the first — identical on every identity table, but use `prod_pristine` for
   the whole-database checksum or the contaminated row shows up as a diff.)

5. **Disk.** A `CREATE DATABASE … TEMPLATE` clone needs the full size of the dump
   (8.6 GB) up front and is blocked by any open connection to the template. Check
   `df` before, not after.

---

## 2. The sequence

Run in this order. Each step is dry-run by default; `--apply` writes.

```bash
# ---- baseline -------------------------------------------------------------
python manage.py contact_identity_audit --snapshot prod-before.csv

# ---- A + B: register each record's own contact details --------------------
python manage.py backfill_missing_person_channels            # dry run
python manage.py backfill_missing_person_channels --apply

# ---- D: Persons holding 3+ humans -----------------------------------------
python manage.py split_collapsed_persons --csv prod-D-plan.csv
python manage.py split_collapsed_persons --limit 20 --apply  # small batch first
python manage.py split_collapsed_persons --apply

# ---- D2: Persons holding 2 humans with conflicting DOBs -------------------
python manage.py split_collapsed_persons --min-names 2 --apply

# ---- DOB on split Persons: BUILT IN since 2026-09-06 ------------------------
#      split_collapsed_persons now passes the group's DOB to Person.resolve and
#      stamps it on any Person it creates. Rehearsal 2: 353 of 354 created
#      Persons carry a DOB; the one without is an Intake-only group (intakes have
#      no DOB column). Passing the DOB also changed 4 outcomes for the better:
#      rehearsal 1 rejoined 4 records to stale, record-less Persons whose DOB was
#      off by days (Charlotte Edge 2020-04-05 vs 06, Michael Bayfield 09-20 vs
#      24 x2, Jack Robbins 04-27 vs 22); the guard now refuses those and the
#      stale Person is purged. Verify after the D/D2 passes:
#        SELECT count(*) FILTER (WHERE dob IS NULL), count(*)
#          FROM "TreatmentPlan_person" WHERE id > <max Person id before the run>;
#      Expect the NULL count to equal the number of created Persons that hold
#      only Intake/Nurture rows.

# ---- C: channels linked to the wrong person -------------------------------
python manage.py repair_stale_person_channels --csv prod-C-plan.csv
python manage.py repair_stale_person_channels --limit 50 --apply
python manage.py repair_stale_person_channels --apply

# ---- REPEAT the last three until a pass changes NOTHING --------------------
#      splitting one collapse exposes another. On BOTH rehearsals:
#      pass 1 = 1,493 splits, pass 2 = 7, passes 3-4 = 0.

# ---- NEW 2026-09-06: Persons named after a human they do not hold ---------
#      The split's ANCHOR keeps the Person id — and used to keep the Person's
#      name and DOB even when every record of that human had just been split
#      off (Person 100335 "Amaina Amhad" b.1992 left holding only Zayd b.2020).
#      One record, one name: invisible to A-D2 and to identity_guard, and
#      Person.resolve matches on name, so that human's next record mints a
#      duplicate. The split now renames its anchor; this pass catches the 535
#      pre-existing Persons in the same shape. Rehearsal 2: 541 renamed, 4 left
#      (each holds TWO humans — never rename by guess; review by hand).
python manage.py split_collapsed_persons --fix-misnamed --csv prod-misnamed-plan.csv
python manage.py split_collapsed_persons --fix-misnamed --apply
#      A second dry run must report 0.

# ---- history stranded on emptied Persons ----------------------------------
python manage.py reattach_stranded_history --csv prod-hist-plan.csv
python manage.py reattach_stranded_history --apply

# ---- housekeeping: obsolete Person rows -----------------------------------
python manage.py purge_recordless_persons --csv prod-purge-plan.csv
python manage.py purge_recordless_persons --limit 100 --apply
python manage.py purge_recordless_persons --apply
#      Expect ~21,906 (rehearsal 2). Rehearsal 1's 21,844 ran the purge BEFORE
#      re-attaching history; this order frees the Persons whose history moved.

# ---- NEW 2026-09-06: the RIGHT primary, not just A primary ------------------
#      The A/B backfill links each record's own email/phone but never touches
#      is_primary. primary_phone / primary_email feed the inbox and the SMS and
#      email senders, so after A/B 5,537 person/kind pairs (105 before) had
#      their own value linked while messages still went to a value none of
#      their records carry — usually a relative's number from the old household
#      linking. --prefer-own moves the primary onto a linked channel the
#      Person's own records carry; a person/kind whose records carry no value
#      of that kind (a child on the family phone) is left alone. Also promotes
#      the ~6,800 person/kinds with no primary and demotes the ~240 with several.
python manage.py repair_primary_channels --dry-run --prefer-own
python manage.py repair_primary_channels --prefer-own
#      A second dry run must report promote 0 / demote 0.

# ---- NEW 2026-09-06 #3: one human, two Persons (identity_guard SPLIT) ------
#      Rehearsed 2026-09-06 (PROD_REHEARSAL_2_2026-09-06.md §7). The FIRST run of
#      this command welded 17 humans into a relative: its identity key came from
#      ANY record a Person held, so Nolan Jeram's Person — holding Sian Jeram's
#      misfiled intake — "matched" Sian's real Person and, having a Patient,
#      swallowed it. Caught ONLY by the row-by-row control comparison; every
#      defect count was 0. Rolled back with Person.unmerge (141/141), guarded,
#      re-run. Two guards now: a key comes only from records carrying the
#      Person's OWN name, and a group with conflicting DOBs is SKIP_DOB_CONFLICT.
#      Merges are logged (ContactMergeLog) and reversible; no Patient row is
#      deleted unless the winner ends up with two rows for one Dentally id (0 on
#      the rehearsal).
python manage.py dedupe_persons --no-partial-names            # dry run: expect ~133 groups
python manage.py dedupe_persons --no-partial-names --commit   # rehearsal: 134 merged
#      A merge carries the loser's STALE links to the winner (17 new C rows on the
#      rehearsal) — so run the C repair and the two-name split to a fixed point
#      again, then the audit. Do NOT run the partial-name pass (`[partial]`
#      groups, prefix/empty-last-name matches) without reviewing its plan by hand.
python manage.py repair_stale_person_channels --apply
python manage.py split_collapsed_persons --min-names 2 --apply
#      ... repeat both until 0 / 0 (rehearsal: 17 links, 6 splits, then 0 / 0)

# ---- E: recover phone numbers Dentally can still supply -------------------
#      needs DENTALLY_ENCRYPTION_KEY from the production .env
python manage.py repair_unusable_phones --landlines
python manage.py repair_unusable_phones --landlines --apply
#      ⛔ ABORT if the dry run shows ANY SKIP_API_ERROR. Verified 2026-09-06:
#      with a missing/invalid key Dentally answers 401, the command counts every
#      patient as SKIP_API_ERROR, prints "would update 0 patients" and EXITS 0.
#      --apply would then be a silent no-op that looks like success. This step
#      was NOT rehearsed on 2026-09-06 for exactly that reason (no key locally);
#      rehearsal 1's 18 rows / 13 patients stand as its only evidence.
#      TWO CAVEATS VERIFIED 2026-09-06:
#      (a) On 7 of the 19 rehearsal fixes the landline was copied into
#          phone_number while secondary_phone_number kept it too, so both
#          columns end up IDENTICAL (35312, 35313, 35507, 51979, 53573,
#          122267, 129553). The originals were all INVALID numbers
#          (checked with phonenumbers: 0 of 7 parse as valid GB), so the
#          substitution is correct — but clear the duplicate secondary.
#      (b) 3 rows (112719, 112720, 113688) got numbers that exist NOWHERE in
#          the source database — they came from live Dentally API calls.
#          The repair is therefore NOT reproducible or verifiable from the
#          database alone. SAVE THE API RESPONSES to a file as you go.

# ---- prove it --------------------------------------------------------------
python manage.py contact_identity_audit --snapshot prod-after.csv \
       --control-from prod-before.csv
python manage.py contact_identity_audit --compare prod-before.csv prod-after.csv
# per-record + per-created-Person checks the audit cannot make (0 problems on
# rehearsal 2; needs a dump_identity_facts.py of the UNTOUCHED copy):
DB_NAME=<control> python _prod_sim/dump_identity_facts.py control.json
python _prod_sim/verify_split_targets.py control.json <max Person id before the run>
python _prod_sim/verify_repair.py control.json after.json
# whole-database content checksums, untouched copy vs repaired (see §3):
_prod_sim/checksum_db.sh <control> before.tsv ; _prod_sim/checksum_db.sh <repaired> after.tsv
join -t$'\t' before.tsv after.tsv | awk -F'\t' '$2!=$4 || $3!=$5 {print $1}'
python manage.py identity_guard --strict     # expect COLLAPSE 0, MISLINKED 0, SPLIT ~252, NAMELESS ~104
```

**Expected numbers (rehearsal 2, this exact sequence):** A/B 5,102 links · D 1,910
groups / 1,912 records / 119 new Persons · D2 1,493 groups / 228 new · C 1,898
links released / 440 primaries · loop pass 2 = 7 · history 575 rows + 34 Persons ·
purge 21,906 · misnamed 541 · primaries 12,326 promoted (5,537 moved to own) /
5,783 demoted · Persons 83,700 → 62,148 · created 354 (353 with DOB) · moved 3,412.

---

## 3. Acceptance — totals are NOT acceptance

A better total with unexplained NEW rows is a **fail**. Require all of:

- [ ] every defect at or near 0, and **`NEW` = 0** on each
- [ ] **control breakage = 0** — and the after-snapshot MUST use
      `--control-from prod-before.csv`, or the check silently re-picks its own
      membership and can hide real breakage
- [ ] **A and B still 0** after the channel repair — the tripwire for over-removal
- [ ] **0 orphaned `person` FKs** across Patient, Intake, Nurture, Note,
      NoteHistory, Activity, ActivityLog, MarketingPatientProfile, PersonChannel
- [ ] record counts unchanged for Patient / Intake / Nurture / Note / ContactChannel
- [ ] **every moved record audited individually** — practice match, target not
      merged away, no DOB conflict with records already on the target, household
      assigned, and the record's own canonicalised email/phone actually linking it
- [ ] **name-key check** after any split — on EVERY Person that holds records, not
      only the created ones: its `(first, last)` key (the tuple `Person.resolve`
      compares, not the joined name) matches at least one of its records.
      `split_collapsed_persons --fix-misnamed` (dry run) reports the count; expect
      only two-human Persons to remain (4 on the rehearsal). The anchor that keeps
      the Person id is where this fails — see the NEW step in §2.
- [ ] **primaries**: `repair_primary_channels --dry-run --prefer-own` reports
      promote 0 / demote 0 after the run
- [ ] **the created Persons carry a DOB** wherever one of their records has one
- [ ] whole-database table diff vs the untouched copy shows only the identity
      tables changed. **Use content checksums, not row counts.** The rehearsal's
      "541 of 546 identical" was a ROW-COUNT comparison; checksumming the actual
      contents found **12 tables differ, 7 of them with identical row counts** —
      3,421 `TreatmentPlan_patient` rows changed in place, 331
      `activityLog_activitylog`, 169 `TreatmentPlan_notehistory`, 124
      `activityLog_activity`, 12 `TreatmentPlan_intake`, 10 `TreatmentPlan_note`,
      1 `dentallyIntegration_dentallyintegration` (that last one is the control
      contamination, not the repair). A row-count check cannot see any of this.
- [ ] **do not trust `--compare` alone for defect C.** It keys on
      `(record_type, record_id)`, which collapses C's 5,411 rows to 3,528 — a
      record fixed on one channel and broken on another never appears.

---

## 4. Expected results, and what is NOT a regression

From the rehearsal — expect these and do not mistake them for damage:

| You will see | Why it is fine |
|---|---|
| A small number of NEW collapsed Persons after a split | A Person **named** "Jamie Guy" holding only *Daniel* Guy's record is a one-name collapse, invisible to any name-count rule, until the split returns Jamie to it. Re-run to a fixed point. |
| New C rows after a split | Pre-existing household fragmentation becoming visible once a shared channel is separately owned. |
| The split tool reporting "created N Persons" | It counts name-groups, not creations. On the rehearsal it said 1,909; only **146** Persons were actually created — 1,763 records rejoined an EXISTING identity, which is the desired outcome. |
| `needs_review` phone channel count not falling | The repair never deletes the old bad ContactChannel. It is not a success metric. |
| Created Persons with `dob = NULL` | **Fixed 2026-09-06** — 1 of 354 on rehearsal 2, an Intake-only group. Any other NULL is a bug. |
| Rehearsal 2 created 4 more Persons than rehearsal 1 | The DOB guard refusing to rejoin a record to a stale record-less Person whose DOB is off by days. Correct — the stale Person is purged. |
| 10 records "still on a Person with a different name" in `verify_repair.py`'s D section | The misnamed-anchor shape; **0 after the `--fix-misnamed` step**. If it is not 0, the step was skipped. |
| 5,537 person/kinds with a wrong primary right after A/B | Expected — A/B links the right value, `--prefer-own` chooses it. **0 after that step.** |
| 6,632 new PersonChannel rows with `role = ''` | Cosmetic against a `mobile`/`email`/`patient` convention; empty-role links went 2,449 → ~9,081. Watch persons 147887/147888, which ended with no primary phone and a blank role where they had `role='mobile', is_primary=true`. |
| Orphaned ContactChannels 524 → 31,491 (60×) | Harmless today — they carry no messaging — but they hold the unique `(practice, kind, value)` slot with no owner. |
| 3 split Persons duplicating an existing same-name Person | Henry Brown, Lydia Lacey, Gemma Smith — the split minted instead of reusing. Check for these after each pass. |
| One bogus Person: 158411 "Newman James" | The repair faithfully propagated a corrupt name split on the intake row itself (findings #1/#8). Household 20997 now holds three identities for two humans. **This is why the name-split code fix must ship BEFORE the repair runs.** |
| ~6,400 history rows "unverifiable" | They point at a deleted record, or at an SMS/Email/CallLog that has no person link. **6,391 of them sit on a Person that holds patients** — visible and fine. Only ~84 are genuinely stranded. |

---

## 5. Decisions already made — do not redo the analysis

| Question | Answer | Detail |
|---|---|---|
| The 25,087 record-less Persons — damage? | **No.** Migration debt from the 2026-07-27 re-import; 98% of the humans still exist. | `RECORDLESS_PERSONS_EXPLAINED.md` |
| Delete them? | Only the provably obsolete ones. The purge keeps any with history and any it cannot explain. | §6 there |
| Defect E — fixable? | **728 of 774 are broken in Dentally itself** (six-digit numbers with no area code). Not our bug, not recoverable. | `DEFECT_E_UNUSABLE_PHONES.md` |
| Practice Mannie's 692 bad numbers? | **Excluded, not deleted** — it is the test practice, and its records re-sync from Dentally so a delete is undone on the next import. | §3 there |
| Use a patient's emergency contact to fix their number? | **No.** It is usually a different person; it would manufacture misattribution. | §2 there |
| The 18 channels with live conversations? | Released — a genuine owner stays linked, so no thread is orphaned. 0 newly ownerless (337 → 296). | `RUNBOOK_STALE_CHANNEL_REPAIR.md` §7 |

---

## 6. Traps that cost real time — read before debugging anything

| Trap | Consequence |
|---|---|
| **A check the fixed code guarantees is a tautology.** Comparing a moved record's name to its target Person's name always agrees, because `Person.resolve` matches on name. | Reported "0 problems" on a repair that had misfiled rows. Verify with DOB / contact evidence the resolver does NOT use. |
| **The generic FK is stronger than any name match.** A history row's `content_type` + `object_id` names the record it is about, and that record owns it. | Name matching put *"Treatment Plan created for Kerrie Maclean"* on **Zuri** Maclean, a relative. |
| **Two content_type conventions.** NoteHistory stores a STRING (`treatment_plan`); Activity/ActivityLog store a ContentType ID. | Handling only one silently resolves nothing for the other — reported "3 usable links" out of 96. |
| **`.update()` fires no signals** and bypasses `auto_now`. | Timestamps cannot date a re-pointing; and reverting with `.update()` leaves a half-fixed state that looks broken. |
| **`Patient.save()` splits E.164** into `country_code` + national number. | Comparing `patient.phone_number` to `channel.canonical_value` reported 13/13 PROBLEM on a correct repair. Re-canonicalise instead. |
| **A silent `continue` on a failed lookup.** | The purge classified 195 Persons as unexplained because two model lookups raised `LookupError` and were swallowed. Fail loudly. |
| **Deleting a PersonChannel can demote `is_primary`.** | 232 people would have shown "no email address on file" in the inbox. The repair now re-promotes in the same transaction. |
| **`treatmentpath_weld_sim` is CONTAMINATED** — a prod snapshot with a local unweld run on top (350 Persons created in a two-minute window; `max(id)` higher than a *newer* dump). | Produced a convincing false trail suggesting the collapses formed after the August fixes. Never use a local DB as a baseline without checking for rows created after its snapshot date. |
| **Dentally's `search` parameter is silently ignored** — HTTP 200, same unfiltered page. | Never treat "no result" as an answer. Page the full list and match locally. |
| **`DentallyIntegration.api_key` is a property that already decrypts.** | Passing it to `Fernet.decrypt()` double-decrypts and raises `InvalidToken`, which looks exactly like a wrong key. |
| **Reserved/invalid test numbers.** `+447700900000` (Ofcom fictitious) and `+447700555666` do not canonicalise. | Fixtures silently create no channel, and tests pass while exercising nothing. Assert your own setup. |
| **`os.environ["DB_NAME"]` lies after `django.setup()`.** `settings.py:561` runs `load_dotenv(override=True)` AFTER `DATABASES` is built. | A script printing the env var labels every run with the `.env` database while querying the one you passed. Use `connection.settings_dict["NAME"]`. |
| **Dentally 401 is not an error to `repair_unusable_phones`.** | `SKIP_API_ERROR`, "would update 0", exit 0 — an apply with a bad key is a silent no-op. Abort on any SKIP_API_ERROR. |
| **"Every created Person is named after a record" is not enough.** The ANCHOR keeps the id and can be named after a human it no longer holds. | 10 such anchors per run, 535 pre-existing, invisible to every audit. Check every Person holding records, on the tuple key. |
| **"Has a primary" is not "has the right primary".** | A/B made the right value available; nothing chose it. 5,537 people's SMS/email kept going to a relative's value. |
| **A merge key built from ANY record a Person holds welds relatives.** A misfiled record makes two people "match". | `dedupe_persons` merged 17 humans into a family member on its first rehearsal run; all five defect counts stayed 0. Only the row-by-row control comparison saw it. |
| **`Person.merge` carries the loser's stale links to the winner.** | 17 new C rows after the dedupe phase. Run the C repair again after any merge. |
| **A verifier that compares the `(first, last)` tuple calls the split-tolerant merge a regression.** | 10 "regressions" that were `Ken\|(Kenneth) Judge` → `Ken (Kenneth)\|Judge` — the fix for audit #1. Both `_prod_sim` verifiers now compare the joined key. |

---

## 7. Still open after this runbook

1. **`identity_guard` is not wired to anything** — it exits 1 on damage but no cron
   or CI runs it. A baseline of the 115 residual issues was written on 2026-09-07
   (`docs/identity-snapshots/prod-2026-09-07/identity-guard-baseline.json`); run
   `identity_guard --baseline <that file>` on a schedule so only NEW damage fails.
2. **63 patients** in real practices whose numbers are unrecoverable at source —
   48 at Danbury. Needs a human to ask the patient; correct in **Dentally**, since
   a local edit is overwritten on the next sync.
3. **84 active Dentally patients with no local patient record** — the app cannot see
   people the practice is currently treating. A sync gap, separate from identity.
4. **131 history rows** on 46 Persons whose owner cannot be determined without
   guessing. Listed in the plan CSV for human review.
5. ~~305 person/kind rows with more than one primary channel~~ — cleared by the
   `repair_primary_channels --prefer-own` step (238 on the rehearsal → 0).
6. The repair code is **uncommitted** locally and NOT in the deployed `main` — the
   production run used a synced copy of the working tree, not the deployed checkout.
   Commit and deploy it, or the next run of any of these commands on the server
   uses the old, unguarded versions.
7. ~~252 humans that exist twice~~ — now the dedupe phase in §2 (rehearsed,
   guarded, reversible): SPLIT 265 → **53**. The remaining 53 are the
   `[partial]`-name groups and pairs with no shared phone; review by hand.
8. **4 Persons named after neither of the two humans they hold** (e.g. 81175
   "Afaaf Rajbee" holding Mohammed Akhter / Mohammed Akhte). In the
   `--fix-misnamed` plan CSV as the residue; a person has to decide.
9. ~~Leads on a relative's Person with no DOB to prove it~~ — fixed:
   `split_collapsed_persons --min-names 2` now also splits a two-name Person when
   the second name has its OWN live Person on a shared channel (positive evidence
   the DOB cannot give). Rehearsal: intake 3097 Sarah Franklin rejoined Person
   119389 "Sarah Franklin"; 5 + 6 records in total. Two-name Persons where no
   such twin exists (77 on the rehearsal) stay untouched — no evidence, no move.
10. **Defect E is unrehearsed on the current code** (needs the production key);
    when it runs, save the Dentally API responses to a file.

---

## 8. Detail documents

| Document | Covers |
|---|---|
| `RUNBOOK_COLLAPSED_PERSON_REPAIR.md` | D and D2 — cause, repair, the full DEV run, the name-mangling defect the audit caught |
| `RUNBOOK_STALE_CHANNEL_REPAIR.md` | C — additive-only linking in both languages, the messaging guard |
| `RECORDLESS_PERSONS_EXPLAINED.md` | The 25,087 — proof they are migration debt, and the housekeeping |
| `DEFECT_E_UNUSABLE_PHONES.md` | E — per-patient verdicts against live Dentally, and the `*_normalized` bug |
| `PROD_REHEARSAL_2026-09-05.md` | Rehearsal 1, the control-group bug, the cross-reference proving nothing was lost |
| `PROD_REHEARSAL_2_2026-09-06.md` | Rehearsal 2 — this document run as written, rehearsal 1 vs 2 compared record by record, the three shape findings and the tools that check them |
| `CONTACT_IDENTITY_FINDINGS_2026-09-04.md` | Original investigation, including conclusions that turned out wrong |
| `CROSS_LANGUAGE_PARITY.md` | Every rule implemented twice (Django + Go) and the fixtures pinning them |

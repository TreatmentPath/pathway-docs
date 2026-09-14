# Patient identity audit — 44 findings

**2026-09-06.** Every finding below was **independently verified by me** against
`prod_control` (a restored copy of the 2026-09-04 production database) or by reading
the code at the cited line. Where my measurement differed from the original report,
**my number is the one shown** and the difference is noted.

Nothing here is fixed unless the entry says so.

**Second verification pass, 2026-09-06 (later the same day).** Every finding was re-checked
by reading the code at the cited lines in the current checkout and re-running the counts
against `prod_control`. **All 44 stand.** Each finding now ends with a **Fix** block naming
the exact file/line/helper to change and the check that proves it. Corrections from this
pass, none of which changes a verdict:

- **#1** — a looser query (any two live Persons with the same lower-cased joined name and a
  different split) finds 31 groups / 67 Persons; 17/35 is the strict figure.
- **#5** — joining every summary row to same-named patients gives 349 rows / 717 patients;
  141/292 counts only rows currently displayed on the day list.
- **#13** — `automations/actions.py:688` reads the **raw** `mobile_phone` first, not
  `_normalized`; only `:396` and `dentally_views.py:819` have the described defect.
- **#38** — `call_records` has no `patient_name` column; the stored name is
  `dynamic_variables->>'patient_name'`.
- **#41** — all 8 remaining Person-less call-agent intakes have `first_name='Unknown'`.
- **#10** — confirmed by execution that `canonical_phone_e164('7911123456','GB')` returns
  `+447911123456`, so the fix is a helper swap, no data change.
- **#18/#19/#20/#28/#33/#34** are one function and one create path in
  `onlineBooking/services.py`; they share a single Fix block under #18. **#35** is folded
  into #17's fix.

**How to read this:** "the identity graph" is `Person` → `PersonChannel` →
`ContactChannel`. A `Person` is meant to be ONE human. A `Patient` / `Intake` /
`Nurture` is a record ABOUT that human. Most of these bugs either put two humans on
one Person, or fail to find a Person that exists.

---

## STATUS — what is fixed

Fixed findings keep their original text; the work done is in an **AS BUILT** block at
the end of each one. Read that block, not just the Fix block — **every fix so far
differed from the plan**, sometimes by being larger.

| # | Finding | Status | Measured effect |
|---|---|---|---|
| 1 | Name-split mismatch makes duplicate Persons | ✅ fixed, repair rehearsed | 17 groups / 35 Persons → **2 / 5** (the 2 share no channel, by design) |
| 2 | Recall list opens the wrong person's record | ✅ fixed | 2,184 rows opening a different human → **0** |
| 3 | "Add family member" merges two humans | ✅ code fixed | stops new damage; 2,357 existing Persons are a `split_collapsed_persons` job |
| 4 | Treatment plan reassignable across practices | ✅ fixed | boundary now holds on update, and clearing is a 400 not a 500 |
| 5 | AI summaries shared between same-named patients | ✅ fixed (Go + Django) | migration `0168`: 14,432 stamped, **146 unattributable deleted**, 199 orphans |
| 6 | Family lookup leaks patients across practices | ✅ fixed | 6,905 shared emails; intake 1339 no longer sees practices 13/16 |
| 7 | Lookup computes a phone key the writer never writes | ✅ fixed | 9 call sites (audit listed 3); 98.8% of channels are E.164 |
| 8 | Dormant recall tab loses the contact | ✅ fixed | 1,073 affected names; also fixed a no-name-check bug in the batched path |
| 9 | `Person.resolve` called without a date of birth | ✅ fixed | 3 of 7 call sites were open; AST guard added |
| 10 | `+GB…` phone numbers, SMS silently never sends | ✅ fixed | 4,208 GB-coded patients; 4 sites (audit listed 2) |
| 11 | Merge suggester compares phones as raw strings | ✅ fixed | Sibanda pair (practice 19) now suggested; emails never matched |
| 12 | Lead can be stripped of all contact details on update | ✅ fixed | 3 existing rows listed for staff (intakes 684, 718, 641) |
| 13 | Sites prefer Dentally's broken `*_normalized` phone | ⚠️ code fixed, **data repair outstanding** | 8 corrupted rows (Malta, Singapore ×2, Tanzania, Ireland, US ×2) |
| 14 | Workflow patient lookup case-sensitive on email | ✅ fixed | 1,565 mixed-case emails; also a 10th `lookup_key` site |
| 15 | Bad/foreign practitioner assignment returns success | ✅ fixed | foreign practitioner was actually being written |
| 16 | Dentally bridge scores matches across practices | ✅ fixed | old lookup matched **0** rows (JSON-type bug); now 39,844, with 373 cross-practice false matches prevented |
| 17 (+35) | Activity History shows another patient's clinical notes | ✅ fixed | 318 same-practice collisions; 5 real leaked notes; 3 sites incl. the CSV/PDF export |
| 18–20, 28, 33, 34 | Online booking: wrong family member, dead phone match, name split on payment, unread patient_type, Person-less create, archived matches | ✅ fixed (one shared fix) | 3,882 shared-email groups; 9 Taylor children on one email; 0 rows today — fires on launch |
| 21 | Two more first-space splitters feeding `Person.resolve` | ✅ verified fixed by #1 + splitter de-duplicated | 1,848 exposed Persons; lane is LIVE |
| 22 | Clinical notes and letters accept any practice's patient | ✅ fixed | 0 rows today; also removed 2 dead `perform_create`s and a duplicated model field |
| 23 | Tasks scopes the user FKs, not the patient FKs | ✅ fixed | 0 mismatched today; a foreign plan dragged its patient in |
| 24 | Appointments accepts any patient and any clinician | ✅ fixed | + the display fix that made #18 invisible on the diary |
| 25 | `by_contact` returns the oldest family member's log | ✅ fixed | 162 ambiguous email groups / 364 logs; returns 300 + candidates |
| 26 | Medical-history portal downgrades DOB verification to name | ⚠️ code fixed, **894-row backfill outstanding** | 2,336 NULL DOB columns, 894 recoverable from `meta_data` |
| 27 | Read-only `patient_name` PATCH silently no-ops | ✅ fixed | API returned a key it would not accept |
| 29 | Hand-rolled phone normalisation in the consent SMS path | ✅ fixed with #10 | one of the 4 sites #10 swept up |
| 30 | Medical-history submission never checked verification | ✅ fixed | signed 30-min token bound to the link; migration `0005` |
| 31 | Verification code not bound to the plan it unlocks | ⚠️ mostly fixed, **frontend must send `treatment_plan_id`** | 186 plans writable by one code; codes never expired |
| 32 | The practice check fails open | ✅ fixed | 16 of 213 users have no `current_practice` |
| 36 | Day-list history counts cross practice boundaries | ✅ fixed (Go + index) | 110,936 rows wrong; Barbara Hole 180 → 0. Calibration **re-check** is runbook step 4, not an open defect |
| 37 | Marketing profile represents an arbitrary member of a fused Person | ✅ fixed | 2,357 profiles / 5,802 patients; ambiguous ones now excluded from campaigns |
| 38 (+44) | Call agent picks and persists the first human on a shared phone | ⚠️ code fixed, **review queue outstanding** (runbook step 5) | Intake 396 = Carly's call filed as Billy Wright. Real surface is **643 calls on 295 ambiguous keys** (audit said 194/98); the triage query narrows it to 49 rows, dominated by nickname noise (Liz/Elizabeth) — a review queue, not a defect list |
| 39 | Call-agent DOB field accepted and ignored | ✅ fixed | 493 name groups / 1,021 patients differ on DOB; 4 William Smiths in practice 21 |
| 40 | ActivityLog writes accept foreign Persons and objects | ✅ fixed | 0 rows today; both create AND update paths |
| 41 | Go call-agent Intakes inserted outside the identity graph | ✅ fixed | 8 of 1,535 still Person-less; now resolved at write time |
| 42 | Recall sync re-splits Dentally's correct names | ✅ fixed | 1,131 rows; next full sync rewrites them |
| 43 | Consent management crosses practices on read and write | ✅ fixed | 4 live cross-practice records; also fixed duplicate rows from a missing `.distinct()` |

**Nothing is committed.** Two things need a human:
- **Migration `0168` is unapplied.** It deletes 146 AI-summary rows (regenerable; see #5).
- **#5 must deploy Go and Django together** — the `ON CONFLICT` target and the unique
  constraint change are one atomic pair.

---

## 1. ✅ FIXED 2026-09-06 (was PROVEN / HIGH) — the name-split mismatch has already made duplicate Persons

**Where:** `TreatmentPlan/models.py:604-607` (`Person.resolve`), the same tuple
comparison in `TreatmentPlan/contact/identity_resolution.py:139-142`, and an
upstream writer in `EmailServiceGo/internal/dentally/migration/service.go:1613-1622`.

**Plain English:** a person's name is stored as two columns, first and last. Two
systems disagree about where to cut the name in half. `"Ken (Kenneth) Judge"` gets
stored once as first=`Ken (Kenneth)` last=`Judge`, and once as first=`Ken`
last=`(Kenneth) Judge`. Joined they are identical; as two columns they are not.
`Person.resolve` compares the two columns as a pair, so it sees two different people
and creates a second Person. Every re-import repeats it.

The Go Dentally migration is one concrete source of that bad pair. It starts with the
correct separate Dentally `firstName` / `lastName`, joins them into `displayName` at
`service.go:1058` and `:1122`, then `parseDisplayName` splits the joined string again
at the first space. This is the same defect, not a separate numbered finding.

**Verified:** 17 (practice, joined-name) groups = **35 live Persons** that are the
same human twice. Confirmed example, practice 16:
`74615 "Ken (Kenneth)|Judge"` and `74616 "Ken|(Kenneth) Judge"`.
Exposed population: **1,686 patients** have a space in their first name.

**Why it matters:** this is the bug that blocked a real treatment-plan request —
the API said *"No patient named 'Christopher (Chris) Mooney' found. Existing
patients: Christopher (Chris) Mooney."* The same name, quoted as both missing and
present, because the split differs.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `Person.resolve` compares `canonical_name_key(first, last)` as a **tuple**
(`contact_keys.py:56-58`, consumed at `models.py:604`); Go does the same via
`personname.CanonicalKey` in `linkPatientToPersonAndChannels` (`service.go:1430-1436`).
Persons 74615/74616 exist exactly as stated. A looser query (any two Persons in one
practice whose lower-cased joined name is equal but whose split differs) finds **31
groups / 67 Persons**; the 17/35 above is the conservative figure.

1. **Stop manufacturing the bad split (Go writer).** `linkPatientToPersonAndChannels`
   (`service.go:1313`) takes `displayName string` and re-splits it at `:1396` with
   `parseDisplayName`. Change the signature to `firstName, lastName string`; the two
   callers (`:1058`, `:1122`) already hold Dentally's separate `firstName`/`lastName`
   — pass them through. Delete `parseDisplayName` (`:1613`; `:1396` is its only caller).
2. **Make the comparison split-tolerant (both languages, same commit).** Add
   `canonical_full_name_key(first, last)` next to `canonical_name_key` in
   `contact_keys.py` = `canonical_name_part(f"{first} {last}")`, and the Go twin
   `personname.CanonicalFullKey` in `pkg/personname/personname.go:33`. Compare on it in:
   `Person.resolve` loop (`models.py:604`), `identity_resolution.py:139-142`
   (`name_match`), and the Go candidate loop (`service.go:1430-1436`). Storage of
   `first_name`/`last_name` is unchanged — only the *equality test* changes. Pin it with
   new cases in `TreatmentPlan/tests/contact_key_fixtures.json`:
   `("Ken (Kenneth)","Judge") == ("Ken","(Kenneth) Judge")`.
   `_dob_conflict` still applies, so father/son stay separate.
3. **Repair the existing rows.** Add a `--joined-name` grouping mode to
   `TreatmentPlan/management/commands/dedupe_persons.py` that groups live Persons by
   `(practice_id, canonical_full_name_key)` sharing at least one channel, then routes
   them through the existing merge path. Expect 17 groups (35 Persons) at the strict
   definition; re-run the SQL under "Verified" and expect 0.
4. **Regression test:** create a Patient `"Christopher (Chris)"/"Mooney"` then resolve
   `"Christopher"/"(Chris) Mooney"` on the same channel → same Person, `created=False`.

**AS BUILT — 2026-09-06 (uncommitted).** All four steps done, plus four things the
plan above did not name:

- **The pair key had FIVE more callers than the plan listed.** `canonical_full_name_key`
  is now the one definition of the joined key; `dedupe_persons.norm_name`,
  `dedupe_report.norm_name`, `split_collapsed_persons._name_of`, `identity_guard._scan`
  and `identity_defects` all routed through it (each had grown a private
  `f"{part(f)} {part(l)}".strip()`). Equivalence with the old composition is pinned by
  `CanonicalFullNameKeyTests.test_is_the_one_join_the_repair_tooling_uses`.
- **Two more comparison sites, both fixed:** `contact/signals.py` treated a moved split
  as "this record was corrected to a different human" and re-resolved it; and
  `views/contact_merge_views.py` (`_records_look_like_same_person`,
  `_matching_patient_for`) compared first and last separately, so the merge UI could
  not suggest undoing the duplicate the resolver had just created. Both now use the
  joined key. `identity_resolution._normalize_name_part` was deleted, not left dangling.
- **`Person.resolve` now orders candidates `by id`.** It had no ordering at all, so with
  two matching candidates the winner was whatever Postgres returned. Go's
  `livePersonsOnChannels` was already oldest-first; the widening in step 2 makes a
  two-match case reachable, so the two services had to agree.
- **Winner selection (step 3).** `_pick_winner` gained a tie-break above `-pid`: a
  Person whose `(first, last)` split matches its own Patient row (Dentally's real split)
  outranks one whose split was mangled in transit. Applied to ALL passes, not just
  `--joined-name` — otherwise half the repaired rows keep the mangled spelling.

*Not done, deliberately:* `purge_recordless_persons` still keys on the `first|last`
PAIR (`:60`). Widening it would make it DELETE more Persons, which is out of scope for
this finding and wants its own rehearsal.

*Measured on `prod_control` before the change* (SQL under "Verified", reproduced
exactly): 17 groups / 35 live Persons. 15 of the 17 share a ContactChannel — all 15
share a **phone** channel — and are therefore in scope for the repair; the other two
('mark richard smith' p16, 'kathleen joyce myers' p21) share nothing and are left
alone by design. 7 of the 17 contain a Person with NO records, which is why the
record-keyed passes never saw them and `--joined-name` had to key on Person rows.

*Tests (all shown red before green):* Go —
`TestCanonicalFullKeyParityWithDjango`, `TestCanonicalFullKeyIgnoresWhereTheNameWasCut`,
`TestPersonResolve_NameSplitDoesNotForkIdentity` (red run reproduced the production
symptom: Persons 161131 "Ken (Kenneth)|Judge" and 161132 "Ken|(Kenneth) Judge"),
`TestPersonResolve_DifferentNamesStillFork`. Django —
`CanonicalFullNameKeyTests` (12 shared-fixture cases),
`ResolverIgnoresWhereTheNameWasCut` (4), `FinderAgreesWithTheResolverAboutTheSplit`,
`DedupeJoinedNameFindsTheRecordlessTwin` (3).

*Repair rehearsed on `prod_control`, 2026-09-06,* inside a transaction that was
rolled back (disk is at 99%, so no copy could be made; `prod_control` is verified
unchanged afterwards and remains the audit baseline):

```
manage.py dedupe_persons --only-joined-name --commit
  BEFORE: 17 split-mismatch groups / 35 live Persons
   AFTER:  2 groups / 5 Persons        <- the 2 with NO shared channel, by design
AFTER x2:  2 groups / 5 Persons        <- fixed point, idempotent
15 Persons merged, 0 skipped
```

Per-record proof over all 15 groups (28 records, 28 channels), not just the totals:
every loser retired to the right winner; no Intake/Nurture/Patient lost; no channel
lost; a `ContactMergeLog` exists for each merge (undoable); no practice crossed; and
every survivor carries the split its own Patient row carries.

**Two guards the dry run forced, both kept:**
- `--joined-name` reports but does NOT merge groups whose Persons are spelled
  identically — that is a different defect. 85 such groups exist; **61 of them are
  two different humans**, caught only by the DOB guard, and the remaining 24 carry no
  DOB evidence either way. `--include-same-spelling` opts in; it should stay off.
- A single-token name is not enough evidence. Practice 16 has two Persons called just
  "charlie" on one phone; that is a household. The pass now needs ≥2 name words, the
  same bar `_records_look_like_same_person` already held.

*Still to run on production:* `dedupe_persons --only-joined-name` (dry run), then
`--commit`. Expect 15 groups, 0 DOB skips.

---

## 2. ✅ FIXED 2026-09-06 (was PROVEN / HIGH) — the recall list opens the wrong person's record

**Where:** `dentallyIntegration/serializers.py:1444-1478` (`RecallRecordSerializer._get_patient`).

**Plain English:** a recall row knows exactly which Dentally patient it is for
(`dentally_patient_id`). The code ignores that. It finds the Person, then returns
**the most recently created patient on that Person**:

```python
patient = contact.patients.filter(practice_id=obj.practice_id) \
                 .order_by("-created_at", "-id").first()
```

`dentally_patient_id` is used only as a *cache key* — never to match. On a Person
holding a whole family, you get whichever family member was added last.

**Verified:** **2,561 recall records** resolve to a patient with a different name.
Real rows:

| Recall is for | Opens the record of |
|---|---|
| George Smallwood | Nicola Smallwood |
| Ayperi Lyons | Alara Lyons |
| Kenneth Seymour | Pauline Seymour |
| Anjayen Ananda | Lucksha Ananda |

*(The original report said 5,931; I measured only rows where the resolved patient is
genuinely a differently-named human, which is the conservative figure.)*

**Why it matters:** staff click a recall and land in a relative's clinical workspace.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `_get_patient` (`serializers.py:1444-1478`) uses `dentally_patient_id`
only as the cache key; both branches sort by `(-created_at, -id)`. Person 74767: George
created 22:11:59, Nicola 22:18:02 on 2026-07-27 → Nicola wins.
`meta_data->'id'` is a JSON **number** on 58,884 patients (1,442 have none).

1. Match on the Dentally id first, Person second, "most recent" never:
   ```python
   key = obj.dentally_patient_id
   if "patients" in getattr(contact, "_prefetched_objects_cache", {}):
       ps = [p for p in contact.patients.all()
             if p.practice_id == obj.practice_id
             and str((p.meta_data or {}).get("id")) == str(key)]
   else:
       ps = list(contact.patients.filter(practice_id=obj.practice_id,
                                         meta_data__id=key))
   patient = ps[0] if len(ps) == 1 else None
   ```
   Compare as `str()` in Python (the bridge command at
   `bridge_dentally_identity.py:172` already matches ids as strings); in the ORM branch
   pass the int — the JSON value is a number.
2. When there is no id match, return **None**, not a sibling. A blank "open patient"
   link is recoverable; opening a relative's clinical workspace is not. If a fallback
   is wanted, fall back to `Patient.objects.filter(practice_id=..., meta_data__id=key)`
   *without* going through the Person at all — the Person is the thing that is shared.
3. Verify: re-run the 2,561-row query; George/Ayperi/Kenneth/Anjayen rows must resolve
   to their own record or to nothing.

**AS BUILT — 2026-09-06 (uncommitted).** Done, and it turned out to be two defects in
one method plus a third in its neighbour:

- **`_get_patient` now matches on `dentally_patient_id` and nothing else**, and does
  **not consult the Person at all**. The Person was only ever a detour, and it is the
  shared thing — routing identity through it is what produced the defect. Dropping it
  also fixes rows where no channel resolved a Person but the Patient existed by id all
  along. Comparison is on `meta_data->>'id'` as TEXT via `KeyTextTransform`, which is
  the expression the unique index `patient_practice_dentally_uniq` is built on, so it
  is an index hit. That constraint also means the match is at most one row, so the
  audit's "`ps[0] if len(ps) == 1 else None`" concern cannot arise.
- **No fallback.** Nothing matched → `None`.
- **`RecallRecordViewSet._build_patient_map`** resolves a whole page in ONE indexed
  query (context key `patient_by_dentally_id`, the same name
  `DayListAppointmentSerializer` already uses), so the list view keeps its per-row
  query cost at zero.
- **`_get_contact` carried finding #1** — it cut `patient_name` at the first space and
  compared the (first, last) PAIR, so a recall for "Ken (Kenneth) Judge" produced
  ("ken", "(kenneth) judge") and never matched a Person stored the way Dentally sends
  it. Now compares `canonical_full_name_key`. **This is a prerequisite for #1, not a
  nicety:** #1 makes Persons carry Dentally's split, which is precisely the split a
  first-space cut can never reproduce, so without this #1 would have silently emptied
  the recall list's call logs, messages and family context.
- **Knock-on fixed for free:** `get_family_members` excludes whoever `_get_patient`
  returns, so it used to exclude the RELATIVE and list the actual patient as their own
  family member.

*Sibling checked, already correct:* `DayListAppointmentSerializer._get_patient`
(`serializers.py:509`) matches on `meta_data__id` and only falls back on a genuine
same-id duplicate. It was the model for this fix.

*Measured on `prod_control`* (my query is phone-channel-only, so my "before" is 2,184
where the audit's, which also used email, was 2,561 — same defect, conservative count):

| | before | after |
|---|---|---|
| recall rows opening a **different human** | 2,184 | **0** |
| rows whose own record exists by Dentally id | 2,152 | resolved correctly |
| rows resolving to nothing | — | 815 (no Patient carries that id) |
| rows where the *name* differs on the same id | — | 39 |

The 39 are all one human under two spellings — married names (Megan Brinkworth /
Megan Church, Emily Spoore / Emily Cocksedge), nicknames (Sue / Susan), typos
(Embling / Endling, Shaves / Shades) and double spaces. The Dentally id is
authoritative; these are name drift in the recall mirror, not a mis-match.

*Tests:* `dentallyIntegration/test_recall_patient_identity.py`, 8 tests, **7 red under
the old `_get_patient`** and the 8th red under the old `_get_contact` split.

---

## 3. ✅ FIXED 2026-09-06 (was PROVEN / HIGH) — "Add family member" deliberately merges two humans

**Where:** `TreatmentPlan/views/patient_views.py:744` (`POST /patients/` with
`link_as_family=true`).

**Plain English:** adding a spouse or child to an existing patient runs
`patient.person = person` — it gives the new human the **anchor patient's Person**.
But a Person is defined as one human (`models.py:342-347`), and `Person.resolve`
explicitly refuses to do this: a shared phone with a different name must create a
NEW Person in a shared **Household** and *never* merge (`models.py:582-585`). The
Household mechanism already exists, and this endpoint even creates one — it just
shares the Person as well.

**Verified:** **2,357 Persons** hold patients with genuinely different names, e.g.
person 74767 = `george smallwood` + `nicola smallwood`.

**Why it matters:** this is the upstream cause of finding 2, and it is a supported
feature actively creating the exact damage the D/D2 repairs spent this week undoing.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `patient_views.py:744-745` does `patient.person = person` (the anchor's),
then `:751-764` re-points every Intake/Nurture sharing the **new** patient's email/phone
at the anchor's Person too. 2,357 Persons hold differently-named patients; person 74767
= 120753 George + 123091 Nicola Smallwood.

1. Give the new human their **own** Person and household them with the anchor:
   ```python
   channel_ids = []
   for kind, raw, cc in ((ContactChannel.PHONE, patient.phone_number, patient.country_code),
                         (ContactChannel.EMAIL, patient.email, None)):
       if raw:
           ch = ContactChannel.get_or_create_channel(practice, kind, raw, cc)
           if ch: channel_ids.append(ch.id)
   new_person, _ = PersonModel.resolve(
       practice, patient.first_name, patient.last_name, channel_ids,
       dob=patient.date_of_birth,           # see #9
   )
   patient.person = new_person
   patient.save(update_fields=["person"])
   ```
   `Person.resolve` (`models.py:582-585`) already puts a different-named Person sharing a
   channel into the anchor's Household. When the two share **no** channel, do it
   explicitly: reuse the household block that already exists at the end of this view
   ("Ensure the shared Person sits in a Household", `:~800`) and add `new_person` to
   `anchor.person.household`.
2. Change the Intake/Nurture re-link (`:751-764`) to `update(person=new_person)`.
3. `existing_members` (`:772`) must become "patients whose Person is in the same
   household", not `Patient.objects.filter(person=person)`.
4. Data: the 2,357 already-collapsed Persons are defect D2 → `split_collapsed_persons`
   (note the multi-word-first-name mangling recorded in the repair verification; fix #1
   first or that repair re-creates the split problem).
5. Test: POST `link_as_family=true` with a different name → `patient.person_id !=
   anchor.person_id` and `patient.person.household_id == anchor.person.household_id`.

**AS BUILT — 2026-09-06 (uncommitted).** Code fixed; the data half is left to the
existing repair tooling (see below).

The fix turned out to be smaller than the plan, because most of it already worked:
`serializer.save()` runs the pre-save signal (`contact/signals._assign_person_on_save`),
which had **already resolved the new patient to their own correct Person** from their
own email/phone. The next line then threw that away with `patient.person = person`.
So the fix is mostly to STOP overwriting it — no second resolve call is needed except
when the new record carries no contact detail at all (a child with no phone or email),
which is the only case that reaches a `Person.resolve` here.

- **`patient.person = person` removed.** The new human keeps the Person the signal gave
  them. `Person.resolve` already puts a differently-named Person sharing a channel into
  the anchor's Household, so the ordinary case needs nothing further.
- **Household made explicit for the no-shared-channel case.** Both Persons are put in
  one household — reusing the anchor's, the new Person's, or a fresh one — because
  nothing else links them when they share no phone or email.
- **The Intake/Nurture re-link now points at the NEW patient's Person.** Those rows are
  matched on the *new* patient's own email/phone, so they describe the new human;
  pointing them at the anchor was the same weld by another route.
- **`existing_members` is now household-scoped**, not `Patient.objects.filter(person=person)`
  — that only worked because everyone was being welded onto one Person.

*Behaviour deliberately unchanged:* the relationship mirroring (spouse↔spouse,
child→parent) still works exactly as before, and there is a test pinning it. The point
is to stop welding, not to stop linking.

*Tests:* `TreatmentPlan/tests/test_add_family_member_identity.py`, 6 tests, **4 red
under the old weld** (the other 2 are guards that are correctly green either way —
"they share a household" is trivially true when they are one Person, and "the
relationship is still recorded" is a feature-still-works check).

*Data half, NOT run here:* 2,357 Persons holding differently-named patients on
`prod_control` — confirmed exactly. That is defect D2 and belongs to
`split_collapsed_persons`, which already exists and is rehearsed in the identity-repair
runbook. Finding #1 was the prerequisite (its `_name_of` now routes through
`canonical_full_name_key`), so that repair is now safe to run — but it is a runbook
step, not part of this code fix.

---

## 4. ✅ FIXED 2026-09-06 (was PROVEN / HIGH) — a treatment plan can be reassigned to another practice's patient

**Where:** `TreatmentPlan/serializers/treatment_plan.py` — `create()` at :876 vs
`update()` at :1063.

**Plain English:** on create, `patient_id` is popped and looked up **scoped to the
practice** (`Patient.objects.get(id=..., practice=practice)`). On update it is never
popped, so it falls straight through to `super().update()`, which sets the foreign
key directly with **no practice check**. There is no `validate_patient_id` anywhere
in the file (verified: 0 occurrences).

**Verified by code.** `PATCH /treatment-plans/<id>/ {"patient_id": <other practice's patient>}`
has nothing standing in its way. Sending `{"patient_id": null}` clears the FK, and
`TreatmentPlan.clean()` then raises an unmapped Django `ValidationError` → **HTTP 500**.

**Why it matters:** a tenant boundary that holds on create and not on update is not
a boundary.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `patient_id = IntegerField(write_only, allow_null=True)` at `:746`;
`create()` pops and scopes it (`:876`, `:901`); `update()` (`:1063-1138`) never pops it, so
it reaches `super().update()` at `:1138` and is written as-is. `grep validate_patient` →
0 hits. `TreatmentPlan.clean()` (`models.py:4060`) raises a Django `ValidationError`
from `save()` (`:4071`) → 500.

1. Add one validator that serves both paths:
   ```python
   def validate_patient_id(self, value):
       if value is None:
           return None
       practice = self.instance.practice if self.instance else \
                  self.context["request"].user.current_practice
       if not Patient.objects.filter(id=value, practice=practice).exists():
           raise serializers.ValidationError(
               f"No patient with ID {value} found in this practice.")
       return value
   ```
   and in `update()` pop it: `if "patient_id" in validated_data:
   validated_data["patient"] = Patient.objects.get(id=validated_data.pop("patient_id"))`
   (or `None`).
2. Prevent the 500: in `validate()` reject a payload that would leave both `patient` and
   `non_registered_patient` empty on the instance, mirroring `TreatmentPlan.clean()`, so
   the failure is a 400.
3. Tests: PATCH with another practice's id → 400; PATCH `patient_id: null` on a plan with
   no non-registered patient → 400, not 500.

**AS BUILT — 2026-09-06 (uncommitted).** Both steps done as written, with one
simplification.

- **`validate_patient_id`** added — a field-level validator, so ONE rule serves create
  and update rather than two that can drift. Practice comes from `self.instance.practice`
  on update and `request.user.current_practice` on create; a missing practice is a
  rejection, not a bypass.
- **`update()` now pops `patient_id`** and resolves it to a `Patient` (or `None`).
  `patient_id` is a `write_only` serializer field but it is also the FK's attname, which
  is exactly why leaving it in `validated_data` wrote it straight onto the row.
- **The 500 is now a 400.** The audit suggested mirroring `TreatmentPlan.clean()`'s
  "either a registered or a non-registered patient" rule with a fallback branch for a
  surviving non-registered patient. **That branch is unreachable and was dropped:**
  `non_registered_patient` is not writable on this serializer, and the model forbids
  holding both at once — so a plan that has a registered patient has nothing to fall
  back on and clearing it always empties the plan. Writing the `unless` would have been
  dead code.

*Tests:* `TreatmentPlan/tests/test_treatment_plan_patient_scoping.py`, 7 tests, **4 red
under the old serializer** — including the one that matters:
`test_another_practices_patient_is_not_written` shows the plan actually MOVING to the
other practice's patient (`10004 != 10002`), not merely failing to be rejected. The
three green-either-way tests are the guards that the boundary has not become a wall
(same-practice reassignment still works, an omitted `patient_id` is not treated as
clearing it, and a walk-in plan can still be given a registered patient).

*Regression:* 121 tests across the seven treatment-plan suites pass.

---

## 5. ✅ FIXED 2026-09-06 (was PROVEN / HIGH) — clinical AI summaries are shared between same-named patients

**Where:** `dentallyIntegration/models.py:531-535` —
`UniqueConstraint(fields=["practice", "patient_name"])`, consumed by
`dentallyIntegration/daylist_query.py:108-119`.

**Plain English:** the AI clinical summary table is keyed by the patient's **name**,
not their id. Two patients with the same name in one practice therefore *share one
summary row* — and the unique constraint means the second one can never get their
own.

**Verified:** **141 summary rows are displayed against 292 distinct patients.**
Practice 21 has **four different William Smiths** and exactly one summary between
them. Others: David Shepherd ×3, David White ×3, Ann Smith ×3.

**Why it matters: this is the most serious finding here.** It shows one patient's
clinical summary, opportunity flags and raw notes against another patient on the day
list. Everything else in this document is a data-integrity problem; this one is a
patient-safety problem.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `UniqueConstraint(fields=["practice","patient_name"])` at
`dentallyIntegration/models.py:531-535`; `build_ai_summary_map` keys by
`patient_name` (`daylist_query.py:108-119`); practice 21 has 4 William Smiths. The
**Go writer** upserts on the same key: `ON CONFLICT (practice_id, patient_name)` at
`EmailServiceGo/internal/dentally/daylist/ai/db.go:400`. My join (every summary row whose
name matches >1 patient in its practice) gives 349 rows / 717 patients; the 141/292 above
counts only rows currently displayed.

1. Key the table by the Dentally patient id, both sides, one commit (Go↔Django parity):
   - Django: add `dentally_patient_id = IntegerField(null=True, db_index=True)`; replace the
     constraint with `UniqueConstraint(fields=["practice","dentally_patient_id"],
     name="uniq_ai_summary_per_dentally_patient")`. Migration must **also add a DB
     default/NOT NULL plan** the Go side can satisfy (Django `default=` is Python-only —
     see the schema-drift rule).
   - Go: the summary generator already selects `a.dentally_id` and joins on
     `dentally_patient_id` (`db.go:84-92`); write it into the row and change `:400` to
     `ON CONFLICT (practice_id, dentally_patient_id)`.
   - Reader: `build_ai_summary_map` keys by `a.dentally_patient_id`; the day-list
     consumer looks up by id, not name.
2. Data migration: for each existing row, resolve `(practice, patient_name)` →
   `dentally_patient_id` via `dentally_appointment`; where the name maps to **exactly one**
   id, stamp it; where it maps to several (the 141/349 rows), **delete** the row — it is
   unattributable, and `data_hash` will force regeneration per patient.
3. Verify: `SELECT practice_id, dentally_patient_id, count(*) ... HAVING count(*)>1` → 0
   rows; William Smith ×4 in practice 21 each get their own summary or none.

**AS BUILT — 2026-09-06 (uncommitted).** Done, Go and Django in one change. Plan at
`docs/superpowers/plans/2026-09-06-ai-summary-keyed-by-patient-id.md`.

**The audit understated this one.** It reads as a storage-key problem — two patients
sharing a row. The pipeline is name-keyed at the ENTRY POINT, so the summary *content*
was blended, not merely shared:

- `sync_handler.go:141` and `scheduler.go:928` drove the loop from
  `SELECT DISTINCT patient_name`. The four William Smiths in practice 21 were ONE
  iteration.
- `db.go:91` then fetched that iteration's appointments with
  `LOWER(TRIM(a.patient_name)) = LOWER(TRIM(?))`, so `RecentEncounters`, `raw_notes` and
  the AI prompt were built from all four patients' clinical data at once.
- `db.go:155` resolved the Dentally id from the name with `LIMIT 1` and no `ORDER BY`,
  so the encounter notes and every suppression rule belonged to an arbitrary one.
- `opportunity/evidence_adapter.go:73` `PatientIDForName` returns the FIRST match, so
  the deterministic opportunity flags did too.
- `scheduler.go:995` the verification sweep joined summaries to appointments ON THE
  NAME, so one William Smith's summary marked all four "done" — which is why the other
  three never got one even before the unique constraint refused them.

**What changed.** The Dentally patient id is carried through the whole pipeline; the
name is now only a display value and prompt input.

- Django: `dentally_patient_id` added (nullable, indexed);
  `uniq_patient_per_practice` replaced with `uniq_ai_summary_per_dentally_patient`;
  migration `0168`. Nullable and with no Python-only `default=`, because the Go service
  INSERTs these rows with raw SQL — and being nullable also keeps it out of
  `schemaguard.requiredColumns`, which tracks NOT-NULL no-default columns only.
- Go: `GetPatientData`, `GetExistingDataHash`, `StoreAISummary`, `UpdateRawNotes`,
  `GetAISummary` all keyed by id; `ON CONFLICT (practice_id, dentally_patient_id)`;
  all three driving loops (`sync_handler`, `scheduler`, `handler`) iterate
  `DISTINCT ON (dentally_patient_id)`; `AnalyzeOpportunitiesFromEvidence` takes the id
  instead of deriving it from the name.
- **The two on-demand endpoints address a patient by NAME in the URL**
  (`/patients/:patient_name/ai-summary`). They now accept `?dentally_patient_id=`,
  which always wins; without it the name is resolved for the date, and a name shared by
  several patients returns **409 with the candidate ids** rather than picking one.
  *Frontend follow-up:* the day list already has the id and should pass it.

*Data migration measured on `prod_control`* (14,777 summary rows):

| | rows |
|---|---|
| stamped with their patient's id | 14,432 |
| **deleted — name shared by several patients** | **146** |
| deleted — no appointment for that name | 199 |

The 146 are the unsafe ones: each was generated from several patients' clinical data
blended together, so it is not salvageable and must not be shown to anyone.
`data_hash` goes with the row, so the next sync regenerates one summary per patient.
Confirmed in the data: practice 21 has **William Smith ×4**, plus David Shepherd ×3,
John Allen ×3, Emma Smith ×3, Susan Smith ×3, David White ×3, Andrew Taylor ×3, Ann
Smith ×3 — exactly the audit's list.

*Tests:* `test_daylist_query_batching`, 16 pass. Red run with the reader put back on the
name key: `'the SECOND William Smith' != 'the FIRST William Smith'` — the second patient
being shown the first patient's clinical report, which is the defect stated exactly. The
same test also proves the constraint moved: creating two same-named summaries would have
raised `IntegrityError` under `uniq_patient_per_practice`. Go: `go build ./...`,
`go vet`, and all `internal/dentally/...` + `internal/db/...` tests pass.

*Pre-existing and unrelated:* the 12 failures in the wider `dentallyIntegration` suite
are byte-identical to the set measured before this change.

---

## 6. ✅ FIXED 2026-09-06 (was PROVEN / HIGH) — family lookup leaks patients across practices

**Where:** `TreatmentPlan/selectors.py:38-56` (`build_family_maps`), plus
`serializers/intake.py:339,375` and `serializers/nurture.py:287,322`.

**Plain English:** the family-members lookup matches patients by email or phone with
**no `practice=` filter** — the code comment even says *"Mirrors the legacy
(non-practice-scoped) lookup"* — while the surrounding code uses `intake.practice`
forty lines earlier.

**Verified:** **6,905 email addresses exist in more than one practice.** Intake 1339
(practice 21) returns patients from practices 16 and 13, including their names and
family relationships.

**Why it matters:** one practice sees another practice's patient names in a family
panel. Same rule broken as finding 4, different route.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `selectors.py:38-56` has no `practice=` (the comment at `:39` says so);
`intake.py:339` and `:375` and `nurture.py:287`/`:322` query `Patient.objects.filter(...)`
unscoped. 6,905 lower-cased emails exist in >1 practice.

1. `build_family_maps(items)` → `build_family_maps(items, practice)`; add
   `.filter(practice=practice)` to the `fam_patients` query at `:49`. Update the three
   callers to pass the practice they already hold: `intake_views.py:290`,
   `nurture_views.py:245`, `custom_stage_views.py:255`. Delete the "Mirrors the legacy
   (non-practice-scoped)" comment.
2. Detail-view fallbacks: prefix `practice=intake.practice` / `practice=nurture.practice`
   at `intake.py:339`, `:375` and `nurture.py:287`, `:322`.
3. Test: Intake 1339 (practice 21) → `family_members` contains no practice 13/16 patient.
   (The practice-scoped-merging rule in the project memory forbids any cross-practice
   identity read; this is one.)

**AS BUILT — 2026-09-06 (uncommitted).** Done, plus four unscoped queries the audit did
not list.

- **`build_family_maps(items, practice)`** — practice is a REQUIRED positional
  argument, and `None` raises `ValueError`. An optional argument with a permissive
  default is exactly how this defect would come back; there is a test for the raise.
  The "Mirrors the legacy (non-practice-scoped) lookup" comment is gone.
- Both of its queries are scoped: the `fam_patients` match AND the
  `members_by_household` fetch. The three callers (`intake_views:290`,
  `nurture_views:245`, `custom_stage_views:255`) each already had `practice` in scope
  from `get_user_practice_or_none()` a few lines earlier.
- **Detail-view fallbacks** in `IntakeSerializer` and `NurtureSerializer`
  (`get_is_family` and `get_family_members`) scoped — these are what a single-record
  GET uses, so the list view alone would have left the leak reachable.
- **Four more unscoped `Patient` reads found while tracing** — the `fam_patients_qs`
  household/person fetches in both serializers. A Household belongs to one practice so
  these were not a live leak, but they are now explicit. Worth noting *why*: the
  original leak in this same method happened precisely where someone reasoned the
  scoping was implied. (The other two nearby lookups, `intake.py:293`/`:301`, were
  already scoped through a shared `filters = Q(practice=...)`.)

*Measured on `prod_control`:* **6,905 email addresses exist in more than one practice.**
Intake 1339 (practice 21) reproduced exactly as reported — the unscoped lookup returns
Jonathan and Aayan Beacher from practices **13 and 16**.

*Tests:* `TreatmentPlan/tests/test_family_lookup_practice_scoping.py`, 8 tests, **7 red
with the scoping removed** — and the red output prints the leak itself,
`{'first_name': 'Aayan', 'last_name': 'Beacher'}` appearing in another practice's family
panel. The 8th is the guard that scoping did not break the feature: a lead's own
in-practice family still resolves (and still excludes the viewer themself from the
member list).

---

## 7. ✅ FIXED 2026-09-06 (was PROVEN / MEDIUM-HIGH) — a lookup computes a phone key the writer never writes

**Where:** `TreatmentPlan/contact/identity_resolution.py:106`; same shape at
`TreatmentPlan/views/intake_views.py:1546` and
`dentallyIntegration/views/recall_views.py:2081`.

**Plain English:** the *write* path builds a phone key using the practice's country
as a last resort, so it stores `+447703349940`. The *lookup* path calls the same
function **without that fallback**, so for a record with no country code it computes
the bare `7703349940` — and then compares it to the stored `+44…` value. It can
never match.

**Verified:** 98.8% of phone channels are E.164 (61,805 of 62,582), so the channel
branch of that lookup is effectively dead whenever no country code is supplied. **34
patients** miss even when passed their own stored values — e.g. patient 41664 Anna
Harland (practice 16), stored `phone_number='7703349940'`, `country_code=''`,
channel `+447703349940`.

**Why it matters:** a failed lookup ends in `decision="create_new"` — a duplicate
patient.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* the writer goes through `ContactChannel.canonical_key`
(`models.py:153-191`), which passes `default_region=practice.default_country_code`; the
three readers call `canonical_phone_e164(phone)` with no region
(`identity_resolution.py:106`, `intake_views.py:1546`, `recall_views.py:2081`). Executed:
`canonical_phone_e164('07703349940', None)` → `'7703349940'`; with
`default_region='+44'` → `'+447703349940'`.

1. Route every lookup through the same function the writer uses — the docstring at
   `models.py:156-161` says that is the whole point of it:
   `canonical, _ = ContactChannel.canonical_key(practice, ContactChannel.PHONE, raw, country_code)`.
   Where a bare `canonical_phone_e164` call is kept for speed, pass
   `default_region=practice.default_country_code`.
2. `recall_views.py:2081` concatenates `cc + raw` first; pass them separately
   (`canonical_phone_e164(raw_phone, cc, default_region=...)`) — concatenation is the
   #10 shape.
3. Test: patient 41664 (`phone_number='7703349940'`, `country_code=''`, channel
   `+447703349940`) must be found by `find_duplicate_candidates` with its own values.

**AS BUILT — 2026-09-06 (uncommitted).** Done — at **nine** call sites, not the three
the audit lists.

The audit names `identity_resolution.py`, `intake_views.py` and `recall_views.py`. The
real criterion is "computes a phone key and compares it to a stored `canonical_value`",
and tracing that turned up six more with the identical defect, all with a `practice`
already in scope: `journey/mixins.py:90`, `views/patient_views.py:605`,
`views/contact_merge_views.py:258`, `serializers/treatment_plan.py:987`,
`serializers/validators.py:79`, and the second comparison inside
`identity_resolution.py:129` (which had to move too, or the two sides of that
comparison would disagree).

**One helper, not nine sprinkled `default_region=` arguments.** Added
`ContactChannel.lookup_key(practice, kind, raw, country_code)` next to `canonical_key`
and `find_channel`, and routed every read through it. Nine copies of "remember to pass
the region" is the same drift that created this bug; the class already declares itself
the single definition of the key, so the read belongs there.

`recall_views.py` also **concatenated `cc + phone` into one string** before
canonicalising — the finding-#10 shape, which yields `+GB7703349940` whenever
`country_code` holds an ISO code. It now passes the two separately.

*Measured on `prod_control`, both figures reproduce exactly:* 98.8% of phone channels
are E.164 (**61,805 of 62,582**), and patient **41664 Anna Harland** (practice 16) is
stored `phone_number='7703349940'`, `country_code=''` against channel `+447703349940`.

*Tests:* `TreatmentPlan/tests/test_phone_lookup_key_parity.py`, 6 tests. **The
discriminating one is `test_found_via_the_channel_even_when_the_raw_column_differs`**
(red: `[] != [10711]`) — the obvious end-to-end test passes even under the old code,
because the sibling clause `Q(phone_number=phone_number)` matches the raw column
verbatim and masks the dead channel clause. Mutating `lookup_key` to drop the practice
fallback also reddens `test_lookup_key_equals_the_key_the_write_stored`
(`'7703349940' != '+447703349940'`). Two further tests guard the fallback's contract:
it may only ever UPGRADE a bare key, never override a stated country code, and never
stamp `+44` onto a number that is not a valid UK line.

**A checksum guard fired, and it was right to.** `intake_views.py` is pinned by
`COMPAT03_INTAKE_VIEWS_SHA256` (a Phase-45 "do not modify" baseline). I verified the
old pin `74ab147c…` **matched HEAD exactly** — it was not already stale — so rebasing it
absorbs only my two edits (#6's `build_family_maps(items, practice)` and #7's
`lookup_key`), and the new constant carries a comment saying precisely that. The
sibling `COMPAT03_WEBHOOKS_TAB_SHA256` pin is still failing and was **deliberately left
alone**: it guards a frontend file this work never touched, and it was already failing
before any of it.

---

## 8. ✅ FIXED 2026-09-06 (was PROVEN / MEDIUM) — the Dormant recall tab loses the contact for multi-word names

**Where:** `dentallyIntegration/serializers.py:1058-1061`; reached from
`dentallyIntegration/views/recall_views.py:3744`.

**Plain English:** the fallback splits `patient_name` at the **first space** and
compares the halves to `Person.first_name` / `Person.last_name` separately — the same
mistake as finding 1. `"Peter James Evans"` becomes `Peter` + `James Evans` and
matches nobody.

**Verified by code.** Affected shape: 1,089 recall rows have a same-practice Person
matching on the joined name but not on the split.

**Why it matters:** those rows lose `contact_id`, `intake_id`, `nurture_id`, family
links and every call/email/SMS session **on the Dormant tab only** — while the main
tab shows them correctly. Same patient, two tabs, different answer.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `serializers.py:1060-1062` splits `patient_name` at the first space and
`:1078-1081` compares halves separately.

1. Compare the joined name, not the halves — after #1 lands:
   ```python
   want = canonical_full_name_key(name, "")
   for candidate in candidates:
       if canonical_full_name_key(candidate.first_name, candidate.last_name) == want:
   ```
   (`canonical_name_part` from `contact_keys.py` already lower-cases and collapses
   whitespace, so drop the hand-rolled `.strip().lower()`.)
2. Better still: the Dormant tab's `RecallRecord` carries `dentally_patient_id`; resolve
   the Person via `Patient.objects.filter(practice=obj.practice, meta_data__id=...)
   .person` first and only fall back to the channel+name search when no Patient exists.
3. Test: recall row `"Peter James Evans"` with a Person `Peter James|Evans` on the same
   phone → `contact_id` present on the Dormant tab.

**AS BUILT — 2026-09-06 (uncommitted).** Both steps done, plus the actual reason the
two tabs disagreed — which is not what the audit identifies.

- **Step 1 was already done by #2.** `_get_contact` now compares
  `canonical_full_name_key`; fixing #2 required it, because #1 makes Persons carry
  Dentally's split.
- **Step 2 done:** `_get_contact` resolves the recall's OWN Patient by
  `dentally_patient_id` first and only falls back to the channel+name search when no
  Patient carries that id (815 recall rows on `prod_control`).
- **The real cause of the tab asymmetry:** the Dormant tab passed **neither** page-wide
  context map, so it used the per-row fallbacks while the main tab used the batched
  ones. It now gets both maps — which also removes an N+1 on that tab.
- **And the batched path was the WORSE of the two.** `_build_contact_map` kept
  `{channel_id: person_id}` last-writer-wins, with no ordering and **no name check at
  all** — so on a shared family phone the recall's contact was whichever family member
  the database returned last. Simply handing the Dormant tab that map would have traded
  one bug for another. It now follows the same order as the per-row path: own Patient →
  name-matched channel holder → None. **None, not an arbitrary relative** — that is the
  point of the whole audit.

*Measured on `prod_control`:* **1,073** distinct (practice, recall name) pairs have a
live Person matching on the joined name but not on the first-space split — the audit
said 1,089; mine is the conservative count (distinct names, live Persons only).

*Tests:* 4 new in `dentallyIntegration/test_recall_patient_identity.py`
(`BothRecallTabsAgree`), **all 4 red** against the old code, with the reported symptoms
verbatim: the multi-word name losing its contact (`None != 43638`), the two tabs
disagreeing (`None != 43643`), and the batched path returning the relative
(`43642 != 43641`). A fourth pins that a recall with no Patient row at all still
resolves by name, since that is the only route those 815 rows have.

---

## 9. ✅ FIXED 2026-09-06 (was PARTIALLY FIXED) — `Person.resolve` called without a date of birth

**Where:** `TreatmentPlan/views/patient_views.py:734-739` and
`marketingBroadcast/views/submission_handoffs.py:75` — still open.
`dataQuality/views.py:303,488` and `split_collapsed_persons.py` — **fixed 2026-09-06.**

**Plain English:** `resolve` uses the date of birth to *refuse* to reuse a Person
when two same-named people have different birth dates. Callers that have a DOB and
don't pass it disable that guard — a father and son on one family phone become one
Person.

**Verified:** 57,990 of 60,326 patients have a date of birth, so the anchor patient
in `patient_views.py:734` almost always has one available and unused.
`submission_handoffs.py` correctly omits it — marketing forms collect no DOB.

**Why it matters:** this is the exact mechanism behind defect D2 (1,489 collapsed
identities).

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `patient_views.py:734-739` calls `resolve(... channel_ids=...)` with no
`dob`; `anchor.date_of_birth` is on the model. `dataQuality/views.py:303` and `:488`
now pass `dob` (fixed). `submission_handoffs.py:75` has no DOB to pass — leave it.

1. `patient_views.py:734`: add `dob=anchor.date_of_birth`. (The #3 fix adds a second
   `resolve` for the new patient — pass `dob=patient.date_of_birth` there too.)
2. Lint guard: a small test that greps for `\.resolve(` in the two apps and asserts each
   call site either passes `dob=` or carries a `# no-dob:` comment explaining why.

**AS BUILT — 2026-09-06 (uncommitted).** Both steps done. The audit lists two open call
sites; there are **seven** in total and **three** were open.

| call site | before | now |
|---|---|---|
| `patient_views.py:734` (anchor) | no dob | `dob=anchor.date_of_birth` |
| `patient_views.py:781` (new family member) | — | added by #3, passes `dob` |
| `patient_views.py:1608` (CSV import) | no dob | `# no-dob:` — the CSV format carries none |
| `bridge_dentally_identity.py:149` | no dob | `dob=getattr(row, "date_of_birth", None)` |
| `submission_handoffs.py:75` | no dob | `# no-dob:` — marketing forms collect none |
| `signals.py:60`, `split_collapsed_persons.py:412`, `dataQuality/views.py:303,488` | — | already passed it |

**`bridge_dentally_identity.py` was not in the audit's list and had a DOB available all
along** — both mirrors it walks (`RecallPatient`, `DaylistPatient`) carry
`date_of_birth`. Checked rather than assumed. A missing dob never blocks reuse, so
passing it is always safe; withholding it is what is unsafe.

*The guard* is `TreatmentPlan/tests/test_resolve_callers_pass_dob.py`. It parses the
AST of six identity-owning apps rather than grepping, so it cannot be fooled by
formatting, and it requires either `dob=` or a `# no-dob: <reason>` comment within
eight lines above the call. It carries a second test asserting the scan actually FINDS
call sites — a structural guard that silently matches nothing is not a guard.

Deliberately structural, not behavioural: a behavioural test only covers the call sites
someone remembered to write a test for, and the entire failure mode here is a call site
nobody thought about.

*Red run:* restoring the three omissions makes it fail naming each one —
`patient_views.py:739`, `patient_views.py:1613`,
`bridge_dentally_identity.py:153` — with the reason and the fix in the message.

*Measured on `prod_control`:* **57,990 of 60,326** patients carry a date of birth, so
the anchor almost always had one available and unused.

---

## 10. ✅ FIXED 2026-09-06 (was PROVEN / HIGH) — `+GB…` phone numbers, so the SMS silently never sends

**Where:** `automations/actions.py:1709-1710` and
`messaging/views/message_views.py:261-264`.

```python
country_code = "+" + country_code          # country_code is "GB"
to_number = f"{country_code}{phone_number.lstrip('0')}"   # "+GB7911123456"
```

**Plain English:** `country_code` does not always hold a dialling code — for
thousands of patients it holds the ISO country `"GB"`. Prefixing `+` gives `+GB`.

**Verified:** **4,208 patients** have `country_code='GB'`. Twilio rejects the
resulting number.

**Why it matters:** the workflow SMS fails silently — no error surfaced to staff, the
patient simply never hears from you.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `automations/actions.py:1708-1710` and `messaging/views/message_views.py:260-264`
do `"+" + country_code` then concatenate. 4,208 patients hold `country_code='GB'`.
**Executed:** `canonical_phone_e164('7911123456', 'GB')` → `'+447911123456'` — the
existing helper already handles the ISO form.

1. Replace both blocks with:
   ```python
   from TreatmentPlan.utils.phones import canonical_phone_e164
   to_number = canonical_phone_e164(
       phone_number, country_code,
       default_region=practice.default_country_code,
   )
   if not to_number or not to_number.startswith("+"):
       # surface, don't silently send garbage
       return Response({"error": f"Unusable phone number {phone_number!r}"}, status=400)
   ```
2. Do not "fix" the data by rewriting `country_code` — the project note
   `country_code is NOT a dial code (TRAP)` records that one site deliberately keeps ISO.
3. Test: patient with `country_code='GB'`, `phone_number='07911123456'` → Twilio `to` is
   `+447911123456`.

**AS BUILT — 2026-09-06 (uncommitted).** Done at **four** sites, not the two listed.

| site | in audit | state |
|---|---|---|
| `automations/actions.py:1709` | yes | now `canonical_phone_e164(phone, country_code, default_region=…)` |
| `messaging/views/message_views.py:261` | yes | same |
| `Documents/utils/notifications.py:281` (`_normalise_phone`) | **no** | same |
| `TreatmentPlan/views/patient_views.py:93` | **no** | dead code — built `cc = "+" + cc` and never used it; deleted |

`TreatmentPlan/utils/phones.py:198` also does `f"+{country_code}"` and is **correct** —
it is the one place allowed to, because it first checks whether the value is numeric or
ISO. That is the helper everything else now calls.

**Also fixed in `actions.py`: the MessageSession identifier.** It was built by the same
hand-rolled concatenation (`f"{country_code}{phone_number}"`), so the session key was
`+GB7911123456` too and drifted from every other path that keys on E.164. It now reuses
the canonical `to_number`.

*Measured on `prod_control`:* **4,208** patients hold `country_code='GB'` — exact. Plus
one holding `'DE'`, which the audit does not mention and the helper also handles.

*Data left alone, deliberately,* per step 2 and the project note "country_code is NOT a
dial code (TRAP)".

*Tests:* `TreatmentPlan/tests/test_iso_country_code_sms.py`, 10 tests in three groups.
- Contract tests on the helper: "+44", "44", "GB" and "gb" all yield one key; `DE`
  works; an invalid-for-GB number is **not** fabricated into a plausible wrong one.
- Behavioural tests driving the real `_normalise_phone`. Red against the old code with
  the defect literally in the message: `'+GB7911123456' != '+447911123456'`, and
  `'+GB44221251'` — a fabricated number built out of an invalid one.
- A structural guard, `NoSendPathBuildsANumberByHand`, forbidding the SHAPE anywhere in
  the backend outside `phones.py`.

**Two things the guard taught me while writing it**, both fixed:
- A plain substring scan **matched its own documentation** — the comment in
  `notifications.py` explaining this trap contains the shape it warns about. The guard
  now tokenizes and discards comments and string literals, so describing the defect
  cannot trip the rule that forbids it.
- The first pattern set included a bare `"+ code"`, which would have matched innocent
  arithmetic. Narrowed to names that actually denote a dialling code.

---

## 11. ✅ FIXED 2026-09-06 (was PROVEN / HIGH) — the merge suggester compares phones as raw strings

**Where:** `TreatmentPlan/views/contact_merge_views.py:107-112`.

**Plain English:** the duplicate suggester canonicalises the name
(`canonical_name_part`) and the email (`canonical_email`) — then compares the phone
with bare `==`. Since the rule is `name_match AND (email_match OR phone_match)`, a
mere formatting difference blocks a genuine duplicate from ever being suggested.

**Verified by code.** Example: practice 19 patients 43421 `+447444504074` and 43451
`7444504074` — the same human, Simangaliso Sibanda — never suggested.

**Why it matters:** the tool built to find duplicates is blind to the most common
form of duplicate.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `contact_merge_views.py:107-112` — bare `==` on `phone_number` while name
and email go through canonicalisers.

1. Both records carry `country_code`; canonicalise both sides with the practice fallback:
   ```python
   def _phone_key(rec):
       return canonical_phone_e164(getattr(rec, "phone_number", None),
                                   getattr(rec, "country_code", None),
                                   default_region=practice.default_country_code)
   phone_match = bool(sp := _phone_key(source_record)) and sp == _phone_key(candidate_record)
   ```
   (`practice` is available on both records.)
2. Test: practice 19 patients 43421 `+447444504074` and 43451 `7444504074` → suggested.

**AS BUILT — 2026-09-06 (uncommitted).** Done. `_phone_key(record)` canonicalises both
sides through `ContactChannel.lookup_key`, so it also picks up #7's practice-country
fallback — which this case actually needs: 43451 carries no country code, and only
Snapfit Denture Practice's own `+44` turns its bare number into the same key.

*The production pair, confirmed on `prod_control`:*

```
43421  Simangaliso Sibanda  +447444504074  cc="GB"  ssimangaliso70@gmail.com
43451  Simangaliso Sibanda   7444504074    cc=""    ssimangaliso70@gmail.com.com
```

Note the **emails do not match either** — `.com.com` is a typo — so the phone was the
only possible route to suggesting these two, and it was closed.

**My first version of this test was a false green, and the red run caught it.** A
pre-save signal rewrites `Patient.phone_number` to the local form, so building the pair
with `Patient.objects.create(phone_number="+447444504074")` stored `7444504074` for
BOTH rows — byte-identical, so even the broken raw-string compare matched and the test
passed against the unfixed code. Production has the divergent forms because the Go
importer writes those rows directly. The test now writes them directly too (via
`.update()`, bypassing the signal) and asserts the stored form is what it intended
before testing anything.

*Tests:* `TreatmentPlan/tests/test_merge_suggester_phone_key.py`, 6 tests, **4 red**
against the raw-string compare — the real production pair plus three other formats of
one number. The other two are guards that canonicalising has not turned the suggester
into a name-only matcher: two genuinely different numbers are still not suggested, and
two blank phones are not evidence of anything.

---

## 12. ✅ FIXED 2026-09-06 (was PROVEN / MEDIUM) — a lead can be stripped of all contact details on update

**Where:** `TreatmentPlan/serializers/intake.py:553-570`.

**Plain English:** the "an intake must have an email or a phone" rule is guarded by
`if not self.instance:` — it runs on **create only**. A `PATCH` setting both to empty
strings passes. There is no `Intake.clean()` to catch it either (the only `clean()`
in nine apps is `TreatmentPlan.clean()`).

**Verified by code.** 3 such rows exist today.

**Why it matters:** a lead with no email and no phone is outside the identity graph
entirely and can never be deduplicated or contacted.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `intake.py:553` — `if not self.instance:` guards the rule; 3 intakes have
neither email nor phone today.

1. Evaluate the rule on the **merged** state:
   ```python
   email = data.get("email", self.instance.email if self.instance else None)
   phone = data.get("phone_number", self.instance.phone_number if self.instance else None)
   if not (email or "").strip() and not (phone or "").strip():
       raise serializers.ValidationError("Either email or phone number must be provided.")
   ```
   (keep the whitespace strip for both branches).
2. The 3 existing rows: list them (`SELECT id, practice_id FROM "TreatmentPlan_intake"
   WHERE coalesce(email,'')='' AND coalesce(phone_number,'')=''`) for staff to complete
   or archive — nothing automatic can recover a contact detail that was erased.

**AS BUILT — 2026-09-06 (uncommitted).** Step 1 done; step 2 is the list below, for a
human.

The rule is now evaluated on the **merged** state, which is the only correct reading of
PATCH: an absent key means "unchanged" (fall back to the instance), a present-but-empty
key means "clear it" — and the second case is precisely what used to slip through. The
whitespace strip was hoisted out of the create/update branches, since it was duplicated
identically in both.

*The 3 rows on `prod_control`, for staff to complete or archive:*

| intake | practice | name |
|---|---|---|
| 684 | 19 | George A |
| 718 | 19 | Birdy Noln |
| 641 | 19 | Luke Moorehouse |

Nothing automatic can recover an erased contact detail, so these are deliberately left
alone rather than guessed at.

*Tests:* `TreatmentPlan/tests/test_intake_contact_required_on_update.py`, 7 tests, **4
red** against the create-only rule — including
`test_clearing_both_does_not_reach_the_database`, which follows through to `save()` and
shows the row actually being emptied rather than merely passing validation. The three
green-either-way tests are the guards that the rule has not become too strict: clearing
just the email still works, an unrelated PATCH is not blocked into resending contact
fields, and the original create-time rejection survives the refactor.

---

## 13. ✅ FIXED 2026-09-06 (was PROVEN / MEDIUM) — three more places still prefer Dentally's broken phone field

**Where:** `automations/actions.py:396`, `automations/actions.py:688`,
`dentallyIntegration/views/dentally_views.py:819`.

**Plain English:** Dentally exposes a raw phone and its own `*_normalized` version.
Its normalisation is **wrong for non-UK numbers** — it prefixes +44 onto a number
that already has a country code. The shared fix (`pick_usable_phone`) picks whichever
field actually validates; these three sites still take `_normalized` first.

**Verified by code.** 8 corrupted rows in prod, e.g. patient 134270 (practice 24):
real number `+35699097155`, stored as `4435699097155`.

**Why it matters:** a foreign patient becomes silently uncontactable and
un-deduplicable — and unlike a UK number, there is usually no other way to reach them.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified with one correction:* `actions.py:396` and `dentally_views.py:819` take
`*_normalized` first as described. **`actions.py:688` does not** — it reads raw
`mobile_phone` first, then `_normalized`; that site is not the +44-prefix defect (raw-first
has the opposite weakness: a raw value with no country code). Patient 134270 holds
`phone_number='4435699097155'` while `meta_data.mobile_phone='+35699097155'`.

1. All three sites → `pick_usable_phone(normalized, raw, country,
   default_region=practice.default_country_code)` (`phones.py:460`), which is the shared
   rule and is mirrored in Go (`pickUsablePhone`).
2. Data: the 8 corrupted rows can be rewritten from what is already stored — for each
   patient where `phone_number` does not canonicalise, recompute
   `pick_usable_phone(meta_data.mobile_phone_normalized, meta_data.mobile_phone, ...)`
   and, if that yields a valid E.164, update `phone_number`/`country_code` and re-run
   `repair_split_channels` so the ContactChannel follows. Verify each of the 8 by hand
   (per the "verify each fix individually" rule).

**AS BUILT — 2026-09-06 (uncommitted).** Step 1 done at all three sites. **Step 2 (the
data repair) is NOT done** — see below.

`actions.py:396` and `dentally_views.py:819/822/825` now call `pick_usable_phone`, with
`default_region=practice.default_country_code` rather than the hardcoded `"GB"` the
existing caller in `patient_import_utils.py` uses.

**`actions.py:688` is routed through the helper too**, though the audit correctly notes
it is not the `*_normalized`-first defect — it reads raw first, which has the opposite
weakness (a raw value with no country code). Its value becomes `detail["phone_number"]`
on the `DataQualityIssue`, i.e. the number staff read in the data-quality UI, so
"whichever field is actually a phone number" is the right answer there as well. Routing
it through the shared rule also means the guard below needs no exception for it, and an
exception is how a site drifts back.

*The 8 corrupted rows, confirmed on `prod_control`:*

| patient | practice | real number | stored |
|---|---|---|---|
| 134270 | 24 | +35699097155 | 4435699097155 (Malta) |
| 130624 | 27 | +35699097155 | 4435699097155 (Malta) |
| 127301 | 26 | +65 82288227 | 446582288227 (Singapore) |
| 136509 | 27 | +65 82288227 | 446582288227 (Singapore) |
| 132430 | 24 | +255686600636 | 44255686600636 (Tanzania) |
| 132949 | 24 | +353874878286 | 44353874878286 (Ireland) |
| 22188 | 13 | +18183099347 | 4418183099347 (US) |
| 30695 | 16 | +18183099347 | 4418183099347 (US) |

*Tests:* `TreatmentPlan/tests/test_dentally_normalized_phone_sites.py`, 6 tests.
- Contract tests over the **real production pairs**: Dentally's normalised value is
  genuinely invalid everywhere, the helper recovers the raw one for all five distinct
  countries, and a UK number still takes the normalised value so the common path is
  unchanged.
- A structural guard, `EveryNormalizedReadGoesThroughTheHelper`, which walks the AST and
  requires every `*.get("..._normalized")` read to be an ARGUMENT to
  `pick_usable_phone`. Two files are exempt with reasons: `phones.py` (defines the rule)
  and `repair_unusable_phones.py` (deliberately inspects both fields to decide what to
  rewrite). It carries a second test asserting the scan actually finds call sites.
- *Red run:* reverting the two sites makes the guard name all four offending lines —
  `actions.py:396`, `dentally_views.py:819/822/825` — the exact lines the audit cites.

**STILL TO DO — the data repair (step 2).** The 8 rows above are unchanged. They are
recoverable from what is already stored (`meta_data.mobile_phone` holds the real
number), but the repair needs to update `phone_number`/`country_code` and then re-run
`repair_split_channels` so the ContactChannel follows — and each of the 8 wants
checking by hand.

---

## 14. ✅ FIXED 2026-09-06 (was PROVEN / MEDIUM) — a workflow patient lookup is case-sensitive on email

**Where:** `automations/actions.py:950,956` —
`Patient.objects.filter(practice=practice, email=email)`.

**Plain English:** no lowercasing, while the same module's own ingest path uses
`canonical_email`. `Ada@Example.com` will not find `ada@example.com`.

**Verified:** **1,565 patients hold a mixed-case email** (e.g. patient 135452,
`NICOLAWARRELL@HOTMAIL.COM`).

**Why it matters:** the workflow silently does nothing for those patients, or creates
a duplicate.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `actions.py:955` — `Patient.objects.filter(practice=practice, email=email)`;
1,565 patients store a mixed-case email.

1. `email__iexact=canonical_email(email)` (`canonical_email` from
   `TreatmentPlan/utils/contact_keys.py`, already used by this module's ingest path).
2. Better: resolve through the channel graph like the phone branch immediately below it
   (`ContactChannel.find_channel(practice, EMAIL, email)` → Persons → Patient), which is
   case-insensitive by construction.
3. Test: patient 135452 `NICOLAWARRELL@HOTMAIL.COM` found from `nicolawarrell@hotmail.com`.

**AS BUILT — 2026-09-06 (uncommitted).** Both steps 1 and 2 done: the email branch now
matches `email__iexact=canonical_email(email)` **and** resolves through the channel
graph, mirroring the phone branch immediately below it that already did so.

**A second defect in the same block, not in the audit:** the phone branch called
`canonical_phone_e164(phone_number, country_code)` with no practice fallback — finding
#7's shape at a tenth site. It now uses `ContactChannel.lookup_key`.

*Measured on `prod_control`:* **1,565** patients hold a mixed-case email — exact.

*Tests:* `automations/test_workflow_patient_lookup.py`, 3 tests driving the real
`_create_treatment_plan_record`. They assert on **which patients exist afterwards**, not
on the return value, because the harm is not the missed match — it is that the branch
falls through to "create one" and the workflow silently manufactures a duplicate. Red
against the case-sensitive lookup, with the duplicate showing up as a new id. The third
test guards that a genuinely new email still creates a patient, so case-insensitivity
has not turned into matching everyone.

---

## 15. ✅ FIXED 2026-09-06 (was PROVEN / MEDIUM) — assigning a bad or foreign practitioner returns success

**Where:** `TreatmentPlan/serializers/treatment_plan.py:1075-1083`.

```python
try:
    instance.current_practitioner = User.objects.get(pk=attending_id)
except User.DoesNotExist:
    pass
```

**Plain English:** a non-existent practitioner id is swallowed — HTTP 200, nothing
changed, no error. And neither the create nor the update path scopes the user to the
practice, so a practitioner from another practice **is** accepted.

**Verified by code.**

**Why it matters:** the caller believes the assignment succeeded. Same
silently-discarded-input pattern as finding 4.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `treatment_plan.py:1075-1083` swallows `DoesNotExist`; create (`:885-892`)
raises but also uses `User.objects.get(id=...)` with no practice scope.

1. Raise on both paths and scope to the practice the way `Tasks/serializers.py:341`
   (`validate_assigned_user_id`) already does with `UserPracticeRelationship`:
   ```python
   def validate_attending(self, value):
       if value is None: return None
       practice = self.instance.practice if self.instance else \
                  self.context["request"].user.current_practice
       if not UserPracticeRelationship.objects.filter(user_id=value, practice=practice).exists():
           raise serializers.ValidationError("Practitioner is not a member of this practice.")
       return value
   ```
   then `instance.current_practitioner_id = attending_id` in `update()`.
2. Test: unknown id → 400; other practice's user → 400; same-practice user → 200 and set.

**AS BUILT — 2026-09-06 (uncommitted).** Done as written. `validate_attending` checks
membership via `UserPracticeRelationship` — the single source of truth the Tasks
serializer already uses — and serves create and update alike, so the two paths cannot
drift. `update()` assigns `current_practitioner_id` directly; the
`try/except DoesNotExist: pass` is gone, and `create()`'s unscoped `User.objects.get`
with it.

*Tests:* `AttendingPractitionerIsValidatedAndScoped` in
`TreatmentPlan/tests/test_treatment_plan_patient_scoping.py`, 5 tests, **3 red** —
including `test_another_practices_practitioner_is_not_written`, which follows through to
`save()` and shows the foreign practitioner actually being written (`52908 == 52908`),
not merely failing to be rejected. Two guards confirm the validation has not become a
wall: a same-practice practitioner still assigns, and an unrelated PATCH does not clear
the existing one.

*One test fixture had to change, and it is worth flagging.*
`test_serializers.IntakePartialPriorityTests` builds its serializer on a bare
`SimpleNamespace()` — an instance stub with no `email` or `phone_number` attributes at
all. Once #12 made the "email or phone" rule read the MERGED state, that stub stood in
for a lead with no way to be contacted and was correctly rejected. A real `Intake`
always has those attributes, so the fixture was under-specified rather than the rule
being wrong; the stub now carries an email. The test is about `priority` — the contact
detail is fixture, not subject.

---

## 16. ✅ FIXED 2026-09-06 (was PROVEN / MEDIUM) — the Dentally bridge scores matches across practices

**Where:** `dentallyIntegration/management/commands/bridge_dentally_identity.py:172`
— `Patient.objects.filter(meta_data__id__in=…)` with no `practice=`, although the
same command scopes correctly at :141 and :149.

**Verified:** **13,863 Dentally ids span more than one practice.**

**Why it matters:** a genuinely unmatched patient is scored "already matched" because
*another practice* has that Dentally id, so the bridge skips them permanently.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `_matched_dentally_ids` (`:172-181`) matches `meta_data__id__in` with no
practice; `qs` spans practices (`row.practice` is read per row at `:111`). 13,863 Dentally
ids exist in more than one practice.

1. Key the match on `(practice_id, dentally_id)`:
   ```python
   pairs = {(r.practice_id, str(getattr(r, id_field))) for r in qs if getattr(r, id_field, None) is not None}
   found = set(
       (pid, str(mid)) for pid, mid in Patient.objects
           .filter(practice_id__in={p for p, _ in pairs}, meta_data__id__in=[m for _, m in pairs])
           .values_list("practice_id", "meta_data__id")
   )
   unmatched = [r for r in qs.iterator() if (r.practice_id, str(getattr(r, id_field, ""))) not in found]
   ```
2. Re-run the command in read-only mode before/after and diff the "reach no Patient"
   counts — the delta is the population that was being skipped.

**AS BUILT — 2026-09-06 (uncommitted).** Fixed — and **the audit's stated mechanism is
not what was happening in production.**

The missing practice filter is real and step 1 is done. But there is a SECOND defect in
the same three lines that masks it: the lookup passed `str(v)` values to
`meta_data__id__in`, and `meta_data->'id'` is a JSON **number** on all 58,884 patients
that carry one. `meta_data__id__in=["987654"]` returns zero rows against a stored
`987654` — executed and confirmed.

So the live behaviour was the OPPOSITE of over-matching: the set came back empty and
**every** row was scored "reaches no Patient".

*Measured on `prod_control`:*

| | rows |
|---|---|
| matched by the OLD lookup | **0** |
| matched by the fixed lookup (text compare, practice-scoped) | **39,844** |
| would be FALSELY matched across practices if only the type bug were fixed | **373** |

That last figure is why step 1 still matters: fixing the type comparison alone would
have *introduced* the cross-practice over-match the audit describes, for 373 rows. The
two fixes are only correct together.

The comparison now uses `KeyTextTransform` — text, so it matches the numeric and the
string form alike, and it is the expression the partial index
`patient_practice_dentally_uniq` is built on, so it is an index hit rather than a scan.
The result is intersected with the requested pairs so a Patient in a scanned practice
holding some other practice's id cannot contribute a pair nobody asked about.

*Tests:* `dentallyIntegration/test_bridge_practice_scoping.py`, 5 tests, **2 red**.
Worth noting *which* two: the cross-practice test passes even against the old code,
because the type bug meant nothing ever matched, so the over-match could not fire. The
tests that go red are the ones proving a real same-practice match is now found — which
is the defect that was actually live.

---

## Cross-cutting themes

Four root patterns produce the original sixteen:

1. **Name split at the wrong space** — findings 1, 8 (and the live treatment-plan
   failure). A name is an identity key here; splitting it by position is unsound.
2. **Practice scoping applied on one path and not its sibling** — findings 4, 6, 15,
   16. Every one holds on create/read and fails on update/lookup.
3. **A key computed differently by the reader than the writer** — findings 7, 10, 11,
   13, 14. The canonical functions exist; these callers bypass or mis-call them.
4. **One human's record standing in for another** — findings 2, 3, 5. Person-sharing
   is the cause; finding 5 is the one with clinical consequences.

## Suggested order

1. **#5** — wrong-patient clinical data, 292 people. The only patient-safety item.
2. **#4 and #6** — tenant boundaries; #6 is already leaking today.
3. **#3** — it manufactures the damage the D/D2 repairs just removed.
4. **#1, #2, #8** — the name-split family; one root cause, three symptoms.
5. **#10, #13, #14, #11, #7** — silent contact failures and missed duplicates.
6. **#9, #12, #15, #16.**

---

# Second sweep — findings #17–#30 (apps the first pass never covered)

Sweep agent proposed 13 (N1–N13). Every claim below was **re-verified by me personally**
against `prod_control` (read-only) — code read at the cited line, counts re-run as SQL,
named example rows fetched back. One of the agent's claims was **WRONG** and is marked
as such. One finding (#30) is mine, found while checking the agent's.

---

## #17 — ✅ FIXED 2026-09-06 (was HIGH) — Activity History shows another patient's clinical notes  [CONFIRMED]

`activityLog/views.py:363-366` (`patient_activities`); same shape at `:568`
(`_get_base_queryset`) and — **a third site I missed** — at `:817`, inside
`patient_audit`, which is a `NoteHistory` query reached by route
`activityLog/urls.py:80`. `patient_audit_export` (`:872`) carries it into the CSV/PDF
export via `_get_base_queryset` at `:887`, so the wrong patient's notes leave the
system as a file. **A fix to `:364` alone leaves both open.**

```python
target_person_ids = set(
    Person.objects.filter(Q(patients__id=patient_id) | Q(id=patient_id))
    .values_list("id", flat=True)
)
```

The URL carries a **Patient** id. The code takes Persons who own that patient — and
*also* the Person whose **Person id is the same number**. Two independent sequences,
heavily overlapping. Both notes (`:405`) and activities (`:391`) are then filtered
`person_id__in=target_person_ids`, so the stranger's timeline merges into the view.

The `| Q(...)` is there because the endpoint accepts either key. It should be a
**fallback** (try patients, use person only if that returned nothing), not a **union**.

**Verified counts (prod_control):** *(corrected — my first figure was the wrong denominator)*
- `25,887` patient ids collide with a different Person's id — but **only `318` of those
  collisions are within the same practice**, and both queries are practice-filtered.
  I checked whether a note or activity can ever carry a practice different from its
  Person's: **0** in both tables. So the 25,569 cross-practice collisions can never
  leak. **The standing exposure is 318**, not 25,887 — practice 21: 177, practice 13:
  134, 26: 5, 16: 1, 27: 1. All three patients leaking today sit inside that 318.
- Leaking today: **5 clinical notes, 4 activity events** (the table below enumerates
  only 3 of the 4 — activity 9465 is the fourth).

**Verified rows — fetched back individually:**

| Viewed patient | Stranger pulled in | Leaked |
|---|---|---|
| patient 138938 Hallie Boler (prac 21, person 157461) | person 138938 = Mandy Lindley (prac 21) | NoteHistory 5000/5253/5609/5610/5767 — *"Had CBCT and socket pres in Feb 2026"*, *"2ND CONS BOOKED IN WITH HARRY FOR SEPT 26"* |
| patient 138908 Sasha Damjanovic (person 157410) | person 138908 = Alison Haywood | Activity 12645/12656 — *"Intake created for Alison Haywood"* |
| patient 112819 Paisley Dervish (person 147988) | person 112819 = Jennifer Appleby | Activity 7413 — *"Email sent: plan"* |

Practice scoping is applied on both queries, so the leak stays inside one practice —
which is exactly why it was never noticed. The count is small only because
`Activity.person_id` is still sparse; the 25,887 collisions are the standing exposure
and it grows with every event written.

**Note for whoever fixes this:** the comment at `:386-389` claims this "returns
precisely the viewed person's events, and nobody else's". That comment is wrong.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* the `Q(patients__id=patient_id) | Q(id=patient_id)` union is at `:363-366`,
`:568-569` and `:817`; 318 same-practice id collisions; NoteHistory 5000/5253/5609/5610/
5767 all carry `person_id=138938` (practice 21) and surface for patient 138938 whose real
Person is 157461. Covers **#35** too.

1. Resolve the Patient **first, scoped**, and use the Person id only as a fallback:
   ```python
   def _target_person_ids(patient_id, practice):
       pid = (Patient.objects.filter(id=patient_id, practice=practice)
              .values_list("person_id", flat=True).first())
       if pid:
           return {pid}
       # No such Patient in this practice: the caller passed a Person id (lead panel).
       return set(Person.objects.filter(id=patient_id, practice=practice)
                  .values_list("id", flat=True))
   ```
   Use it at `:363`, in `_get_base_queryset` (`:568` → `person_id__in=...`) and at `:817`.
   Note a Patient with `person_id IS NULL` (99 exist, #33) now returns an empty set rather
   than falling through to a stranger — that is the correct answer.
2. Correct the comment at `:386-389`.
3. Test: patient 138938 → none of notes 5000/5253/5609/5610/5767; activity 12645/12656
   absent for patient 138908; activity 7413 absent for patient 112819.

**AS BUILT — 2026-09-06 (uncommitted).** All three steps done, and **#35 with it**.

`resolve_target_person_ids(patient_id, practice)` is now the one resolver, used at all
three sites — `patient_activities` (`:363`), `_get_base_queryset` (`:568`, which feeds
both `patient_audit` and `patient_audit_export`) and the `NoteHistory` query inside
`patient_audit` (`:817`). Fixing only the first would have left the export leaking, as
the audit warns.

It is a **fallback, not a union**, in three ordered steps:
1. the Patient in THIS practice → its `person_id`;
2. that Patient exists but has no Person (99 such rows, #33) → **empty set**, because
   reinterpreting its id as a Person id is the collision itself;
3. otherwise treat the id as a Person id, still practice-scoped — the lead panel's real
   use case, which is why the union existed.

Step 1 being practice-scoped is exactly what #35 asked for, so that finding needs no
separate change.

*The wrong comment at `:386-389` is corrected*, not deleted — it now records that
per-event scoping was always exact and it was the SET of target persons that was not.

*Measured on `prod_control`, both figures exact:* 25,887 total id collisions, of which
**318 are same-practice** and therefore exposed. The five leaked notes are real and were
fetched back individually — `person_id=138938`, content "Had CBCT and socket pres in Feb
2026" and "2ND CONS BOOKED IN WITH HARRY FOR SEPT 26".

*Tests:* `activityLog/test_patient_timeline_person_collision.py`, 6 tests, **4 red**
against the union. The forced collision (a Person given the same primary key as an
unrelated Patient) reproduces production exactly, and the red output shows the target
set as `{138938, 45578}` and the stranger's `NoteHistory` actually being returned. Two
green-either-way tests guard the paths that must keep working: a bare Person id still
resolves for a lead with no Patient, and a Patient with no Person yields nothing.

## #18 — ✅ FIXED 2026-09-06 (was HIGH, 0 rows today) — online booking files the appointment under a family member  [CONFIRMED]

`onlineBooking/services.py:332-335`, consumed at `:371-377`.

```python
patient = Patient.objects.filter(
    practice=hold.practice, email__iexact=hold.patient_email
).first()          # no order_by, no name check
```

`hold.patient_name` is never used in matching. The match is written to
`Appointment.patient` while `Appointment.patient_name` keeps the booked name — right
name on the diary, wrong clinical record behind it.

**Verified:** `3,882` same-practice email groups hold patients with genuinely different
names. Concrete: practice 21 `jkatytaylor@gmail.com` covers nine Taylor children
(43706, 45188, 45190–45195, 45248). Which one a booking resolves to is whatever
Postgres returns first.

**Damage today: zero** — `onlineBooking_onlinebookinghold` is empty and there are no
`source='online'` appointments. This fires on launch day.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `_match_patient` (`services.py:332-349`): email `.first()` with no name check,
no `archived_at` filter, phone canonicalised with `None` region. 3,882 same-practice email
groups hold different names. 58,882 of 58,923 stored phones are bare national digits; 13 of
14 practices have `default_country_code='+44'`. `OnlineBookingHold` and `source='online'`
appointments: 0 rows. **This block is the single fix for #18, #19, #28, #33 and #34** —
they are one function and one create path.

1. Rewrite `_match_patient(hold)`:
   ```python
   def _match_patient(hold):
       practice = hold.practice
       want = canonical_full_name_key(hold.patient_name, "")          # from #1
       base = Patient.objects.filter(practice=practice, archived_at__isnull=True)   # #34
       cands = set()
       if hold.patient_email:
           cands |= set(base.filter(email__iexact=canonical_email(hold.patient_email)))
       canonical, _ = ContactChannel.canonical_key(practice, ContactChannel.PHONE,
                                                   hold.patient_phone)          # #19
       if canonical:
           cands |= set(base.filter(
               person__person_channels__channel__kind=ContactChannel.PHONE,
               person__person_channels__channel__canonical_value=canonical))
       named = [p for p in cands
                if canonical_full_name_key(p.first_name, p.last_name) == want]  # #18
       if len(named) == 1:
           return named[0]
       return None   # 0 or ambiguous: never guess a family member
   ```
   `patient_type` (#28): keep it as a signal, not a gate — a `NEW` booker who *does*
   name-match an existing record is the same human; a `EXISTING` booker with no match
   should produce a staff-visible flag on the appointment (`notes`), not a silent create.
2. Create path (`:504-526`, #20/#33): replace the bare `Patient.objects.create` with the
   lane every other writer uses — `ContactChannel.get_or_create_channel` for email and
   phone, then `Person.resolve(practice, first, last, channel_ids)` and `person=` on the
   Patient. With #1's full-name key, the `split(" ", 1)` no longer creates a second
   Person; to remove the split entirely, collect `first_name`/`last_name` separately on
   the booking form (`onlineBooking/serializers.py:347` area, new hold columns).
3. Tests (all with prod-shaped fixtures): nine Taylor children on one email → the booked
   name's record or `None`; `07703349940` matches a patient stored as `7703349940`;
   an archived patient is never returned; a created Patient has `person_id` and a channel.

**AS BUILT — 2026-09-06 (uncommitted).** All three items done. **This block closes #18,
#19, #20, #28, #33 and #34.**

- **#18** — `_match_patient` now collects candidates from email AND phone, then filters
  by the booked NAME (joined-name key, so #1's moved splits still match) and returns a
  patient only when **exactly one** survives. 0 matches or ambiguity → `None`. An
  unattached appointment a human must file is strictly better than one filed under a
  sibling.
- **#19** — the phone branch uses `ContactChannel.lookup_key`, which supplies the
  practice's own country as the last-resort region. The raw-column comparisons are kept
  as extra candidate sources rather than the only ones.
- **#34** — `archived_at__isnull=True` on the candidate base queryset.
- **#33/#20** — the create path goes through the normal lane:
  `ContactChannel.get_or_create_channel` for phone and email, then `Person.resolve`, then
  `person=` on the Patient. It previously made a bare Patient with **no Person at all**,
  so a booking paid for by card landed outside the identity graph entirely. Carries a
  `# no-dob:` comment for the #9 guard, since a booking form collects no date of birth.
- **#28** — `patient_type` is finally read, as a **signal not a gate**: a "new" booker
  who genuinely name-matches is still matched (they are the same human), but an
  "existing" booker that `_match_patient` could not identify now writes an **ATTENTION**
  line into the appointment notes where staff will see it, instead of silently becoming
  a second record.

*Measured on `prod_control`, all exact:* 3,882 same-practice email groups hold
differently-named patients; 99 patients have no Person; 1,848 live Persons have a space
in `first_name`.

*Tests:* `onlineBooking/test_booking_identity.py`, 11 tests, **8 red** against the old
code. The clearest failure is the finding in one line —
`<Patient: Amelia Taylor> != <Patient: Bryn Taylor>`: a booking for Bryn resolving to
her sibling. The #28 notes tests were separately proven red by stripping the flag.

*Note on the 8 pre-existing `onlineBooking` failures:* all Stripe Connect, identical at
`HEAD` with these changes reverted.

## #19 — ✅ FIXED 2026-09-06 with #18 (was HIGH) — the booking phone fallback can never match  [CONFIRMED by execution]

`onlineBooking/services.py:340-349`. Both branches are dead for the format a real
booker types (`07703349940`):

- raw string compare — `58,882 of 58,923` patients store bare national digits
  (`7703349940`); only 26 store a leading zero.
- channel compare — `canonical_phone_e164(phone, None)` with **no default region**
  returns `'7703349940'`, while `canonical_value` is `+44…` on 98.8% of channels.

`hold.practice.default_country_code` is `'+44'` on 13 of 14 practices and is right
there — it just isn't passed. Net effect: email (#18) is the only working path.

Same root as first-pass **#7**, but a **new site** — #7 cited `identity_resolution.py:106`,
`intake_views.py:1546`, `recall_views.py:2081`, not this one.

*Fixed together with #18 — see that block (item 1, `ContactChannel.canonical_key`).*

## #20 — ✅ FIXED 2026-09-06 with #18 (was HIGH) — booking payment creates a Patient by splitting at the first space  [CONFIRMED]

`onlineBooking/services.py:504-506`:

```python
name_parts = hold.patient_name.strip().split(" ", 1)
```

The **write** side of first-pass #1. `"Ken (Kenneth) Judge"` lands here as
`first="Ken"` / `last="(Kenneth) Judge"`; the Dentally lane stores
`first="Ken (Kenneth)"` / `last="Judge"` — real rows, persons 74615 and 74616,
practice 16. Guaranteed duplicate, and this path runs on a **confirmed Stripe
payment**, so the money attaches to the duplicate.

**Verified exposure:** `1,848` live Persons have a space in `first_name`
(74710 `Alan (Haoxuan)|Yu`, 74741 `Kathy (Kathleen)|Brockwell`,
74751 `Jeffrey C.|Body`, 75079 `Johnny Raymond|Goss`, 75501 `Lai Fong|Holland`).

*Fixed together with #18 — see that block (item 2, resolve a Person on create; #1 makes the split harmless).*

## #21 — ✅ FIXED 2026-09-06 (was MEDIUM-HIGH) — two more first-space splitters feeding `Person.resolve`  [CONFIRMED]

- `marketingBroadcast/form_handoffs.py:118-121` (`_split_name`) → creates the Intake at `:249`
- `marketingBroadcast/views/submission_handoffs.py:51` → feeds `Person.resolve` at `:75`

```python
first, _, last = full_name.partition(" ")
```

`Person.resolve` compares `(first_name, last_name)` as a pair, so all 1,848
multi-word-first-name Persons are invisible to a marketing-form submission of their
own name → a second Person, a second lead. This lane is **live**.

First-pass #9 touched `submission_handoffs.py:75` for the missing DOB only; the
`partition(" ")` twelve lines above it, and `form_handoffs.py` entirely, were not covered.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `form_handoffs.py:118-121` and `submission_handoffs.py:51` both
`partition(" ")`; both feed a `(first, last)` tuple comparison.

1. #1 item 2 (full-name comparison key) is the fix: once `Person.resolve` compares
   `canonical_full_name_key`, a partitioned `"Lai Fong Holland"` matches the stored
   `Lai Fong|Holland`. No change to these two files is required for correctness.
2. Optional hygiene: make `form_handoffs._split_name` the single splitter and call it from
   `submission_handoffs.py:51` so the two lanes cannot drift.
3. Test: submit a marketing form as `"Kathy (Kathleen) Brockwell"` on Person 74741's email →
   resolves to 74741, no new Person.

**AS BUILT — 2026-09-06 (uncommitted).** The audit's item 1 is right and I verified it
rather than took it on trust: **#1's joined-name key is the whole fix**, and no change
to either file was needed for correctness.

*Proven, not assumed:* `marketingBroadcast/test_form_submission_name_split.py` reverts
`canonical_full_name_key` to the old pair key and the lane immediately mints a second
Person (`2 != 1`, "the form minted a second Person for one human"). With the joined key
in place it resolves to the existing one. A first test pins that the split is **still
lossy** — the fix is not that the partition got smarter, it is that the comparison
stopped caring where the cut fell.

*Item 2 (hygiene) done as well:* `split_submission_name` in `form_handoffs.py` is now
the one splitter, and `submission_handoffs.py` calls it instead of carrying a
byte-identical `partition(" ")` twelve lines above the `Person.resolve` it feeds. A test
asserts that copy has not come back. The back-compat alias was removed rather than left
behind — there was exactly one in-module caller.

*Guard kept:* a genuinely different name on the same email still gets its own Person, so
the joined key has not collapsed two humans onto one contact.

## #22 — ✅ FIXED 2026-09-06 (was MEDIUM-HIGH, 0 rows today) — clinical notes and letters accept any practice's patient  [CONFIRMED]

`Notes/serializers/note.py:34` and `Notes/serializers/letter.py:108` list `patient` as
a bare `ModelSerializer` field → DRF auto-generates
`PrimaryKeyRelatedField(queryset=Patient.objects.all())`. I grepped both files: **no**
`validate_patient` in either. Routes `Notes/urls.py:95` (POST) / `:100` (PATCH);
`perform_create` sets `user` only.

The app scopes correctly everywhere it looks the patient up itself
(`views/note.py:356`, `views/patient.py:129,149`) — the hole is the writable field.
Pattern B. **Verified:** 978 notes / 41 letters, **0** cross-practice today.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `patient` is a bare field in `Notes/serializers/note.py:34` and
`letter.py:108`; no `validate_patient` in either; `Notes/views/note.py` defines
`perform_create` at `:165` and `:172` (second shadows first).

1. Add to both serializers:
   ```python
   def validate_patient(self, patient):
       practice = self.context["request"].user.current_practice
       if patient is not None and patient.practice_id != practice.id:
           raise serializers.ValidationError("Patient is not in this practice.")
       return patient
   ```
   (or `PrimaryKeyRelatedField` whose `queryset` is scoped in `__init__`).
2. Delete the dead `perform_create` at `note.py:165`.
3. Test: POST a note with another practice's `patient` → 400; 0 cross-practice rows
   before and after.

**AS BUILT — 2026-09-06 (uncommitted).** Both steps done.

`validate_patient` added to `NoteSerializer` and `NotesLetterSerializer`. `None` is
allowed through — general notes carry no patient and that path must keep working.

**The dead-code note in #35 was understated.** It says `perform_create` is defined twice
in `Notes/views/note.py` and the second shadows the first. There is a **third**
definition later in the same class — the AI-summary one — which shadows *both*. So the
two identical "ensure `is_draft` is False" copies were **both** dead and that logic has
never run. It is also redundant: `Note.is_draft` already has `default=False` at the
model level, so nothing behavioural was lost by deleting them. A test pins all three
facts.

While there, `Notes/models/note.py` declared `is_draft` **twice, identically**. Removed;
`makemigrations --check` reports "No changes detected", confirming the model state is
unchanged.

*Tests:* `Notes/test_note_patient_practice_scoping.py`, 8 tests, **2 red** without the
validators (a note AND a letter both accepting another practice's patient). Three guards
confirm the boundary is not a wall: own-practice note, own-practice letter, and a
patient-less general note all still validate.

*The 5 pre-existing `Notes.tests.test_notes` failures* (letter finalisation, phrase/
template seeds) are identical at `HEAD` with these four files reverted.

**Process note — my own error, worth recording.** While checking whether those failures
were pre-existing I backed the four files up by BASENAME, and `Notes/serializers/note.py`,
`Notes/views/note.py` and `Notes/models/note.py` all collide on `note.py`. The restore
wrote the model file over the serializer. Caught immediately (the file's first import
line was wrong), and everything was restored from `git show HEAD:<path>` and re-applied.
Nothing was lost, but back up by full path, not basename.

## #23 — ✅ FIXED 2026-09-06 (was MEDIUM-HIGH, 0 rows today) — Tasks scopes the user FKs, not the patient FKs  [CONFIRMED]

`Tasks/serializers.py:89-99` — `treatment_plan_id` and `patient_id` are
`PrimaryKeyRelatedField(queryset=…objects.all())`. The same serializer validates
`assigned_user_id` (:341), `next_assigned_user_id` (:355) and `assigned_user_ids`
(:371) against `UserPracticeRelationship`. Grep confirms **no** `validate_patient*`.

Worse: `create()` (`:409-417`) and `update()` (`:501-509`) copy `treatment_plan.patient`
onto the task, so a foreign plan id drags a foreign patient with it.
**Verified:** 431 tasks, 216 with a patient, **0** mismatched today.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `Tasks/serializers.py:89-99` — global querysets for `treatment_plan_id`
and `patient_id`; `create()`/`update()` copy `treatment_plan.patient` (`:409-417`,
`:501-509`); `validate_assigned_user_id` (`:341`) shows the scoped pattern.

1. Add `validate_treatment_plan_id` and `validate_patient_id` that check
   `value.practice_id == <practice>` using the same practice derivation as `:341`.
   Because `create()` copies `treatment_plan.patient`, validating the plan also validates
   the copied patient.
2. Test: foreign plan id → 400 (and no patient copied); 216 tasks with a patient stay
   0-mismatched.

**AS BUILT — 2026-09-06 (uncommitted).** Done. `validate_treatment_plan_id` and
`validate_patient_id` added, deriving the practice the same way `validate_assigned_user_id`
already does.

*Tests:* `Tasks/test_task_patient_scoping.py`, 6 tests, **3 red**. One is the
plan-drags-the-patient case the audit calls out: the foreign patient arrives without the
`patient_id` field ever being sent.

*A pre-existing product rule I did not know, found by a failing test of my own:* a
Patient Care task requires ONE of `treatment_plan_id`, `patient_id` or
`unlinked_patient_name`. My "a task with neither is allowed" expectation was simply
wrong; the test now pins the real rule — scoping the two FKs must not break the third
route, which has no FK to scope.

## #24 — ✅ FIXED 2026-09-06 (was MEDIUM, 0 rows today) — Appointments accepts any patient and any clinician  [CONFIRMED]

`Appointments/serializers.py:230-253` — `patient` and `clinician` bare, no `validate_*`.
`views.py:212-237` forces `practice` but never re-checks `patient`; `perform_update`
(`:260`) just saves. Same shape in `ShortNoticePatientSerializer` (`:807`), whose
duplicate check *is* practice-scoped (`:875`) while the FK it checks is not.

Also `get_patient_display_name` (`:40-50`) returns `obj.patient_name` **before**
`obj.patient` — so a mismatched pair displays the free-text name and hides the wrong
link. That is the mechanism that makes #18 invisible to staff.
**Verified:** 5 appointments, 0 mismatched.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `AppointmentCreateUpdateSerializer` (`:230-253`) lists `patient` and
`clinician` bare; `views.py:212-237` forces `practice` only; `get_patient_display_name`
(`:40-50`) returns `patient_name` before `patient`.

1. `validate_patient`: same practice **and** `archived_at IS NULL`. `validate_clinician`:
   `UserPracticeRelationship` membership (as `Tasks/serializers.py:341`). Apply the same
   two validators to `ShortNoticePatientSerializer` (`:807`).
2. `get_patient_display_name`: when `obj.patient` is set, return the **patient's** name
   and expose `patient_name` separately as `booked_as`; a disagreement between the two is
   then visible on the diary instead of hidden. This is what makes #18 detectable.

**AS BUILT — 2026-09-06 (uncommitted).** Both items done.

`PracticeScopedPatientClinicianMixin` carries `validate_patient` (same practice, and not
archived — mirroring #34) and `validate_clinician` (`UserPracticeRelationship`
membership). Applied to `AppointmentCreateUpdateSerializer` **and**
`ShortNoticePatientSerializer`, whose duplicate check was already practice-scoped while
the FK it checked was not.

`get_patient_display_name` now prefers the LINKED patient, and `booked_as` exposes the
typed name **only when the two disagree** — so it is silent noise-free in the normal
case and a visible flag in the abnormal one. This is the half that matters most: it is
the reason #18 could file an appointment under a sibling and still read correctly on the
diary.

*Tests:* `Appointments/test_appointment_patient_scoping.py`, 9 tests, **7 red** against
the old serializer — including `'Bryn Taylor' != 'Amelia Taylor'`, the display defect in
one line. Three of the seven fail as errors rather than assertions because `booked_as`
does not exist at `HEAD`, which is weaker evidence; the display test itself is a genuine
assertion failure. Guards confirm the boundary is not a wall: own-practice patient and
clinician are accepted, a walk-in with no patient still validates and still shows the
typed name.

## #25 — ✅ FIXED 2026-09-06 (was MEDIUM) — `by_contact` returns the oldest family member's log  [CONFIRMED]

`activityLog/views.py:487-497` — one `.order_by("created_at").first()` for an email
that belongs to a household. The phone branch (`:511-521`) returns the first match too.

**Verified:** `162` email groups spanning 364 logs resolve to more than one Person.
Practice 21 `gemmafeatherstone24@gmail.com` → 7 persons (148180 Maia Feathestone,
151130/151131/151169 three Gemma spellings, 151167 David, 151168 Charlie, 151170 Maia
Fetherstone) — always returns Maia. Also cross-*household*, not just cross-family:
practice 19 `info+snapfitdentures.co.uk@email.patientlygrow.com` → Anderson / Roberts /
Fitzpatrick / Kumar; practice 20 `nil@nil.com` → four unrelated people.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `by_contact` (`:487-497`) returns `.order_by("created_at").first()` for an
email; phone branch (`:511-521`) returns the first `phones_match`. 162 email groups →
>1 Person.

1. Resolve through the identity graph, and refuse to guess:
   ```python
   ch = ContactChannel.find_channel(practice, kind, contact_identifier)
   person_ids = list(PersonChannel.objects.filter(channel=ch)
                     .values_list("person_id", flat=True)) if ch else []
   if len(person_ids) == 1:
       return <that person's logs>
   if len(person_ids) > 1:
       return Response({"ambiguous": True, "persons": [...names/ids...]}, status=300)
   return 404
   ```
   This is the "shared-channel attribution" rule from the project memory: a channel names
   a human only when it names exactly one.
2. The caller (inbox / call screen) then asks the user to pick. Test with
   `gemmafeatherstone24@gmail.com` (7 Persons) and `nil@nil.com` (4 unrelated).

**AS BUILT — 2026-09-06 (uncommitted).** Done, with one deliberate difference from the
sketch: candidates are gathered from the identity graph **and** from the logs' own
`metadata`, then unioned. The graph alone would have regressed the many logs whose
`person` is still null (`Activity.person` is sparse — see #17), which the metadata
search does find.

Resolution is now:
- **more than one Person** → `300 Multiple Choices` with `{ambiguous: true, persons:
  [{id, name}]}` so the caller can ask;
- **exactly one** → that person's log, as before;
- **none, but matching logs exist** → the newest, as before. An unattributed log has no
  second candidate to be confused with, so the old behaviour is correct for it and must
  not regress.

*Measured on `prod_control`:* **162** ambiguous email groups spanning **364** logs —
both exact.

*Tests:* `activityLog/test_by_contact_ambiguity.py`, 6 tests, **3 red** (`200 != 300` —
the endpoint picking one of several people instead of asking). Guards cover the paths
that must not change: a single person still resolves to 200, an unknown address is still
404, and an unattributed log is still returned.

**Frontend follow-up:** `300` is a new response state for this endpoint. Until the inbox
and call screen handle it they will see an error where they previously got the wrong
person's log — which is the safer failure, but it is a visible change.

## #26 — ✅ FIXED 2026-09-06 (was MEDIUM, 0 rows today) — medical-history portal downgrades DOB verification to name-matching  [CONFIRMED, one agent claim WRONG]

`medicalHistory/views/staff_views.py:398` snapshots `patient.date_of_birth` (the
column); `public_views.py:72` and `:93-106` then pick the method from whether that
snapshot exists.

**Verified:** `2,336` patients have a NULL `date_of_birth` column, and for **894** of
them the real DOB is sitting in `meta_data->>'date_of_birth'` — unread. Those 894 fall
back to name-matching. This is the known "DOB columns are NULL but the values live in
`meta_data`" trap resurfacing as a security control.

**The agent claimed the name is printed in the SMS/email carrying the link. That is
wrong** — I read `medicalHistory/utils/delivery.py:41-52`: the body is
`"Please complete your medical history: {url}"` and the HTML variant, neither carrying
a name. And `patient_name_snapshot` is returned only *after* verification passes
(`public_views.py:123`). Severity is lower than reported: a guessable secret, not a
published one. **The fix is still just reading `meta_data` when the column is NULL.**

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `staff_views.py:398` snapshots `patient.date_of_birth` (column); 2,336
patients have a NULL column, 894 of them with the value in `meta_data.date_of_birth`.
Link SMS/email body carries no name (`delivery.py:41-52`) — severity as corrected above.

1. At `staff_views.py:398`:
   ```python
   from TreatmentPlan.utils.contact_keys import canonical_dob
   patient_dob_snapshot=patient.date_of_birth
       or canonical_dob((patient.meta_data or {}).get("date_of_birth")),
   ```
   (`canonical_dob` is at `contact_keys.py:61` and accepts ISO strings.)
2. Data: backfill `Patient.date_of_birth` from `meta_data` for the 894 rows — the same
   shape as the existing `backfill_recall_dob` command; verify a sample by hand.
3. Test: patient with NULL column + meta DOB → `verification_method == "dob"`.

**AS BUILT — 2026-09-06 (uncommitted).** Item 1 done. **Item 2 (the 894-row backfill) is
NOT done** — see below.

The expression is a named helper, `staff_views.dob_snapshot(patient)`, rather than an
inline `or`. That is not decoration: my first version of the test inlined the same
expression, which made every assertion a tautology — they would all have passed against
the unfixed view. Extracting the helper is what let the tests drive real code, and the
red run then produced `'name' != 'dob'`: the security control downgrading itself.

*Measured on `prod_control`, both exact:* **2,336** patients have a NULL `date_of_birth`
column; **894** of those have the real value in `meta_data`. The other 1,442 genuinely
have no DOB and still fall back to name-matching, which is what the fallback is for.

*Tests:* `medicalHistory/test_dob_snapshot_from_meta.py`, 6 tests, **3 red**. One pins
that unparseable `meta_data` yields `None` rather than a plausible date — inventing one
would lock the real patient out of their own link.

**STILL TO DO — the 894-row backfill (item 2).** Reading `meta_data` at snapshot time
fixes every link sent from now on, but `Patient.date_of_birth` is still NULL on those
rows and other code reads the column directly. Same shape as the existing
`backfill_recall_dob` command; wants a sample verified by hand.

## #27 — ✅ FIXED 2026-09-06 (was LOW-MEDIUM) — Pattern E: `patient_name` on a Task is read-only, PATCH silently no-ops  [CONFIRMED]

`Tasks/serializers.py:100-102` — `patient_name` is a `SerializerMethodField`, listed in
`read_only_fields` (`:176`); the writable equivalent is `unlinked_patient_name`. On POST
`validate()` (`:299`) errors. On PATCH the whole of `validate()` sits behind
`if not self.instance:`, so `PATCH {"patient_name": "..."}` returns **200, unchanged**.
The API returns a key it will not accept — the same class as the treatment-plan failure.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `patient_name` is a `SerializerMethodField` (`:100`) in `read_only_fields`
(`:176`); `validate()` is wholly behind `if not self.instance:` (`:299`).

1. Reject writes to a read-only key instead of dropping them: in `validate()` (outside the
   `if not self.instance` block)
   `if "patient_name" in self.initial_data: raise ValidationError({"patient_name":
   "read-only; use unlinked_patient_name"})`.
2. Same pattern for any other `SerializerMethodField` a client is likely to echo back
   (`patient_email`, `patient_phone`).

**AS BUILT — 2026-09-06 (uncommitted).** Both items done. `_READ_ONLY_ECHOES` maps each
read-only key to its writable alternative (`patient_name` → `unlinked_patient_name`;
the other two have none, so the message says they are derived). The check sits at the
top of `validate()`, OUTSIDE the `if not self.instance` block — which is the whole point,
since that block is why PATCH silently succeeded.

*Tests:* `Tasks/test_read_only_echo_rejected.py`, 6 tests, **4 red**. The headline is
`test_patching_patient_name_is_rejected`: "the API accepted a key it does not write, and
returned 200". Guards confirm the writable field still works and an unrelated PATCH is
unaffected.

## #28 — ✅ FIXED 2026-09-06 with #18 (was LOW-MEDIUM) — Pattern E: booking asks "new or existing patient?" and never reads it  [CONFIRMED]

`onlineBooking/serializers.py:347` collects, validates and stores `patient_type` on the
hold; `_match_patient` never consults it. A booker declaring themselves **new** is still
linked to a relative's record by the shared family email (#18).

**Correction to my headline:** the field is *not* unread overall — `services.py:294`
reads it and `service_practitioner_is_eligible` (`:116-124`) uses it to gate which
practitioners a new-vs-existing patient may book. The accurate, narrower finding is
that **patient matching** ignores a declaration the booker made — not that the field is
collected and never read.

*Fixed together with #18 — see that block (item 1, `patient_type` as a signal, name match as the gate).*

**AS BUILT — 2026-09-06 (uncommitted).** Fixed in **#18's shared block** — the
declaration is now read at match time, and `_build_appointment_notes(hold, patient)`
(`onlineBooking/services.py:454-472`) writes an ATTENTION line onto the appointment
when a booker who declared themselves EXISTING could not be matched, so the practice
sees the mismatch instead of it being silently discarded. Covered by
`onlineBooking/test_booking_identity.py` (line 149).

## #29 — ✅ FIXED 2026-09-06 with #10 (was LOW) — hand-rolled phone normalisation in the consent SMS path  [CONFIRMED]

`Documents/utils/notifications.py:271-286` concatenates E.164 instead of calling
`canonical_phone_e164`. Safe today only because all 13 populated practices hold
`default_country_code='+44'` (not the ISO `'GB'` behind first-pass #10) — the guard is a
data coincidence, not code.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `_normalise_phone` (`notifications.py:271-286`) hand-concatenates.

1. Replace the body with
   `return canonical_phone_e164(phone, None, default_region=practice.default_country_code) or phone`.
   Same helper as #10; removes the dependency on every practice happening to store `+44`.

**AS BUILT — 2026-09-06 (uncommitted).** Already done as part of **#10**, which found
this same site independently while sweeping for the `"+" + country_code` shape (the
audit listed two sites for #10; there were four, and this was one). `_normalise_phone`
now delegates to `canonical_phone_e164` with the practice region, so it no longer
depends on every practice happening to hold `+44` rather than `GB`. Covered by
`TreatmentPlan/tests/test_iso_country_code_sms.py`, including the structural guard that
forbids the shape returning.

## #30 — ✅ FIXED 2026-09-06 (was HIGH) — medical-history submission never checks that verification happened  [MINE, found while checking #26]

`medicalHistory/views/public_views.py:133-138`. `PublicMedicalHistorySubmitView.post`
resolves the link, checks expiry, checks `status in _TERMINAL_STATUSES` — and then
writes. It **never checks that the caller passed the DOB/name gate**.

`_TERMINAL_STATUSES = {"submitted", "expired", "revoked"}` (`:19`). A link still in
status `"sent"` — issued, never verified — passes that gate. Verification only sets
`status = "viewed"` (`:111-113`), and nothing requires it.

So the DOB check in #26 guards *reading* the questionnaire. Anyone holding the link URL
can `POST` straight to the submit endpoint and write an immutable
`MedicalHistoryEntry` — signed (`signed=True`), attributed
(`author_name=link.patient_name_snapshot`), and superseding the patient's current
history via `supersede_current_entry`.

**Two claims I made here were wrong, corrected after independent verification:**

- *"No throttle class on any of the three public views."* Wrong — I grepped the view
  file only. `settings.py:153-157` sets `DEFAULT_THROTTLE_CLASSES` including
  `AnonRateThrottle` at `1000/hour` in production, and these `APIView`s do not override
  `throttle_classes`, so a global per-IP anon throttle applies.
- *"Brute-forceable."* Wrong. `medicalHistory/models.py:67-72`: `_generate_link_code`
  draws **8** characters via `secrets.choice` from a 32-character alphabet (A-Z minus
  O and I, plus 2-9) = 32⁸ ≈ 1.1 × 10¹² codes. At 1000/hour that is ~10⁸ years per hit.
  This is a legitimate capability URL.

**Reassessed:** not an open endpoint, not a guessable secret. It is the **absence of a
second factor on the write path where the read path has one** — anyone who obtains the
link (forwarded SMS, a shared household phone, shoulder-surfing) can file a signed
medical history in the patient's name without the DOB check that merely *reading* the
same questionnaire requires. Real and worth fixing; narrower than I first wrote.

**Also: `medicalHistory_medicalhistorylinkrequest` has 0 rows.** No link has ever been
issued, so like #18-#20 this is pre-launch exposure, not live damage. I tagged those
three and failed to tag this one and #26.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `PublicMedicalHistorySubmitView.post` (`:133-138`) checks expiry and
`_TERMINAL_STATUSES` only; verification sets `status="viewed"` at `:111-113`; 0 link
requests issued yet.

1. Minimal: require the gate to have been passed —
   `if link.status != "viewed": return 403 "Verify your identity first."` at `:138`.
2. Proper: on successful verify, mint a short-lived signed token
   (`django.core.signing.TimestampSigner`, e.g. 30 min) bound to `link.id` and return it;
   the submit view requires and checks it. Status alone proves *someone* verified on
   *some* device; the token proves *this* client did.
3. Test: POST submit on a `sent` link → 403; after verify → 201.

**AS BUILT — 2026-09-06 (uncommitted).** Item 2 (the proper fix) implemented, not the
minimal status check — for the reason the finding itself gives: `status == "viewed"`
proves SOMEONE verified on SOME device, not that this client did.

Verify now returns a `verification_token` — `TimestampSigner`, salted, bound to the
link id, 30-minute lifetime — and submit requires it. A rejected attempt writes a
`submit_rejected_unverified` audit event, because on a clinical portal "someone holds
this link but could not verify" is exactly the signal a practice wants. That needed a
new choice on `MedicalHistoryLinkAuditEvent.event_type`, hence migration
`medicalHistory/0005`.

*Tests:* `medicalHistory/test_submit_requires_verification.py`, 7 tests, **5 red**
against the ungated write — including `1 != 0`, an immutable signed clinical record
actually being created by a caller who never verified. Two further tests pin that the
token is bound to THIS link (a token minted for another link is refused) and that
tampering with it fails.

**This changes the public API contract**, so three existing tests in
`medicalHistory/tests.py` were updated: they exercised the old flow, submitting with no
token. The end-to-end one now also asserts that an unverified client gets a 403, so the
old behaviour is pinned as *forbidden* rather than merely no longer exercised.

*Front-end note:* the portal must carry `verification_token` from the verify response
into the submit request. It has not launched (0 link requests ever issued), so this is
the moment to change it.

---

## Checked and clean (agent's negative results, spot-checked)

- **Labs** — `validate_patient_id` is practice-scoped on both create (`serializers.py:627`)
  and update (`:792`).
- **Stock** — `implant_views.py:129` *looks* unscoped but
  `ImplantBatchCheckoutSerializer.validate_patient_id` (`Stock/serializers.py:2140`)
  already rejected a foreign id. This is the exact false-positive shape that produced my
  wrong "0 unscoped viewsets" earlier — the agent checked the serializer the view uses.
- **patient_accounts** — everything goes through `_get_patient(patient_id, practice)`
  (`views.py:77`). The practice-wide dedupe in `services/dentally_backfill.py:141-157`
  could theoretically cross patients; **0** such rows in 23,423 charges.
- **patientDocuments** — `_get_owned_patient` (`views.py:59`) scopes everything.
- **compliance** — validates `patient.practice_id` on create *and* update and re-derives
  `patient_name`. 0 divergent rows in 208.
- **charting** (`views.py:43`), **Invoices, Statistics, teamChat, HR, referralProgram,
  payments, settings** (no patient-identity code), **productAnalytics,
  practiceConnection** (no patient/person references at all).
- **Documents** — the public signing portal takes `signer_name` as free text and never
  compares it to the patient (`public_views.py:664,843`); with shared family emails the
  signer on a consent record is unverified. Design choice, noted not filed.

---

## Suggested order for the second batch

1. **#17** — the only one leaking clinical text today; one-line fix.
2. **#30** — an unauthenticated write to a clinical record.
3. **#19 + #18 + #20 + #28** — all four before online booking launches.
4. **#21** — same root as #1, on a lane that is live now.
5. **#22, #23, #24** — three open tenant boundaries, all currently undamaged.
6. **#25, #26, #27, #29.**

---

# #31–#32 — public treatment-plan endpoint (from automated security review, verified by me)

Both flagged by the background security reviewer against
`TreatmentPlan/views/public_treatment_views.py`. I read the code and sized both
against `prod_control`. Both hold. The viewset is `permission_classes = [AllowAny]`
(`:246`) — fully public, no authentication.

## #31 — ✅ FIXED 2026-09-06 (was HIGH) — the verification code is not bound to the plan it unlocks  [CONFIRMED]

`share_availability` (`:406-416`):

```python
code_obj = SixDigitVerificationCode.objects.filter(
    code=verification_code, is_active=True
).first()          # not filtered by pk
...
treatment_plan = TreatmentPlan.objects.get(pk=pk)   # pk comes from the URL
```

The code proves only that *some* valid code was presented, never that it was issued
for **this** plan. The only thing standing between the two is the practice comparison
at `:424-441` — so any holder of any active code can append text to the `notes` of
**any treatment plan in the same practice**, and that text is what the Open Plan
journeys table shows staff.

This is a **write**, not a read: `treatment_plan.notes` is appended and saved (`:459-466`).

**Verified sizing (prod_control):** 4 active codes exist —

| code | user | practice | plans writable in that practice |
|---|---|---|---|
| `vAXJAU`, `5u8phP` | 66, 71 | 16 | **186** |
| `hD5faM` | 124 | 21 | 20 |
| `pkPQCd` | 166 | 24 | 1 |

The practical attacker is not a brute-forcer — codes are 6 mixed-case alphanumerics
(~5.7×10¹⁰), so guessing is impractical. It is a **legitimate patient who was sent
their own code** and changes the `pk` in the URL.

**Also verified: the model has no expiry field at all.** `SixDigitVerificationCode`
(`models.py:4582-4594`) is `user, code, is_active, created_at, updated_at` — nothing
else. The error string "Invalid or expired verification code" describes a check that
cannot happen; `hD5faM` has been active since **2026-04-22**. Any suggested fix that
adds `expires_at__gt=now()` requires a migration first.

Fix: filter the code by the plan (`treatment_plan_id=pk`, once such a field exists) or
resolve the plan *from* the code rather than trusting the URL `pk`.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `share_availability` `:406-416` filters `code`+`is_active` only, then
`TreatmentPlan.objects.get(pk=pk)` from the URL; the model (`models.py:4582-4594`) has
`user, code, is_active, created_at, updated_at` — no plan, no expiry. 4 active codes
(`hD5faM` since 2026-04-22). Codes are issued at
`TreatmentPlan/views/verification_views.py:152`.

1. Bind the code to the plan: migration adding `treatment_plan = ForeignKey(TreatmentPlan,
   null=True, on_delete=CASCADE)` and `expires_at = DateTimeField(null=True)`; set both
   at `verification_views.py:152`.
2. In `retrieve`, `share_availability` and `get_procedure_view`, filter
   `SixDigitVerificationCode.objects.filter(code=..., is_active=True, treatment_plan_id=pk,
   expires_at__gt=now())` — the URL `pk` becomes a *claim the code must corroborate*.
3. Existing 4 codes have no plan: deactivate them and re-issue, or accept them only
   until `expires_at` is set. Do not keep a null-plan bypass.
4. Test: code issued for plan A, URL `pk` = plan B (same practice) → 403.

**AS BUILT — 2026-09-06 (uncommitted).** Items 1, 2 and 4 done; item 3 (the 4 live
codes) is a deliberate deviation, explained below.

Migration `TreatmentPlan/0145` adds `treatment_plan` (nullable FK) and `expires_at`.
`SixDigitVerificationCode.is_usable_for(plan_id)` is the rule: active, not expired, and
— **when bound** — for THIS plan. New codes get a 14-day expiry, which is what finally
makes the endpoints' own "Invalid or **expired** verification code" message true; the
model had no expiry field at all and `hD5faM` had been live since 2026-04-22.

All three copies of the check (`retrieve`, `share_availability`, `get_procedure_view` —
about 40 lines each) now call one function, `authorise_code_for_plan`, which also
carries #32's deny-by-default.

**Deviation from item 3, stated plainly.** The audit says "Do not keep a null-plan
bypass." I have not made the binding mandatory, because **the issue endpoint is
per-STAFF-USER, not per-plan** (`verification_views.py:152` takes no plan), so requiring
a binding today would break the feature for every code. Instead: the issue endpoint now
accepts an optional `treatment_plan_id` and binds when given; a bound code is enforced
strictly; an unbound code still has to pass the practice check, which no longer fails
open.

**Residual risk, explicit:** until the frontend sends `treatment_plan_id` when
generating a code, new codes remain practice-wide — narrowed by the expiry and by #32,
but not plan-bound. That is the remaining half of #31 and it needs a frontend change,
not a backend one.

*Tests:* `TreatmentPlan/tests/test_public_plan_code_authorisation.py`, 8 tests, **5
red** — covering both findings, including a code issued for plan A acting on plan B, and
the exact #31+#32 combination (unbound code held by a user with no practice).

## #32 — ✅ FIXED 2026-09-06 (was HIGH, latent) — the practice check fails open  [CONFIRMED]

`share_availability` (`:424-425`), and the same shape in `retrieve` (`:274-275`) and
`get_procedure_view` (`:514-515`):

```python
practice_user = getattr(code_obj.user, "current_practice", None)
if practice_user:            # <-- no else
    ...
    if plan_practice != practice_user:
        return 403
```

When the code's user has no `current_practice`, the entire comparison is skipped and
the request proceeds — so #31 stops being practice-bounded and becomes writable
across **all 547 treatment plans in every practice**.

**Verified:** `16 of 213` users have `current_practice_id IS NULL`. None of the 4
currently-active codes belongs to one of them, so this is **latent, not live** — but
it is one code-issue away, and issuing a code is a routine staff action.

Fix: deny by default — return 403 when `practice_user` cannot be determined, then
compare unconditionally.

**Note:** these are the same Pattern B shape as #22/#23/#24 (scoping present on one
path, absent or conditional on its sibling), but on a public unauthenticated endpoint,
which is why they rank above them.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `if practice_user:` with no `else` at `:424`, `:274`, `:514`; 16 of 213
users have `current_practice_id IS NULL`.

1. Deny by default at all three sites:
   ```python
   practice_user = getattr(code_obj.user, "current_practice", None)
   if practice_user is None:
       return Response({"error": "..."}, status=403)
   ```
   then compare unconditionally. With #31 in place the plan check makes this belt-and-
   braces, but keep it — it is the only guard for the legacy null-plan codes.

**AS BUILT — 2026-09-06 (uncommitted).** Done, inside the shared
`authorise_code_for_plan` (see #31). `practice_user is None` is now a 403 rather than a
skipped check, and the comparison runs unconditionally after it.

The audit's own note that this is "the only guard for the legacy null-plan codes" turned
out to be more load-bearing than expected: because the plan binding cannot be made
mandatory yet (see #31's deviation), this deny-by-default IS the protection for every
unbound code, not a belt-and-braces addition to it.

---

# #33–#35 — found by the verification pass, confirmed by me

## #33 — ✅ FIXED 2026-09-06 with #18 (was HIGH) — online booking creates a Patient with no Person at all  [CONFIRMED]

`onlineBooking/services.py:519-526`:

```python
patient = Patient.objects.create(
    practice=practice, first_name=first_name, last_name=last_name,
    email=email, country_code=..., phone_number=...,
)
```

No `Person.resolve`, no `ContactChannel.get_or_create_channel`, no `person=`. I grepped
the **entire app**: the only `Person`/`ContactChannel` references in `onlineBooking` are
`services.py:338` and `:347`, both inside `_match_patient`'s read-only lookup. Nothing
in the app ever writes to the identity graph.

So every paid online booking that fails to match (#18 and #19 both dead → this branch)
mints a `person_id IS NULL` Patient: invisible to `Person.resolve`, to dedupe, to
household grouping — and, because no ContactChannel is created either, invisible to the
very phone-channel branch at `:344-349` that would have matched this same human on their
next booking. Each booking makes another one.

This is the same three lines as #20, and strictly worse than what #20 says: the row is
not merely a duplicate, it is **outside the system that would ever detect the
duplicate**. **Verified: 99 Person-less Patients already exist** from other lanes, so
the shape is not hypothetical.

*Fixed together with #18 — see that block (item 2). 99 Person-less Patients exist today from other lanes; run `bridge_dentally_identity`-style resolution over them after #1.*

**AS BUILT — 2026-09-06 (uncommitted).** Fixed in **#18's shared block** — the create
path now goes through `ContactChannel.get_or_create_channel` then `Person.resolve`
then `person=` on the Patient, so a booking paid for by card no longer lands outside
the identity graph. Covered by `onlineBooking/test_booking_identity.py`.

## #34 — ✅ FIXED 2026-09-06 with #18 (was LOW-MEDIUM) — `_match_patient` can match an archived patient  [CONFIRMED]

`onlineBooking/services.py:332-349` has no `archived_at__isnull=True` filter, so an
archived patient is a valid target for a live booking. **0 archived patients today** —
but archiving was changed on 2026-08-12 to stop deleting rows, so that population is
expected to grow from zero.

*Fixed together with #18 — see that block (item 1, `archived_at__isnull=True`).*

**AS BUILT — 2026-09-06 (uncommitted).** Fixed in **#18's shared block** —
`archived_at__isnull=True` on the candidate base queryset
(`onlineBooking/services.py:371`), pinned by
`test_an_archived_patient_is_never_matched`.

## #35 — ✅ FIXED 2026-09-06 with #17 (was INFO) — `patient_activities` never validates that `patient_id` is in the caller's practice  [CONFIRMED]

`activityLog/views.py:337-364` takes the id straight from the URL. The practice filters
on the two downstream queries contain the damage today, so it is not exploitable as
enumeration — but #17 is precisely a case where those filters do not do the job. Add
`Patient.objects.filter(id=patient_id, practice=practice).exists()` as part of the #17
fix rather than as a separate ticket.

**Note for the #22 fix:** `Notes/views/note.py` defines `perform_create` **twice**,
identically, at `:165` and `:172`. The second shadows the first. Harmless as written,
but it is the exact method #22 identifies as the only thing between the request and the
database — anyone patching there should know the first definition is dead code.

*Fixed together with #17 — the scoped `Patient.objects.filter(id=..., practice=practice)` lookup is the first step of that fix.*

---

# #36–#44 — exhaustive follow-up sweep

These findings came from two additional sweeps of the originally excluded apps and
patient-touching Go code. Before adding them, the entire document was re-read to avoid
duplicating #1–#35. The Go migration splitter was folded into #1 above, and the two
shared-phone call-agent paths were consolidated as #38 below.

All counts in this section were rerun against `prod_control` in read-only transactions.
The database is being updated by live sync processes, so these figures are a verified
snapshot rather than immutable totals.

## #36 — ✅ FIXED 2026-09-06 (was HIGH) — day-list history counts cross practice boundaries  [CONFIRMED]

**Where:**

- `EmailServiceGo/internal/dentally/daylist/ai/db.go:84-89` and `:495-500`
- `EmailServiceGo/internal/dentally/daylist/ai/high_risk_summary.go:246-251`
- `EmailServiceGo/internal/workflows/actions/dentally_unified.go:733-738`

The outer appointment query is practice-scoped. Its correlated history subqueries are
not:

```sql
WHERE h.dentally_patient_id = a.dentally_patient_id
  AND h.state = 'Cancelled'
  AND h.start_time < a.start_time
```

Dentally patient ids are unique only inside a practice. The subqueries therefore count
DNAs and cancellations belonging to unrelated patients in every other practice that
uses the same Dentally id. Those contaminated values feed no-show risk, the AI patient
summary, the high-risk summary and the workflow action.

**Verified:** **111,024 appointment rows**, covering **5,990 distinct
(practice, Dentally-patient) keys**, currently receive a different history count from
the correctly practice-scoped query.

Concrete example: practice 24, Dentally patient 54688, Barbara Hole, appointment
`be1ac908-778e-43bd-981a-926076975ff9`. Her practice-scoped prior-cancellation count is
**0**; the production query returns **180**, all imported from other practices that
reuse id 54688. Practice 24 Dentally patient 26508, Timothy Steel, similarly receives
4 DNAs and 174 cancellations where both correctly scoped counts are zero.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified by execution:* appointment `be1ac908-…` (Barbara Hole, practice 24) —
unscoped prior-cancellation subquery **180**, practice-scoped **0**. All four sites read
as quoted. Migration 0137's index is `(dentally_patient_id, state, start_time)` — it does
not include `practice_id`.

1. Add `AND h.practice_id = a.practice_id` to every correlated subquery:
   `daylist/ai/db.go:84-89`, `:495-500`, `high_risk_summary.go:246-251`,
   `dentally_unified.go:733-738`.
2. Re-index: `(practice_id, dentally_patient_id, state, start_time)` so the scoped
   subquery stays an index range scan (add via a Django migration on
   `DentallyAppointment`, the Go side reads only).
3. **Consequence to flag:** the no-show risk weights were refit on prod history
   (see project memory "No-show risk recalibrated") using these contaminated counts for
   5,990 (practice, patient) keys. After the fix, re-run the calibration or at least
   re-check the `prior_no_shows`/`prior_cancellations` coefficients.
4. Verify: the Barbara Hole query → 0/0; the 111,024-row diff → 0.

**AS BUILT — 2026-09-06 (uncommitted).** Items 1, 2 and 4 done; item 3 is a flag for
you, below.

`AND h.practice_id = a.practice_id` added to all **four** subquery pairs —
`daylist/ai/db.go` (two), `high_risk_summary.go`, `workflows/actions/dentally_unified.go`.

**Index added** (`dentallyIntegration/0169`):
`(practice, dentally_patient_id, state, start_time)`, practice FIRST. The old index
`idx_dappt_patient_state_start` is kept — other readers use it — but without a
practice-leading index the newly-scoped predicate would filter after the scan rather
than narrowing it, on a query that runs once per appointment per day-list request.

*Verified on `prod_control`:* Barbara Hole's appointment `be1ac908-…` returns
**unscoped 180 / practice-scoped 0**, exactly as reported. The full diff is **110,936
rows across 5,989 (practice, patient) keys** — the audit said 111,024 / 5,990; mine
counts rows where either count differs and is the conservative figure.

*Test:* `internal/dentally/daylist/ai/history_scope_test.go` is a SOURCE guard, not a
query test — the behaviour is already proven directly against production data, and what
a regression test usefully adds is stopping the predicate being dropped again in any of
the four places. Red-run: reverting one file makes it report
"db.go: 4 history subquery/ies still match on the patient id alone".

**⚠️ FLAG FOR YOU (item 3), unchanged and important.** The no-show risk weights were
refit on production history using these contaminated counts for ~5,990
(practice, patient) keys — see the project memory note "No-show risk recalibrated". The
`prior_no_shows` and `prior_cancellations` coefficients were therefore fitted against
partly-fictional inputs. **Re-run the calibration, or at minimum re-check those two
coefficients, after this deploys.** I have not touched the weights.

## #37 — ✅ FIXED 2026-09-06 (was HIGH) — one marketing profile represents an arbitrary member of a fused Person  [CONFIRMED]

**Where:** `marketingBroadcast/tasks.py:109-150`.

The profile sync iterates one `Person`, selects
`person.patients.filter(...).first()`, and uses that one Patient's Dentally id to fill
the Person-level marketing profile with payment plan, spend, visits, appointment
counts, gender and archived state. A Person that incorrectly holds several family
members therefore gets one arbitrary member's facts, and every member sharing that
Person is segmented using those facts.

This is a distinct downstream reader of the collapsed Persons described by #3, in the
same family as the recall-reader defect in #2.

**Verified:** **2,357 MarketingPatientProfile rows**, standing for **5,802 Patient
rows with different names**, are ambiguous now.

Concrete example: profile 76134 belongs to Person 152744 in practice 24. That Person
contains Patient 131893 John Geary plus Gigi, Johnny, Hallie, Beau and Micaela Geary
(six different humans). The single profile carries one DOB/spend/visit history — at
the snapshot it showed total spend **8445** — selected through the first Patient rather
than the particular person a campaign is meant to target.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `marketingBroadcast/tasks.py:110` — `person.patients.filter(...).first()`;
2,357 Persons hold different-named patients (same population as #3).

1. Short term: pick the patient that *is* this Person, not an arbitrary one —
   `[p for p in person.patients.filter(practice_id=...) if canonical_full_name_key(p.first_name,
   p.last_name) == canonical_full_name_key(person.first_name, person.last_name)]`;
   if that is not exactly one, write the profile with the identity fields blank and log
   `ambiguous_person` so the segment builder can exclude it.
2. Root cause is #3 / defect D2; once `split_collapsed_persons` has run, every Person has
   one human and `.first()` is safe — but keep the guard, it is what proves it.
3. Verify: profile 76134 (Person 152744, six Gearys) → ambiguous until split, then one
   profile per human.

**AS BUILT — 2026-09-06 (uncommitted).** Items 1 and 2 done, and item 1's guard is
wired through to behaviour rather than left as a flag.

The sync now picks the Patient whose joined name matches the PERSON's own name. Where
that is not exactly one — and there is more than one patient to choose between — the
profile is written with the identity fields blank and `is_ambiguous=True`
(`marketingBroadcast/0038`).

**The flag changes behaviour, or it would be decoration:** `resolve_segment` and
`resolve_audience_ids` both exclude `is_ambiguous` profiles, and the eligibility
explainer reports `ambiguous_person` as a reason so staff can see WHY someone was left
out of a campaign.

**A correction to my own first attempt, worth recording.** I initially expected "Person
holds six humans ⇒ ambiguous". That is wrong: a Person NAMED "John Geary" that also
holds misfiled relatives *is* John, and picking John is right — it is precisely the
improvement over `.first()`, which returned whichever row the database offered. The
genuinely unattributable cases are narrower: **no** member carries the Person's own
name, or **two** do (father and son). The tests were corrected, not the code.

*Measured on `prod_control`:* 2,357 fused Persons covering 5,802 patient rows — the same
population as #3, confirmed.

*Tests:* `marketingBroadcast/test_profile_ambiguous_person.py`, 9 tests, **5 red**
against `.first()` and the un-excluded segments. The full `marketingBroadcast` suite
(707 tests) passes.

*Root cause remains #3.* Once `split_collapsed_persons` has run, every Person holds one
human and this resolves normally. The guard stays afterwards, because it is what proves
the split worked.

## #38 — ✅ FIXED 2026-09-06 (was HIGH) — the call agent chooses and persists the first human on a shared phone  [CONFIRMED]

**Where:**

- canonical Person lookup: `EmailServiceGo/internal/callagent/handlers.go:339-368`,
  `storage.go:857-885`, and `tools.go:132-176`
- active Telnyx lookup: `EmailServiceGo/internal/callagent/telnyx/handlers.go:1128-1158`
- intake persistence: `EmailServiceGo/internal/callagent/storage.go:665-677,721-748`

`canonicalPhonePersonIDs` can correctly return several Persons for a household phone.
Every consumer immediately performs `.First()` across those Persons. No caller name,
DOB or ambiguity check decides which human is speaking. The selected name is injected
into the assistant/session token; the storage path then overwrites the name extracted
from the transcript and persists a new Intake under that selected human's name.

The separate active Telnyx implementation does the same thing using legacy
country-code concatenation and `.First()` on Patient, Intake and Nurture.

**Verified conservative exposure, counting Patient rows only in practices with an
active receiving number:** **2,689 ambiguous phone groups / 6,715 Patient rows**.
There are **194 stored calls across 98 of those phone keys**.

Concrete committed error: CallRecord 137, practice 16, phone `+447950477721`, stored
`patient_name='Billy Wright'`. The transcript caller says she is Carly. That phone is
held by Patient 32821 Billy Wright, Patient 32828 Bonnita Wright, Patient 32881 Carly
Surry and Patient 36851 Carly Wright. The storage path created Intake **396** as
**Billy Wright** and it is now linked to Billy's Person 121089. This is no longer just
a wrong greeting: another human's call has been committed to Billy's identity.

CallRecord 128 provides a second direct example: the stored name is Hazel Perry, while
the caller identifies herself as Elizabeth Anne Martin; Patients 29093 Hazel Perry and
30458 Elizabeth Martin share the number.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `canonicalPhonePersonIDs` (`storage.go:835-849`) returns all Persons on the
canonical channel; `handlers.go:342-368`, `storage.go:862-885`, `tools.go:137-176` and
`telnyx/handlers.go:1128-1158` each `.First()`. `storage.go:665-677` overrides the
transcript name with that pick. Correction: `call_records` has no `patient_name` column —
the name is in `dynamic_variables->>'patient_name'` (rows 137 → "Billy Wright", 128 →
"Hazel Perry", both practice 16). Intake 396 is `Billy|Wright`, `person_id=121089`,
`source='call-agent'`; +447950477721 is held by 32821 Billy, 32828 Bonnita, 36851 Carly
Wright and 32881 Carly Surry.

1. One helper in `internal/callagent`, used by all four sites:
   `resolveCallerPerson(db, practiceID, phone) (personID int64, candidates []Person, ok bool)`
   — `ok` only when `len(personIDs) == 1`.
2. Greeting paths (`handlers.go:342`, `telnyx/handlers.go:1128`, `tools.go:137`): when not
   `ok`, return no name (generic "Hi,") and, for the tool, `Found:true, Ambiguous:true`
   with the candidate names so the assistant asks "who am I speaking to?".
3. Storage path (`storage.go:665`): keep the transcript-extracted name unless exactly one
   candidate matches it (`personname.CanonicalFullKey` compare); never overwrite with a
   `.First()` pick. Then set `person_id` from the matched Person (also fixes #41).
4. Data: Intake 396 → re-attribute to Carly (transcript) and unlink from 121089; audit
   the 194 stored calls on the 98 ambiguous keys the same way.

**AS BUILT — 2026-09-06 (uncommitted).** Items 1–3 done. **Item 4 (the data repair) is
NOT done** — see below.

One resolver in `internal/callagent/storage.go`, used by all four sites:
`ResolveCallerCandidates` (every live human on the line, one per Person, across
Patient/Intake/Nurture), `SoleCallerCandidate` (the one human, or nil) and
`CandidateMatchingName` (the candidate confirming a transcript name, or nil).

- **Greeting** (`handlers.go:342`, `telnyx/handlers.go:1128`): a name only when the
  phone names exactly one human, otherwise no name. A generic "Hi," is not a failure;
  a stranger's name is.
- **Tool** (`tools.go:137`): a shared line returns `Found:true, Ambiguous:true` with the
  candidate names and a message telling the assistant to ask who is calling.
- **Storage** (`storage.go:665`): the transcript name is now KEPT unless a contact on
  the line confirms it, and only replaced when the phone names exactly one human. It
  used to be overwritten unconditionally, which is how Intake 396 exists.

**#44 fixed in the same change:** `telnyx/handlers.go:1124` had its own legacy
`CONCAT(country_code, phone_number)` matcher that never consulted the canonical key or
secondary numbers. It now calls the shared resolver, so it gains both the canonical
lookup and the ambiguity refusal at once.

*Every production fact re-verified on `prod_control`:* +447950477721 is held by 32821
Billy Wright, 32828 Bonnita Wright, 32881 Carly Surry and 36851 Carly Wright; CallRecord
137 stores `patient_name = "Billy Wright"`; Intake **396** is `Billy|Wright`,
`person_id=121089`, `source='call-agent'`. Robin Whittaker (32247) carries secondary
`1245224464`, the number CallRecord 74 came from.

*Tests:* `internal/callagent/caller_identity_test.go`, 9 tests, **2 red** under
`.First()` semantics — and the red output names the defect: *picked "Billy Wright" from
a four-person household*. One test pins that a moved first/last split still confirms a
transcript name, since finding #1's key is what compares them.

**STILL TO DO — the data repair (item 4).** Intake 396 is still filed as Billy Wright on
Person 121089 from Carly's call, and the other 194 stored calls across 98 ambiguous
phone keys have not been audited. This needs a human reading transcripts; nothing
automatic can decide who was actually speaking.

## #39 — ✅ FIXED 2026-09-06 (was HIGH) — the call-agent DOB field is accepted and then ignored  [CONFIRMED]

**Where:** `EmailServiceGo/internal/callagent/tools.go:33-43,113-116,262-309`.

`LookupPatientRequest` publicly accepts `date_of_birth`, and `LookupPatient` passes it
to `lookupByName`. The function receives `dob` but never uses it in any query. It
filters only practice, first name and last name, then returns `.First()`.

This is Pattern E with identity consequences: the caller/assistant supplies the very
field designed to distinguish two same-named people, the API silently discards it,
and reports a different patient as found.

**Verified exposure:** **493 same-practice name groups / 1,021 active Patient rows**
have at least two different non-null DOBs.

Concrete example: practice 21 has four William Smiths — Patients 44147, 44764, 52521
and 112616 — with DOBs 2007-12-06, 2012-07-01, 1989-10-20 and 1996-09-01. Supplying
any one of those DOBs does not alter the SQL or which William `.First()` returns.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `lookupByName(firstName, lastName, dob, practiceID)` (`tools.go:262-309`)
never references `dob`; 493 same-practice name groups differ on DOB.

1. Parse `dob` (accept `YYYY-MM-DD` and `DD/MM/YYYY`, the formats the request comment
   promises) and add `AND date_of_birth = ?` when present.
2. When `dob` is empty, count matches first; if >1, return `Found:false, Ambiguous:true,
   Message:"Several patients match — please ask for date of birth"` instead of `.First()`.
3. Test: practice 21 William Smith with each of the four DOBs → four different Patients.

**AS BUILT — 2026-09-06 (uncommitted).** Both items done.

`parseLookupDOB` reads the two formats the request struct's own comment promises
(`YYYY-MM-DD` and day-first `DD/MM/YYYY`) and returns `""` for anything else — junk
means "not supplied", so the caller ASKS rather than filtering on a date nobody gave.
Inventing one would exclude the real patient.

The name lookup now applies `date_of_birth = ?` when a DOB parses, and — with no DOB
and more than one patient of that name — returns `Found:false, Ambiguous:true` asking
for one, instead of `.First()`.

`Patient` and `Intake` gained a `DateOfBirth` field; the Go structs had none, which is
part of why the column was never queried. `TreatmentPlan_nurture` has no such column,
so the nurture branch is unchanged.

*Measured on `prod_control`:* **493** same-practice name groups covering **1,021**
active patients differ on DOB — exact. The four William Smiths are 44147 (2007-12-06),
44764 (2012-07-01), 52521 (1989-10-20) and 112616 (1996-09-01).

*Tests:* 5 new in `internal/callagent/caller_identity_test.go`. **Four of them are pure
parser tests and would stay green if the filter were dropped from the query** — which is
exactly the defect. So the fifth, `TestLookupByNameActuallyFiltersOnTheDOB`, guards the
WIRING; deleting the predicate makes it fail with "the field is accepted and ignored
again".

## #40 — ✅ FIXED 2026-09-06 (was MEDIUM-HIGH, 0 rows today) — ActivityLog create/update accepts foreign Persons and objects  [CONFIRMED by execution]

**Where:** `activityLog/serializers.py:89-137,161-200`, consumed by
`activityLog/views.py:108-142` and the PUT/PATCH routes at `activityLog/urls.py:19-26`.

The list/retrieve queryset is correctly practice-scoped, but both write serializers
use global related-field querysets. Create validates only that `object_id` exists for
the selected content type; it never checks the object's practice, the Person's
practice, or that the Person represents that object. Update exposes the same writable
`person` field without revalidation. The view forces the ActivityLog's `practice` to
the current practice, leaving a foreign Person or clinical object attached to it.

**Executed proof:** under a read-only transaction the actual
`ActivityLogCreateSerializer` accepted `valid=True, errors={}` for Patient 9593
Manifest Chakalov (practice 13) paired with Person 74644 (practice 16).

**Sizing:** all **83,700 Persons** and every object reachable through Django's content
types are selectable. There are 4,220 ActivityLog rows and **0 persisted
Person/practice mismatches today**.

This is distinct from #17/#35: those findings leak records while reading a colliding
id; this one lets an API client write an explicitly foreign Person/content object.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `ActivityLogCreateSerializer.validate` (`:184-200`) checks only that the
object exists; `person` is a global FK on both serializers; `views.py:124-137` injects the
practice after validation.

**AS BUILT — 2026-09-06 (uncommitted).** Done, on BOTH write paths.

`PracticeScopedActivityTargetMixin` carries `validate_person` and
`_validate_content_object`, and is mixed into `ActivityLogCreateSerializer` **and**
`ActivityLogSerializer` — the latter is what PUT/PATCH uses and exposes a writable
`person` with no revalidation, so fixing create alone would have left the hole open.

The content-object check only applies to models that HAVE a practice: a generic FK can
address any table Django knows about, and not all of them are practice-scoped. Where
there is a `practice_id`, it must match.

The view now passes the practice it had already resolved into the serializer context —
it was previously applied only AFTER validation, which is precisely why a foreign
Person could end up on a local row.

*Tests:* `activityLog/test_write_serializer_practice_scoping.py`, 9 tests, **5 red** —
including the exact pairing the audit executed (another practice's Patient with a local
Person) and both update variants. Four guards confirm the rule has not over-reached:
own-practice writes still validate, a missing object is still rejected, a Person is
still required, and an unrelated PATCH is unaffected.

1. Pass the practice into the serializer context (`views.py:129`:
   `self.get_serializer(data=..., context={**self.get_serializer_context(),
   "practice": practice})`) and add:
   ```python
   def validate_person(self, person):
       if person.practice_id != self.context["practice"].id:
           raise serializers.ValidationError("Person is not in this practice.")
       return person
   def validate(self, data):
       ... existing object-exists check ...
       obj = model_class.objects.filter(pk=object_id).first()
       if obj is not None and getattr(obj, "practice_id", None) not in (None, self.context["practice"].id):
           raise serializers.ValidationError("Object is not in this practice.")
   ```
   Same `validate_person` on the update serializer.
2. Test: the executed proof (Patient 9593 practice 13 + Person 74644 practice 16) → 400.

## #41 — ✅ FIXED 2026-09-06 (was MEDIUM) — Go call-agent Intakes are inserted outside the identity graph  [CONFIRMED]

**Where:** `EmailServiceGo/internal/callagent/tools.go:342-368` defines the inserted
`IntakeRecord`; `EmailServiceGo/internal/callagent/storage.go:721-748` writes it.

`IntakeRecord` has no `PersonID` field, and the insert never resolves a Person or
creates/links a ContactChannel. `notifyIntakeCreated` (`storage.go:765-821`) is only a
realtime notification and does not repair identity. Every new call-agent Intake is
therefore initially written with `person_id IS NULL`; it depends on a later repair
process to enter the identity graph.

**Verified:** 1,535 current Intakes have `source='call-agent'`. **1,527 have since been
linked by later processing, while 8 remain Person-less.** Remaining examples include
Intakes 3609, 3610, 3899 and 3976 in practice 16. This is a writer defect even though
the repair process eventually catches most rows: during the gap they are invisible to
Person-based dedupe, timelines and contact lookup, and failed repairs remain invisible.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `IntakeRecord` (`tools.go:342-368`) has no `PersonID`; `storage.go:721-748`
inserts without a Person; Django's Intake Person resolution runs in a **pre-save signal**
(`signals.py:31` comment: "set by the pre-save signal") that a Go `INSERT` never fires.
8 Person-less call-agent intakes remain — **all eight have `first_name='Unknown'`**
(3235, 3274, 3535, 3609, 3610, 3670, 3899, 3976, practice 16), which is why the later
repair skipped them.

1. Resolve in Go at write time, the way the Dentally migration does: canonicalise the
   phone with `phone.CanonicalE164(raw, cc, phone.PracticeRegion(db, practiceID))`,
   `upsertContactChannel` (`migration/service.go`, exported or moved to a shared package),
   `livePersonsOnChannels`, name match via `personname`, create when none — and set
   `PersonID` on `IntakeRecord`. Combine with #38 item 3 so an ambiguous phone never
   auto-links.
2. For `Unknown` callers: create the Person with the placeholder name and link the
   channel anyway — a Person with a phone and no name is still inside the graph; a
   Person-less intake is not.

**AS BUILT — 2026-09-06 (uncommitted).** Item 1 done. **Item 2 deliberately narrowed** —
see below.

`IntakeRecord` gained `PersonID`, and `resolveOrCreatePersonForCaller` runs at write
time: canonical phone → find-or-create `ContactChannel` → live Persons on it → name
match → create when none → link `PersonChannel`. The identity KEYS are the shared,
fixture-pinned `phone.CanonicalE164` and `personname.CanonicalFullKey`, so this cannot
drift from Django on the thing that matters; the surrounding orchestration is kept small
rather than made a second copy of the Dentally lane's.

Attribution rules, and combined with **#38** so an ambiguous line never auto-links:

| caller | line | result |
|---|---|---|
| named, one matching contact | — | link to them |
| named | nobody on the line | create + link |
| named, no match | others on the line | create + link — a new family member on a shared phone is ordinary |
| named, TWO contacts share that name | — | **0** — never pick between two humans (#38) |
| no usable name | line is new | create + link |
| no usable name | line already names real people | **0** |

**The narrowing of item 2, stated plainly.** The audit says to create an `Unknown` Person
and link the channel regardless. I do that only when the line is NEW. On a line that
already names real people, minting an "Unknown" Person adds a phantom household member
to a real family — and all **8** surviving Person-less rows are exactly that case
(3235, 3274, 3535, 3609, 3610, 3670, 3899, 3976, all practice 16, all
`first_name='Unknown'`). A null is recoverable; a phantom sibling is not.

*Measured on `prod_control`, exact:* 1,535 call-agent Intakes, 1,527 linked, **8**
Person-less, all `Unknown`. Both `ON CONFLICT` targets I rely on are real unique
constraints (`unique_person_channel`, `unique_channel_per_practice`).

*Tests:* 4 more in `internal/callagent/caller_identity_test.go` pinning the attribution
rules, plus `TestTheIntakeInsertSetsAPerson`, which guards the WIRING — the resolver can
be perfect while its result is never put on the record, which is exactly what #41 was.
Removing `intake.PersonID = &personID` fails it with "every new row lands outside the
identity graph again".
3. Data: link the 8 rows by hand (each phone → its channel → Person, or a new Person).

## #42 — ✅ FIXED 2026-09-06 (was MEDIUM-HIGH) — recall sync re-splits Dentally's correct first and last names  [CONFIRMED]

**Where:** `EmailServiceGo/internal/dentally/recall/engine.go:994-996,1097-1105`.

The recall fetcher receives separate Dentally `first_name` and `last_name`, joins them
into one `Name`, then `SplitN(..., " ", 2)` splits that joined name again before the
`recall_patient` upsert. It permanently changes the field boundary even though the
writer began with the correct pair.

This is not a duplicate of #8: #8 is the Django Dormant-tab *reader*. This is the Go
writer corrupting `recall_patient` during every sync.

**Verified:** **1,122 recall_patient rows / 1,122 distinct practice-Dentally keys**
currently have the same joined name as their Patient but a different first/last pair.

Examples in practice 13:

| Dentally id | Patient id / Person id | Correct Patient fields | Recall fields |
|---:|---|---|---|
| 295 | 41445 / 128647 | `Kate (Katherine)` / `Churchouse` | `Kate` / `(Katherine) Churchouse` |
| 432 | 41335 / 128573 | `Lucy Claire` / `Littlewood` | `Lucy` / `Claire Littlewood` |
| 551 | 41260 / 128501 | `Andrew David` / `Fox` | `Andrew` / `David Fox` |
| 793 | 41095 / 128347 | `Robert (Robbie)` / `Howchin` | `Robert` / `(Robbie) Howchin` |

The corruption is masked wherever code immediately rejoins the fields, but every
separate-field comparison or first-name personalization observes the wrong identity.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified by execution:* 1,122 `recall_patient` rows have the same joined name as
their Patient but a different first/last pair; `engine.go:994-996` joins, `:1100-1105`
re-splits.

1. Carry the pair through: add `FirstName`, `LastName` to the recall patient struct at
   `:994`, populate them from Dentally's `first_name`/`last_name`, and write those at
   `:1100` instead of `SplitN(p.Name, " ", 2)`. Keep `Name` for display only.
2. The next full recall sync rewrites all 1,122 rows; verify with the query above → 0.
3. Same shape as #1 item 1 — both Go writers should share one rule: *never re-split a name
   you received split*.

**AS BUILT — 2026-09-06 (uncommitted).** Item 1 done; item 2 is a consequence of the
next sync, noted below.

`recallPatientData` carries `FirstName`/`LastName` populated from Dentally's own fields,
and the upsert writes those. `Name` is explicitly display-only now.

**The first-space split survives as a guarded fallback, deliberately.** The stored/owed
path reads a row back that has only `name` and no pair, so something must split there —
but only when BOTH parts are genuinely absent. A test pins that the `SplitN` sits inside
that guard and cannot run ahead of it, because an unguarded fallback would overwrite the
correct pair on the very next sync.

*Measured on `prod_control`:* **1,131** recall_patient rows carry the same joined name as
their Patient row but a different split (the audit said 1,122; mine is a slightly broader
query, same defect). The four practice-13 examples reproduce exactly.

*Tests:* `internal/dentally/recall/name_split_test.go`, 3 tests, **2 red** when the
re-split is restored. One asserts the four real production names round-trip through the
whitespace collapsing unchanged — and, importantly, that each fixture actually
*reproduces* the defect when joined and re-split, so a single-word first name cannot
make the test vacuous.

*Item 2:* the next full recall sync rewrites all 1,131 rows; re-run the query above and
expect 0. No separate backfill command is needed — the sync is the backfill.

*Item 3 — the shared rule now holds in both Go writers:* the Dentally migration lane
stopped joining-and-re-splitting under #1, and this is the recall lane doing the same.

## #43 — ✅ FIXED 2026-09-06 (was HIGH) — consent management crosses practices on read and write  [CONFIRMED by execution]

**Where:** `Notes/views/consent.py:116-149`,
`Notes/serializers/consent.py:8-73`, routes at `Notes/urls.py:414-450`.

`ConsentStatusViewSet` and `ConsentAlertViewSet` scope through
`patient__notes__user=user`, not the current practice. A user who has authored a note
in a different practice can still list/read/update/delete its consent status and
retrieve/dismiss its active alerts after changing practices.

The status serializer exposes writable `patient` and `note` FKs from global querysets,
with no same-practice or same-patient validation. `ConsentAlertSerializer` declares
the same unscoped fields plus `consent_status`, but the current URL map does not expose
alert create/update, so this report does not count that declaration as a writable
exploit. The alert list/actions are still affected by the cross-practice queryset
above. This is a new consent-specific hole, not a repeat of #22's ordinary Note and
NotesLetter serializers.

**Existing cross-practice exposure at the verified snapshot:**

- ConsentStatus 17 and ConsentAlert 10 concern Patient 29495 Aayan Beacher, practice
  16, but are reachable through User 8 whose current practice is 9.
- ConsentStatus 24 and ConsentAlert 13 concern Patient 20965 Adam Monaghan, practice
  13, but are reachable through User 66 whose current practice is 16.

That is **4 live clinical-consent records**: two statuses and two active alerts.

**Writable proof:** in a read-only transaction, the real `ConsentStatusSerializer`
accepted Patient 9593 Manifest Chakalov (practice 13) together with Note 1138, which
belongs to Patient 29495 Aayan Beacher in practice 16: `valid=True, errors={}`. No
`.save()` was called. The exposed FK populations are **60,326 Patients and 978 Notes**;
there are zero persisted Patient/Note mismatches today.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

**AS BUILT — 2026-09-06 (uncommitted).** Fixed on both the read and the write side.

Both viewsets now scope by `patient__practice=<caller's current practice>` instead of
`patient__notes__user=user`. A user with no current practice gets an empty queryset
rather than everything they ever touched.

`ConsentStatusSerializer.validate` checks three things, and the third is the one that
matters clinically: `patient` is in this practice, `note` is in this practice, **and the
note actually belongs to that patient**. A consent record asserts that THIS patient
consented in THIS note; the audit's executed proof was a local patient paired with
another patient's note, which the practice checks alone would not have caught.

**A fourth bug found while writing the test, not in the audit:** the old queryset joined
through `patient__notes` with no `.distinct()`, so a patient with three notes returned
the same ConsentStatus three times — `[15, 15, 15]`. Scoping by practice removes the
join and the duplication with it; there is a test pinning it.

*Tests:* `Notes/test_consent_practice_scoping.py`, 7 tests, **5 red** — the left-behind
practice's status being listed, its alert being dismissible, the writable FK holes, and
the duplicate rows. Two guards confirm own-practice reads and writes still work.

*Scope note:* `ConsentAlertSerializer` declares the same unscoped FKs, but the URL map
exposes no alert create/update, so it is not a writable hole today. Its list and dismiss
actions were affected by the queryset and are fixed by that change.

*Re-verified by execution:* `ConsentStatusViewSet`/`ConsentAlertViewSet`
(`consent.py:116-149`) scope by `patient__notes__user=user`; status 17 (patient 29495,
practice 16) is reachable by user 8 (current practice 9); status 24 (patient 20965,
practice 13) by user 66 (practice 16); alerts 10 and 13 likewise. Serializers expose
`patient` and `note` from global querysets with no cross-check.

1. Queryset: `filter(patient__practice=self.get_user_practice())` — `PracticeAccessMixin`
   is already imported in this file (`consent.py:~5`), add it to both viewsets and drop
   `patient__notes__user`. (Author-of-a-note is not an access rule anywhere else in the app.)
2. Serializer `validate(self, data)`: `note.patient_id == patient.id` and
   `patient.practice_id == current practice`; same on `ConsentAlertSerializer` even though
   no write route exposes it today.
3. Test: the executed proof (Patient 9593 + Note 1138) → 400; user 8 listing statuses in
   practice 9 → status 17 absent.

## #44 — ✅ FIXED 2026-09-06 with #38 (was MEDIUM-HIGH) — active Telnyx phone lookup computes a different key from the identity writer  [CONFIRMED]

**Where:** `EmailServiceGo/internal/callagent/telnyx/handlers.go:1124-1158`, called
when building assistant variables at `:349-375` and in the alternate bridge at
`:747-761`.

The active Telnyx resolver strips a few characters and hand-concatenates
`country_code + phone_number`. It does not call the canonical phone helper, consult
`ContactChannel.canonical_value`, or inspect secondary phone channels. A stored `+44`
plus `07469...` becomes `4407469...`, while the writer's key is `447469...`; a call to
a Patient's correctly linked secondary number is also missed.

**Verified:** **82 stored calls** use canonical phone channels linked to **24 Patient
rows** that the current Patient/active-Intake/active-Nurture resolver cannot find.
**39 of those CallRecords already have a blank `patient_name`.**

Concrete example: CallRecord 74, practice 16, caller `+441245224464`. ContactChannel
102949 canonically links that number through Person 120578 to Patient 32247 Robin
Whittaker. Robin's Patient row has primary `+44 / 7727178263` and secondary
`1245224464`. The resolver checks only the primary concatenation, stores a blank name,
and asks the caller to identify himself. The transcript confirms Robin Whittaker and
confirms `+441245224464` as his number.

This is separate from #38: #38 chooses the wrong human when several match; #44 fails
to recognize a human whom the canonical identity graph identifies unambiguously.

**Fix (second verification pass, 2026-09-06 — evidence at the cited lines):**

*Re-verified:* `telnyx/handlers.go:1128-1158` — `CONCAT(REPLACE(country_code,'+',''),
phone_number)` compared to the stripped caller id; no canonical helper, no channel lookup,
no secondary number. The correct implementation already exists **in the same service**:
`canonicalPhonePersonIDs` (`callagent/storage.go:835`) is what `callagent/handlers.go:342`
uses first.

**AS BUILT — 2026-09-06 (uncommitted).** Fixed together with **#38** — the two findings
met in one function. `getPatientNameByPhone` now calls the shared
`callagent.ResolveCallerCandidates`, so it gains the canonical key (including secondary
numbers, which it never checked) AND the refusal to pick from a shared line. See #38's
AS BUILT block.

1. Replace the body of `getPatientNameByPhone` with the `canonicalPhonePersonIDs` →
   Person → Patient/Intake/Nurture lookup (i.e. the same code as `callagent/handlers.go:339-368`),
   with the #38 ambiguity rule. Delete the three CONCAT queries; keep them only as a
   fallback if a measured population still needs them (the doc's 82 calls / 24 patients
   suggests the canonical path finds more, not fewer).
2. Test: caller `+441245224464` → Robin Whittaker (Patient 32247, via secondary number
   through ContactChannel 102949).

---

# #45–#47 — found by driving the real UI against a restored production copy

The original 44 came from reading code and querying data. These three came from
running the app: booting Django against `prod_rehearsal2` with outbound messaging
intercepted, and using the product the way a receptionist would. Full evidence in
`docs/superpowers/plans/2026-09-06-identity-e2e-results.md`.

## #45 — ✅ FIXED 2026-09-06 (was HIGH) — one patient can hold several AI summaries under name variants

The constraint replaced by migration 0168 was UNIQUE on the raw
`(practice, patient_name)` string — case- and whitespace-sensitive. So one human
legitimately accumulated several clinical summaries: `Adam Ribbits`/`ADAM RIBBITS`,
`Alan Graham`/`Alan  Graham` (doubled space), `Dulcie Sharp`/`Dulcie sharp`,
`George Daley`/`GEORGE DALEY`, `madeleine Canty`/`Madeleine Canty`,
`Sarah Hayter Ltd`/`Sarah Hayter LTD`. **12 pairs across 6 practices.** The day list
renders whichever the query returns first, and the two can hold different content.

This is #5's defect on the opposite axis: #5 was *several patients sharing one
summary*, this is *one patient holding several*.

**AS BUILT.** Migration 0168 now collapses duplicates per
`(practice, dentally_patient_id)`, keeping the most recently updated row, before
adding the new constraint. It also surfaced two defects in the migration itself:
it was **undeployable** (the constraint could not be built over the duplicates —
it would have aborted the production deploy) and it held ACCESS EXCLUSIVE on the
summary table for 15+ minutes via an N+1 loop. Rewritten set-based: data step now
completes in under 90 seconds. Verified on `prod_rehearsal2`: 14,393 rows, all
stamped, one summary per patient exactly.

## #46 — ✅ FIXED 2026-09-07 (was MEDIUM-HIGH) — an outbound SMS on a shared family line was recorded against nobody

Sending from a named patient's panel stores a session with `patient_id` NULL and
`participant_name` set to the raw phone number, and creates no Activity row at all.
Evidence: Tate Wheeler (7 Persons on `+447508199526`) produced
`Entity smsmessage #2426 has no person. Skipping ActivityLog creation.`
The control — Mackalla Williams on an unshared line — was attributed correctly.

The identity was known and discarded: the frontend requested
`/messaging/contacts/119734/thread/?patient_id=32069`. The send path derives identity
solely from the channel via `sole_person_for_channel`, which correctly refuses to name
anyone on an ambiguous channel — so the guard from #38/#41 is working, but the user's
explicit choice of recipient is never consulted.

Consequences on a family line: no activity-log entry, the inbox thread is titled with
a phone number, "Last Contacted" never updates, and a reply lands in an anonymous
thread with 7 candidates.

**Fix (not yet applied):** the send endpoint should accept the `patient_id`/`person_id`
the caller already supplies and stamp the session and SMS with it, falling back to
`sole_person_for_channel` only when the caller gives nothing.

## #47 — ✅ FIXED 2026-09-06 (was HIGH) — a CARRY implemented as a DISCOVER mints duplicate Persons

Rename an intake (Actions → Edit), then move it (Actions → Move → Nurture), and the
move creates a **second Person** for a human who already had one. Reproduced on
`prod_rehearsal2`: intake person 74676, nurture created with person 158648, phone
channel `+447576763975` went from 1 Person to 2. A control move *without* a rename
reused the person correctly, isolating the cause to name-based re-resolution.

Harm proven on the same patient 13 minutes apart: SMS 2427 (before) linked to activity
session 110; SMS 2428 (after) was recorded against nobody. **Two ordinary UI actions
turned a cleanly-tracked patient into an untrackable one** — and manufactured exactly
the ambiguity #38 and #46 are about.

The audit's fixes were all working: `sole_person_for_channel` refused to guess exactly
as designed. Nothing stopped the app *creating* the ambiguity.

**AS BUILT.** Not a weakening of `Person.resolve` — its "different name → new Person in
the shared Household" rule is what stops two family members on one phone being welded
together (#3), and must stay. Instead, carry paths no longer call it:

1. `_assign_person_on_save` honours an explicitly supplied Person on a record with no
   pk, and still links that record's channels. Without this the call-site fixes are
   silently inert, because `_preserve_person_and_link_channels` requires `instance.pk`
   and so never ran on creates — which is why the rename alone was always safe.
2. `ConversionMixin.carried_person()` and `automations.actions._carried_person()`
   supply the known identity at all **8** carry sites: 5 journey conversions and 3
   workflow handlers. The workflow handlers took `source_record_type` /
   `source_record_id` and never read them, while the Go engine sends both on every
   call.
3. Two AST guards fail the build if a new conversion or handler forgets; each includes
   a self-check proving the detector can fail.

Re-verified through the UI after the fix: the same rename-then-move on Tina Brown left
intake, nurture and patient all on person 158284, with one Person still on the channel.

---

# SIGN-OFF — fixes verified 2026-09-07

Eight findings closed today. **Every one carries a RED RUN**: the test was run against
the unfixed code and shown to FAIL before the fix was trusted. A test that has never
failed proves nothing, and two of these caught real mistakes in my own fixes.

| # | Sev | What | Red-run evidence | Suite |
|---|-----|------|------------------|-------|
| #78 | CRITICAL | Websockets authenticated the user, never authorised the practice | 3 fail: "foreign practice joined the feed" | 8/8 |
| #52 | CRITICAL | Inbound email trusted an unauthenticated `practice_id` | 3 return **201** — forged email accepted | 6/6 |
| #69 | CRITICAL | Day List / Recall leaked patient PII across a practice switch | 5 of 6 fail | 6/6 |
| #70 | HIGH | Clinical notes read AND written by patient NAME | id branch proven exact | 5/5 |
| #71 | HIGH | Universal search opened a different human (the Alex Cooper report) | 5 of 6 routing tests fail | 12/12 |
| #64 | HIGH | Inbox sends carried no recipient id | receiving contract already covered | 16/16 |
| #66 | HIGH | Inbox tasks filed under a name string | (shares #64's change) | — |
| #67 | MEDIUM | Practice switch left the previous WhatsApp config live | (shares #69's registry) | 6/6 |

**Full verification run, 2026-09-07:** backend 46/46 across the five new suites;
`messaging` 179/179; frontend 42/42. Pre-commit clean on every touched file;
`tsc -p tsconfig.app.json` clean on every touched frontend file.

**Pre-existing failures, confirmed NOT caused by this work.** `Notes` + `TreatmentPlan`
report 11 failures. Proven pre-existing by removing the new test file and re-running:
the identical 5 Notes failures remain (letter finalisation — the known
`Notes/tasks.py:30` `.service`/`.services` import bug — plus three seed-data tests). The
6 in TreatmentPlan are the previously catalogued set: the missing `journey_automation`
module, two `patient_practice_dentally_uniq` fixture errors, the stale
archive-deletes-the-row assertion, a `display_name` contact-signals failure, and the
COMPAT-03 checksum tripped by unrelated uncommitted next-appointment work.

**Corrections made while fixing — recorded because the original claims are what a reader
would otherwise act on:**
- **#78** first listed five vulnerable consumers. It was THREE. `dentallyIntegration`
  does validate; I had grepped for `url_route` without reading on.
- **#70**'s reported symptom was wrong. Executing it showed the workspace's
  `"John Smith"` query matched NOTHING (the columns are separate), so notes went
  MISSING rather than another patient's appearing. Cross-patient disclosure needs a
  single-token query. Fix unchanged, reasoning stronger.
- **#72** does not contradict #68: `useTreatmentPlanCreation` carries the id correctly
  and `AddOpenPlanDialog` does not. Same endpoint, two callers, one broken.

**Two traps worth carrying forward:**
- A **green test was protecting #71**: `phone.test.ts` asserted
  `getPatientRouteId({ id: 42 }) === "42"` — the defect, written down and passing.
- My **first #52 red run passed**, which meant the tests were worthless: I had guessed
  the URL and every request 401'd before reaching the view. The second attempt then
  showed the fix would REJECT LEGITIMATE MAIL, because practices receive at
  `<prefix>@<own domain>` and I had only implemented local-part resolution.

**#52 has a deployment precondition.** It has NOT been checked against production
recipients. If any live inbound address resolves by neither slug-prefix nor a
`PracticePreferredDomain` row, this change returns 400 instead of misrouting — patient
email would be dropped. Query prod for distinct recent `EmailMessage` recipients and
confirm each resolves BEFORE deploying this one.

**CORRECTED 2026-09-07 (this claim was WRONG).** An earlier version of this entry said
the production container predated the `attribute_to` call sites and that #64 was
therefore inert. It is not. The running container HAS them, at
`messaging/views/message_views.py:357` and `:2026`.

The error: the check was `grep -rn attribute_to /app/TreatmentPath/messaging/ | head`.
`models.py` contributes 2 matches and the test file 8 — exactly 10 — so `head` truncated
the output immediately before the two `message_views.py` lines, and "called only from
tests" was inferred from a cut-off result. A truncating pipe is not evidence of absence.

The backend half is deployed and live. What is NOT in production is the FRONTEND half
written today, which simply has not shipped yet — an ordinary deploy, not a stale image.


---

# #48–#63 — excluded-app and sibling-path sweep (read-only production proof)

These findings were checked against **all of #1–#47 before inclusion**. They are new
call sites or new failure modes, not restatements of an earlier root pattern. Every
database check below ran against `prod_control` in a read-only transaction. No save was
performed during serializer proofs; routed write-path proofs stopped at a mocked
external/write boundary.

## #48 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — custom letter routes disclosed another practice's clinical letter

**Where:** `Notes/views/letters.py:298-327`, `:1001-1034`, `:1302-1336`.

The ordinary queryset was corrected under #22, but its siblings were not. The custom
`get_object()` used by `render_content` filters by the **author's** practice membership,
not the patient's practice, and the custom `retrieve()` bypasses `get_queryset()` with
the same author-based rule. Staff in practice 16 can therefore read/render a letter for
a practice-13 patient when its author is visible in practice 16.

**Concrete production proof:** NotesLetter **29**, authored by User **71**, contains
clinical content for Patient **9593, Manifest Chakalov** (practice **13**). The author's
current practice is **16** and the letter's legacy `practice_id` is NULL. Executing the
real routed `retrieve` as unrelated User **66** in practice 16 returned **200**;
executing `render_content` returned **200** as well.

**Affected rows:** **1 live letter / 1 patient**. **Pattern B.**

## #49 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — custom note-by-patient routes disclosed a former practice's clinical note

**Where:** `Notes/views/note.py:369-412`, `:414-476`.

`unique_patient_names` and the default branch of `notes_by_patient` filter by note
author only. `notes_by_patient` becomes practice-scoped only when a caller opts into
`practice_scope=true`; its normal/default request is not. This bypasses the corrected
main Note queryset from #22.

**Concrete production proof:** Note **662**, authored by User **66** (current practice
**16**), belongs to Patient **20965, Adam Monaghan** (practice **13**). The actual
`notes_by_patient?patient_id=20965` route returned **200** with Note 662, and
`unique_patient_names` returned `{id: 20965, name: "Adam Monaghan"}`.

**Affected rows:** **1 non-draft clinical note / 1 patient** today (17 author/current-
practice mismatches exist, but 16 are drafts and were not counted as readable here).
**Pattern B.**

## #50 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — Intake/Nurture API fields selected arbitrary siblings from a fused Person

**Where:** `TreatmentPlan/serializers/intake.py:235-305`,
`TreatmentPlan/serializers/nurture.py:206-257`; consumed by
`perfect-pixel-playground-project/src/components/patients/patient-panel/PatientPanelOverviewTab.tsx:176-191,273-306`
and `src/components/compact/IntakeTable.tsx:1361-1374,1541`.

An Intake or Nurture is a specific human, but `patient_id`, the related journey ID and
`dentally_site_id` are chosen as the first/recent record on its Person. On a Person that
still contains several family members, the returned IDs can belong to another human;
`patient_id` and `dentally_site_id` can even name two different Patient rows in the same
response. The frontend uses them for patient panels, tasks and Dentally deep links.

**Concrete production proof:** active Intake **3097, Sarah Franklin** (practice 16)
serializes `patient_id=30839`, which is **Simon Franklin**. Intake **709, Chris Davies**
(practice 19, Person 127132) serializes Patient **39775, Chris Davies** as `patient_id`
but the Dentally UUID of Patient **39888, Christopher Davis** as `dentally_site_id`.

**Affected rows from the actual serializer methods:** Intake `patient_id` is a
different canonical name for **53 rows, 16 active**; Intake Dentally UUID is wrong-name
for **53 rows, 16 active**. The two IDs disagree on **472 Intakes, 11 active**. Nurture
has **7** wrong-name Patient/UUID selections (none active) and **64** internal-ID/UUID
disagreements (**8 active**). **Pattern D.**

## #51 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — live edit surfaced re-split stored names at the first space

**Where:**
`perfect-pixel-playground-project/src/components/patients/patient-panel/PatientEditableFields.tsx:65-95`,
`PatientPanelOverviewTab.tsx:322-327`, `usePatientPanelController.ts:231-259,294-345`,
`src/components/compact/IntakeTable.tsx:1681-1707`,
`CustomJourneyTable.tsx:1606-1644`, and `src/components/WorkflowPanel.tsx:339-342,645-664`.

These components receive a joined display name, split the first token into
`first_name`, and PATCH/POST the two columns. Merely opening a name editor for
`Ken (Kenneth)|Judge` prefills `Ken|(Kenneth) Judge`; saving writes that false boundary.
This is a different ingress from #1 (Dentally migration), #20 (booking payment), #21
(marketing form handoff), and #47 (identity carry after a rename).

**Concrete execution proof:** current `PatientSerializer`, partial-updating Patient
**9622, Ken (Kenneth)|Judge** (practice 16, Person 74615), accepts
`first_name=Ken, last_name=(Kenneth) Judge` unchanged (`valid=True`). The exact Nurture
payload built by `WorkflowPanel` validates the same wrong split. Other real examples
are Patient **20531 James Edward|Barr**, Patient **21740 Sarah Jane|Smith**, Intake
**537 Kathy (Kathleen)|Coleman**, and active Nurture **521 Scott Antony|Newland**.

**Affected/at-risk rows:** **1,686 active Patients + 1 active Nurture = 1,687 live
rows**; 46 historical Intakes are also prefilled incorrectly by edit screens. A direct
same-Person/same-joined-name boundary-mismatch query found **0 rows that can currently
be attributed to these UI paths**, so this is proven reachable with a zero-current-
damage qualifier. **Pattern A.**

## #52 — ✅ FIXED 2026-09-07 (was CRITICAL) — inbound email trusts an unauthenticated practice ID

> **Status corrected 2026-09-07.** This heading said OPEN/REVERTED, which described the
> first attempt only. The recipient-authoritative version WAS reverted; the hole was then
> closed a different way — caller authentication with `X-Workflow-Secret` — which is
> shipped and green. See the second AS BUILT block below.

**Where:** `messaging/urls.py:391-395`,
`messaging/views/message_views.py:2208-2225,2354-2373,2677-2695`, and
`EmailServiceGo/internal/email/worker/inbound/queue_processor.go:191-208`.

`receive_email_v2` is `AllowAny` with authentication disabled. When `practice_id` is
present, it trusts that integer instead of proving it from the recipient domain. The Go
worker adds only `Content-Type`, no shared secret or signature. The selected practice
then owns the EmailMessage and can also own a generated Intake, invoice or Task. This
is a practice-boundary identity-routing defect, not merely webhook hardening.

**Concrete execution proof:** an anonymous request with a neutral recipient and
`practice_id=16` selected **Danbury Dental Care (practice 16)** and reached
`_create_email_message`; the audit replaced that method with a stop sentinel before
any write. The request user was `AnonymousUser` and the actual routed action reached
the stop point.

**Affected scope/count:** all **14 practices** are addressable by ID. The database has
**2,484 incoming EmailMessages across 4 practices**. No row can presently be proven
forged, so the attributable affected-row count is **0**. **Pattern B / reader-writer
trust-boundary mismatch.**

## #53 — ✅ FIXED 2026-09-07 (was HIGH, 0 attributable sends) — email-template preview is scoped; send is not

**Where:** `messaging/views/template_views.py:573-637` (scoped preview) versus
`:721-835` (unscoped send), and `messaging/serializers.py:1017-1038`.

Preview validates the plan/Patient against the current practice. Its sibling `send`
loads `TreatmentPlan.objects.get(id=...)` or `Patient.objects.get(id=...)` globally;
the serializer checks only that exactly one ID was supplied. A practice-16 template
can therefore render and address another practice's patient, then records that foreign
patient/plan in practice-16 history.

**Concrete execution proof:** Template **13, “Enquiry Followup #1”** (practice 16)
accepted Patient **139014, Craig Almond** (practice 27) and the real view reached the
mailer with Craig's address. The same serializer accepted TreatmentPlan **1027**
(practice 27) for Patient **116528, Mihail Neykob**. The mailer was mocked to stop;
nothing was sent or saved.

**Affected/at-risk rows:** relative to template 13, **34,862 emailable Patients** and
**361 TreatmentPlans** are outside its practice. Persisted attributable bad sends:
**0**. **Pattern B.**

## #54 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — patient CSV import treated a shared family contact as proof of one arbitrary patient

**Where:** `TreatmentPlan/views/patient_views.py:1387-1416,1489-1523`; frontend entry
point `perfect-pixel-playground-project/src/hooks/useCSVImport.ts:395-402`.

The importer builds one dictionary entry per email/phone. It does not include names in
the prefetched data, so a later row overwrites an earlier family member and a CSV row
with that shared contact is declared an existing Patient regardless of its name. The
new person is silently skipped or associated with whichever relative survived in the
dictionary.

**Concrete production proof:** practice 16 has Patients **9594 Jonathan Beacher** and
**29495 Aayan Beacher** on `jonathanwbeacher@gmail.com`. Rebuilding the exact current
dictionary selects Patient **29495 Aayan**, so a CSV row for Jonathan is treated as
Aayan before the importer ever examines the submitted name.

**Affected rows:** **3,882** shared-email/different-name keys covering **9,823 active
Patients**; **6,701** shared-phone/different-name keys covering **16,770 active
Patients**; **18,725 distinct active Patients** are in at least one of those sets.
**Pattern D.**

## #55 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — phone-only plan association ORed away its practice/match predicate

**Where:** `TreatmentPlan/views/patient_views.py:1838-1906`.

The query starts as `Q(practice=current)`. Email is ANDed onto it, but phone is ORed.
For a phone-only Patient this becomes “every plan in this practice OR a matching phone,”
so the endpoint associates every practice plan with that Patient.

**Concrete production proof:** Patient **20480, Hugh Gilmore** (practice 13; phone,
no email) produces **all 10 practice-13 plans** and none belongs to Hugh. Examples in
the returned queryset include Plan **62** for Manifest Chakalov, Plan **269** for Carter
Morton and Plan **898** for Test5 User.

**Affected/at-risk rows:** **19,635 active phone-only Patients**. Current non-empty
`treatment_plan_ids` attributable to this path: **0**. **Pattern B / query composition.**

## #56 — ✅ FIXED 2026-09-07 (was MEDIUM-HIGH) — patient-account charges accepted a practitioner from another practice

**Where:** `patient_accounts/serializers.py:215-226` and
`patient_accounts/views.py:219-285,657-690`.

The charge endpoint correctly scopes the Patient, but `practitioner_id` is a bare
integer and `_resolve_practitioner` performs a global User lookup. The created ledger
entry and charge can therefore attribute one practice's patient treatment to a staff
member who belongs only to another practice.

**Concrete execution proof:** the real `CreateChargeSerializer` accepts Patient
**9594, Jonathan Beacher** (practice 16) with User **113, Andrew Connolly**, whose only
practice is **20** (`valid=True`). No save was called.

**Affected rows:** **23,423 PatientCharges**, **4,985** with a practitioner. One live
row already has this invalid shape: Charge
`e9d58c6e-0f20-4782-84b7-3d93959a6d2b`, Patient **53867 Susan Albest** (practice 21),
is attributed to User **144 Will Westphal**, who has no membership in practice 21.
That row is Dentally-backed, so it proves the data shape exists but is **not attributed
to this endpoint**. **Pattern B.**

## #57 — ✅ FIXED 2026-09-07 (was MEDIUM-HIGH, latent) — consent signing accepts another patient's plan/appointment as context

**Where:** `Documents/serializers.py:340-355` and
`Documents/views/signing_views.py:308-325,392-437`.

The send action scopes `patient_id` and template IDs, but `appointment_id` and
`treatment_plan_id` are raw IDs with no practice or patient consistency check. They
are written directly onto the signing request, so the patient's consent summary can
claim it belongs to another human's treatment or appointment.

**Concrete execution proof:** `SendConsentSerializer` accepted Patient **9594,
Jonathan Beacher** (practice 16) with TreatmentPlan **1109** (practice 24) for Patient
**136312, Vishal Patel** (`valid=True`). No save was called.

**Affected/at-risk rows:** **361 plans** lie outside practice 16. There are **2 current
SigningRequests**, both with null plan/appointment context, so persisted affected rows
today: **0**. **Pattern B.**

## #58 — ✅ FIXED 2026-09-07 (was MEDIUM) — legacy conversation reader lowercases the key, then compares case-sensitively

**Where:** `messaging/urls.py:238-247` and
`messaging/views/message_views.py:1399-1450,3539-3555`.

The route lowercases a requested email, then uses exact equality against stored
message and Patient email fields. Writers retain mixed case. A valid Patient's email
thread therefore returns 200 with no messages and no Patient ID.

**Concrete execution proof:** User **113** in practice 20 requested the conversation
for Patient **55623, Sandra Sumner**, stored as `FRSumner8@gmail.com`. Three matching
EmailMessages exist, but the real route returned **0 messages** and `patient_id=None`.

**Affected rows:** **26 patient email keys / 34 EmailMessages** contain mixed case and
are hidden by this reader. This endpoint appears legacy/deprecated, which limits
severity but not the reproduced defect. **Pattern C.**

## #59 — ✅ FIXED 2026-09-07 (was MEDIUM) — invoice update exposed a global practitioner FK

**Where:** `Invoices/serializers.py:792-807`; the active view selects this serializer at
`Invoices/views/invoice_views.py:365-378`.

`allocated_practitioner` is a writable `PrimaryKeyRelatedField(User.objects.all())`.
The invoice queryset is practice-scoped, but the chosen practitioner is not.

**Concrete execution proof:** Invoice
`78cd0dc9-8266-444a-846d-1083d59dff3d` (practice 16, Directmeds) validates with User
**113 Andrew Connolly**, whose only practice is 20. No save was called.

**Affected rows:** **951 invoices** exist; **0 currently have an allocated
practitioner**, so persisted affected rows are **0**. **Pattern B.**

## #60 — ✅ FIXED 2026-09-07 (was MEDIUM) — Patient's preferred clinician FKs were globally writable

**Where:** `TreatmentPlan/serializers/patient.py:45-58`; exposed by
`perfect-pixel-playground-project/src/components/patients/workspace/PatientDetailsTab.tsx:730-731`.

Both `preferred_dentist_id` and `preferred_hygienist_id` use `User.objects.all()` and
have no practice-membership validation. A practice-scoped Patient can be saved with a
clinician belonging exclusively to another practice.

**Concrete execution proof:** partial `PatientSerializer` validation for Patient
**9594 Jonathan Beacher** (practice 16) accepted User **113 Andrew Connolly** (only
practice 20) for both fields (`valid=True`). No save was called.

**Affected rows:** all active Patients are writable through this surface; currently
**0 Patients** have either preferred-clinician FK populated, so persisted affected rows
are **0**. **Pattern B.**

## #61 — ✅ FIXED 2026-09-07 (was MEDIUM, latent) — marketing webhook ignores its exact recipient and picks the first Person on the email

**Where:** `marketingBroadcast/views/webhook_views.py:76-93,143-175`.

Campaign delivery already has an exact `BroadcastRecipient.person`, but bounce/
complaint/opt-out handling discards that identity, resolves by email channel and calls
`.first()`. A shared family email therefore changes consent for an arbitrary relative.

**Concrete execution proof:** in practice 13, `4.peacocks@gmail.com` belongs to Person
**110979 Jude Peacock** and Person **110980 Rebecca Peacock**. The current helper
returns Jude solely because he is first; a webhook about Rebecca would update Jude.

**Affected/at-risk rows:** **7,893 shared email channels**. There are currently **0
BroadcastCampaigns and 0 BroadcastRecipients**, so affected persisted campaign rows
today are **0**. **Pattern D.**

## #62 — ✅ FIXED 2026-09-07 (was PROVEN / MEDIUM) — consent-to-Dentally sync chose one arbitrary Patient on a fused Person

**Where:** `marketingBroadcast/consent_ledger.py:121-128,153-155` and
`marketingBroadcast/tasks.py:7-38`.

The profile task fixed under #37 detects ambiguous multi-Patient Persons, but this
sibling consent lane still calls `person.patients...first()`. Enabling Dentally
marketing sync would write one Person's consent to whichever family member's Dentally
ID happens to come first.

**Concrete production proof:** **2,357 Persons** contain more than one distinct
Dentally patient ID, spanning **5,802 IDs**. Those are real multi-human candidates on
which the function has no rule for selecting the consenting patient.

**Affected rows today:** **0** — no practice currently enables
`enable_dentally_marketing_sync`, and no MarketingConsent row is attached to those
fused Persons. The control flow and exposed population are proven; live writes are
dormant. **Pattern D.**

## #63 — ✅ FIXED 2026-09-07 (was MEDIUM, feature disabled) — booking OTP prefill reveals whichever family member is first

**Where:** `onlineBooking/views.py:575-626,629-670`; consumed by
`perfect-pixel-playground-project/src/pages/booking/hooks/usePublicBooking.ts:300-316`.

After proving control of an email address, `_build_verified_response` calls `.first()`
for that email and returns that Patient's first name, last name and phone. It also lists
confirmed holds by email only. On a shared family address, one relative can receive
another relative's prefill and the family's bookings. This is the read/prefill sibling
of #18, which covered appointment filing.

**Concrete execution proof:** practice 21 has nine different Taylor Patients on
`jkatytaylor@gmail.com`: Patient **43706 Twyla-Storie Taylor**, **45188 River-Raven
Taylor**, **45190 Blake Taylor**, **45191 Jackson Taylor**, **45192 Reggie-Lee Taylor**,
**45193 Brogan Taylor**, **45194 Lilly-Mai Taylor**, **45195 Brooke Taylor**, and
**45248 Millie-Jai Taylor**. The actual helper returns Twyla-Storie's identity.

**Affected/at-risk rows:** across the seven practices with booking profiles, **3,450
shared-email/different-name groups cover 8,515 active Patients**. All **7 profiles are
currently disabled**, and the example currently has 0 upcoming holds, so live exposed
rows today are **0**. **Pattern D.**

---

# #79–#81 — found by driving the app as a user, after the 44 were closed

**RENUMBERED 2026-09-07.** This block was originally numbered #48–#50, which
collided with the #48–#63 sweep above — two different findings shared each of
three numbers, so "is #49 fixed?" had two correct answers. Renumbered to
#79–#81; the #48–#63 sweep keeps its original numbers. Old -> new: #48 -> #79,
#49 -> #80, #50 -> #81.

## #79 — ✅ TOOLING BUILT 2026-09-06 (HIGH) — a conversation titled with the WRONG family member  [was #48]

**User-reported from the UI:** *"in task it says Monique Shimwell, in inbox it says
Grant Shimwell, and it has no activity log or anything."* Reproduced exactly.

`messaging_messagesession` 301 holds `participant_identifier = 7392136791` — Monique's
own number, per her patient record; Grant's row carries a different one — with
`participant_name = "Grant Shimwell"` and `patient_id` NULL, over messages that open
*"Good Morning Monique"*. Channel 104648 links both of them, `sole_person_for_channel`
rightly declines to name either, and the contacts serializer falls back to the stored
name, which is wrong. `patient_id` NULL is why there is no activity history.

**71 sessions, 297 messages, 4 practices** carry a label naming one member of a shared
channel while a different member owns the number — every one with no patient link.

**Live code no longer does this**: a session opened during testing on the 7-person
Wheeler line got `participant_name = "+447508199526"`, a number rather than a guess.
This is historical damage from the old arbitrary-`.first()` behaviour.

**AS BUILT.** `messaging/management/commands/repair_mislabelled_sessions.py`, dry-run
by default, runbook step 6. Its first dry run is the argument against auto-applying:
of 49 raw candidates, 19 were the SAME human written differently (Lynn/Lynne,
Dawn/Dawn Campbell, Mrs Tracy/Tracey) and one pair had an identical name — a duplicate
Person, not a mislabel. It now classifies DIFFERENT / SAME / DUPLICATE_PERSON and
repairs only the 29 genuine ones. Left for a human to review, as #38 was.

## #80 — ✅ FIXED 2026-09-06 (HIGH) — outbound email bypassed the sandbox, and was unattributed  [was #49]

Two problems, one discovery. Sending email from a patient panel left the intercept log
empty: **outbound email does not use Django's `EMAIL_BACKEND` at all.**
`messaging/email_service.py` POSTs to the Go email-service, which holds the provider
credentials. Any rehearsal against restored production data that only swaps
`EMAIL_BACKEND` will deliver real email to real patients. It failed here only because
that service happened to be down.

`send_plain_email` also had #46's gap: it read `intake_id`/`nurture_id`/
`treatment_plan_id` for touch points and never attributed the session.

**AS BUILT.** The dev sandbox blocks the email-service paths (matched on path, not
host, so a staging URL cannot slip past). Attribution added to the email path, plus
`patient_id` on the frontend email sender. Verified on the Shimwell household address:
session 1993 went from `busterandmonique@gmail.com` / NULL to **Monique Shimwell /
31714**, with activity row 19602.

## #81 — ✅ FIXED 2026-09-06 (HIGH) — automated sends were filed against nobody  [was #50]

The #46/#80 fixes cover what a human clicks. The unattended paths — recall automation,
appointment confirmations, journey sequences — are the bulk of message volume and none
of them attributed. On a family phone every automated recall and confirmation produced
a conversation with no patient link and no history: the Shimwell symptom arriving
continuously rather than once.

Each already knew the recipient: `RecallRecord` and `DentallyAppointment` carry
`dentally_patient_id` (11,146 of 11,430 patients in practice 16 hold the matching
`meta_data->>'id'`), and `JourneySequenceEnrollment` carries a `person` FK.

**AS BUILT.** `patient_for_dentally_id` and `patient_for_person` beside
`patient_for_journey_record`; attribution added to recall, both confirmation paths and
journey dispatch, each wrapped so labelling can never break a send.

Chasing it exposed a fourth instance of the same shape: `_log_confirmation_activity`
already resolved the patient by Dentally id, used it only as an `if not person: return`
guard, then called `on_record_created`, which re-derived identity from the channel and
got nobody — the answer sitting in a local variable two lines above.

**Still open:** `Tasks/tasks.py:294` and `automations/actions.py:1809` do not attribute.
Lower volume, not exercised in the sweep, left rather than changed blind.

---

# #64–#68 — ROUND TWO: the frontend, audited for the first time

Everything above is backend. Round two extended the audit to
`perfect-pixel-playground-project` (branch April27), and the first three findings
explain symptoms that looked like backend bugs and are not.

Method note: two attempts to delegate this sweep to sub-agents produced nothing (one
returned a plan and no findings, the other was stopped before completing), so these
were read by hand. Coverage is therefore partial and stated honestly at the end.

## #64 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — Inbox sends carried no recipient id, so attribution could never fire

`InboxPage.tsx:939-943` sends an SMS with `phone_number` and nothing identifying WHO
it is for. This is structural, not an omission at one call site: `SendSmsRequest`
(`components/inbox/types.ts:254`) and `SendEmailRequest` (`:244`) have no id field at
all.

The backend attribution added in #46 reads `patient_id` / `intake_id` / `nurture_id` /
`treatment_plan_id` off the request (`messaging/views/message_views.py:355-362`). The
Inbox sends none of them, so `attribute_to` is unreachable from the main Inbox. The
journey panels attribute correctly only because they happen to send those ids for the
touch-point counter.

**This is the cause of the symptom reported on +447789791781** — three humans share
that line (Sharon Harper, Talia Harper-smith, and an unnamed caller), the session has
`patient = None`, and the panel correctly falls back to the bare number because nothing
ever told it who the message was for.

**It also corrects a claim made during the #80 work:** deploying the backend
attribution fix does NOT fix manual Inbox sends. The backend cannot attribute what the
frontend never sends.

The id is in hand and discarded — `selectedContact.patient_id` is read from the same
object as `normalized_phone`, one line earlier.

**Fix:** add an optional recipient id to `SendSmsRequest`/`SendEmailRequest` and pass
`selectedContact.patient_id` at the two send sites. No backend change needed; the
receiving code already exists.

## #65 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — "Add to Nurture" discarded the patient and forced a re-DISCOVER

`WorkflowPanel.tsx:658-660`. The Open and Active branches both pass
`...(numericPatientId ? { patient_id: numericPatientId } : {})`. The Nurture branch,
three lines above them, sends only `first_name, last_name, email, phone_number`.

CARRY implemented as DISCOVER — the single biggest root cause in round one (#47),
still live in the frontend. Staff open a known patient, add them to Nurture, and the
backend re-resolves identity from contact details, which is how duplicate Persons get
minted.

Unlike #64 this is not a frontend-only fix: `patient_id` and `person_id` on
`NurtureSerializer` are `SerializerMethodField` (`serializers/nurture.py:70-73`) —
**read-only**. There is currently no way for the Nurture create API to accept a known
identity, so every "Add to Nurture" from a known patient is a forced re-discovery.

**Fix:** add a writable identity field to the Nurture create path (the signal at
`contact/signals.py` already honours an explicit `person` on create), then pass
`numericPatientId` from the Nurture branch.

## #66 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — Inbox tasks were filed under a name string, not a patient

`InboxPage.tsx:1050-1057`:

    patient_id: null, // Phase 49 will wire patient IDs from thread

It falls through to `payload.unlinked_patient_name` (`:1081`) — a NAME, which is never
an identity key. The task never appears on the patient's record and breaks silently if
the name changes.

This is the `openTaskModal` bug recorded as fixed on 2026-09-03. That fix landed on a
different component; this instance is still live. `ContactListItem` exposes
`patient_id`, `intake_id`, `nurture_id` AND `activity_log_id`
(`types/contactMessaging.ts:22-25`) — all four available, none used.

## #67 — ✅ FIXED 2026-09-07 (was PROVEN / MEDIUM) — practice switch left the previous practice's WhatsApp config live

`hooks/useWhatsAppConfig.ts:8-20` is a bare module-level singleton with no practice
key; `fetchConfig` short-circuits on `_configFetched`. A `refetch` that resets it
exists (`:49-54`) but only Settings calls it, and only for mid-session revocation —
`InboxPage.tsx:102` and `PatientInboxTab.tsx:128` never do.

Switch from a WhatsApp-enabled practice to one without and the WhatsApp tab stays
available on the new practice, backed by the old practice's config.

Fifth instance of the documented practice-switch stale-state class (AuthContext,
useFetchWithAuth, FeatureAccessContext, useTeamChat were the first four).

## #68 — ✅ FIXED 2026-09-07 (was MEDIUM) — the frontend fabricated "Unknown" as a first name

`hooks/useTreatmentPlanCreation.ts:120` and `components/WorkflowPanel.tsx:340` both do
`name.split(" ")[0] || "Unknown"`.

Measured on production 2026-09-07: **249 Persons and 424 Intakes** carry the literal
string "Unknown" as a name. The call agent is one producer; these two are others. The
codebase elsewhere already knows "Unknown" is not a name — `split_collapsed_persons`
treats any name starting with "unknown" as blank. Storing it as a real name is the seed
of the fused-Person class repaired under #51.

Not currently welding strangers together (249 separate Persons, only one household
holds two), so MEDIUM rather than HIGH. A nameless human should carry a blank name.

## Round two — checked and CLEAN (recorded so they are not "fixed" later)

- **Treatment-plan creation carries identity correctly.** It sends `patient_data` AND
  `payload.patient` (`useTreatmentPlanCreation.ts:137-139`), and the backend prefers the
  id (`serializers/treatment_plan.py:985-994`, `if patient_id: … elif patient_data:`).
  No duplicate is minted. Only the no-patient-selected path is a problem, and that is
  #68.
- **`_contactsFetched` in `hooks/useContactInbox.ts` is dead code** — declared at `:10`,
  set at `:88`, never read. It looks exactly like #67 and is harmless.

## Round two — COVERAGE

Examined: the frontend Inbox in full; the WorkflowPanel journey-add paths; the
module-level cache class across the whole frontend; and the treatment-plan creation
chain end to end into the backend serializer.

NOT reached: the Go service; backend background writers (Celery/beat/signals/commands);
unauthenticated ingress; realtime and bulk egress; and most of the Journeys boards,
patient workspace, contacts, global search, merge UI, Day List, Recall and
Confirmations.

That is roughly a third of the intended round-two scope. #64–#66 are enough to act on,
but the remainder is genuinely unaudited rather than audited-and-clean.

---

# #69–#74 — ROUND TWO, second pass (independent audit, cross-checked)

Found by an independent read-only audit (codex `gpt-5.6-sol`) run over the same
frontend, deliberately NOT told what #64–#68 were, so it could not anchor on them.
It was pointed at the five areas the first pass never reached.

**Every finding below was re-verified by hand at `file:line` before being recorded.**
Seven were reported, seven held up. Six are new; the seventh corroborated #65 from
different call sites and is folded into it.

Two of these are more serious than anything in #64–#68.

## #69 — ✅ FIXED 2026-09-07 (was PROVEN / CRITICAL) — Day List and Recall leaked patient PII across a practice switch

`pages/day-list/hooks/useDayListData.ts:10` —

    const dayListCache = new Map<string, {...}>();   // keyed by DATE ONLY

`pages/recalls/useRecallsData.ts:43` and `pages/recalls/useDormantRecalls.ts:15` repeat
the shape, keyed only by query string. None carries a practice id.

The soft practice switch (`components/settings/PracticeSwitcher.tsx:138-145`) clears
React Query and `clearAllCachedData()` — neither touches these module-level Maps. Its
own comment claims it wipes "practice-scoped caches (React Query + useCachedData) so
nothing from the previous practice is served"; these three caches are outside both.

**Scenario:** open today's Day List in practice A, switch to practice B, reopen the
same date within the 3-minute TTL. Practice A's patient names, phones, emails and
clinical summaries render inside practice B. If the refetch then fails, they stay on
screen.

Sixth instance of the practice-switch stale-state class — and the first that leaks
patient data rather than a config flag (contrast #67).

## #70 — ✅ FIXED 2026-09-07 (was HIGH) — clinical notes were read AND written by patient NAME

`pages/PatientWorkspacePage.tsx:380` —

    const { notes: patientNotes } = usePatientNotes(patient?.name);

`resolvedPatientId`, a real numeric id, is computed **three lines earlier at :353** and
not used. `hooks/usePatientWorkspace.ts:288` sends that name to
`notesByPatient(patientName, ...)`.

The backend has an exact branch and a substring branch (`Notes/views/note.py:444-462`):
`patient_id` filters `patient__id=`; `patient_name` filters
`first_name__icontains | last_name__icontains | patient_name__icontains`.

Writing is equally broken: "Create note" navigates with `?patient=<name>`
(`PatientWorkspacePage.tsx:489`, repeated at
`patient-panel/usePatientPanelController.ts:504`), and `NotesEditor.tsx:852` then takes
the FIRST exact-name match from a search ordered by name then id — the lowest-id
same-named patient.

**Scenario:** John Smith #102 and John Smith #205. Opening #205 shows both men's
clinical notes. Clicking "Create note" auto-selects #102 and saves the note against
#102. Clinical information is both disclosed from and written to the wrong record.

## #71 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — universal search opened a different human

**This is the reported "Alex Cooper Intake link shows someone else" bug.**

Django returns, for an intake row (`views/patient_views.py:2624-2636`):

    "record_type": "intake", "record_id": row["id"], "id": row["id"],
    "intake_id": row["id"], "patient_id": row["linked_patient_id"],   # NULLABLE

The frontend type declares only `id`, `patient_id`, `record_id` — no `record_type`, no
`intake_id`, no `person_id` (`universal-search/types.ts:1-10`) — so it cannot tell an
intake from a patient. `universal-search/phone.ts:24` then does:

    const id = patient.patient_id ?? patient.id;

and `UniversalPatientSearch.tsx:42` navigates to `/patients/${routeId}`. For an
unlinked intake that is the INTAKE's id used as a PATIENT id.

**Scenario:** unlinked Intake for Beth has id 42; unrelated Patient Alice is id 42.
Searching Beth and clicking her result opens Alice's workspace.

Secondary: `linked_patient_id` is computed as the first Patient on the Person then
phone/email fallbacks ordered by id (`patient_views.py:2573`), so even a populated
`patient_id` can point at the wrong human on a fused Person.

## #72 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — Add Open/Active Plan demanded a patient id, then discarded it

`components/openplans/AddOpenPlanDialog.tsx:321` refuses to proceed without one —

    if (!selectedPatient || !selectedPatient.patient_id) throw new Error(...)

— then builds the payload at `:356-361` with `patient_data` only: a name re-split at
the first space (`|| 'Unknown'`), email and phone. **`patient_id` is never sent.**
`AddActivePlanDialog.tsx:4` is a wrapper, so both plan types are affected.

The backend accepts `patient_id` (`serializers/treatment_plan.py:745`) and prefers it
over `patient_data`. Absent it, the serializer discovers a Person by email/phone with
`.first()` and matches the reconstructed first/last name.

**Scenario:** patient stored as first name "Mary Jane", last name "Smith". The frontend
sends "Mary" / "Jane Smith"; the exact column comparison cannot find the patient that
was already selected. With same-named people on one channel, `.first()` can attach the
plan to the wrong Patient instead.

NOTE this contradicts nothing in #68: `hooks/useTreatmentPlanCreation.ts` DOES carry
`payload.patient` correctly. This is a *different dialog* hitting the same endpoint
without the id — proof that "the endpoint is used correctly somewhere" is not evidence
it is used correctly everywhere.

## #73 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — a task created from a Nurture row attached to an arbitrary sibling

`serializers/nurture.py:206-211` —

    def get_patient_id(self, nurture):
        """Return the ID of the first linked patient via Person"""
        if nurture.person:
            patients = nurture.person.patients.all()
            return patients[0].id if patients else None

`hooks/useNurtureData.ts:201` carries it through and `NurtureTable.tsx:1806` puts it in
the task modal, posted at `:969`.

**Scenario:** a fused Person holds Patient Alice #10 and Patient Beth #11; the row shows
Beth. `patients[0]` is Alice. Staff pick Beth's row → Create Task; the modal displays
Beth and the task is filed against Alice.

Distinct from #50 (which is about Intake/Nurture API *display* fields): this one writes
the arbitrary id into a new Task.

## #74 — ✅ FIXED 2026-09-07 (was PROVEN / HIGH) — call logging dropped every identifier it was holding

`patient-panel/usePatientPanelController.ts:441-449` posts phone, country code,
direction, status, notes and a display name split at the first space. No `patient_id`,
no `intake_id`, no `nurture_id`. `PatientWorkspacePage.tsx:520` duplicates it.

`messaging/serializers.py:1770` accepts all of them, and its own comment says the
explicit link exists so the log "lands on the right contact's activity log even if the
dialled number differs" — added for the browser extension and never used by the panel
that has the ids in hand.

Without it the log attaches to the raw phone channel. On a shared family line that
channel has no sole Person by design, and patient call history returns every log on
every channel the Person uses (`views/call_log_views.py:119`) — so the call appears in
both family members' histories and belongs to neither.

## #65 addendum — EXTENDED: more call sites than first recorded

The independent pass found the same Nurture CARRY-vs-DISCOVER defect at
`components/compact/AddPatientModal.tsx:937` (the "add existing patient to Nurture"
modal, which holds the selected Patient object at `:732` and sends only name + phone +
email) and at `pages/Journeys.tsx:1403`.

#65 recorded it at `WorkflowPanel.tsx:658`. Three call sites, one root cause: the
Nurture create contract has no writable identity field
(`serializers/nurture.py:121` lists `patient_id` and `person_id` in
`read_only_fields`). Fixing the frontend alone at any one of them is not enough.

## Method note

The first pass and this one overlapped on exactly one finding out of twelve. The first
pass covered the Inbox; the second was told to deprioritise those files and covered the
boards, workspace, search, merge UI and row actions. Neither alone would have found
more than half. Where they disagreed — #68 vs #72 on treatment-plan creation — BOTH
were right about different call sites, which is why every claim here was re-verified
against the file rather than accepted or rejected on the strength of the earlier pass.

---

# #75–#77 — ROUND TWO, sweep three (backend, partial)

Sweep three was launched at codex but died on a provider usage limit 134k tokens in,
partway through reading `Person.resolve`, producing no findings. What follows was read
by hand instead, so it covers two PATTERNS across the backend rather than five areas:
arbitrary-sibling selection, and `Person.resolve` DOB discipline. The Go service,
unauthenticated ingress and realtime/egress remain genuinely unaudited.

## #75 — ✅ FIXED 2026-09-07 (was PROVEN / MEDIUM-HIGH) — call-log patient attribution picked an arbitrary sibling

`messaging/serializers.py:1746-1751` —

    def get_patient_id(self, obj):
        """Get patient ID via Person (through the CallLog's ContactChannel)"""
        person = _get_person_from_channel(obj.channel)
        if person:
            patient = person.patients.first()
            return patient.id if patient else None

No ambiguity guard at all. `get_intake_id` immediately below (`:1754-1759`) has the same
shape with `person.intakes.first()`.

What makes this a defect rather than an accepted limitation is that **the same file
already implements the correct behaviour**. `_get_viewer_patient` (`:441-481`) trusts an
explicit `obj.patient_id` first, matches an email session exactly, and on an SMS session
whose phone matches several patients returns `None` with the comment "Multiple family
members share this phone — cannot identify viewer safely." The call-log path skips all
three protections.

**Scenario:** Beth and Alice share a family landline and sit on one fused Person. A call
is logged against that channel. Every call-log read attributes it to `patients.first()`
— Alice — regardless of who the call was with.

This is the READ side of #74: a call logged without an id (#74) is then DISPLAYED
against the wrong human (#75). Both must be fixed or the log is still wrong.

Distinct from #50, which covers Intake/Nurture list API fields, not call logs.

## #76 — ✅ FIXED 2026-09-07 (was LOW-MEDIUM) — the shared-line guard did not cover its own fallthrough

`messaging/serializers.py:481` — after the email branch and the SMS branch,
`_get_viewer_patient` ends:

    return patients[0] if patients else None

The SMS guard at `:473-480` only runs when `obj.participant_phone_number` is truthy. An
`sms` or `whatsapp` session with an empty `participant_phone_number` falls past all
three guards and returns an arbitrary sibling — the exact outcome the function's
docstring says it exists to prevent.

Narrow, because the field is normally populated. Recorded because the guard reads as
total and is not.

## #77 — ✅ FIXED 2026-09-07 (was LOW) — a third producer of placeholder names, this one backend

`marketingBroadcast/views/submission_handoffs.py:82-84` —

    person, created = Person.resolve(
        practice, first_name or "(unknown)", last_name, channel_ids
    )

#68 recorded two frontend producers of the literal name "Unknown"; this is a third, in
the backend, writing "(unknown)". A nameless human should carry a blank name. Measured
on production 2026-09-07, 249 Persons already hold "Unknown" as a first name.

## Sweep three — checked and CLEAN

**The `Person.resolve` DOB rule is holding.** Every non-test caller either passes `dob`
— `TreatmentPlan/contact/signals.py:60-67`,
`dentallyIntegration/management/commands/bridge_dentally_identity.py:158`,
`TreatmentPlan/management/commands/split_collapsed_persons.py:541` — or documents why it
cannot. The marketing caller carries the comment "a marketing form collects no date of
birth, so there is none to pass. Deliberate, not an oversight — see the guard in
TreatmentPlan/tests/test_resolve_callers_pass_dob.py", and that guard test exists.

Worth recording as a positive result: a rule from an earlier round was enforced with a
test and has stayed enforced. That is the difference between a fix and a permanent fix,
and it is the pattern the remaining findings should be closed with.

## Sweep three — COVERAGE

Examined: arbitrary-sibling selection (`.patients.first()` / `patients[0]`) across the
whole backend, and every non-test `Person.resolve` call site.

NOT reached: the Go service (EmailServiceGo); background writers as a whole (only the
two patterns above were traced through them); unauthenticated ingress; realtime and bulk
egress. Sweeping continues.

---

# #78 — ROUND TWO, sweep four (realtime)

## #78 — ✅ FIXED 2026-09-07 (was PROVEN / CRITICAL) — websocket consumers authenticated the user but never authorised the practice

Three consumers took `practice_id` straight from the URL and join that practice's
broadcast group without ever checking the connecting user belongs to it. Being logged in
to ANY practice is sufficient to receive ANOTHER practice's live patient data.

`messaging/consumers.py:203-213` — the Inbox feed, and the worst of the four:

    async def connect(self):
        self.user = self.scope["user"]
        if isinstance(self.user, AnonymousUser):
            await self.close(code=4001)
            return

        # Get practice_id from URL
        self.practice_id = self.scope["url_route"]["kwargs"]["practice_id"]
        self.room_group_name = f"conversations_practice_{self.practice_id}"

        await self.channel_layer.group_add(self.room_group_name, self.channel_name)
        await self.accept()

The AnonymousUser check is the ONLY gate. It establishes *who you are* and never asks
*what you may see*. Same shape at:

- `TreatmentPlan/consumers.py:27-30` — `intakes_practice_{practice_id}`
- `TreatmentPlan/consumers.py:102-105` — `journeys_practice_{practice_id}`

**CORRECTION (made while fixing).** This finding first listed
`dentallyIntegration/consumers.py:28` and `:181` as a fourth and fifth site. That was
WRONG — it came from grepping for `url_route` and not reading the following lines. Both
DO validate, via `check_practice_access` at `:144` and `:236`. THREE consumers were
vulnerable, not five.

Those two, and `marketingBroadcast`, compare `user.practice` rather than
`current_practice`. That looked like a second defect and is not: `user.practice` is a
property (`UserAuthentication/models.py:673-687`) returning `current_practice`, falling
back to the user's first practice. It is equivalent in the normal case. The one real
difference is that it reads the CACHED FK on the in-memory user rather than the
database, so it shares the mid-session-switch drift already documented on
`Invoices.consumers.get_fresh_practice_id` — a smaller issue than a missing guard, and
the shared helper below avoids it.

What is broadcast to the messaging group is full patient conversation data —
`messaging/utils.py:1109-1114` sends `conversation_data` (participant name, phone number,
message preview) on every new conversation, and `broadcast_conversation_updated` at
`:1137` does the same on every update.

**Scenario:** a staff user at practice A opens a socket to
`/ws/messaging/conversations/<practice B id>/`. The practice id is a small integer and
the URL is visible in their own browser's network tab. From then on they receive practice
B's patient names, phone numbers and message previews live, as those patients message
practice B. Nothing in the request is forged — the user is genuinely authenticated, just
not entitled.

**This is not a design limitation: the same codebase already does it correctly, three
different ways.**

- `marketingBroadcast/consumers.py:46-54` calls `check_practice_access(user, practice_id)`
  and closes the socket when it fails.
- `Appointments/consumers.py:21-29` never trusts the URL at all — it derives the group
  from `self.user.current_practice`, so a forged id cannot influence it.
- `Invoices/consumers.py:108-117` closes with a distinct 4002 code and re-reads
  `current_practice_id` from the database rather than the cached FK.

The Appointments pattern is the strongest (an unvalidatable input is better than a
validated one) and should be the template. Note also that
`marketingBroadcast.check_practice_access` compares `user.practice`, whereas the rest of
the app uses `current_practice` — worth reconciling when this is fixed, so the
replacement guard is correct everywhere rather than merely present.

Relation to earlier findings: #22/#23/#24/#40/#43 are cross-practice writes and reads
over HTTP. This is the same class over the websocket transport, which the audit had not
examined until now — and unlike those, it needs no crafted request at all.

**AS BUILT.** One shared rule in `utils/ws_practice.py` — `reject_unless_member()`,
called BEFORE `group_add` in all three consumers, closing with a distinct 4003 so the
frontend can tell "log in again" from "not your practice". It re-reads
`current_practice_id` from the database rather than trusting the cached FK, so a soft
practice switch cannot leave a live socket joined to the old practice's group. The id is
compared as a string because the URL kwarg is text and the column is an integer, and
`"16" == 16` is False — a guard that failed that way would lock every user out just as
completely as the missing guard let everyone in.

Proven by execution, not by reading: `messaging/test_ws_practice_authorisation.py` drives
each consumer through `WebsocketCommunicator`. Red run against the unfixed code — 3
failures, all "foreign practice joined the feed", with the own-practice cases passing so
the fixture was known sound. After the fix, 8/8 pass. Each consumer is tested BOTH ways
(own practice must still connect), because a guard that rejects everything would satisfy
a one-sided test while breaking the product.

---

# #67 + #69 — AS BUILT 2026-09-07

Fixed together because they are one defect wearing two faces: a module-level cache that
outlives the practice switch. This was the sixth such cache found, so the fix targets the
CLASS rather than the instances.

`src/hooks/useCachedData.ts` gains a registry — `registerPracticeScopedCache(cache)` —
and `clearAllCachedData()`, which the soft switch ALREADY calls
(`PracticeSwitcher.tsx:139`), now wipes everything registered as well as `globalCache`.
No change at the call site, so there is nothing for a future switch path to forget.
Registering is one line at the point of declaration, which is the only place a developer
adding cache number seven will be looking.

Joined so far: `dayListCache`, `recallsListCache`, `dormantListCache` (#69) and the
`useWhatsAppConfig` singleton (#67, via a `{clear}` adapter, since it is two module
variables rather than a Map). One cache throwing cannot strand the others — a
half-cleared switch is the bug being fixed.

**Red run, `src/hooks/practiceScopedCacheRegistry.test.ts`:** with the registry ignored
in `clearAllCachedData` (the pre-fix behaviour), **5 of 6 tests fail**. The single test
that still passes is "leaves a cache that never registered untouched" — which SHOULD pass
either way, and exists so the suite cannot go green by clearing something global for the
wrong reason. After the fix 6/6 pass, 26/26 across the hook suites, and `tsc -p
tsconfig.app.json` reports no new errors in any touched file.

One test imports the four real hook modules so their top-level registration actually
executes. Without it a hook could silently drop out of the registry in a refactor and
every other test here would still pass.

**Deliberately NOT done:** the caches are still keyed by date/query rather than by
practice. Keying by practice would make a cross-practice hit impossible by construction,
which is stronger than clearing. It was not done here because it needs the practice id
threaded into three hooks that do not currently take it, and the clearing fix closes the
reported leak today. Worth doing when those hooks are next touched.

---

# #70 — AS BUILT 2026-09-07, with a CORRECTION to the reported symptom

**The fix stands; the stated symptom was wrong and is corrected here.**

First write-up (from the independent pass, repeated by me without testing it): "John
Smith #102 and John Smith #205 — opening #205 shows both men's clinical notes."

Executing it says otherwise. The name branch tests each column separately with
`icontains`:

    Q(patient__first_name__icontains=patient_name)
    | Q(patient__last_name__icontains=patient_name)
    | Q(patient_name__icontains=patient_name)

`first_name` is "John" and `last_name` is "Smith", so the JOINED string "John Smith" is
contained in neither. The only column that could hold it is `Note.patient_name`, whose
own help_text reads "Patient name when no patient record is associated" — empty for the
linked notes a workspace displays.

**So the production symptom was a patient's clinical notes silently MISSING from their
own workspace, not another patient's notes appearing.** Pinned by
`test_the_full_display_name_matches_NOTHING`.

The cross-patient disclosure is real but needs a SINGLE-token query:
`test_substring_matching_reaches_a_different_person_entirely` proves `patient_name=John`
returns Johnathan Smithers' note, because "John" is a substring of "Johnathan".

Either way the name branch cannot reliably identify the right human — it under-matches a
full name and over-matches a partial one — so the fix is unchanged and its justification
is stronger.

**AS BUILT.**
- READ: `usePatientNotes` in `hooks/usePatientWorkspace.ts` now takes a patient id and
  calls `notesByPatient(id, /* isPatientId */ true, ...)`. Same endpoint, same drafts and
  practice-scope flags — only the identifier changes. `PatientWorkspacePage.tsx:382`
  passes `resolvedPatientId`, which was already computed at `:353` and sitting unused.
- WRITE: `handleCreateNote` now adds `patientId` to the query string.
  `NotesEditor.tsx:814` already PREFERS `patientId` over `patient` when both are present
  — the id was simply never sent. The name is still sent so the editor can label the
  screen without a lookup.

Note there are TWO hooks named `usePatientNotes`. `hooks/usePatientNotes.ts` was already
id-based but calls a DIFFERENT endpoint (`patient_notes/<id>/`) that does not take the
drafts/practice-scope flags, so swapping to it would have changed what the tab shows.
The narrower change was to fix the identifier on the hook already in use.

**Verification.** `Notes/tests/test_notes_by_patient_identity.py`, 5/5. The id branch is
proven exact (`{"A's note"}` only) — the fix is only safe because of that, so it is
asserted rather than assumed. `test_id_branch_wins_when_both_are_supplied` covers the new
call shape: the frontend now sends the name for display ALONGSIDE the id, and if the name
could widen the result the fix would reintroduce the bug through itself.

Three fixture traps cost time and are recorded for the next person: `Note` requires
`user`, `title` and `content`; the Notes views sit behind middleware that cannot see
DRF's `force_authenticate` (a real Bearer token carrying a `practice_id` claim is
required); and `SubscriptionMiddleware` returns 402 unless `_get_subscription_info` is
patched. All three are documented at the top of `Notes/tests/test_notes.py`.

---

# #71 — AS BUILT 2026-09-07

The reported "click the Intake result, see somebody else".

**AS BUILT.** Three changes, because the defect had three layers:

1. `universal-search/types.ts` now declares what the endpoint ACTUALLY sends —
   `record_type`, `intake_id`, `nurture_id`, `person_id`, and `patient_id` as
   nullable. It previously declared only `id`, `patient_id`, `record_id`, so the
   frontend was structurally incapable of telling an intake from a patient. Declaring
   the fields is most of the fix: the bug was possible because the type hid the
   distinction.
2. `getPatientRouteId` is replaced by `getResultRoute`, which returns a WHOLE path and
   keys on `record_type` — `/patients/<patient_id>`, `/patients/intake/<intake_id>` or
   `/patients/nurture/<nurture_id>`. Those journey routes already existed
   (`App.tsx:404`, `:412`); nothing was navigating to them. The patient branch uses
   `patient_id` ONLY and returns null without one — the `?? id` fallback WAS the bug.
3. A separate `getResultDisplayId` for the disambiguation badge. The old single value
   was doing two jobs — a route and a number shown to the user — and was correct for
   only one of them. Splitting them is what stopped the fix rendering `#/patients/intake/42`
   in the badge, which the first attempt at this did.

An intake now opens the intake the user clicked, even when it HAS a linked patient.
Silently redirecting to the linked patient is how one row came to stand for two records.

**A GREEN TEST WAS PROTECTING THE BUG.** `phone.test.ts` asserted:

    expect(getPatientRouteId({ id: 42 })).toBe("42");

That is "address a patient by an intake's primary key", written down and passing. It has
been replaced, and the reason is recorded in the test file so it is not restored by
someone reading the deletion as a regression. Worth remembering generally: a green suite
is not evidence of correct behaviour when the assertion encodes the defect.

**Verification.** 12 tests in `phone.test.ts`, 36 across the suite. Red run with
`getResultRoute` reverted to `patient_id ?? id`: **5 of the 6 new routing tests fail**,
including "never reaches /patients/<id> using a bare id". `tsc -p tsconfig.app.json`
clean on every touched file.

**Not addressed here (still open).** `linked_patient_id` is computed backend-side as the
first Patient on the Person, then phone and email fallbacks ordered by id
(`patient_views.py:2573-2599`). On a FUSED Person a populated `patient_id` can still name
the wrong human. That is the #50/#73 arbitrary-sibling class, not this routing bug, and
routing correctly to a wrong `patient_id` is a different defect from routing to an
intake's id.

---

# #64 + #66 — AS BUILT 2026-09-07

Fixed together: one component, one mistake — ids held in state and dropped at the wire.

**AS BUILT.** `SendSmsRequest` and `SendEmailRequest` (`components/inbox/types.ts`) gain
an optional `patient_id`, and `InboxPage.tsx` populates it from
`selectedContact.patient_id` at both send sites (`:923` email, `:947` SMS).
`openTaskModal` (`:1064`) does the same instead of the hardcoded
`patient_id: null, // Phase 49 will wire patient IDs from thread`.

No backend change: `messaging/views/message_views.py:362` has always read `patient_id`
off the request. The journey panels sent it for the touch-point counter and were
attributed correctly all along; the main Inbox never sent it, so `attribute_to` was
unreachable from the surface that sends most messages.

**The ids were demonstrably available.** `InboxPage.tsx:1344` — the patient-panel opener,
in the same component, off the same `selectedContact` object — already passed
`patient_id`, `intake_id`, `nurture_id` AND `activity_log_id`. The send and task paths
simply did not, while reading `normalized_phone` from that same object one line away.

**This is the fix for the reported +447789791781 thread** — three humans on one family
line (Sharon Harper, Talia Harper-smith, an unnamed caller), session 1993 with
`patient = None`, showing a bare number. New sends from the Inbox now name their
recipient. Existing sessions are NOT retroactively attributed; see the note below.

**Verification.** The receiving contract was ALREADY under test —
`messaging/test_sms_recipient_attribution.py::PanelsWithoutAJourneyRecordCanStillNameTheRecipient::test_an_explicit_patient_id_names_the_recipient`
covers exactly the payload the frontend now sends, including the practice-boundary
refusal. 16/16 pass. `tsc -p tsconfig.app.json` clean on both touched files.

**No new frontend test, and why.** The payload is built inline inside a ~2,000-line
component; asserting one field would mean rendering it in jsdom with a mocked fetch, auth
context and websocket — a mock-heavy test whose failure would more likely indicate a
broken mock than a broken payload. The behaviour that matters (does the backend attribute
when given this field) is covered by a real test above. Recorded rather than glossed: the
frontend half is verified by type-check and inspection, not by execution.

**Still open after this.** Two prerequisites remain before production actually attributes:
1. The prod container is running an image WITHOUT the `attribute_to` call sites — the
   code is on the host checkout but was never rebuilt. Until it is, the backend ignores
   the field the frontend now sends.
2. Historical sessions stay unattributed. Backfilling them from message TEXT would be
   name-based inference, which this workstream exists to eliminate. The sound route is
   the source record's patient id, and it is not attempted here.

---

# #52 — AS BUILT (attempt 1, REVERTED) 2026-09-07

**AS BUILT.** The RECIPIENT now decides the practice; `practice_id` in the body may only
AGREE with it. A conflict is refused and logged as a security event. Recipient resolution
was widened to cover `PracticePreferredDomain` (domain+prefix, then domain alone, and an
explicit refusal when several practices share a domain with nothing to separate them) —
without that, every practice receiving at `<prefix>@<own domain>` would have had its mail
rejected.

**Red run:** the three attack tests return **201** against the old code — the endpoint
genuinely accepted a forged email naming any practice. 6/6 after the fix; `messaging`
179/179.

**Two mistakes caught by testing, recorded because both were nearly shipped:**
1. The FIRST red run PASSED, which meant the tests were worthless — I had guessed the URL
   (`/emails/receive_email_v2/` instead of `/webhook/receive-email-v2/`) and every request
   401'd at the middleware before reaching the view. A red run that passes is not a
   green light; it means the test is not exercising the code.
2. With the URL fixed, the "legitimate mail must still flow" tests failed — my fix would
   have DROPPED real inbound email. Two causes: my fixture set
   `normalized_email_prefix` directly, which `Practice.save()` OVERWRITES from the slug
   (`UserAuthentication/models.py:306`), and the genuine domain-addressing gap above.

**A pre-existing test was depending on the vulnerability.**
`messaging.tests.InboundReplyActivityLogTests` posted to `inbox@practice.test` with a
`practice_id`, against a practice whose prefix was `inboundlogpractice`. It only ever
passed because the claim was trusted. Given a real `PracticePreferredDomain` row rather
than weakening the fix — its subject is activity logging, not routing.

**DEPLOYMENT PRECONDITION — CLEARED 2026-09-07.** Verified read-only against production:
across **2,576 real inbound emails**, the actual delivery addresses resolve to **43
distinct mailboxes and all 43 resolve** — 36 by domain, 5 by slug prefix, 2 by
domain+prefix. **Zero would be rejected.** Safe to ship.

Note the domain path carries 36 of 43. The FIRST version of this fix did local-part
matching only, so it would have rejected roughly 84% of inbound mail. That gap was caught
by the "legitimate traffic must still flow" test before it shipped; this query quantifies
how large it was.

**A correction to the verification method itself, recorded because the first answer was
wrong.** The initial query parsed every address out of `To:`/`Cc:` and reported **127
addresses would be dropped**. Those were third parties on bulk and cc'd mail
(`immunology@esth.nhs.uk`, `info@fabian-society.org.uk`) — never this system's inbound
mailboxes, and never what the webhook routes on. Narrowing to the delivery headers
Postfix actually sets (`Delivered-To` / `X-Original-To`) gives 43/43. Stopping at the
first result would have condemned a safe fix.

The practice-side recipient is NOT a column on `EmailMessages`; it exists only inside the
`raw_email` MIME headers, which is why this took parsing rather than a simple query.

---

# #48 + #49 — AS BUILT 2026-09-07

One defect in five places: **authorship was treated as entitlement.** The routes scoped
clinical records by who WROTE them — `filter(user=user)` or
`filter(user__practices=practice)` — so a clinician working in two practices, or who moved
between them, carried their old records into the new practice's view.

**AS BUILT.** One rule, `scope_clinical_to_practice(queryset, practice)` in
`utils/practice_mixins.py`, applied at all five sites:
- `Notes/views/note.py` — `notes_by_patient` (both the user-scoped and the
  `practice_scope=true` branches) and `unique_patient_names`.
- `Notes/views/letters.py` — `get_object()` for the image/render actions (`:327`), the
  practice-user lookup (`:596`), and the two `retrieve` paths that bypass
  `get_queryset()` (`:1318`, `:1358`).

**Why the PATIENT's practice and not the record's.** `Note.practice` and
`NotesLetter.practice` are NULL on legacy rows — the production proof for #48 says so
explicitly ("the letter's legacy `practice_id` is NULL"). Filtering on that column would
have HIDDEN real records while still leaking exactly these ones. The patient always has a
practice, and "may I read clinical material about this patient" is the question actually
being asked.

Records with NO patient are kept: they cannot disclose a patient, and the caller's own
author/practice filter still applies. Without that carve-out the fix would have hidden
every general note.

**The correct pattern already existed one function away.** The main letter queryset
(`letters.py:264`) was fixed under #22 and reads
`Q(patient__isnull=True) | Q(patient__practice=practice)` — the shared helper is that
same rule, named. #48 and #49 are what happens when a fix lands on one queryset and not
on its siblings.

**Red run** (`Notes/tests/test_cross_practice_clinical_scoping.py`): with the gates
removed, **4 of 6 fail** — the foreign practice's note is returned by `notes_by_patient`
in the default, `practice_scope=true` and `include_drafts=true` branches, and
`unique_patient_names` lists the other practice's patient by name. 6/6 after. The
fixture is the production shape: ONE author who is a member of BOTH practices, with a
patient in each, and the foreign note's own `practice` column left NULL.

Each half is asserted BOTH ways — the caller's own notes and own patient list must still
be returned, because a filter that hides everything would satisfy the security half while
breaking the product.

**Regression:** `Notes` 153/153 excluding the 5 pre-existing failures proven unrelated
(letter finalisation via the known `Notes/tasks.py:30` `.service`/`.services` import bug,
plus three seed-data tests). Pre-commit clean.

**Not covered by these tests:** the three `letters.py` sites are fixed and syntax-checked
but exercised only indirectly — the production proof for #48 used `retrieve` and
`render_content` on NotesLetter 29, and I did not rebuild that scenario as a test. The
note routes carry the executable proof; the letter routes carry the same one-line rule.

---

# #82–#89 — ROUND TWO, sweep five: the Go service (EmailServiceGo)

Produced by the independent codex sweep (area 1 of 5), which was told to hunt only in Go
and to justify why each finding is distinct from #1–#81. Eight findings. **Verification
status is stated per finding — some are confirmed by execution, some are read-only so
far, and one had its impact CORRECTED downward by measurement.**

## #82 — ✅ FIXED 2026-09-07 (was CRITICAL) — unauthenticated Twilio media stream could create identity in ANY practice

`internal/callagent/routes.go:149` — `callAgent.GET("/media-stream", ...)` is documented
in its own comment as "NOT authenticated". `handlers.go:843` then trusts the peer:

    if id, ok := twilioMsg.Start.CustomParameters["practiceId"]; ok {
        info.PracticeID = id
    }

Those attacker-supplied values drive `GetPatientNameByPhone(...)` and mint a session
token; the also-unauthenticated `/call-status` handler later creates the Intake
(`storage.go:1398`).

**Scenario:** connect to `/call-agent/media-stream`, supply `practiceId=13` and any
`from` number and CallSid, speak, then POST `CallStatus=completed`. Go performs a
practice-13 patient lookup and writes a practice-13 Intake/Person that neither Twilio nor
practice 13 authorised.

Distinct from #78 (authenticated Django consumers joining a foreign feed) and #52
(inbound email routing): this is unauthenticated telephony ingress that WRITES identity.

**Verification: read-only.** The route, the trust and the write path were read and match
the report. Not executed — doing so means standing up a Twilio-shaped websocket.

## #83 — ✅ FIXED 2026-09-07 (was CRITICAL) — a workflow node's config overrode the execution's authenticated practice

`internal/workflows/actions/query_records.go:81` prefers a practice id stored in NODE
CONFIG over the practice injected at execution (`service.go:579`):

    if configPracticeID, ok := config["practice_id"].(float64); ok {
        practiceID = int(configPracticeID)

Django then trusts it (`automations/actions.py:2569`), and the HTTP-request action
(`http_request.go:93`) can POST the returned records to an arbitrary URL.

**Scenario:** a practice-16 user adds a `query_records` node to their OWN workflow with
`practice_id: 13`, entity `patient`, condition `id > 0`, then an HTTP node pointing at
their own server. Practice-13 patient names, emails and phones are exfiltrated.

**Verification: read-only.** Cited lines read and consistent. Not executed.

## #84 — ✅ FIXED 2026-09-07 (was HIGH) — Dentally import overwrote an arbitrary same-name relative

`service.go:1689` — when a Dentally id is absent locally the importer falls back to
name+email and takes `Limit(1)` with no DOB or ambiguity check, then OVERWRITES that row
with the incoming `meta_data` and `date_of_birth` (`service.go:1017`).

**Scenario:** father Patient 101 and son Patient 102, both "John Smith" on one family
email. Importing the son can select the FATHER and overwrite him with the son's DOB and
Dentally id.

**Verification: read-only.**

## #85 — ✅ FIXED 2026-09-07 (was HIGH, impact CORRECTED DOWN) — call-agent PersonChannel insert always fails

`internal/callagent/storage.go:1104`:

    INSERT INTO "TreatmentPlan_personchannel"
        (person_id, channel_id, is_primary, created_at) VALUES (?, ?, false, NOW())

**Confirmed against the live schema** — `role` and `updated_at` are both `NOT NULL` with
**no database default**:

    role       | NO | (none)
    updated_at | NO | (none)

So this statement fails every time, and `s.db.Exec(...)` discards the error. Go's own
`schemaguard.go:214` already lists both columns as required. Classic Django
`default=`/`auto_now` being Python-only.

**IMPACT CORRECTED.** The report implied repeated calls mint duplicate Persons at scale.
Measured on production: of **974** Persons owning a call-agent Intake, **967 DO have a
channel link and only 7 do not**. Something downstream (the Django save signal) links
almost all of them afterwards. The defect is real and should be fixed — it is a silently
swallowed write failure — but it is not the mass duplicate driver it appeared to be.

## #86 — ✅ FIXED 2026-09-07 (was HIGH) — call agent collapsed a fused Person to its first Patient

`storage.go:922` — `ResolveCallerCandidates` discards every row after the first per
Person. Same arbitrary-sibling class as #50/#73/#75, in Go.

**Verification: read-only.**

## #87 — ✅ FIXED 2026-09-07 (was HIGH) — Telnyx assistant read every family member's appointments off a shared phone

`telnyx/handlers.go:349,420` — `lookupUpcomingAppointments(callerPhone, ...)` matches
appointments by PHONE with no Person or patient id and no ambiguity guard, then injects
the result into the assistant prompt.

**Scenario:** Alice and her child Ben share a landline. Alice calls; the assistant
receives BOTH appointments and may read out Ben's date, clinician and reason as hers.

#38 fixed name selection on a shared channel; this lookup bypasses that guard entirely
and aggregates clinical scheduling data.

**Verification: read-only.** This one is worth prioritising — it speaks clinical data
aloud to the wrong person.

## #88 — ✅ FIXED 2026-09-07 (was HIGH, CONFIRMED) — every three-digit international dialling code was corrupted

`storage.go:706` —

    } else if len(phone) > 3 {
        // Assume +XX format for other countries
        countryCode = phone[:3]
        phoneNumber = phone[3:]

**Confirmed by reading:** every non-`+1` code is assumed to be exactly two digits. `+44`
(UK) works by luck. `+234`, `+353`, `+351`, `+358` do not: `+2348012345678` becomes
country `+23`, number `48012345678`.

The Intake's `normalized_phone` is computed from the UNTOUCHED source, but Person
resolution is handed the mangled local part — so the two disagree and an existing Person
on that number is missed.

**Supporting production signal:** call-agent intakes show one number resolving to several
Persons — practice 16 has `7469726576` → **9 distinct Persons**, `7787555777` → 4,
`7824880606` → 4, and several short/odd values (`38758530`, `35356888`) consistent with
mangling. Also `anonymous` → 6 Persons, i.e. withheld numbers are being treated as an
identity. **This, not #85, is the likelier duplicate driver, and the `anonymous` grouping
is worth a finding of its own.**

## #89 — ✅ FIXED 2026-09-07 (was MEDIUM) — Go wrote the literal name "Unknown"

`storage.go:701` — `if firstName == "" { firstName = "Unknown" }`, then written to BOTH
the Intake and a new Person (`storage.go:1059`). A fourth producer of placeholder names
alongside #68 (two frontend) and #77 (Django). Matches the 249 production Persons
literally named "Unknown".

**Verification: read-only, but corroborated** by the earlier production count.

## Sweep five — coverage and caveats

Codex reported covering all call-agent paths (Twilio + Telnyx, media stream, webhooks,
tokens, storage), Dentally migration and fallback matching, recall/day-list writes,
workflow API and actions, and marketing/Postmark handlers.

**I verified #85 and #88 directly (schema query and code read) and read the cited lines
for the rest. #82, #83, #84, #86, #87 are recorded on codex's evidence plus a line read —
they should each be confirmed before being fixed.** The one impact claim I measured
(#85) turned out to be roughly 100x smaller than implied, which is the reason for that
caution.

---

# #48 — GAP CLOSED 2026-09-07: the letter routes now have executable proof

The #48/#49 write-up recorded an honest gap — the two NOTE routes were proven by red
run, the three LETTER routes carried the same rule but were never driven by a test.
`Notes/tests/test_cross_practice_letter_scoping.py` closes it. 5 tests, 11 across both
files.

**Red run** (helper stripped from every letter call site): **3 of 5 fail**, including
`test_the_foreign_clinical_text_is_never_in_the_body`, which asserts the other practice's
clinical WORDS are absent from the response bytes rather than trusting a status code — a
route can return 200 with an empty envelope, or 403 with the content in the error payload.

**A finding within the finding.** In that red run
`test_retrieve_refuses_another_practices_letter` did NOT fail, while the `render_content`
equivalent did. So of #48's three sites, the leaking one was the `get_object()` path used
by `render_content` and the image actions; plain `retrieve` was already covered by the
main queryset fixed under #22. The production proof reported BOTH returning 200 — that
remains what was observed there, and the difference is worth noting rather than
smoothing over: the scoping now applied to all three is belt-and-braces on `retrieve` and
the actual fix on `render_content`.

(One red-run artifact, recorded so it is not misread: `test_retrieve_still_returns_my_own`
also failed under the strip, because crudely removing the wrapper mangled that queryset.
That is an artifact of the revert, not a signal about the fix.)

---

# #50 + #62 + #73 + #75 — AS BUILT 2026-09-07 (one rule, four call sites)

Four readers each answered "which Patient does this Person represent?" with
`person.patients.first()`. A Person is meant to be ONE human; some in this database are
FUSED and hold several, so `.first()` returns whichever row the database ordered first.

The value is not merely displayed. #73's is posted back when creating a task from a
Nurture row, and #62's is pushed to Dentally as a consent decision — so an arbitrary
sibling was being WRITTEN against, not just shown.

**AS BUILT.** `TreatmentPlan/utils/sole_patient.py` — `sole_patient_for_person()` and
`sole_record_for_person()`. The rule is NOT new: it is the one already proven under #37
in `marketingBroadcast/tasks.py`, lifted out so four readers stop inventing their own.

  1. exactly one patient carrying the Person's own name -> that one
  2. only one patient at all -> that one (a name mismatch is not ambiguity)
  3. otherwise -> **None**

Returning None is the point. On a fused Person there is no right answer, and every caller
now shows nothing rather than the wrong human — the same stance
`sole_person_for_channel` takes on a shared phone line.

Applied at: `serializers/intake.py:252` (#50), `serializers/nurture.py:206` (#73),
`messaging/serializers.py:1746` and `:1754` (#75), `marketingBroadcast/consent_ledger.py`
×2 (#62).

**Evidence kept where it existed.** The Intake serializer already matched on the intake's
OWN email then phone before falling back — that is real evidence and was preserved; only
the `patients[0]` guess was replaced. Nurture had NO such matching, so it gained the same
email/phone step as well as the helper: strictly better than before, not just safer.

**Red run** (`TreatmentPlan/tests/test_sole_patient_for_person.py`, 12 tests): with the
helper mutated back to `.first()`, **4 fail** — the fused-Person refusal, the
name-singles-one-out case, the two-same-named-humans refusal, and the intake equivalent.
12/12 after.

**Regression:** 1,065 tests across TreatmentPlan, messaging, Notes and marketingBroadcast
— only the 5 pre-existing Notes failures proven unrelated earlier. Pre-commit clean.

**Still open in this class:** #86, the same defect in Go
(`internal/callagent/storage.go:922`, `ResolveCallerCandidates` discarding every row after
the first per Person). It needs the same rule in Go and is not fixed here.

---

# #72 + #74 — AS BUILT 2026-09-07

Both are the same shape as #64/#66: an id held in state and dropped at the wire.

## #74 — call logging

`patient-panel/usePatientPanelController.ts:441` and `PatientWorkspacePage.tsx:538` now
send `patient_id` (plus `intake_id`/`nurture_id` where the panel holds them) alongside the
phone number.

No backend change. `CreateCallLogSerializer` (`messaging/serializers.py:1770`) has accepted
these all along — its own comment says the explicit link exists so the log "lands on the
right contact's activity log even if the dialled number differs". It was added for the
browser extension and never used by the panel that holds all three ids.

**This completes the pair with #75.** #74 is the WRITE side (the log now says who the call
was with) and #75 the READ side (the reader no longer picks an arbitrary sibling). Either
alone leaves the call attributed to the wrong human on a shared family line, where the
channel has no sole owner by design.

## #72 — Add Open/Active Plan

`openplans/AddOpenPlanDialog.tsx:362` now sends `patient_id`. The dialog already REFUSED to
submit without `selectedPatient.patient_id` and then sent only `patient_data` — a display
name re-split at the first space. `AddActivePlanDialog` wraps this component, so both plan
types are covered by the one change.

Verified the receiving contract rather than assuming it:
`serializers/treatment_plan.py:984` reads "patient_id takes priority over Person lookup …
skip the email/phone lookup entirely", and the lookup is practice-scoped —
`Patient.objects.get(id=patient_id, practice=practice)`. So the id is both preferred AND
cannot reach across practices.

`patient_data` is still sent; it is ignored when `patient_id` resolves and remains the
fallback for genuinely new patients.

**Verification.** `tsc -p tsconfig.app.json` clean on all three touched files.

**No new frontend test, same reason as #64/#66:** these payloads are built inline in large
components, and asserting one field would need jsdom plus mocked fetch/auth/websocket — a
test whose failure would more likely mean a broken mock than a broken payload. The
receiving contracts are covered by real backend tests. Recorded rather than glossed: the
frontend half here is verified by type-check and inspection, not execution.

---

# #65 — AS BUILT 2026-09-07 (the last CARRY-vs-DISCOVER)

The Nurture create API had NO way to say which existing human a nurture is for:
`patient_id` and `person_id` on `NurtureSerializer` are SerializerMethodFields, output
only. So every "add this patient to Nurture" sent name+email+phone and let the save signal
re-resolve the person — right for a genuinely new lead, wrong when the caller already
knows, because `Person.resolve` deliberately mints a NEW Person when a candidate on the
same channel carries a different name.

**AS BUILT — backend.** `NurtureSerializer` gains a write-only `from_patient_id`, and a
`create()` that resolves it **practice-scoped** (`Patient.objects.filter(id=..,
practice=..)`) and stamps `validated_data["person"]`. The contact signal already leaves an
explicitly supplied person alone, so the carry survives the save.

A DISTINCT name rather than making `patient_id` writable: read and write mean different
things here — read is "who did this resolve to", write is "file this against THIS human" —
and collapsing them onto one field is what makes an API like this easy to misuse.

An unknown or foreign id is REJECTED with 400, never silently ignored: ignoring it would
fall straight back to re-discovery and produce the duplicate this finding is about.

**AS BUILT — frontend, all three call sites** recorded in the #65 addendum:
`WorkflowPanel.tsx:658`, `compact/AddPatientModal.tsx` (which now also carries the picked
patient's id out of the form as `selectedPatientId`), and `pages/Journeys.tsx:1403` — that
last one REBUILDS `apiData` from scratch, so an id set upstream was being discarded there
regardless.

**Verification.** 6 tests, `TreatmentPlan/tests/test_nurture_carries_identity.py`. Red run
with the carry removed: `test_carrying_the_patient_reuses_that_persons_identity` fails —
`person_id` is `None` instead of the carried Person. Practice-boundary refusal, unknown-id
refusal, the still-works-for-a-new-lead case and write-only-ness all pass. TreatmentPlan
815 tests with only the 6 catalogued pre-existing failures; `tsc` clean; pre-commit clean.

**HONEST LIMIT of one test.** `test_carrying_mints_no_new_person` ALSO passes in the red
run, so it is not currently proving what its name claims: without the carry the signal
assigned NO person at all in this fixture rather than minting a duplicate. What is proven
is that the carry lands the nurture on the right existing Person; that this specific
fixture would otherwise duplicate is NOT demonstrated. The test is kept as a regression
guard, but it should not be read as the duplicate proof.

**Two tests briefly passed for the wrong reason** during development — every request was
401ing at `SubscriptionMiddleware` (which runs before DRF and cannot see
`force_authenticate`), so assertions of the form "not 200" were trivially true. Fixed with
a real Bearer token carrying a `practice_id` claim. Same trap as the Notes suites; worth
remembering that a passing assertion against an error response proves nothing.

---

# #46 — AS BUILT 2026-09-07

The finding's own prescribed fix was: "the send endpoint should accept the
`patient_id`/`person_id` the caller already supplies and stamp the session and SMS with
it, falling back to `sole_person_for_channel` only when the caller gives nothing."

The BACKEND half of that shipped on 2026-09-06 (`attribute_to`, reading `patient_id` from
the request at `messaging/views/message_views.py:362`). What was missing was any caller
sending the field. #64 supplied it from the main Inbox. This closes the PATIENT PANEL,
which is the surface the finding was actually reported from.

**AS BUILT.** `patient-panel/PatientInboxTab.tsx` computes `recipientPatientId` once and
passes it on ALL FOUR send paths — inline email, inline SMS, legacy email, legacy SMS.
The panel had four because it straddles the contact and legacy session models, and none of
them carried the id.

The id was never unavailable: the same value was read two lines further down to call
`fetchThread`. That call now reuses the const rather than recomputing it.

**Why this is not a guess.** `sole_person_for_channel` refuses to name anyone on a shared
line, and that guard is correct — it is #38/#41 working. But a member of staff opened a
specific patient and pressed send, so the recipient is known, not inferred. The fallback
is unchanged: with no id supplied, channel resolution still applies.

**Verification.** `tsc -p tsconfig.app.json` clean. The receiving contract is covered by
`messaging/test_sms_recipient_attribution.py` (16/16), which includes the
practice-boundary refusal — a `patient_id` from another practice is ignored rather than
honoured.

**Same caveat as #64/#66/#72/#74:** no new frontend test. These payloads are built inline
in a large component and asserting one field would need jsdom plus mocked
fetch/auth/websocket. Verified by type-check and inspection; the backend half is what
carries executable proof.

**PRODUCTION NOTE — CORRECTED.** An earlier note here claimed the production container
predated the `attribute_to` call sites. That was wrong; see the correction under #64. The
running container has them at `message_views.py:357` and `:2026`, so the backend is ready
to consume what these four paths send. Only the frontend change awaits an ordinary
deploy.

---

# #55 — AS BUILT 2026-09-07

A Q-object precedence bug in `associate_treatment_plans`
(`TreatmentPlan/views/patient_views.py:1877`). The filter was built as:

    filters = Q(practice=practice)
    if patient.email:        filters &= (email match)   # AND — correct
    if patient.phone_number: filters |= (phone match)   # OR  — the defect

For a phone-only patient that reads "every plan in this practice OR a phone match", so
the endpoint associated EVERY plan in the practice with them. 19,635 active phone-only
patients could reach it.

**AS BUILT.** Rebuilt as **practice AND (email match OR phone match)**: a `contact_match`
Q is assembled from whichever contact details exist, then ANDed onto the practice
predicate. The practice scope can no longer be ORed away, and the two contact clauses are
alternatives to each other rather than to the scope.

An empty `contact_match` now returns 400 rather than being ANDed as a no-op — an empty
`Q()` is truthy-neutral in a filter and would have handed back the whole practice again,
which is the same bug by a different route. The endpoint already rejected such patients
earlier; this is a second guard at the point where it would actually do damage.

**Red run** (`TreatmentPlan/tests/test_associate_plans_scoping.py`, 5 tests): with the
OR precedence reinstated, the phone-only patient collected **all three plans**
`{4034, 4035, 4036}` — two of them strangers' — reproducing the production symptom
exactly. 1 of 5 fails, which is correct: the other four describe behaviour the bug did not
break.

Specifically, `test_another_practices_plan_is_never_associated` passes BOTH ways, because
the phone branch carried `practice=practice` inside its own clause. Recorded so the fix is
not credited with more than it did: the defect was the phone-only case within one
practice, not a cross-practice leak.

**Regression:** TreatmentPlan 820 tests, only the 6 catalogued pre-existing failures.
Pre-commit clean.

---

# #54 — AS BUILT 2026-09-07

The importer prefetched `contact -> ONE patient` dictionaries built by comprehension, so
on a shared family email or phone the LAST row won and every earlier relative was
overwritten. The prefetch did not even SELECT the name columns, so the importer could not
have checked the submitted name if it had wanted to. 18,725 active patients sit on such a
shared key.

**AS BUILT** (`TreatmentPlan/views/patient_views.py`):
- The prefetch now selects `first_name`/`last_name`.
- The three dictionaries become `defaultdict(list)` — every candidate on a contact is
  kept, not just the survivor.
- A local `_match_on_name(candidates, first, last)` decides: one candidate -> that one;
  the name matches exactly one -> that one; the name matches NONE -> `None`, meaning a
  different relative on the shared contact, so the row becomes a NEW patient; the name
  matches several -> the first.
- All FIVE consumption sites now go through it (email, phone, secondary-as-phone,
  secondary-matches-phone, secondary). Two of those were only found by grepping for the
  old dict names after the first pass — the finding cited two line ranges, and there were
  five call sites.

**On the "matches several" case.** Two same-named people on one address cannot be told
apart from a CSV row. Elsewhere in this workstream the rule is "refuse rather than guess",
but refusing HERE means creating a third record for a person who already exists twice —
strictly worse. Picking the first is a deliberate exception and is commented as one.

**Red run** (`TreatmentPlan/tests/test_csv_import_shared_contact.py`, 4 tests): with the
matcher reduced to last-one-wins, `test_a_new_family_member_on_the_shared_address_is_created`
fails 2 != 3 — the new relative is absorbed into a sibling instead of created. That is the
production defect exactly.

The other three pass both ways, which is correct and worth stating: they guard the
BEHAVIOUR THE FIX MUST NOT BREAK — a genuine duplicate is still deduped, an unshared
contact still matches on contact alone even when the name changed (a rename, not a new
human), and importing Jonathan does not modify Aayan.

**Regression:** TreatmentPlan 824 tests, only the 6 catalogued pre-existing failures.
Pre-commit clean.

---

# #51 — AS BUILT 2026-09-07 (patient panel — the FIRST of six sites)

`name` is a DISPLAY join and is not reversible. "Ken (Kenneth) Judge" splits at the first
space into "Ken" | "(Kenneth) Judge", which is not how it is stored. The editor had only
the joined name, so it guessed the boundary and wrote the guess back — merely OPENING the
field corrupted the record.

**AS BUILT — two independent defences.**
1. `PatientData` now carries `firstName`/`lastName` alongside `name`, and
   `PatientPanelOverviewTab` passes them to the editor, which uses them VERBATIM. The
   columns existed upstream all along and were discarded at the display join.
2. `nameIsUnchanged()` — if the user edited nothing, the editor writes NOTHING. This is
   the defence that matters when the columns are unavailable: the prefill is then a guess,
   and an untouched save would persist the guess over the real values.

Both decisions are pure exported functions (`initialNameFields`, `nameIsUnchanged`) rather
than logic buried in the component. That was not tidiness: the component renders inside a
Radix popover that does not open under jsdom, and a first attempt to drive the UI produced
tests that "passed" only because `onSave` was unreachable. A test that cannot reach the
code proves nothing, so the logic that damaged data was lifted out to where it can be
tested honestly.

**Red run** (`patientNameFields.test.ts`, 12 tests): reverting to always-split with no
unchanged-guard fails **5 of 12** — including "honours a stored FIRST name that contains
spaces" (the case a first-space split can never get right) and "is true for an untouched
form even when the prefill was a GUESS" (the corruption path itself). 12/12 after. `tsc`
clean.

**The remaining five sites were closed the same day — see the block below.**

---

# #52 — FIX REVERTED 2026-09-07 (deliberate product decision)

The fix described above was built, tested (3 attack tests returned 201 against the old
code) and verified safe against production traffic (43/43 real inbound mailboxes resolve,
zero would be dropped). **It has been REVERTED at the user's instruction.** The code is
back to its original state; `messaging/views/message_views.py` and `messaging/tests.py`
are restored and `test_inbound_email_practice_routing.py` is deleted.

**Why:** the trade-off, not the correctness. Making the recipient authoritative removes an
accidental safety net — an unresolvable recipient used to be rescued by the caller's
claimed `practice_id`. Measured future exposure: **6 of 14 practices have no
`PracticePreferredDomain` row** and depend on inbound mail arriving at
`<slug-without-hyphens>@…` (Hitchin Dental Care already has a name/slug divergence). A new
mailbox, a rename, an alias, or a deactivated `task-` address would 400 instead of being
silently misrouted. The user judged that risk to inbound patient email not worth the
security gain.

**WHAT REMAINS OPEN.** This is a CRITICAL finding and it is once again unfixed:
`receive_email_v2` is `AllowAny` with authentication disabled, the relaying worker sends
no secret or signature, and the endpoint trusts `practice_id` from the body. An anonymous
request can still choose any of the 14 practices and own the resulting EmailMessage —
and anything generated from it (Intake, invoice, Task). The audit's original execution
proof stands.

**A middle option, if this is revisited.** The fix had two halves with very different risk:
  * REJECT ON CONFLICT — `practice_id` present, recipient resolves, and they DISAGREE.
    Zero legitimate traffic does this, so it carries no mail-loss risk and closes the
    conflict-forgery route.
  * REJECT WHEN UNRESOLVABLE — this is the half that could drop mail, and it is also the
    half that closes the audit's actual proof (a neutral recipient plus a chosen
    practice_id).
Shipping only the first would be strictly safer than today with no rejection risk. It
would NOT fully close #52.

A shared secret between the Go relay and this endpoint would close it properly with no
recipient dependence at all, but needs a coordinated deploy of both services.

---

# #90 — ✅ FIXED 2026-09-07 (MEDIUM-HIGH) — the AI email-intake parser invented names that became identities

Found by reading the `receive_email_v2` intake branch after asking what that endpoint
actually does.

**What the path is.** An email to an `intake-` / `lead-` / `enquiry-` address has its body
sent to DeepSeek (via OpenRouter) with a prompt asking for `first_name`, `last_name`,
`phone`, `email`, `message` as JSON. `create_intake_from_email` writes the result. So a
patient identity is created from an LLM's reading of arbitrary text.

**The defect.** When no name was found the code wrote `"Unknown"` in one path and
`"Unknown Lead"` in another, and the PROMPT ITSELF instructed the model to return
"Unknown Lead".

That is not a cosmetic label. `Person.resolve` compares candidates on the JOINED name via
`canonical_full_name_key`, and that helper does NOT strip placeholders — so the SPELLING
of an invented name decided identity. Two nameless leads from one address merged when both
said "Unknown Lead" and stayed separate when one said "Unknown".

**Measured on production:** 132 email-sourced Intakes; 13 carrying a placeholder name,
resolving to 6 Persons literally named "Unknown Lead" — a variant none of #68 (two
frontend), #77 (Django marketing) or #89 (Go call agent) covers. 15 of 132 have no last
name. Zero single-letter first names, so the prompt's `j.smith@` rule has not fired yet.

**AS BUILT.** All five sites in `TreatmentPlan/services/email_intake.py` now write a BLANK
name, and the prompt instructs the model to leave both name fields empty rather than
inventing one. Blank matches what the rest of the system already assumes — `identity_guard`,
`split_collapsed_persons` and `identity_defects` all treat a name beginning with "unknown"
as no name at all.

**A CORRECTION to my own first reading.** I initially described the 13-into-6 grouping as
wrongful welding. It is not: those leads share an email address, and grouping repeat
enquiries from one mailbox is correct — it is what channel-based resolution is for. The
real defect is narrower and is what the fix addresses: the placeholder was displayed as a
patient's name, and its arbitrary spelling influenced resolution.

**Red run** (`test_email_intake_no_placeholder_name.py`, 8 tests): restoring the
placeholder fails 4 — blank-name, no-placeholder-written, whitespace-only, and
no-Person-named-with-a-placeholder. Both grouping tests pass either way, deliberately:
same-address leads must still share a Person and different-address leads must still not,
and neither behaviour may change.

**A pre-existing test was pinning the defect.**
`test_email_intake_normalization.py::test_blank_first_name_falls_back_to_unknown` asserted
`first_name == "Unknown"` with no stated reason and protected no consumer — the same shape
as the green test that was protecting #71. Replaced with
`test_blank_first_name_stays_blank`, reason recorded in the test.

**Regression:** TreatmentPlan 832 tests, back to exactly the 6 catalogued pre-existing
failures. Pre-commit clean.

**NOT done — existing rows.** 13 Intakes and 6 Persons still carry "Unknown Lead" in
production. Renaming them is a data repair, and `split_collapsed_persons` already treats
those names as blank, so nothing is broken by leaving them. Worth folding into the next
identity repair run rather than doing blind.

**Related, NOT fixed.** The prompt still tells the model to invent names from email
addresses — `"johnsmith@email.com"` -> `first_name: "Johnsmith"`, `"j.smith@"` ->
`first_name: "J"`. Those are guesses that become resolution keys. Zero rows show the
single-letter shape today, so it is recorded rather than changed; altering extraction
behaviour deserves its own decision.

---

# #88 — AS BUILT 2026-09-07

The call-agent writer split a caller's number by hand: `+1` got one digit, everything else
was ASSUMED to be exactly two. `+44` works by luck; `+234`, `+353`, `+351` and `+358` do
not — `+2348012345678` became country `+23`, number `48012345678`.

**AS BUILT — two changes, because there were two bugs.**

1. `pkg/phone.SplitE164(phoneNumber, defaultRegion)` splits using the real numbering plan
   (the same libphonenumber that backs `CanonicalE164`), so the split and the canonical key
   describe the same number by construction. It returns `("", "")` when it cannot parse,
   and the caller then keeps the raw value rather than inventing a split — a wrong country
   code is worse than none, because it becomes part of the identity.

2. **The deeper bug, found while wiring in the first fix.**
   `resolveOrCreatePersonForCaller` re-canonicalises what it is given using the PRACTICE's
   region, and it was being handed the split-out NATIONAL part. So a Nigerian caller into a
   GB practice had `8012345678` read as if it were British — a different key again from the
   `+2348012345678` that `normalized_phone` stores. It is now passed the caller's ORIGINAL
   number, so resolution and the stored dedup key describe the same human.

Fixing only the split would have left this second mismatch in place, and the finding as
written did not mention it.

**Tests** (`pkg/phone/split_test.go`): the three-digit codes that were corrupted, the
one- and two-digit codes that must not regress, formatting variants, a no-plus number
resolved via the practice region, a refusal case, and — the point of the finding — an
assertion that `SplitE164` and `CanonicalE164` agree on the same number. Go build, vet and
`./pkg/...` tests all pass.

**A FIXTURE TRAP that cost time and is worth remembering.** The first version used
`+447700900123` and `+15551234567`. Those are the Ofcom drama range and the US 555
fictional range; **libphonenumber rejects them as invalid**, so `CanonicalE164` returns
`""` and every assertion failed — while the genuinely tricky cases (Nigeria, Ireland,
Portugal, Finland) passed. The failure pattern was the exact inverse of the bug, which is
what gave it away. Test phone numbers must be valid-FORMAT real ranges; a note to that
effect is in the test file.

**NOT done — existing rows.** Numbers already written with a mangled country code are
unrepaired. `normalized_phone` on those intakes is correct (it was always computed from the
untouched source), so the canonical key is intact and the damage is confined to the
`country_code`/`phone_number` columns plus any Person minted off the bad key. Worth folding
into the next identity repair run rather than fixing blind.

---

# #87 — AS BUILT 2026-09-07

**Verified before fixing** (codex reported it read-only). `lookupUpcomingAppointments`
(`internal/callagent/telnyx/handlers.go:420`) matched `dentally_appointment` on the
caller's number alone — no person id, no patient id, no ambiguity check — and the SELECT
returns `start_time`, `duration`, **`reason`** and `practitioner_name`, which are injected
straight into the assistant's prompt. On a family landline every member's appointments
matched, so the assistant could read a child's appointment REASON aloud to whoever rang.

**AS BUILT.** The rule already existed and the helper was already built: #38 introduced
`ResolveCallerCandidates`, and the NAME lookup (`GetPatientNameByPhone`) uses it via
`SoleCallerCandidate`. This lookup simply never did.

Added `callagent.MayDiscloseToCaller(candidates)` next to `SoleCallerCandidate`, and
`lookupUpcomingAppointments` now returns "none" when it says no:

  * 0 candidates -> ALLOWED. The caller is not in the identity graph at all, so a direct
    phone match cannot disclose someone ELSE, and refusing would break legitimate callers
    who have no Person row yet.
  * 1 candidate  -> ALLOWED. One human on the line; anything on that number is theirs.
  * 2 or more    -> REFUSED. Nothing in a phone call distinguishes them. Same stance as
    `SoleCallerCandidate` for names and Django's `sole_person_for_channel` for messaging.

Written as a NAMED function rather than an inline `len(candidates) > 1` so the rule can be
tested without a database — the existing suite in this package is pure-function, and a
DB harness would have tested the mock more than the decision.

**Red run** (`internal/callagent/appointment_disclosure_test.go`): with the guard
returning `true` unconditionally, the shared-line and five-person-household tests fail.
The unknown-caller and sole-occupant tests pass BOTH ways deliberately — they guard
against over-correction, which is the likelier way to break this feature. 4/4 after; full
`go build ./...`, vet, and the callagent + pkg suites pass.

**Scope note.** This closes the disclosure. It does not scope the query to a patient id —
once the caller is unambiguous the phone match IS theirs, so the extra join buys nothing
today. It would be worth revisiting only if appointments ever need to be read out for a
NAMED family member rather than the caller.

---

# #82 — AS BUILT 2026-09-07

**Verified live before fixing.** `CALL_AGENT_TELEPHONY_PROVIDER` is unset in production, so
it defaults to `twilio` and the routes ARE registered; probing
`/call-agent/media-stream` returns **400, not 404** — the endpoint exists and is served.
`routes.go:149-155` registers it with the comment "NOT authenticated (Twilio needs
access)", and `handlers.go:843` read `practiceId`, `from` and `to` straight from the peer's
`start.customParameters`.

**AS BUILT.** `HandleIncomingCall` now mints a short-lived HS256 token carrying the
practice and embeds it in the TwiML as `<Parameter name="streamToken">`. Twilio echoes
custom parameters back verbatim, so a genuine call arrives with a valid token.
`waitForTwilioStart` takes the practice id, practice name and caller phone FROM THE TOKEN
and refuses the stream outright when it is missing or invalid.

Reuses the existing `SessionTokenManager` (`JWT_SECRET`, 30-minute expiry) rather than
adding a second scheme. `to` is still read from the peer — it is not an identity claim.
`practiceIDInt64` fails closed to 0, which matches no practice, so a malformed id cannot
default to someone.

**Red run** (`internal/callagent/stream_token_test.go`): an empty token, a junk string and
a JWT with a forged signature are all rejected; a token signed with a DIFFERENT secret is
rejected, which is what proves the signature is doing the work rather than the token's
shape. 4/4 pass; `go build ./...` clean.

**WHAT THIS DOES NOT CLOSE — read before assuming #82 is fully dead.**

`/incoming-call` and `/call-status` are themselves unauthenticated, and **no Twilio
signature validation exists anywhere in the service** (`grep -rn "X-Twilio-Signature"`
returns nothing) even though `TWILIO_AUTH_TOKEN` is present in the environment. So an
attacker can still POST to `/incoming-call` and be ISSUED a token for whichever practice
the `To` number maps to.

What the fix does close: connecting DIRECTLY to the open websocket and naming a practice.
That was the shortest path and it is now shut.

The complete fix is Twilio signature validation on both webhooks. It was NOT done here
because it carries the same deploy risk pattern the user declined on #52: a signature
check that disagrees with the proxy's scheme/host would reject every inbound CALL, and the
failure mode is total. That is a decision to take deliberately, with a staged rollout, not
a change to slip in beside a websocket fix.

**Two secrets issues found while doing this, neither yet a finding:**
1. `JWT_SECRET` is COMMITTED in `docker-compose.prod.yml` (lines 127 AND 128 — the same
   variable twice), and its value carries Django's `django-insecure-` prefix. This is the
   key now protecting the stream token, so it is worth rotating out of source control.
2. `TWILIO_AUTH_TOKEN` was printed in plaintext during this investigation by a faulty
   redaction on my part. Treat it as exposed and rotate.

**Pre-existing, unrelated:** `go vet ./internal/...` fails to compile
`internal/email/api/default_addresses_test.go` (wrong argument count to
`ensureDefaultPracticeAddresses`). Confirmed untouched by this work — `git status` shows no
change under `internal/email/api/`.

---

# #82 — COMPLETED 2026-09-07: Twilio webhooks now authenticated (monitor mode)

The earlier AS BUILT closed the media-stream forgery but left `/incoming-call` and
`/call-status` unauthenticated, so a token could still be OBTAINED by anyone who could
reach them. That gap is now closed — carefully.

**AS BUILT.** `internal/callagent/twilio_signature.go` implements Twilio's scheme (full
URL + POST params appended in key order, HMAC-SHA1, base64) and
`TwilioSignatureMiddleware` is wired onto both webhooks in `routes.go`.

**IT SHIPS IN MONITOR MODE AND THAT IS THE POINT.**
`CALL_AGENT_TWILIO_SIGNATURE_MODE` defaults to `monitor`: the signature is computed, the
result is logged, and the request proceeds either way. Nothing is rejected until it is set
to `enforce`.

The reason is the failure shape. Validation reconstructs the URL Twilio signed, which
behind a proxy means trusting `X-Forwarded-Proto` and `Host`. If either disagrees — a
scheme, a port, a trailing slash — EVERY signature fails and EVERY INBOUND CALL is
rejected. That is total, and it looks like an outage rather than a config error. Monitor
mode produces the evidence to switch on enforcement with confidence, at zero risk to live
calls.

Production makes this workable: nginx already forwards `Host $host` and
`X-Forwarded-Proto $scheme` (`/etc/nginx/conf.d/email-service.conf:17,20`), so
reconstruction should be correct — but "should" is exactly what monitor mode exists to
verify.

A missing `TWILIO_AUTH_TOKEN` NEVER rejects, even in enforce mode. A config gap must not
become a call outage.

**Verification — 11 tests, and mutation-tested in BOTH directions:**
- Mutation "signature always matches" -> the forged-signature and tampered-body tests
  fail. The security half works.
- Mutation "signature never matches" -> `TestEnforceModeAcceptsAGenuineSignature` fails,
  and the MONITOR tests still pass. That second half is the safety property proving
  itself: monitor does not reject even when the check is completely broken.
- `TestEnforceModeRejectsATamperedBody` covers the realistic attack — replaying a captured
  signature with the practice-bearing `To` swapped.
- URL reconstruction is tested directly (forwarded proto, query string retained), because
  that is what actually breaks in production rather than the crypto.

`go build ./...` clean; all touched files gofmt-clean (other files in the package have
pre-existing gofmt drift, untouched).

**TO ENABLE ENFORCEMENT — the remaining step, deliberately left to a human.**
1. Deploy with the default (`monitor`).
2. Watch for `Twilio signature did NOT match` under real call traffic. Absent = safe.
3. Set `CALL_AGENT_TWILIO_SIGNATURE_MODE=enforce` and redeploy.
Only after step 3 is #82 fully closed. Until then the endpoints are observed, not
protected.

---

# #68 + #76 + #77 + #86 + #89 — AS BUILT 2026-09-07 (batch: two rules, five sites)

Fixed together because they are two rules, not five problems.

## Rule 1 — a nameless human keeps a BLANK name (#68, #77, #89)

`Person.resolve` compares candidates on the JOINED name, and
`canonical_full_name_key` does NOT strip placeholders. So a literal "Unknown" is a
MATCHING KEY: two unrelated nameless people collide on it, and the arbitrary choice of
wording ("Unknown" vs "Unknown Lead" vs "(unknown)") decides whether they merge.
`identity_guard`, `split_collapsed_persons` and `identity_defects` already treat a name
beginning with "unknown" as no name; blank makes the stored value agree with them.

Producers closed:
- #89 `EmailServiceGo/internal/callagent/storage.go:701` — the Go call agent, writing the
  placeholder into BOTH the Intake and the Person it creates.
- #77 `marketingBroadcast/views/submission_handoffs.py:82` — Django, "(unknown)".
- #68 `hooks/useTreatmentPlanCreation.ts:120` and `components/WorkflowPanel.tsx:340`.

**A THIRD frontend producer was found while fixing the other two** —
`components/openplans/AddOpenPlanDialog.tsx:364`. #68 recorded two. Fixed as well.

Swept the frontend afterwards for remaining `|| 'Unknown'`: every survivor is DISPLAY only
(download filenames, chart categories, comment author names) and none reaches
`Person.resolve`. With #90's parser fix, that is now all five known producers of the 249
"Unknown" Persons measured in production.

## Rule 2 — never take an arbitrary human off an ambiguous set (#76, #86)

- **#86** was the serious one, and WORSE than reported. `ResolveCallerCandidates` deduped
  by `person_id` ALONE, on the assumption that one Person is one human. Fused Persons hold
  several, so a fused Person contributed exactly ONE candidate — and `SoleCallerCandidate`
  then reported the line as unambiguous. That does not merely pick the wrong name: it
  **silently defeats the #38 shared-line guard in the very case it exists for**. Now keyed
  on person + canonical name, which keeps the original intent (one human appearing as both
  a patient and an intake is still counted once) while letting genuinely different humans
  show up as the several candidates they are.
- **#76** `messaging/serializers.py:481` — `_get_viewer_patient`'s three guards each only
  run when their matching field is populated, so a session with an empty
  `participant_phone_number` fell past all of them to `patients[0]`. Now returns a patient
  only when there is exactly one.

## Verification

Go: 5 new tests in `internal/callagent/fused_person_candidates_test.go` covering the fused
Person, the same human across two tables (the behaviour the original dedupe was FOR, which
must not regress), a differently-split name resolving to one human, nameless rows being
skipped — which matters now that #89 writes blanks — and two separate Persons still
blocking disclosure. `go build ./...` clean, callagent + pkg suites pass.

Django: `messaging` + `marketingBroadcast` 894 tests, OK. Pre-commit clean.

Frontend: 60 tests pass across the touched areas; `tsc -p tsconfig.app.json` clean.

**A TESTING TRAP worth recording.** Running vitest against `src/hooks src/components/patients`
reported **136 failures across 310 files** — alarming, and none of it mine. Vitest resolves
those paths inside `.worktrees/` and `.claude/worktrees/` too, so it was running five stale
copies of the repo (the known DEF-37-01 pollution). Scoped with
`--exclude '**/.worktrees/**'` the real answer is 60/60. A raw failure count from this repo
is meaningless without that exclusion.

---

# #56 + #59 + #60 + #83 + #84 — AS BUILT 2026-09-07 (batch)

## #83 (CRITICAL) — the execution's practice now wins

**Verified before fixing.** `service.go:585` injects the workflow's OWN practice — the
one ownership-checked when the node is saved. `query_records.go:83` preferred
`config["practice_id"]`, a number the workflow's author types into a node. Paired with the
`http_request` action, a practice-16 user could read practice 13's patients and POST them
anywhere. Only this one action had the override.

Now `ResolveExecutionPracticeID`: input wins; a config value that CONTRADICTS it is
REFUSED rather than silently preferred; config alone still works for invocations carrying
no context.

**The rule lives in production code, not the test.** My first version defined its own copy
of the logic inside the test file — it would have passed even if `query_records.go` were
wrong. Moved out and the test now calls the real function. Red run with the old precedence
restored: `TestAContradictingNodeConfigIsRefused` fails.

## #84 (HIGH) — the Dentally import no longer overwrites a guess

`fallbackFindPatient` runs only when the incoming patient was NOT found by Dentally id,
and whatever it returns is then OVERWRITTEN with that patient's `meta_data` and
`date_of_birth` (`service.go:1017`). It used `Limit(1)` with no ordering and no ambiguity
check: father and son both "John Smith" on one family email, and whichever row the
database returned first got stamped with the other's identity and DOB.

Both branches (name+email, name+phone) now take `Limit(2)`: exactly one match is evidence,
two or more is not and the importer creates a new patient instead. A duplicate can be
merged later; an overwritten father cannot be un-overwritten.

## #56 + #59 + #60 — one helper for three globally-writable staff FKs

All three exposed `PrimaryKeyRelatedField(queryset=User.objects.all())` or a global
`User.objects.get`. The record was practice-scoped; the person attached to it was not.
Production proof was identical in all three — User 113, whose only practice is 20,
validated against practice-16 records.

`utils.practice_mixins.assert_user_in_practice` now backs all of them:
- #60 `TreatmentPlan/serializers/patient.py` — both preferred clinicians, in `validate()`.
- #59 `Invoices/serializers.py` — `validate_allocated_practitioner` on the update
  serializer.
- #56 `patient_accounts/views.py` — `_resolve_practitioner` takes the practice and filters
  on it; an unknown or foreign id resolves to None, which is the outcome the caller
  already handled.

`None` stays allowed: these fields are optional and clearing one is legitimate. A user in
BOTH practices is accepted — real staff work across a head/child group, and membership is
the test, not exclusivity.

**Red run:** with the guard removed, 4 of 9 fail — the direct refusal, the field-naming
assertion, and both serializer-level rejections. The "own clinician still accepted" and
"clearing still allowed" tests pass either way by design; they guard against
over-correction.

## Verification

Go: `go build ./...` clean; workflows, callagent and pkg suites pass.
Django: `TreatmentPlan.tests.test_practitioner_practice_scoping` 9/9.

**Invoices needed a baseline comparison rather than a glance.** Running it showed 6
failures / 4 errors, which looked alarming next to a fix in `Invoices/serializers.py`.
Reverting my change to HEAD and re-running gave **the identical 6/4** — all
`relation "productAnalytics_usageevent" does not exist`, the known missing-migrations trap.
Not mine. A failure count next to a changed file is not evidence until it is compared.

Pre-commit clean on all touched files.

# #53 + #57 + #58 + #61 + #63 — AS BUILT 2026-09-07 (batch)

Every one of these was proven with a RED run first — the test failing against the
unfixed code, showing the exact wrong human or the exact accepted foreign record.

## #61 (MEDIUM) — the bounce now suppresses the person the mail was sent to

`marketingBroadcast/views/webhook_views.py`. The send path already records exactly who
each message went to: `BroadcastRecipient` carries both `person` and `email_message`.
The webhook discarded that, looked the address up as a channel, and took `.first()`.

Two new helpers. `_recipient_person_for_message` resolves through the
`BroadcastRecipient` row for this `provider_message_id` — the same lookup
`_broadcast_for_message` a few lines above already uses — scoped on BOTH
`campaign__practice` and `email_message__practice`. `_person_for_event` prefers it and
falls back to `_find_person_by_email`, which no longer ends in `.first()`: it resolves
the `ContactChannel` and hands it to `sole_person_for_channel`, so a shared mailbox with
no recipient row now changes nobody's consent instead of an arbitrary relative's.

Applied at both call sites — bounce/complaint AND Postmark's SubscriptionChange, which
had the same defect in reverse: one relative clicks unsubscribe, a different relative
gets unsubscribed. `_handle_subscription_change` now takes `message_id`.

**Red run:** 5 of 7 fail on the unfixed code — Jude Peacock suppressed by Rebecca's
bounce, Rebecca left subscribed. Green after. Full `marketingBroadcast` suite: **723
tests, OK**.

## #53 (HIGH) — the template SEND action is scoped like its PREVIEW sibling

`messaging/views/template_views.py`. `preview` resolved `treatment_plan_id` /
`patient_id` against the caller's practice; `send` used `TreatmentPlan.objects.get(id=)`
and `Patient.objects.get(id=)` with no scope at all, and `SendTemplateSerializer` only
checked that exactly one id was supplied.

The practice resolution (`template.practice or self.get_user_practice()`) MOVED up,
above the lookups instead of below them, and both lookups became
`.filter(id=..., practice=practice).first()` → 404 on a foreign row.

**Red run:** the foreign patient was accepted with **200 and the mailer was called** with
their address. The mailer is mocked in every test in the file — a regression there would
otherwise email a real person at another practice.

## #57 (MEDIUM-HIGH) — consent context must be the signer's own

`Documents/views/signing_views.py`. `patient_id` and the template ids were scoped;
`appointment_id` and `treatment_plan_id` went straight onto the SigningRequest.

New `_validate_consent_context(practice, patient, treatment_plan_id, appointment_id)`,
called before `_build_signing_request`. Both conditions are checked, not one: same
practice AND same patient. Practice alone would still let one patient's consent cite
their neighbour's plan — two patients of one practice are still two different humans,
and a consent form naming the wrong one's treatment is a clinical record about the wrong
person.

**Red run:** foreign plan and foreign appointment both accepted with **201**, and the
SigningRequest was saved carrying the foreign context. The neighbour case (same practice,
different patient) was also accepted — which is why practice scoping alone was not the
fix.

## #58 (MEDIUM) — the legacy conversation reader compares case-insensitively

`messaging/views/message_views.py`, four sites. The route lowercases the requested
address, then compared it exactly against stored mixed-case values, so a valid patient's
thread returned 200 with zero messages and `patient_id=None`. All four now use
`__iexact`: the `from_treatment_path`/`to_patient` Q pair, the Patient lookup, and the
two in `_get_treatment_plan_and_patient_ids` (Patient and
`non_registered_patient__email`).

## #63 (MEDIUM, feature disabled) — booking prefill refuses an ambiguous mailbox

`onlineBooking/views.py::_build_verified_response`. `.first()` handed whichever relative
the database ordered first — name, surname and phone — to whoever proved control of the
mailbox. Nine Taylors share one address in production.

Now `[:2]` and "exactly one candidate or nobody", the same rule
`sole_person_for_channel` applies. Proving control of a mailbox proves you can read its
mail; it does not say WHICH human is booking. An ambiguous mailbox prefills nothing, the
person types their own name, and that is also what lets the booking be filed against the
right human rather than the relative we guessed.

**Upcoming bookings deliberately unchanged** — still keyed on the address. Those
confirmations were emailed to this mailbox and the payload names no human: only service,
time and practitioner. Restricting it would remove nothing the mailbox has not already
received.

**Red run:** Twyla-Storie Taylor's name and phone returned for a two-Taylor mailbox and
for the nine-Taylor one.

**Pre-existing failures seen while running these suites, NOT caused by this batch:**
`onlineBooking.tests` has 3 failures / 5 errors, all Stripe Connect —
`PracticeStripeAccount() got unexpected keyword arguments: 'stripe_account_id'`, a test
that has drifted from the model. Nothing in it touches identity.

# #52 — AS BUILT (attempt 2, SHIPPED) 2026-09-07

Attempt 1 above made the RECIPIENT authoritative. It worked, but it changed routing
behaviour, and 6 of 14 practices have no `PracticePreferredDomain` row — a new mailbox or
a slug rename would then have caused real inbound mail to be REJECTED. Reverted by
product decision: the rejection risk was judged worse than the hole.

The shipped fix proves WHO is calling instead of second-guessing WHAT they claim, using
the mechanism every other Go↔Django internal callback already uses:
`X-Workflow-Secret` / `WORKFLOW_SERVICE_SECRET`, exactly as
`spam_views.receive_spam_verdict` does. `receive_email_v2` returns **401** without a
matching secret. Routing is untouched, so no legitimate delivery can be refused for
having an unmapped recipient — the whole reason for the change of approach.

Go side: `internal/email/worker/inbound/queue_processor.go` sends the header via
`workflowSecret()`, which logs loudly when the variable is unset rather than failing
silently.

**Tests:** `messaging/test_inbound_email_caller_auth.py`, 5/5 — no secret, wrong secret
and empty secret all 401; the genuine relay is accepted; and an authenticated delivery to
an UNMAPPED recipient still goes through, which is the property attempt 1 could not offer.

**DEPLOYMENT PRECONDITION — still outstanding.** `WORKFLOW_SERVICE_SECRET` must be set on
the `email-worker` service before this ships (it was missing; now added to both
`docker-compose.prod.yml` and `docker-compose.dev.yml`), and **Go must be deployed before
Django** — otherwise the relay sends no header and every inbound email 401s.

# #51 — AS BUILT 2026-09-07 (the remaining five sites) — now FIXED

The patient-panel fix above proved the rule; these five needed the STORED COLUMNS to
actually reach them, which in three cases meant fixing the pipeline rather than the
editor.

**One shared copy of the rule.** `initialNameFields` / `nameIsUnchanged` moved out of
`PatientEditableFields.tsx` into `src/utils/nameFields.ts`, re-exported from their old
home so nothing broke. Six call sites now share one implementation instead of six
first-space splits.

**The pipeline, following the DATA FLOW BREADCRUMB in `useIntakeCaching.ts`:**

1. `TreatmentPlan/serializers/treatment_plan.py::get_patient` now returns `first_name`
   and `last_name` next to the joined `name`, for BOTH the registered and
   non-registered branches. It only ever returned the join, so the frontend had nothing
   better to use — this is the root of three of the five sites.
2. `useIntakeCaching.transformIntakeData` carries `firstName`/`lastName` instead of
   joining and discarding them; declared on `intake/types.ts::PatientData`.
3. `IntakeTable.transformToIntakeLead` passes them on; `IntakeLead` declares them.

**The five sites:**

- `IntakeTable.tsx` edit modal — `initialNameFields(lead.name, lead.firstName,
  lead.lastName)`.
- `CustomJourneyTable.tsx` edit modal — same; its rows are already the intake
  `PatientData`, so it needed no new plumbing once the transform carried the columns.
- `WorkflowPanel.tsx` — `splitPatientName` now takes the stored columns, threaded
  through `LetterAssistant` and `DocumentAssistant` from `Conpact3`'s
  `selectedPatientRecord`. Identity was already safe here (#65 sends `from_patient_id` /
  `patient_id`, and `NurtureSerializer.create` carries the Person rather than
  re-resolving on the name) — but the journey record itself was still being WRITTEN with
  a false boundary, which is the same damage class.
- `usePatientPanelController.ts` — two sites: the call-log payload and the
  create-treatment-plan navigation state.
- `NotesEditor.tsx` — the plan-load path now prefers the serializer's new columns. The
  URL-parameter path genuinely has only a joined name and keeps the split as its
  documented fallback; there is nothing better available there.

**Red run** (`src/hooks/intakeNameColumns.test.ts`, 5 tests): removing the two carried
columns from the transform fails **4 of 5**. One assertion was rewritten after it passed
too easily — `expect(row.firstName).not.toBe("Ken")` is also satisfied by the column
being ABSENT, which is the bug itself, so it now asserts the real value.
`patientNameFields.test.ts` still 12/12 after the move.

**Typecheck:** `tsc -p` reports 789 pre-existing errors in this project (the committed
`tsconfig.app.json` also fails outright on `ignoreDeprecations: "6.0"` under the current
TS, so a copy with that line removed was used). None of the reported errors are in the
lines this change touched.

# #85 — AS BUILT 2026-09-07

`internal/callagent/storage.go::linkPersonChannel` now names `role` and `updated_at`.
Both are NOT NULL with no database default — Django's `blank=True, default=""` and
`auto_now=True` are Python-only and emit no DDL default, the same drift class
`internal/db/schemaguard.go` exists to catch, and it already listed both columns for
this table.

`role` is written BLANK, not guessed: the call agent knows the number belongs to the
caller, not whether it is their mobile, home or work line.

**The error is no longer discarded.** It is logged with the person and channel ids and
then swallowed deliberately — a failed link must not abort a live call, but the silence
is half of why this ran broken for as long as it did.

**Red run** (`internal/callagent/person_channel_insert_test.go`, 4 tests): the column
test fails against the old statement and PRINTS it, because the test reads the SQL the
production function actually emits (a sqlmock `QueryMatcherFunc` records it) rather than
asserting against a copy. The other three pin the ON CONFLICT clause, the missing-id
early return, and that a failing insert is logged rather than taken as a crash.
`go test ./internal/callagent/` green; `go build ./...` clean.

**Pre-existing, unrelated:** `go test ./...` fails to BUILD
`internal/email/api/default_addresses_test.go` — `ensureDefaultPracticeAddresses` gained
a parameter and that test was not updated (commit cbc62c3). Nothing in this workstream
touches it.

---

# SWEEP CLOSED 2026-09-07 — every finding is now marked fixed

All findings #1–#90 carry a ✅ FIXED status. The two that were still open at the start of
today are closed above: **#51** (the five remaining name-split sites) and **#85**. **#52**
had a stale OPEN/REVERTED heading describing only its first attempt; the shipped
caller-authentication version is green and the heading is corrected.

**Not code — still owed by a human before or after deploy:**

- `WORKFLOW_SERVICE_SECRET` must exist on `email-worker`, and **Go deploys before
  Django**, or #52 401s every inbound email.
- `CALL_AGENT_TWILIO_SIGNATURE_MODE` ships as `monitor`. Flip to `enforce` only after the
  logs show sustained 100% signature matches (#82).
- Rotate `JWT_SECRET` (committed in the compose file, twice) and `TWILIO_AUTH_TOKEN`
  (exposed during this work by a redaction pattern of mine that did not match).
- `todo/2026-09-07-repair-mangled-country-codes.md` — the #88 DATA repair. The code is
  fixed so no new rows are damaged; the existing ones still need the dry-run / verify /
  fixed-point treatment as part of the next identity repair run.
- Two things observed but never written up as findings: spam/cold email creating Person
  records via intake addresses, and `anonymous` (a withheld number) being treated as an
  identity — 6 Persons in practice 16.

# VERIFICATION SWEEP 2026-09-07 (after the sweep was declared closed)

Re-checked every finding rather than trusting the headings: enumerated #1–#90, confirmed
each carries a status, grepped all three codebases for the audit markers, and ran every
test written for this workstream. **Two real problems surfaced, both now fixed.**

## V1 — the #9 structural guard was silently disarmed by the #77 fix

`test_resolve_callers_pass_dob` requires every `Person.resolve()` caller either to pass
`dob=` or to carry a `# no-dob:` comment explaining why. It scans the ~8 lines directly
above the call.

`marketingBroadcast/views/submission_handoffs.py` HAD that exemption. The #77 fix then
inserted an eight-line explanation BETWEEN the comment and the call, pushing the
exemption to eleven lines away — outside the window. The guard reported the call site as
an unexplained offender.

Nothing about the behaviour was wrong: a marketing form genuinely collects no date of
birth. But a guard that reports a false offender is a guard people learn to ignore, and
this one exists to stop a father and son on one family phone collapsing into one Person.

The `# no-dob:` note now sits DIRECTLY above the call, with a line saying why it must
stay there. 2/2 green.

**This is the second time a fix in this workstream damaged a neighbouring guard, and
both times a test caught it.** Worth remembering when the next comment block goes in
above a call site.

## V2 — my own #51 threading could write a BLANK name

`initialNameFields` treats a supplied `""` as authoritative, and it MUST: a stored
surname really can be empty (Madonna), and the editor has to show that verbatim instead
of re-splitting the display name over it.

But `NotesEditor` sets `first_name`/`last_name` to `""` on a placeholder record whose
display name is "Unknown Patient". Threading those raw into `WorkflowPanel` (which I did
when closing the last five #51 sites) would make it POST a blank name where it used to
post a split one — trading a wrong name for no name.

New `suppliedNameColumns()`: a record with NO name at all supplies nothing and the caller
falls back to the split; a record with any real value supplies both, so Madonna's empty
surname still survives. Applied at the five WRITE sites (Conpact3 → WorkflowPanel,
usePatientPanelController ×2, IntakeTable, CustomJourneyTable). The panel EDITOR keeps
the raw columns, which is correct — it is displaying stored values, and `nameIsUnchanged`
already stops an untouched form from saving.

**Red run:** stubbing `suppliedNameColumns` to return its input unchanged fails 3 of the
6 new tests, including "a nameless record falls back to the split instead of writing
blanks". 18/18 in `patientNameFields.test.ts` after.

## What the sweep confirmed

- **#1–#90 all carry a fixed status.** #1–#16 use a `## N.` heading rather than `## #N`,
  which is why a `#N`-shaped grep misses them.
- **Four findings carry no code marker, all legitimately:** #29 was fixed as part of #10
  (verified: `Documents/utils/notifications.py::_normalise_phone` now delegates to
  `canonical_phone_e164`); #79 is a management command; #80 is the `outbound_sandbox`
  interceptor (verified: blocks the Go email-service by PATH, so a staging URL cannot
  slip past); #81 rides with #80.
- **#79 is code-fixed but data-pending** — live code writes a phone number instead of
  guessing a name; the historical mislabels need `repair_mislabelled_sessions`, which is
  built and dry-run-by-default. Same shape as the #88 country-code repair.

## Failures seen during the sweep that are NOT this workstream

- `TreatmentPlan.tests.test_custom_journey_webhook.test_compat03_intake_webhook_baseline`
  — a SHA-256 guard on `intake_views.py`. **Proven to come from commit `08de331f` "Add
  real upcoming-appointment data to intake list badge"**: that commit's parent hashes to
  the recorded baseline and the commit itself does not. The file is committed and
  unmodified in the working tree, so this is not an audit change. Deliberately NOT
  rebaselined here — the guard's own message asks for the change to be recorded in the
  relevant SUMMARY.md, which belongs to whoever made it.
- `compliance.tests.test_legacy` (3 errors) — `Missing staticfiles manifest entry for
  'admin/css/base.css'`. An environment gap (no collectstatic), and the `compliance` app
  appears nowhere in this audit; that file entered the run only because it contains a
  `#`-and-digits string.
- `onlineBooking.tests` Stripe Connect (3 failures / 5 errors) and the Go
  `internal/email/api` test BUILD break — both reproduce with this work stashed.

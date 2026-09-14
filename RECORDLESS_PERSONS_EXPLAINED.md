# The 25,087 record-less Persons — explained and proven

**Status:** fully traced against an untouched production copy, and the housekeeping
rehearsed on a repaired copy — 2026-09-05.
**Verdict: migration debt from the 2026-07-27 re-import. Not identity damage.**

Earlier notes called this "group-practice migration debt" as an assertion. This
document is the proof, and it corrects two wrong intermediate guesses along the way.

---

## 1. What they are

A `Person` holding **no** Patient, Intake or Nurture record. On the 2026-09-04
production dump: **25,087**, of which **25,064 were created on a single day —
2026-07-12**.

They are concentrated almost entirely in two practices:

| Practice | Record-less Persons | Live patients | Total Persons |
|---|---|---|---|
| 26 Hacton Dental Care | 14,324 | 6,676 | 20,088 |
| 27 Church View Dental Clinic | 10,627 | 11,782 | 20,152 |
| everything else combined | ~136 | — | — |

## 2. The proof

### Step 1 — they were NOT born empty

`backfill_persons_households` (the 2026-07-12 backfill) builds each Person **from**
existing Patient/Intake/Nurture records grouped by name. **Every Person it created
had at least one record at the moment of creation.**

So a record-less Person means its records were later deleted or re-pointed. That
single fact rules out "the backfill just made spare Persons".

### Step 2 — the records were deleted and re-created

| | |
|---|---|
| Patients in 26+27 created **2026-07-27** | 11,778 + 6,660 |
| Patients in 26+27 created **before** 2026-07-27 | **16** |
| Patients re-imported into 24+26+27 on 2026-07-27 | **24,400** |
| Stranded Persons in 26+27 | **24,951** |

**Every current patient in practices 24, 26 and 27 was created on 2026-07-27.**
Sixteen rows survive from before that date, out of ~25,000. The pre-existing
patient population was wiped and re-imported wholesale, and the stranded-Person
count matches the re-imported patient count to within 2%.

2026-07-27 is the same 24,555-patient bulk import that produced defect D — see
`RUNBOOK_COLLAPSED_PERSON_REPAIR.md` §2.

### Step 3 — the humans still exist

Working outward from the 25,087:

| Evidence the human still exists | Count |
|---|---|
| A live Person with the same name holds patients | **19,060** |
| Otherwise: shares a contact channel with a live patient | **3,672** |
| Otherwise: present in `recall_patient` / `daylist_patient` | **1,860** |
| **No local trace at all** | **496** |

**24,591 of 25,087 (98%) are demonstrably still in the system.** And for 12,998 of
them the replacement Person was created *after* the stranded one — the signature of
a re-import minting a fresh identity rather than reusing the existing one.

## 3. About the 496

402 are Church View (27), 65 Hacton (26). All created 2026-07-12. Mostly ordinary
names; a few are junk (one has an email address in the `first_name` field).

**"No local trace" is NOT evidence they were lost.** The Dentally-side mirrors are
badly incomplete — `recall_patient` holds **756 rows for practice 26's 6,676
patients** (11% coverage). A person missing from a table that only covers a tenth
of the practice tells you nothing.

The honest position: these 496 were patients before 2026-07-27 and were not
re-imported under the same name. That is consistent with people who left the
practice, were archived in Dentally, or came back with a different name spelling.
Confirming which needs a **read-only Dentally API lookup**, which has not been done.

## 4. Two guesses this investigation killed

Recording these because both sounded right and both were wrong:

1. **"They're duplicates of live Persons in the same practice."** Only ~2% have a
   same-practice name twin (221 of 14,324; 288 of 10,627). The twins are in the
   *other* practices of the group — which is what a consolidation produces, and is
   why the same-practice test was the wrong test.

2. **"They were invisible to dedup because they had no channels."** The opposite is
   true: **24,980 of 25,087 (99.6%) DO have channels.** Meanwhile 4,688 Persons
   created the same day that *do* hold patients have none.

## 5. What this means for the cleanup

- They are **inert**: no patient, intake or nurture records; their wrong channel
  links were released by the defect-C repair; none of the five defects (A, B, C, D,
  D2) touch them.
- Deleting them is **not** required to fix anything. It is housekeeping.
- Per the practice-scoping rule, a Person in Hacton and one in SmileHQ for the same
  human are **not** duplicates to merge — they are separate practice-scoped
  identities, one of which is now empty.
- **Do not bulk-delete before resolving the 496.** They are the only population
  where "empty Person" might mean "patient we no longer have".


---

## 6. The housekeeping — rehearsed 2026-09-05

`TreatmentPlan/management/commands/purge_recordless_persons.py`. Dry run by
default; `--apply` writes. **Deletion is not reversible — take a backup first.**

### What it deletes, and what it refuses to

A record-less Person is deleted only when the human is **provably still in the
system**:

| Verdict | Count | Meaning |
|---|---|---|
| `DELETE_LIVE_NAME_TWIN` | 21,592 | a live Person of the same name holds patients |
| `DELETE_PRESENT_IN_DENTALLY_MIRROR` | 195 | name present in recall/daylist mirrors |
| `DELETE_SHARES_CHANNEL_WITH_LIVE_PATIENT` | 48 | shares a contact channel with a live patient |
| `KEEP_HAS_HISTORY` | **99** | carries a Note / NoteHistory / Activity / ActivityLog |
| `KEEP_UNEXPLAINED` | **106** | no evidence the human is still here — see §3 |

Also kept: anything in a `merged_into` chain or referenced by
ContactMergeDismissal / ContactMergeLog. Record-emptiness is re-checked **inside
the transaction**, so a Person that gains a record mid-run is skipped, not deleted.

### Result on the repaired production copy

| table | before | after | delta |
|---|---|---|---|
| **Person** | 84,050 | 62,206 | **−21,844** |
| MarketingPatientProfile | 83,700 | 61,856 | −21,844 |
| PersonChannel | 143,238 | 105,983 | −37,255 |
| Patient / Intake / Nurture | 60,326 / 4,236 / 617 | **unchanged** | 0 |
| Note / NoteHistory | 4,326 / 8,588 | **unchanged** | 0 |
| ContactChannel | 114,886 | **unchanged** | 0 |
| Household | 15,890 | **unchanged** | 0 |

- **0 orphaned `person` FKs** across all 9 referencing tables
- **all five defects still measure 0** — A, B, C, D, D2
- 205 record-less Persons remain by design (99 history + 106 unexplained)

MarketingPatientProfile falls exactly 1:1 with Person. It holds derived Dentally
analytics (appointment counts, spend, dates) and **no consent** — consent lives in
`marketingBroadcast_marketingconsent`, which has **zero** rows for these Persons.

### A bug caught in this command before it ran

The first version looked the mirrors up as apps `"recall"` and `"daylist"`. Both
models actually live in `dentallyIntegration`, so both lookups raised
`LookupError`, were silently swallowed by a `continue`, and contributed **zero**
names — printing `names present in Dentally-side mirrors: 0`.

The effect was not random: 195 Persons who ARE in Dentally were being classified
`KEEP_UNEXPLAINED`. Conservative here, but the same silent-`continue` pattern in a
rule that *permits* deletion would have deleted too much. The command now raises
rather than continuing when a mirror lookup fails or returns nothing.

### Reading a Dentally API key — do it through the model, not Fernet

`DentallyIntegration.api_key` is a **property that already decrypts**
(`dentallyIntegration/models.py`); the ciphertext lives in `_api_key_encrypted`,
whose db_column is `api_key`. Passing `integration.api_key` to `Fernet.decrypt()`
therefore double-decrypts and always raises `InvalidToken` — which looks exactly
like a wrong key and cost this investigation an hour of chasing the wrong thing.

```python
os.environ["DENTALLY_ENCRYPTION_KEY"] = "<key from the prod .env>"
key = DentallyIntegration.objects.get(practice_id=24).api_key   # already plaintext
svc = DentallyAPIService(Practice.objects.get(id=24))           # use this, not requests
```

The head practice (24) key covers the whole group; Dentally reports **35,227
patients** across it.

### The `search` parameter is silently ignored

`get_patients(search=...)` returns HTTP 200 and **the same unfiltered first page**.
Probed with `search="ZZZNOSUCHNAMEZZZQQ"`: 5 rows returned, `meta.total` 35,227 —
identical to the unfiltered call.

**Never look a patient up by name through this API and treat an empty/!empty result
as an answer.** Page the full list and match locally.


---

## 7. The unexplained residue — RESOLVED against the Dentally API

Paged the full patient list (35,227 across the group; 21,084 active / 14,143
inactive) and matched locally, because `search` is ignored.

| Verdict | Count | What it means |
|---|---|---|
| **In Dentally, ACTIVE** | **84** | a real, current patient — we simply hold no `TreatmentPlan_patient` row |
| In Dentally, inactive/archived | 12 | left the practice; correctly absent |
| **Not in Dentally by exact name** | **39** | see §7.1 — **none of them are junk** |

**No patient was lost.** Every genuine human in the residue is still in Dentally.

### 7.1 The 39 "not in Dentally" — a claim that did not survive checking

They were first written up here as "junk rows, safe to delete". **That was wrong on
both counts**, and per-row evidence is what showed it:

**29 carry an ActivityLog row** — every one an `"Email sent: …"` entry, practices
13 and 21, Person ids 151xxx. These are Persons created from **email
correspondence**, not from Dentally patients. They are correctly absent from a
patient list, they hold real history, and the purge already keeps them
(`KEEP_HAS_HISTORY`).

**10 have real names** — 5 people, each appearing once in practice 26 and once in
27. They failed the exact-name match because the LOCAL name is misspelled:

| Local name | Dentally | Active |
|---|---|---|
| Svetlana **Kordila** | Svetlana **Hordila** | yes |
| James **Frances** | James **Francis** | yes |
| Cristina **Teocam** | Cristina **Teocan** | yes |
| Zayd **Amhad** | Zayd **Ahmad** | yes |
| Pankaj **Sidreia** | Pankaj **Sadheura** (the only "Pankaj" in Dentally) | yes |

**All five are active patients.** Exact matching alone would have condemned ten
rows belonging to real, current patients.

**Deletable junk in the residue: zero.** The purge command's own dry run confirms
it: of these 39, **0 are marked for deletion** (29 `KEEP_HAS_HISTORY`,
10 `KEEP_UNEXPLAINED`).

### 7.2 Two findings worth their own tickets

1. **84 active Dentally patients have no local patient record.** The app cannot
   see people the practice still treats. A sync gap, not identity damage.
2. **Local names are garbled against Dentally** on at least these 5. Worth a
   fuzzy-match sweep — an exact-match cleanup would silently destroy real rows.

### The one thing worth acting on

The **84 active Dentally patients with no local patient record** are a *sync gap*,
not identity damage — the app cannot see people the practice still treats. Worth
raising separately from this workstream.

### Housekeeping consequence

**Nothing further to delete.** All 205 Persons the purge kept are justified:
99 carry history, 106 have no proof the human is gone. The residue investigation
found zero additional deletable rows.

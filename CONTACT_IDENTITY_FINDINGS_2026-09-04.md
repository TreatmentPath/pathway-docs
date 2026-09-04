# Contact identity — findings, 2026-09-04

Living document. Everything here is **measured**, not inferred, unless a line says
otherwise. DEV and PROD figures are given separately; where they agree, the problem is
real in production at full scale.

---

## 0. The one-paragraph summary

The system decides "who is this?" by looking a contact detail up in a registry
(`ContactChannel` → `PersonChannel` → `Person`). An email or phone written on a patient's
file is **not** in that registry unless something puts it there. Three separate things go
wrong: details never registered, details registered in the wrong practice, and — most
seriously — **details registered against the wrong human**. The last one means the app
shows a patient under someone else's name.

---

## 1. Scale (PROD, read-only via `sim_readonly`)

| | PROD | DEV |
|---|---|---|
| Patients | 60,326 | 60,506 |
| Persons | 83,697 | — |
| Registered contact details | 108,612 | — |
| Person↔detail links | 136,671 | — |

DEV mirrors PROD closely enough to develop against (see §2 — the two differ by ≤6 rows on
every measure taken).

---

## 2. The three defects, measured

| Defect | PROD | DEV |
|---|---|---|
| **A.** Patient's own **email** not registered against them | **3,547** | 3,549 |
| **B.** Patient's own **phone** not registered against them | **7,488** | 7,486 |
| **C.** A contact detail registered against the **WRONG person** | **~5,460** (see correction) | **5,454** |

> **CORRECTION 2026-09-04.** C was first reported as 12,452. That figure counted JOIN ROWS:
> one channel matching several owner records was counted once per match. The deduplicated
> figures are **5,454 distinct (channel, person) links** across **3,572 distinct channels** —
> less than half the original claim. 5,454 is the repair unit. Caught by
> `contact_identity_audit`, which uses DISTINCT, disagreeing with the ad-hoc query.

**C is the serious one** and was found last. A and B mean "we won't recognise them". C
means "we will recognise them **as somebody else**".

> Phone figures reconstruct the canonical number in SQL rather than running the app's real
> formatter, so treat B and C's phone halves as ±a few percent. The email figures are exact.

### What C is NOT

Household sharing is deliberate and correct: a child carrying a parent's email, four family
members on one phone. The raw count of "detail attached to someone whose own record doesn't
carry it" is ~53,000 — but almost all of that is legitimate household behaviour. The 12,452
figure **excludes** anyone in the same household as the detail's true owner, so it is
cross-household attribution only.

---

## 2b. DEFECT D — whole families collapsed into ONE Person (found last, worst)

**PROD, measured.** Persons holding records with 2+ distinct human names:

| Distinct names on one Person | Persons |
|---|---|
| 6 | 2 |
| 5 | 38 |
| 4 | 194 |
| 3 | 583 |
| 2 | 1,573 |

The 2-name cases are ambiguous (a maiden/married name is one human — e.g. Person 75013
holds both "jo lee" and "jo o'brien"). **817 Persons hold 3 or more distinct names**, and
those are not name changes.

### Verified example — Person 152744, "John Geary"

```
patient  first    last    dob          phone         email
135585   Micaela  Geary   1984-01-04   7368557184    gearymicaela@gmail.com
131893   John     Geary   1985-02-25   7887705702    gearymicaela@gmail.com
133908   Beau     Geary   2012-04-26   7368557184    gearymicaela@gmail.com
133907   Hallie   Geary   2014-01-10   7368557184    gearymicaela@gmail.com
133906   Johnny   Geary   2018-06-02   7368557184    gearymicaela@gmail.com
133905   Gigi     Geary   2021-02-15   (none)        gearymicaela@gmail.com
```

**Six humans — two adults and four children, six different birth dates — are ONE Person
record.** All created 2026-07-27. The Person row itself has `dob = NULL`.

Other confirmed cases: Person 90301 (six El-Boukas), 75759 (five Cordells), 76214 (a
blended Jones / Clifford-Tucker family), 77083 (five unrelated-looking surnames).

### Why the current design did NOT do this

`Person.resolve` reuses a Person only on an EXACT name match. "Micaela Geary" and "Beau
Geary" do not match, so today's code would create separate Persons in a shared Household —
the correct outcome. This damage therefore predates the current resolution logic. The Go
source documents the previous behaviour it replaced:

> *"SELECT person_id FROM TreatmentPlan_personchannel WHERE channel_id IN (...) LIMIT 1 —
> no name check, no DOB check, no merged_into filter and no ORDER BY. Any patient sharing a
> family phone was absorbed into whichever Person the database happened to hand back."*

That produces exactly this. **INFERRED, not proven** — the rows carry no marker saying
which code wrote them.

Note the DOB defence cannot help retroactively: the surviving Person row has `dob = NULL`,
and `_dob_conflict` treats a missing DOB as "no conflict".

### Why this outranks defects A–C

A and B mean "we won't recognise them". C means "we'll recognise them as someone else".
**D means six people ARE one person in the database** — their records, consent, messages
and appointment history hang off a single identity.

## 2c. DEFECT C's chain — corrected

A subagent reproduced the full sequence on DEV and **corrected the causality in §4 Cause 4**.
My original account had the call-agent starting the chain. It does not:

- Running the exact PROD order (unlinked channel -> Go intake -> staff edit) does **NOT**
  reproduce. At that point the channel is unlinked, so `lookupNameByPhone` returns nothing
  and the intake is correctly named and correctly resolved.
- The sequence that DOES reproduce PROD channel-for-channel (including `is_primary=false`)
  starts with **a record already on Alex's Person having its phone edited to Jacqui's
  number**. The call-agent misnaming is a *consequence* of that link, not its cause.
- `lookupNameByPhone` (`EmailServiceGo/internal/callagent/storage.go:852`) is real and does
  overwrite the extracted caller name — its first branch matches by **person_id**, taking
  any record on any Person linked to the caller's phone channel, regardless of that
  record's own number. Confirmed by replicating both queries in SQL.
- `IntakeDetailView.perform_update` (`intake_views.py:492`) is a plain `serializer.save()`
  with no re-resolution, so a real authenticated PATCH reproduces it identically. **The API
  layer mitigates nothing.**
- All three record types (Intake, Patient, Nurture) reproduce.
- **The household compounds it**: a later genuine record for Jacqui on her own number finds
  Alex as a channel candidate, sees a different name, and creates a new Person *inside
  Alex's Household*. Two unrelated humans become a family.
- The misattributed link never renames the messaging session (`link_channels` uses
  `bulk_create`; `sync_new_record_name_to_sessions` is create-only), which is why PROD
  session 1364 still shows bare digits. **The wrong identity is latent in the data rather
  than visible in the inbox header.**

Alex Cooper's Person 120461 turns out to hold SIX records — his own patient row plus five
unrelated call-agent intakes (Jacqui Rogan, Alison Sims, three "Unknown" callers) on five
different numbers, four of them updated within four minutes of each other on 2026-07-15.

Side finding from the same run: a raw insert into `TreatmentPlan_intake` fails on
`is_marketing_funnel` NOT NULL with no DB default — a Go/Django `default=` drift of exactly
the class the pre-commit parity guard exists to catch. Worth checking the Go `IntakeRecord`
struct lists that column.

---

## 3. Worked examples (real records, DEV)

### 3.1 Jacqui Rogan — defect C, the serious one — **CONFIRMED ON PROD**

Patient 30022 (practice 16), phone `+447864538288`. Verified on PROD read-only: that
number is registered to **Alex Cooper**. Her practice-13 copy (patient 21508) has the
number registered to nobody at all — so the same patient hits defect C in one practice and
defect B in the other.

```
 patient | practice | name         | phone      | number_registered_to
   21508 |       13 | Jacqui Rogan | 7864538288 | (nobody)
   30022 |       16 | Jacqui Rogan | 7864538288 | Alex Cooper
```


That number is registered as **channel 190505 — linked to Alex Cooper's Person (120461)**.
Alex Cooper's own number is `+447376361234`; his Person carries both.

Consequence: `messaging_messagesession` 1364 is an inbound conversation on Jacqui's number,
`patient_id` NULL, `participant_name` = the bare digits. **The app shows the wrong identity
for her number.** Alex Cooper and Jacqui are in different households, so this is not
household sharing.

Her email is fine — `srogan689@gmail.com` on file, same registered.

### 3.2 Samantha Alexander — defects A + B (via practice duplication)

Patient 31085 (practice 16), email `alexandersamb@gmail.com`.
Her Person (119010) is registered under `jamiespears@hotmail.co.uk` and Jamie's phone.

Household 16213 is the Spears family — Harrison, Freddie, Jamie, Samantha, all registered
under Jamie's email. For the children that is correct (they have no email of their own).
For Samantha it is not: she has her own address on file and it was never registered.

Her address **does** exist as a channel — id 151223, **practice 13**, created
`2026-07-12 21:35`. Her patient record is in **practice 16**. Registration is
practice-scoped by design, so the practice-16 copy simply never got one.

### 3.3 Christopher Dale — defect A, simplest form

Patient 39322, `candfdale@btinternet.com` on file, **no email registered at all**.

---

## 4. Root causes

### Cause 1 — the 2026-07-12 backfill left populations behind (~3,080 of defect A)

Every email channel in the system was first created on **2026-07-12**. That job did not
finish the whole population. Practices **21** (2,265) and **19** (813) are largely missing,
and nothing has gone back for them.

This is **already documented in your own code** —
`TreatmentPlan/contact/channel_resolution.py`:

> *"The channel row was never created at all, even though the address or number sits on the
> Patient record — a Feb bulk import and the 2026-07-12 backfill each left populations
> behind. (866 emails, 3,833 phones)."*

So this defect class was known; §2 is its current size.

### Cause 2 — the same human in two practices (~316 of defect A)

Practices 13 and 16 are a head/child pair holding the same patients. Contact registration
is deliberately practice-scoped (a channel must never cross practices). The July backfill
registered the practice-13 copies; the practice-16 copies were left without. 282 of
practice 16's 384 cases are exactly this.

**This is not a scoping bug** — the scoping is right. The practice-16 copies just need
their own registration.

### Cause 3 — registered but not linked (399 of defect A)

The detail exists as a channel in the correct practice and simply isn't attached to the
person.

### Cause 4 — attributed to the wrong person (defect C) — CAUSE NOT YET ESTABLISHED

The strongest candidate is documented in the Go source itself. Before the current
resolution rewrite, `EmailServiceGo/internal/dentally/migration/service.go` resolved a
person as:

```sql
SELECT person_id FROM TreatmentPlan_personchannel WHERE channel_id IN (...) LIMIT 1
```

Its own comment describes the consequence: *"no name check, no DOB check, no merged_into
filter and no ORDER BY. Any patient sharing a family phone was absorbed into whichever
Person the database happened to hand back."*

That would produce exactly defect C. **But I have not proved these 12,452 rows came from
that code path** — that needs either row-level timestamps tied to a sync run, or the sync
logs. Until then this is a hypothesis, not a finding.

---

## 5. What was fixed today (all uncommitted)

| Area | Change |
|---|---|
| Ingress harness scorer | Refs keyed to the DB row their delivery landed on, not their email. 7 of 14 scenarios could not previously fail. |
| Harness phone namespacing | Per-run phone shift so old runs' data can't block new ones. |
| CSV import | A row that dedups onto an existing patient now contributes its channels instead of discarding them — the dedup was manufacturing the next duplicate. |
| Canonical keys | ONE definition per key: `TreatmentPlan/utils/contact_keys.py` + Go `pkg/email`, `pkg/personname`. ~20 inline Django copies and 3 name normalizers (with 2 behaviours) removed. |
| Name key bug | `Person.resolve` compared names without collapsing internal whitespace while the duplicate *finder* did — so the finder reported matches the resolver refused. Fixed both sides. |
| Go writers | callagent, Dentally migration, recall engine, daylist store all use the shared keys. |
| Public form ingress | `marketingBroadcast/views/public_form_views.py` stored the submitted address raw. |
| Cross-language parity | `CROSS-LANGUAGE PARITY` markers on all 5 Django↔Go duplicated implementations + `docs/CROSS_LANGUAGE_PARITY.md`. |

---

## 6. Open, in priority order

1. **Defect C (12,452 rows).** Contact details attributed to the wrong human. Establish the
   cause before repairing — a blind repair could detach details that are correct.
2. **Defect B (7,488).** Phones — the larger of the two "not registered" defects, and the
   one I under-reported for most of this investigation.
3. **Defect A (3,547).** Emails. Causes 1 and 3 are safe to backfill; Cause 2 needs a
   decision (it means deliberately registering the same human once per practice).
4. **Dentally tables outside the graph.** `recall_patient`, `recall_record`,
   `daylist_patient`, `dentally_appointment` have no person or channel FK. 3,303 distinct
   addresses in them are registered nowhere. `bridge_dentally_identity` (read-only by
   default) is built but has never been run with `--apply`.
5. **Truth-table case 5.** Same person, all-new email and phone, can never be matched —
   name is not a candidacy signal. Product decision, not a code fix.

---

## 7. Corrections made during this investigation

Recorded because each one changed the conclusion:

- **"242 addresses invisible"** → the real figure is **3,303**. I had measured whether the
  *person* was reachable; dedup keys on the *address*.
- **"Email only"** → phones are ~2× worse (7,488 vs 3,547) and went unmeasured for most of
  the session.
- **"Missing/unlinked"** → Jacqui's case is neither: the detail is linked, to the wrong
  human. That is defect C, found only because a specific patient was queried by name.
- **"The August call-agent finding is outdated"** → it was correct. Zero call-agent intakes
  get a Person at insert; 1,445 of 1,447 are welded only when Django later re-saves them,
  median <24h but a tail to 59 days.
- **Cross-language drift introduced mid-session** — fixing Django's name key left Go's copy
  on the old rule for several hours. Caught by sweep, not by a test. This is why §5's
  parity markers exist.

---

## 8. How to re-run these measurements

DEV DSN is in `ingress-test-engine/.env` (`dev_dsn`). PROD is read-only:

```
psql "postgresql://sim_readonly@100.95.79.104:5432/treatmentpath_db?sslmode=prefer"
```

The queries for §2 are inline in this session's transcript; the two that matter are
(a) patients whose own email/phone is absent from their Person's channels, and
(b) channels linked to a Person where the value is another Person's own detail and the two
are not in one household.

---

## 9. Verification harness — `contact_identity_audit`

Read-only. Snapshots the IDENTITY of every affected row to CSV, and diffs two snapshots.

```
python manage.py contact_identity_audit --snapshot before.csv
...apply a fix...
python manage.py contact_identity_audit --snapshot after.csv
python manage.py contact_identity_audit --compare before.csv after.csv
```

**Why identity and not counts.** `before: 3,549 / after: 3,549` reads as "no effect" but is
equally consistent with fixing 3,549 rows and breaking 3,549 different ones. The diff
reports FIXED / PERSISTING / **NEW** per defect; NEW is the column that matters. Proven on
crafted fixtures: identical totals, wholly different row sets -> correctly reported 3 fixed,
3 NEW, plus a control row that broke.

**Control group.** 2,000 rows that are CORRECT today are snapshotted as `CONTROL_HEALTHY`.
Any that stop being healthy are reported as regressions. A repair with a better total but a
broken control row is not safe.

**One definition per defect**, shared by both snapshots, so before/after cannot drift in how
they define "broken".

### Baseline captured (DEV, 2026-09-04) — `docs/identity-snapshots/dev-before.csv`

| Defect | Rows |
|---|---|
| A email unregistered | 3,549 |
| B phone unregistered | 7,486 |
| C wrong person | 5,454 |
| D collapsed person | 817 |
| CONTROL healthy | 2,000 |
| **total** | **19,306** |

Take the equivalent PROD snapshot before any repair there — the connection is read-only, so
snapshotting PROD is safe.

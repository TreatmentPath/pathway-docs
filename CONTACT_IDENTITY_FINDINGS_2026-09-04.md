# Contact identity — findings and production plan

**Investigated 2026-09-04. DEV figures verified against PROD (read-only). Nothing written to production.**

Everything here is **measured** unless a line says otherwise. Section 7 lists the
conclusions that were wrong along the way and why — several headline numbers moved by
an order of magnitude under checking, so the reasoning trail matters as much as the
result.

---

## 1. The four defects, final state

| | What it is | Size | Status |
|---|---|---|---|
| **A** | A patient's own **email** not registered against their identity | **0** | ✅ fixed by backfill |
| **B** | A patient's own **phone** not registered | **0** | ✅ fixed by backfill |
| **C** | Person rows with contact channels but **no records** | 25,094 | ✅ **explained — not damage** |
| **D** | One Person holding **3+ different humans** | **817** | ⚠️ open — repair built, rehearsed |
| **E** | Phone numbers that cannot be canonicalised | ~1,690 people | ⚠️ 268 recoverable, rest is bad source data |

**D is the only genuine identity damage remaining.**

Scale for context (PROD): 60,326 patients · 83,697 Persons · 108,612 channels · 136,671 links.

---

## 2. Root causes — all traced to specific events

Every cause is a **historical event whose code is already fixed**. Nothing is actively
producing new damage except the one live defect in §3.

### D — the family collapse (817 Persons, ~2,728 people)

Measured by *when records were attached*, not when the Person row was created — an
earlier reading of `Person.created_at` gave the wrong answer:

| Day collapse completed | Persons |
|---|---|
| **2026-07-27** | **812** |
| 2026-02-11 | 1 |
| 2026-09-02 | 1 (test fixture) |

2026-07-27 was a single 24,555-patient bulk import (adjacent days: 7, 1, 14, 14).

The resolution in force matched on **channel only**. From the Go source's own
description of what it replaced:

> *"SELECT person_id FROM TreatmentPlan_personchannel WHERE channel_id IN (...) LIMIT 1
> — no name check, no DOB check, no merged_into filter and no ORDER BY. Any patient
> sharing a family phone was absorbed into whichever Person the database happened to
> hand back."*

**Worked example — PROD Person 152744, "John Geary":** six humans, six birth dates
(1984, 1985, 2012, 2014, 2018, 2021), two parents and four children on one family
email, under one identity named after the youngest child.

**Fixed:** Go `4e6b27a` (2026-08-17), Django `e1d172da` (2026-08-18).

### C — record-less Persons (25,094): group-practice migration debt

**Not identity damage.** Practice 24 (SmileHQ) is a group head; 26 (Hacton) and 27
(Church View) are child sites. The Go sync could not handle group practices until
`7d9f93f` (2026-07-06); `dentally_site_assignment` rows were only created 2026-08-03.

Sequence: patients imported into child practices → Persons built for them by the
2026-07-12 backfill → patients **hard-deleted** and re-imported under the head → Person
rows left behind with their channels.

Evidence:

- 99.4% of record-less Persons are in practices 26 and 27
- 25,065 of 25,094 created **2026-07-12**
- Practice 26: **20,070 Persons vs 6,675 patients**
- Only **19 archives** in practice 26 → deletion was hard, not archival
- Practices 26/27 have **no Dentally integration**; the key lives on head practice 24
- Practice 24's integration was created **2026-07-27** — the import day itself

**Confirmed to 99.97%:** of 19,907 stranded practice-26 Persons with name+DOB, the same
human still exists as a patient in practice 24/26/27 for **19,855**. Of the 52 that did
not match exactly: **26** same name with a corrected DOB, **21** renamed but a contact
channel matches a live patient, **5** genuinely absent (a patient leaving the practice
is a normal outcome).

**Live consequence:** 14,030 phone channels still hang off these obsolete Persons, so an
inbound call from one resolves to an empty identity. That is a cleanup decision, not a
repair.

### E — unusable phones (~1,690 people, 3,437 records)

Concentrated in practices **13 (Practice Mannie)** and **16 (Danbury)** — 99% of cases,
16.7% and 16.3% of their patients. Every other practice is at 0.1–0.2%. Those two hold
the same humans (1,712 shared Dentally ids), so it is ~1,724 distinct people.

Source: the **February 2026** Dentally import, run from Django by user 66 in 17-minute
bursts, months before the Go migration service existed (2026-04-14).

Split by whether the number is recoverable from data we hold:

| | Count |
|---|---|
| **Recoverable** — a Dentally value reaches real E.164 | **268** |
| Unrecoverable — no value reaches E.164 | 3,473 |
| Recoverable from later syncs (`dentally_appointment` 47, `recall_patient` 5) | 52 |

The 268 have a specific, fixable cause: **`+44` forced onto numbers that already carried
a different international prefix.**

```
stored 4435699097155  ← source +35699097155    (Malta)
stored 44034617277712 ← source 0034617277712   (Spain, 00 prefix)
stored +GB7544805726  ← source +447544805726   (the +GB bug)
```

The rest is Dentally's own data being incomplete — `224012` with no area code,
`0778601318` at 10 digits where a UK mobile needs 11. Not recoverable from any database.

---

## 3. The one LIVE defect — contact-correction misattribution

Everything above is historical. This one fires **today**, every time staff correct a
contact detail.

`TreatmentPlan/contact/signals.py::_preserve_person_and_link_channels`: on update it
keeps the record's existing Person and links the record's **current** contact details to
that **old** Person. It computes `_contact_fields_changed` on the line above and does
not use it to gate the linking.

**Traced production case:** Jacqui Rogan's number `+447864538288` is registered to **Alex
Cooper**. His Person also holds intakes named "Jacqui Rogan", "Alison Sims" and three
"Unknown" callers.

**Fix applied:** re-resolve when the name **and** contact both changed. Both conditions
are required — each guards a legitimate edit:

- contact changed, name same → same human, new mobile. Keep their Person.
- name changed, contact same → a spelling fix. Re-resolving would split one human in two.

**20 tests; mutation-proven** — removing the gate fails exactly 9.

---

## 4. Guards, and what actually proves them

| Guard | Proof |
|---|---|
| Django `Person.resolve` | `test_family_collapse_cannot_recur.py` replays the real six-Geary shape. **Mutation: remove name/DOB check → 5 of 6 fail** |
| Go resolution | `TestPersonResolve_SharedPhoneDoesNotWeldFamilyIntoOnePerson`. **Mutation: neuter checks → fails** ("no Person named 'Autumn Wilkinson'... got Ian Wilkinson") |
| Contact-correction gate | 20 tests. **Mutation: remove patch → 9 fail** |
| Canonical email/name/phone keys | Shared fixtures both languages read; mutation-proven both sides |

**Caution:** the first Go mutation attempt broke the *build* (unused variables), which
proves nothing. A build failure looks like a passing mutation test if you don't read the
output. Always mutate in a way that compiles.

**Remaining structural risk:** `Person.resolve` (Django) and
`linkPatientToPersonAndChannels` (Go) are two implementations of the same rule, kept in
step only by a `CROSS-LANGUAGE PARITY` comment. The keys have shared fixtures; the
*resolution logic* does not. That is the gap that caused this, and it is still open —
see `docs/CROSS_LANGUAGE_PARITY.md`.

---

## 5. Tooling built (all read-only unless `--apply`)

| Command | Purpose |
|---|---|
| `contact_identity_audit --snapshot / --compare` | Snapshots the **identity of every affected row**; diff reports FIXED / PERSISTING / **NEW** |
| `validate_person_channel_links` | Per-link verification with downstream checks |
| `split_collapsed_persons` | The D repair. Creates and re-points only — **never deletes** |
| `identity_guard` | Cause-agnostic damage detector, exits 1 on new damage |
| `bridge_dentally_identity` | Links Dentally rows reaching no Patient |
| `backfill_missing_person_channels` | Pre-existing; fixed A and B |

**Why snapshots record row identity, not counts:** `before 3,549 / after 3,549` reads as
"no effect" but is equally consistent with fixing 3,549 rows and breaking 3,549 others.
Proven on fixtures: identical totals, different rows → correctly reported 3 fixed, 3 NEW.

`split_collapsed_persons` uses `Person.resolve`, so a human who already exists rejoins
their real identity rather than gaining a duplicate — verified on DEV, where Jacqui's
intake returned to her existing Person 118602.

---

## 6. Production plan

Fresh prod copy already pulled: `_prod_sim/fresh-2026-09-04/` (1.9 GB, archive stamped
2026-09-04 16:00:01, `pg_restore` verified, 5,578 objects).

```
# 1. rehearse on the restored prod copy FIRST
python manage.py contact_identity_audit --snapshot before.csv
# deploy identity fixes, then:
python manage.py split_collapsed_persons --csv plan.csv        # writes nothing
python manage.py split_collapsed_persons --person <id> --apply # one, inspect by hand
python manage.py split_collapsed_persons --limit 20 --apply
python manage.py split_collapsed_persons --apply
python manage.py contact_identity_audit --snapshot after.csv
python manage.py contact_identity_audit --compare before.csv after.csv
```

**Acceptance:** D falls, **NEW is zero on every defect**, no control row breaks. A better
total with any NEW rows is not a pass.

**Order matters:** deploy the code fixes first, or the repair competes with a live
defect. Run outside sync windows — a Dentally import mid-repair re-resolves records while
they are being moved.

**Known limits of the repair, both deliberate:** person-level rows (Notes, Activity) stay
on the anchor Person because they carry no per-human marker; and stale channel links are
not removed, so after repair Jacqui's number is registered to *both* her and Alex Cooper.

---

## 7. Conclusions that were wrong, and what corrected them

Recorded because each changed the answer, and because the pattern matters more than any
single number.

| Claim | Reality | Caught by |
|---|---|---|
| Defect C = 12,452 misattributions | **Not misattribution at all** — 25,094 record-less Persons from a group-practice migration | Reading actual rows instead of counting query output |
| C = 5,454 (after first correction) | Still wrong; the join inflated it and the definition was wrong | The audit tool disagreeing with the ad-hoc query |
| "242 addresses invisible" | **3,303** | Measured person-reachability, not address-reachability |
| Email is the problem | Phones are ~2× worse and went unmeasured for most of the session | Asking about a specific patient (Jacqui) |
| 370 phones recoverable | **268** — bare `needs_review` keys counted as successes | Requiring the result to start with `+` |
| Backfill damaged 356 links | **Zero.** Three rounds of false alarms | Inspecting all 27 flagged links individually |
| 26 control rows "broke" | Artefact of my own harness — `ORDER BY id LIMIT 2000` shifted between runs | Checking three of them |
| 772 collapses from the 12 Jul backfill | **812 from the 27 Jul import** | Grouping by record attachment, not `Person.created_at` |
| "Reuse the stranded Persons" | They are in **different practices** — cross-practice merging | Checking practice ids |
| "The repair tool invents Persons" | It uses `Person.resolve` and reuses | Reading the code I wrote |
| Dentally search confirms patients | The API **returns HTTP 200 and ignores the filter** — a nonsense query returns the same first page | Querying `ZZZNOSUCHNAMEZZZ` |

**The recurring failure:** measuring one thing and reporting it as another; trusting a
number because the query succeeded. Every correction came from looking at individual
rows rather than aggregates.

**New Dentally gotcha for the runbook:** `/v1/patients` accepts `q=`,
`filter[last_name]=` and `search_criteria=` with **HTTP 200 and silently ignores them**.
Existing note says 401 = bad key, 404 = route missing; add: **200 ≠ filtered**.

---

## 8. Reproducing any measurement

DEV DSN: `dev_dsn` in `ingress-test-engine/.env` (read with an absolute path — the shell
cwd resets between calls). PROD read-only:

```
psql "postgresql://sim_readonly@100.95.79.104:5432/treatmentpath_db?sslmode=prefer"
```

All defect definitions live in one place —
`TreatmentPlan/contact/identity_defects.py` — which routes through
`ContactChannel.canonical_key`, the real chokepoint. **Never reimplement a canonical key
in SQL:** the first version rebuilt the phone key as
`'+'||country_code||phone_number`, could not apply the practice's default country, and
produced three separate false alarms in one afternoon.

**Test-data trap:** `+4477009000xx` is Ofcom's reserved fictitious range.
`canonical_key` rejects it outright, so a fixture using it silently creates **no phone
channel** and every phone assertion passes or fails for the wrong reason.

# Defect E — phone numbers that cannot be canonicalised

**Status:** every affected patient checked individually against **live Dentally**,
2026-09-05. Recoverable cases repaired and verified on the production copy.
**Not yet run on production.**

---

## 1. The real size — earlier estimates were wrong

| Claim | Reality |
|---|---|
| "~1,690 people affected" | **1,020 people / 777 channels / 774 patients** |
| "268 recoverable" | **10 recoverable to a valid mobile** (+34 to a landline) |

Neither earlier figure survived contact with the data. Both came from pattern
inference; the numbers below come from fetching all 774 patients from Dentally one
at a time.

## 2. Per-patient verdict, against the source of truth

| Verdict | Patients |
|---|---|
| **UNRECOVERABLE — Dentally's own number is invalid too** | **728** |
| PARTIAL — no valid mobile, but a valid home/work number | 34 |
| **RECOVERABLE — Dentally holds a valid mobile** | **10** |
| no Dentally id | 2 |

### Why the 728 genuinely cannot be fixed

**705 of them are SIX-DIGIT numbers in Dentally itself.** Live example, patient 2311:

```
home_phone            = '223863'      <- six digits, no area code
home_phone_normalized = '+44223863'   <- Dentally's own bad normalisation
work_phone            = '227531'
mobile_phone          = ''
```

Old-style local numbers from before area codes were required. Dentally prefixes
`+44` and produces nonsense. **Nothing stored anywhere can reconstruct the area
code**, and inventing one means calling a stranger.

The remaining 23 are off-by-one-digit errors at source — `0753388862` (a UK mobile
needs 11 digits), `073781631659` (one too many). You cannot know which digit was
dropped or added.

**This is not our bug.** The importer faithfully copied bad Dentally data.

### A shortcut that was considered and rejected

6 of the 728 have a *valid emergency-contact* number. An emergency contact is
normally a **different person** — a partner or parent. Adopting it as the patient's
own number would manufacture exactly the misattribution the rest of this workstream
exists to remove. Do not do it.

## 3. It is overwhelmingly ONE practice — and it is the test practice

| Practice | Affected patients | % of its patients |
|---|---|---|
| **Practice Mannie (13)** | **692** | **6.54%** |
| Danbury Dental Care (16) | 53 | 0.46% |
| Fareham Road Surgery (21) | 12 | 0.10% |
| Church View Dental Clinic (27) | 11 | 0.09% |
| Hacton Dental Care (26) | 6 | 0.09% |

Practice Mannie runs at **ten times** the 0.4–0.8% background rate of every real
practice. It is the owner's test practice, and its records still sync from Dentally
(Danbury's key reaches the same account), so **deleting them would be undone on the
next import**. It is therefore EXCLUDED from the measure rather than deleted —
`TEST_PRACTICES` in `repair_unusable_phones.py`.

**Excluding it, defect E is 82 affected patients, not 774.**

## 4. The repair

`TreatmentPlan/management/commands/repair_unusable_phones.py` — dry run by default.

```bash
# needs DENTALLY_ENCRYPTION_KEY from the production .env
python manage.py repair_unusable_phones --landlines            # dry run
python manage.py repair_unusable_phones --landlines --apply
```

It queries Dentally per patient (the `search` parameter is silently ignored, so
bulk filtering is impossible), and adopts a valid mobile — or a landline with
`--landlines`, which restores calling but **not** SMS.

### Result on the production copy

18 patient rows updated, **13 distinct patients verified individually: 13 OK, 0 problems.**

| Before | After |
|---|---|
| `44034617277712` | `+34617277712` (Spain) |
| `446582288227` | `+6582288227` (Singapore) |
| `4435699097155` | `+35699097155` (Malta) |
| `44470003` | `+442074207379` (landline) |
| `44754635509` | `+447546355096` |

Most recoverable cases are **foreign numbers mangled by a `44`-prefix bug** —
`44` + `0` + the real international number.

All five identity defects remained **0** afterwards.

## 5. Two traps this exposed — both in MY verification, not the repair

**1. Comparing a canonical value against a national number.** The first check
compared `channel.canonical_value` (`+442039686001`) with `patient.phone_number`
(`2039686001`) and reported **13/13 PROBLEM** on a repair that was actually fine.
`Patient.save()` deliberately splits E.164 into `country_code` + national number
(`models.py`, via `parse_phone_number`). Any check must re-canonicalise through
`ContactChannel.canonical_key(practice, kind, phone_number, country_code)` — never
compare the raw column.

**2. Expecting `needs_review` count to fall.** The repair does not delete the old
bad ContactChannel row, so that count cannot drop. It is not a success metric.

**3. Reverting with `.update()` leaves a mixed state.** The revert restored the bad
`phone_number` without firing signals, so the good channel created by the first run
survived and those patients then fell OUT of the next selection — they looked broken
when they were half-fixed. If you revert a phone repair, revert the channels too or
re-run and re-verify.

## 6. What remains

- **65 patients** in real practices whose numbers are unrecoverable at source.
  Fixing these requires a human to ring the patient and ask. They are not a code
  defect and should not be counted as one.
- Practice Mannie's 692 stay as-is by decision (test practice, re-syncs anyway).

---

## 7. A LIVE code bug this uncovered — fixed 2026-09-05

### The bug

Dentally exposes both a raw phone field and its own `*_normalized` version. Every
caller took `*_normalized` whenever it was non-empty. **Dentally's normalisation is
wrong for non-UK numbers** — it prefixes `+44` onto a number that already carries a
country code:

```
mobile_phone            = "+35699097155"    <- correct (Malta)
mobile_phone_normalized = "+4435699097155"  <- Dentally's own bad value
```

We stored the second one. It is not a valid number anywhere, so the patient became
**silently uncontactable by SMS** — no error, no warning, nothing in a log.

### It was in THREE places

| | |
|---|---|
| Go | `internal/dentally/migration/service.go` → `extractPhones` |
| Django | `dentallyIntegration/tasks.py` (mobile, home, work) |
| Django | `dataQuality/patient_import_utils.py` |

### The fix

One shared rule per language, with parity markers pointing at each other:

- Go: `pickUsablePhone(normalized, raw, isoCountry)`
- Django: `TreatmentPlan/utils/phones.py` → `pick_usable_phone(...)`

Order is deliberate and preserves existing behaviour:

1. `normalized` **when it canonicalises** — unchanged for every number where
   Dentally was right, which is the overwhelming majority;
2. otherwise `raw` when it canonicalises — recovers the international cases;
3. otherwise the old fallback, so genuinely broken source data behaves exactly as
   before and still surfaces as `needs_review`.

### Proof it works and breaks nothing

Both test suites use the **real production values** captured from the Dentally API:

| | |
|---|---|
| Go `TestExtractPhones_*` | 9 cases — 4 foreign fixed, 3 UK unchanged, unrecoverable unchanged |
| Django `test_pick_usable_phone` | 5 tests, same cases |
| **Mutation** | restore "always prefer normalized" → the Malta case fails on BOTH sides |
| Django identity + phone regression | **56 tests OK** |
| `dataQuality` | **40 tests OK** |
| `dentallyIntegration` | 400 tests, 4 failures + 2 errors — **proven pre-existing**: reverting the change reproduces exactly the same 6, and they are appointment-targeting tests |
| Go full suite | all packages pass; `internal/email/api` build failure is pre-existing (`ensureDefaultPracticeAddresses` arity in a test) and in a file this work never touched |

### One thing deliberately NOT changed

`extractPhones`' doc comment promises an E.164 dial code, but for a national-format
number `ParsePhoneToDialCode` returns the ISO code (`"GB"`, not `"+44"`). That is
pre-existing, and **harmless**: `ContactChannel.canonical_key` resolves `"GB"` and
`"+44"` to the identical canonical value (`+447546355096`, `needs_review=False` —
verified against production data). Changing it would be risk without benefit. The
Go test asserts the real behaviour and says why.

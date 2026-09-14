# Code sweep — duplicated logic and the defects it was hiding

**2026-09-05.** A read-only sweep for "same thing implemented in several places",
prompted by the identity workstream. Three live bugs came out of it. All fixed and
mutation-proven; the wider consolidation is proposed, not done.

---

## 1. The bugs — all one root pattern

**`country_code` is NOT always a dial code.** 3,167 patients (5.4% of those with a
phone) store the ISO code `"GB"` rather than `"+44"`. Four places built a phone with
`f"{country_code}{phone}"`, which for those patients produces `GB4473781631659`.

| Where | What it did | Severity |
|---|---|---|
| `TreatmentPlan/contact/sync.py` `_normalize_phone_parts` | **WROTE `phone_number="+GB7544805726"` to the database** | data corruption |
| `Appointments/serializers.py` `ShortNoticePatientSerializer.get_patient_phone` | handed the frontend `GB4473781631659` | user-visible |
| `TreatmentPath/activity_change_utils.py` `format_contact_phone` | activity log read `GB44250159` | user-visible |
| `TreatmentPlan/contact/sync.py` `_build_session_identifier` | session key loses the country code | **not changed — see §3** |

### The database one is the serious one

`sync.py` did:

```python
cc = resolve_country_code(country_code, practice)
if not cc.startswith("+"):
    cc = "+" + cc                      # resolve_country_code("GB") == "GB"
full_phone = f"{cc}{phone_number}"     # -> "+GB7544805726"
```

`resolve_country_code` passes an ISO code straight through, so the guard produced
`"+GB"`. That value is then written by
`sync_contact_values -> patient_qs.update(phone_number=...)`, so **any of the 3,167
ISO-coded patients corrupts the moment a contact edit propagates.** Two patients were
already carrying `+GB...` values from exactly this path.

**Fix:** parse the canonical value the function already computed
(`full_phone = normalized_phone`) instead of rebuilding it. `canonical_phone_e164`
resolves ISO and dial codes alike, so `"GB"` and `"+44"` now give identical results.

**Repaired:** both patients, verified letters gone and an E.164 channel linked.
27 Intake rows also contain letters (`anonymous`, `jona`, `03i4`) — those are junk
user input, not this bug, and are left alone.

## 2. A shared channel named an arbitrary human

`messaging/views/contact_views.py` `_resolve_identity` kept its own copy of a rule
that `TreatmentPlan/contact/channel_owner.py` had already replaced:

```python
link = channel.person_channels...first()      # no ORDER BY
person = link.person if link else None
```

On a family phone that returns an arbitrary member — and the method then returns
**that person's** channels as "this contact". Measured on the 2026-09-04 production
copy: of 1,711 channels carrying a conversation, **372 have two or more owners**.

**Fix:** call `sole_person_for_channel(channel)`. Ambiguous channels now resolve to
no person and the contact stays the lone channel, per that module's rule — *an
unknown name is recoverable; a wrong one frozen forever is not.*

## 3. Deliberately NOT changed

`_build_session_identifier` (same file) also concatenates, and with `cc="GB"` the
messaging session key becomes bare `7546355096` instead of `+447546355096`, losing
the country code. **586 sessions already carry bare-digit identifiers.** Changing
the derivation would make new messages compute a different key and **split existing
conversations**. That is a data migration, not a bug fix, and needs its own plan.

## 4. Evidence

| Gate | Result |
|---|---|
| `manage.py check` | no issues |
| `makemigrations --check` | **no changes detected** — no model was altered |
| Mutation (revert each fix) | **5 tests fail** across the three fixes |
| `Appointments` + channel tests | 66 → 66, OK (unchanged baseline) |
| identity + phone suite | 73 OK |
| `messaging` | 144 OK |
| `TreatmentPlan.tests` | 6 failures / 3 errors **with** the changes; **8 / 3** with them reverted — the extra 2 are the new tests failing against un-fixed code. **Zero regressions; two fixed.** |

New tests: `TreatmentPlan/tests/test_phone_display_country_code.py` (14) and a new
class in `messaging/test_shared_channel_attribution.py` covering the endpoint path
(the existing file tested the helper but never `_resolve_identity`).

## 5. Found, not touched — the consolidation backlog

Per the organizing-django-modules rule, deduplication is a separate task from the
bug fixes above. Ranked by risk-of-defect, not by tidiness:

| Concern | State | Why it matters |
|---|---|---|
| **Phone canonicalisation** | healthy — one definition, **30 importers** | the model to copy |
| **Email canonicalisation** | 19 importers; 16 inline `.strip().lower()` remain | all in auth/admin login paths, deliberately out of scope |
| **Name key** | `canonical_name_part` has 8 importers, but **157 inline** `f"{first} {last}"` and **6 separate** `full_name` helpers | a name is an identity key here; divergent spellings caused D2 |
| **DOB parsing** | `canonical_dob` has 4 importers; ~150 hand-rolled `strptime`/`fromisoformat` | mostly reporting code, low identity risk |
| **Practice scoping** | `PracticeFilterMixin` in 28 of 181 view files | practice-scoping is a hard rule; every hand-rolled filter is a chance to miss it |
| **Channel owner** | `sole_person_for_channel` now used by the endpoint; the other 7 `filter(channel=…)` sites legitimately collect ALL owners | no action needed |

**Recommended next:** a single `full_name` / name-key helper, then widening
`PracticeFilterMixin`. Both touch ~180 files, so they want their own branch and their
own before/after test baseline — not to be folded into a bug-fix pass.

---

## 6. Name joining consolidated — done 2026-09-05

Scoped to the patient domain (TreatmentPlan, dentallyIntegration, Notes,
activityLog, messaging, marketingBroadcast). HR, compliance, finance, Invoices,
Stock, settings and UserAuthentication are deliberately out.

### One definition: `TreatmentPlan/utils/names.py` → `full_name(first, last)`

None-safe, collapses internal whitespace, strips. Sits beside `phones.py` and
`contact_keys.py`.

**It is NOT the identity key.** Matching two people is
`contact_keys.canonical_name_key`, which lower-cases and is parity-locked with Go.
`full_name` is for DISPLAY. Keeping them apart matters in both directions: a
display name that started lower-casing would be a visible regression, and a match
key that stopped lower-casing would split one human into two.

### The four sites that now delegate

| Model | Was | Note |
|---|---|---|
| `Patient.full_name` | `f"{first} {last}"` — **no strip** | the odd one out; 26 production patients rendered as `"Funmi "` |
| `Person.display_name` | `.strip()` + phone fallback | keeps its "Unknown"/phone fallback |
| `RecallPatient.full_name` | `.strip()` | — |
| `DentallyPractitioner.display_name` | `.strip()` + title prefix | staff, not a patient, but the same join; keeps its title |

Every property keeps its name and signature, so **no call site changed**. Only the
bodies became one-line delegations.

### Deliberately untouched

- `Patient.person_display_name` — already forwards to `Person.display_name`
- `models.py:4324`, `4493` `display_name` — these are **treatment category /
  procedure** names. Same method name, different concern; folding them in would be
  matching on a word rather than on what the code does.
- The **86 inline `f"{first} {last}"`** builds in-domain (TreatmentPlan 43,
  messaging 19, dentallyIntegration 10, Notes 8, activityLog 3, marketing 3).
  Deferred on purpose: converting them in the same commit as the helper would make
  a bisect impossible if the baseline moved.

### Behaviour change, flagged not slipped

`Patient.full_name` now strips. Those 26 patients render `"Funmi"` instead of
`"Funmi "`. That is the fix, and it is the only behavioural difference.

### Evidence

| Gate | Result |
|---|---|
| `manage.py check` | no issues |
| **`makemigrations --check`** | **no changes detected** — machine proof no model was altered |
| TreatmentPlan | baseline 663 / 6F / 3E → **674 / 6F / 3E** (+11 new tests, same failures) |
| dentallyIntegration | baseline 400 / 4F / 2E → **400 / 4F / 2E** |
| messaging | baseline 144 OK → **144 OK** |
| Mutation (stop stripping in the helper) | **8 of 11 fail, including all 5 model tests** — the models really do delegate |

## 7. PracticeFilterMixin — checked, nothing to do (recorded so it is not re-derived)

`PracticeFilterMixin` appears in only **28 of 181** view files, which looks like a
scoping hole. It is not one.

Every ViewSet exposing `Patient`, `Person`, `Intake` or `Nurture` was scanned for
practice scoping. **Zero were unscoped.** The other files scope a different way —
they override `get_queryset()` and filter there instead of using the mixin.
Different style, identical protection.

Consolidating them would be a cosmetic preference touching ~150 view files with no
behavioural gain, and practice scoping is the one rule where a mistake leaks one
practice's patients to another. **Not worth the risk. Do not re-open without new
evidence** — e.g. an actual cross-practice leak, or a new viewset that genuinely
forgets to scope.

---

## 8. Duplicate-risk sweep — Django + Go, 2026-09-06

A targeted audit of everything that could still mint a duplicate Person or
misattribute identity.

### The structural fact that frames all of it

| Table | Unique constraint |
|---|---|
| ContactChannel | `(practice_id, kind, canonical_value)` ✅ |
| PersonChannel | `(person_id, channel_id)` ✅ |
| Patient | `(practice_id, meta_data->>'id')` partial ✅ — blocks duplicate Dentally imports |
| **Person** | **NONE** |

**Person duplication is prevented by CODE, not by the database.** Every caller of
`Person.resolve` is therefore load-bearing — which is what made the finding below
matter.

### FOUND AND FIXED — resolve() called without a date of birth

`Person.resolve` uses the DOB to REFUSE reuse when two same-named people have
different birth dates. Six callers exist; only the signal path was passing one:

| Caller | Passed dob? | Has a dob available? |
|---|---|---|
| `contact/signals.py` (the save path) | yes | — |
| **`dataQuality/views.py` x2** | **no** | **YES — Dentally always returns `date_of_birth`** |
| **`split_collapsed_persons.py`** | **no** | **YES — the records being split carry one** |
| `patient_views.py` CSV import | no | no — the CSV has no DOB column (verified) |
| `marketingBroadcast/submission_handoffs.py` | no | no — form submissions collect none (verified) |

The two marked in bold could merge a parent and child who share a surname and a
family phone — defect D2, which accounted for 1,489 collapsed identities on
production. Both now pass a DOB (`dob_from_detail()` reads it from the Dentally
payload; the split command takes the first dob its record group actually has).

The other two omit it correctly — there is genuinely no DOB to pass, verified
rather than assumed.

### Checked and clean

| Risk | Finding |
|---|---|
| Person created outside `resolve` | only 3 repair commands, all deliberate and documented |
| Direct `person_id` writes bypassing signals | only the repair commands, all documented |
| `bulk_create` on Patient/Intake/Nurture (fires no signals) | one live path — `patient_views.import_csv` — and it **does** call `PersonModel.resolve` before bulking. Verified, not trusted: the grep for `Person.resolve` missed it because of an aliased import. The rest are test-data generators. |
| Go writing `TreatmentPlan_patient.person_id` | exactly one site, inside `linkPatientToPersonAndChannels` |
| Go ContactChannel insert | takes an already-canonical value; `needs_review` derived from it |
| Go mirror tables (recall, daylist, callagent) | all now canonicalise (2 / 1 / 3 call sites) |
| Hand-rolled email normalisation in Go | one site, `daylist/appointment/sync.go:570` — trims but does not lowercase. Its only consumer matches with `__iexact`, so it is harmless. Noted, deliberately not changed. |

### Still open (not defects, but worth knowing)

- **`Person` has no unique constraint.** Nothing at the database level stops a
  duplicate. Adding one is not straightforward — two people can legitimately share
  a name in a practice — but it means correctness rests entirely on `resolve`.
- `recall_patient` and `daylist_patient` carry patient names and contacts but sit
  **outside the identity graph** (no `person_id`). They cannot create duplicate
  Persons, but they are not deduplicated either.

# Utils consolidation — build once, use everywhere

One definition per behaviour. The 90-finding patient-identity audit kept turning up
the same root cause: the SAME logic implemented several times, slightly differently,
so a reader's lookup key never matched the writer's.

Two homes:
- **global** `TreatmentPathBackend/TreatmentPath/utils/` — used by 2+ apps
- **app** `<app>/utils/` — used only inside that app

## Scan queue — ONE directory per agent run

Not bundles. An agent given four apps at once does a shallow job on all four, so each
run gets exactly one absolute path and is killed afterwards. Both tracks (the codex
issue-hunt and the haiku duplicate-inventory) walk the same list.

Base: `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/`

| # | Directory | Why it is on the list | Haiku | Codex |
|---|---|---|---|---|
| 1 | `TreatmentPlan/` | The identity core: Person, Patient, Intake, Nurture, ContactChannel | DONE | DONE |
| 2 | `dentallyIntegration/` | Recall + day list; imports identity from an external system | DONE | DONE |
| 3 | `messaging/` | Sessions, SMS/email threading — attributes a message to a human | DONE | DONE |
| 4 | `marketingBroadcast/` | Consent, forms, recipient profiles — acts ON a chosen human | DONE | SKIPPED - rerun |
| 5 | `onlineBooking/` | Creates identity from a public form | DONE | DONE (0) |
| 6 | `Appointments/` | Appointments filed against a human | DONE | DONE (0) |
| 7 | `Notes/` | Clinical notes filed against a human | DONE | **SKIPPED - rerun** |
| 8 | `Documents/` | Consent/signing filed against a human | DONE | queued |
| 9 | `activityLog/` | Displays history per human | RUNNING | RUNNING |
| 9b | `automations/` | **Workflow engine — user-flagged.** Prior findings #13 (actions.py:396 vs :688 disagree on raw-vs-normalized phone), #14 (workflow patient lookup case-sensitive on email), #83 (node config overrode the execution's practice). 4,248 lines, 40 identity-shaped call sites | queued | queued |
| 9c | `dataQuality/` | **User-flagged.** The app that DETECTS duplicates — if its detection key differs from `Person.resolve`'s, the two disagree about what a duplicate IS: it reports pairs dedup will not merge, and misses pairs it would. 1,755 lines | queued | queued |
| 10 | `Tasks/` | Tasks filed against a human | queued | queued |
| 11 | `medicalHistory/` | Portal submissions creating/matching a human | queued | queued |
| 12 | `EmailServiceGo/internal/callagent/` | Go, shared DB — its own copies of the keys | queued | area 1 done |
| 13 | `EmailServiceGo/internal/dentally/` | Go sync writer | queued | area 1 done |
| 14 | frontend `src/components/compact/` | Journey tables that split/join names | queued | queued |
| 15 | frontend `src/components/patients/` | Patient panel + universal search | queued | queued |

## Canonical helpers (the "originals" — anything else doing this is a duplicate)

Global `utils/`:
- `practice_mixins.py` — `PracticeAccessMixin`, `scope_clinical_to_practice()`, `assert_user_in_practice()`
- `ws_practice.py` — `current_practice_id()`, `reject_unless_member()`

`TreatmentPlan/utils/`:
- `contact_keys.py` — `canonical_email()`, `canonical_name_part()`, `canonical_name_key()`, `canonical_full_name_key()`, `canonical_dob()`
- `names.py` — `full_name()` (the ONE display join)
- `phones.py` — `canonical_phone_e164()` (the ONE phone dedup key) + 11 others
- `sole_patient.py` — `sole_patient_for_person()`, `sole_record_for_person()`
- `contact/channel_owner.py` — `sole_person_for_channel()`

Go equivalents that must stay byte-parity with the above:
- `pkg/phone` — `CanonicalE164()`, `SplitE164()`, `PracticeRegion()`
- `pkg/personname` — `CanonicalFullKey()`

## Process

1. Spawn ONE agent for the next app. Never two at once.
2. It writes `docs/utils-consolidation/<app>.md` — inventory only, no code.
3. Compile the fixes into `FIX-LIST.md`.
4. Kill that agent, spawn a fresh one for the next app.

**The most valuable output is not "these are duplicated" but "these copies DISAGREE."**
Two divergent copies of a lookup key is the exact bug class this workstream exists for.

## Codex runs still owed

The codex track is one directory behind in two places — both skipped because a previous
run was still busy when the next was due, and not re-queued:

- `marketingBroadcast/` — never started
- `Notes/` — never started (the haiku scan there produced C11, so a second pass is worth it)

Re-run both before calling the sweep complete.

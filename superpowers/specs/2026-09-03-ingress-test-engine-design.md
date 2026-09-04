# Ingress Test Engine — Design

**Date:** 2026-09-03
**Status:** Approved in brainstorming session
**Project root:** `/home/mannie/Desktop/Projects/treatmentpath/ingress-test-engine/` (new standalone sub-project)

## 1. Purpose

A regression harness for the patient-identity pipeline. We feed **synthetic data whose
correct answer we already know** through the app's real entry points, then score how well
the system did:

- did it **merge** records that are the same human?
- did it **keep apart** records that are different humans (even when they share a phone)?
- how many verdicts were **correctly flagged** — as a graded scorecard, not just pass/fail?

Every time an ingest lane or identity logic changes, re-running the suite shows whether the
score held or dropped. It is the whole-identity-pipeline analogue of the existing
`phone_fixtures.json` parity test.

## 2. Architecture

Standalone Python package, no Django imports. The backend is a black box with one
read-only window:

```
ingress-test-engine/
├── pyproject.toml            # deps: requests, psycopg2 (read-only), pyyaml, fastapi, uvicorn, click
├── engine/
│   ├── generator.py          # synthesizes records + ground truth from parameters
│   ├── archetypes.py         # the 8 archetype definitions (below)
│   ├── drivers/              # one driver per entry point (full matrix in §9)
│   │   ├── base.py
│   │   ├── legacy_webhook.py     # ep-legacy · IntakeWebhookReceiveView
│   │   ├── custom_webhook.py     # ep-custom · custom journey webhook
│   │   ├── form_handoff.py       # ep-mkt · marketing form handoff
│   │   ├── ai_email.py           # ep-ai · create_intake_from_email shim
│   │   ├── add_patient.py        # ep-add · Add-Patient dialog (IntakeListCreateView)
│   │   ├── patients_crud.py      # ep-crud · PatientSerializer.create
│   │   ├── family_member.py      # ep-family · add-as-family-member
│   │   ├── move_convert.py       # ep-move + ep-conv · Move/Convert + journey conversions
│   │   ├── csv_import.py         # ep-csv · CSV bulk import
│   │   ├── dq_import.py          # ep-dq · DQ force-import
│   │   ├── automations.py        # ep-auto · automations engine actions
│   │   ├── booking.py            # ep-book · online booking (payment + abandoned session)
│   │   ├── message_session.py    # ep-msg · inbound email/SMS/WhatsApp auto-create
│   │   ├── archive_restore.py    # ep-restore · archive restore
│   │   ├── merge_unweld.py       # ep-merge · merge / unweld
│   │   ├── backfills.py          # ep-backfill · management-command backfills
│   │   ├── admin_edit.py         # ep-admin · Django Admin edits
│   │   └── go_sync.py            # go-sync · Dentally sync raw SQL (V2)
│   │   └── go_callagent.py       # go-call · call-agent GORM intake (V2)
│   │   └── go_workflows.py       # go-wf · workflow updates via Django REST (V2)
│   ├── runner.py             # executes scenarios: fire → wait for async effects → snapshot DB
│   ├── scorer.py             # compares actual Person/Channel graph vs ground truth
│   ├── db.py                 # read-only Postgres access (DEV), graph extraction
│   ├── auth.py               # practice-scoped JWT minting for the QA practice
│   ├── report.py             # HTML scorecard generation
│   └── server.py             # FastAPI app powering the web UI (localhost:8090)
├── web/
│   └── index.html            # simulator UI (compose / run / scorecard tabs)
├── scenarios/                # saved YAML scenarios (generator can emit these)
└── reports/                  # generated run reports (run-<timestamp>.html)
```

## 3. Archetypes (derived from the identity models)

Identity = `Person` + `ContactChannel` (unique per practice+kind+canonical_value) +
`PersonChannel` (is_primary) + `Household`. Every simulatable system state is a
combination of these:

| # | Archetype | Model-level meaning | Correct outcome |
|---|-----------|--------------------|-----------------|
| 1 | Unique person | new channels, new Person | 1 Person, 2 channels |
| 2 | Same person, new email | new ContactChannel adopted by existing Person | same Person, +1 email channel |
| 3 | Same person, reformatted phone | canonicalizes to existing channel key | same Person, **no new channel** |
| 4 | Same person, name variant | channels match despite name drift | same Person, updated name |
| 5 | Different persons, shared phone | 2 Persons, 1 phone channel, 2 PersonChannels (one primary) | **no merge** |
| 6 | Family of N | Household + N Persons sharing a phone channel | 1 household, N persons |
| 7 | Re-arriving merged person | value of a `merged_into` Person reappears | resolves to survivor |
| 8 | Cross-lane same human | same human via webhook AND AI email AND booking | still one Person |

**Phone generation rule (all archetypes):** valid phone numbers are produced with the
`phonenumbers` library (Google libphonenumber port) — for a chosen region the generator
picks a valid number pattern and verifies it with `phonenumbers.is_valid_number`, so every
"real" number is callable-format correct for ANY country code, not just GB. Noise variants
(formatting archetypes 3, 5) re-render the same number via `phonenumbers.format_number`
(E.164, NATIONAL with spaces, etc.). Separately, the generator has a **rubbish-number
knob** (`invalid_phones: N`): numbers that violate the library — digit-transposed,
truncated, letter-containing, `+` in the middle, empty — feeding adversarial family A1 and
proving lanes reject/degrade instead of creating junk channels.

Plus three **adversarial families** (these exercise failure handling, not identity
resolution — an error here crashes the ingest pipeline):

| # | Adversarial family | What it feeds the lane | Correct outcome |
|---|--------------------|------------------------|-----------------|
| A1 | Malformed payloads | missing fields, wrong types, garbage phone strings, 100KB fields | lane returns 4xx or degrades; **no 500, no partial row** |
| A2 | Delivery chaos | duplicate webhook delivery (retry), duplicate arriving *before* the original, out-of-order pairs | idempotent: no twin records |
| A3 | Concurrent arrival | same human via two lanes simultaneously (threaded) | exactly one Person afterwards |

A2/A3 need ordering controls in the runner (`order: sequential|reverse|concurrent`) and an
idempotency assertion type (`expect: no_duplicate_person`).

## 3b. Engine self-tests — battle-testing the engine (rigor requirement)

The engine is itself critical infrastructure — a buggy scorer that always says "pass"
would be worse than no engine. The self-test battery covers five attack surfaces:

1. **Scorer conformance (by construction).** For every archetype, generator-emitted
   fixtures are scored in pure-python form (no DB) against a "perfect system" grouping
   (must be 100% accurate) and a "twins system" grouping (every same-truth pair must be
   `missed_merge`).
2. **Property-based testing (hypothesis).** The generator/scorer pair is fuzzed over
   random parameter combinations (persons 0–50, families 0–10, family sizes 1–8, any
   duplicate mix, regions from a libphonenumber region list, seeds). Invariants that must
   hold for EVERY draw: determinism under fixed seed; truth covers every ref; pair
   classification is total (no pair unclassified); accuracy ∈ [0,1]; score of the identity
   mapping == 1.0.
3. **Adversarial inputs to the engine itself.** Malformed scenario YAML (missing truth,
   refs in `truth.persons` with no record, records with unknown `entry_point`, duplicate
   refs, `order` misspelled, negative counts, family_size 0) must produce clean
   validation errors — never a crash mid-run or, worse, a silently-empty pass. Scorer
   pathological inputs: empty scenario (defines accuracy = 1.0 by convention, documented),
   single ref, 10k refs (performance ceiling), truth/actual key mismatch.
4. **Fault injection (runner).** Driver transport errors, timeouts, HTTP 500s, and a
   shim that dies mid-run must produce a per-record error result and a scored-but-failed
   run — never a partial silent pass, never an unhandled exception with no report.
5. **Canary mutation test** (runnable via `python -m engine canary`): the setup registers
   a deliberately-broken matcher variant; running the suite against it MUST produce a
   lower score. If the harness can't fail, it's void.

Plus **driver smoke tests**: each driver fires against the QA practice with a trivial
unique-person fixture and asserts the record actually landed — proves the driver works
before identity scenarios rely on it.

Archetype 8 (per-record entry-point choice) and entry-point noise variants (raw vs E.164
phones, email case/whitespace) are orthogonal knobs the generator applies to any archetype.

## 4. Fixture format

One YAML file = one scenario. Ground truth is written by the author **or emitted
automatically by the generator** (which knows the truth because it constructed the data).

```yaml
scenario: family-shared-phone
entry_points:
  A: legacy_webhook
  B: add_patient
records:
  - { ref: A, first_name: Alice, last_name: Shaw, email: "alice@ex.com",  phone: "+447700900123" }
  - { ref: B, first_name: Bob,   last_name: Shaw, email: "bob@ex.com",    phone: "07700 900123" }
truth:
  persons: { A: p1, B: p2 }          # two different people...
  shared_channels: [{ A: B, kind: phone }]
  household: [A, B]
expect:
  person_count: 2
  channel_count: { phone: 1, email: 2 }
```

`records[].ref` is the scenario-local identity; `truth.persons` maps refs → canonical
synthetic persons. Two refs sharing a `pN` means "same human" — that is the entire ground
truth the scorer needs.

## 5. Scorer

After the run, the engine snapshots the actual graph (Persons, ContactChannels,
PersonChannels, Households, record rows) from DEV Postgres (read-only role) and classifies
every ref pair:

| Verdict | Meaning |
|---------|---------|
| **Correct merge** | truth: same person, system put them on one Person ✅ |
| **Missed merge** | truth: same person, system made two Persons ❌ (false negative) |
| **Correct unique** | truth: different people, system kept them apart ✅ |
| **Wrong merge** | truth: different people, system merged them ❌ (false positive — most dangerous class) |

Outputs: per-scenario verdict table + run totals (precision, recall, accuracy), plus the
actual graph for drill-down. `expect.*` blocks give exact-value assertions on top
(channel counts, canonical stored values) — a failing exact assertion is a scenario FAIL;
a mismatch in pair classification lowers the scorecard.

## 6. Runner semantics

1. Namespace every contact value per run (e.g. `qa+<runid>-<ref>@example.com`, and a
   unique digit tail on phones) so re-runs never collide.
2. Fire records in declared order through their drivers. Auth: practice-scoped JWT for
   UI lanes; raw webhook POST for ingress lanes; direct function shim for
   `create_intake_from_email` (that lane has no HTTP trigger without a real inbox —
   the shim runs via `manage.py shell` on DEV).
3. Wait for async effects (Celery tasks, signal propagation) with a per-scenario timeout
   and poll-based settle detection.
4. Snapshot the DB graph, run the scorer, render the report.

## 7. Web UI (localhost:8090)

FastAPI backend + single-page frontend (same visual language as the existing diagram
pages). Three tabs:

- **Compose** — dials: number of unique persons, families (and sizes), duplicates per
  archetype, per-record entry point, format-noise knobs, practice + country. Live preview
  table of the records to be sent, with truth badges (same-person groups highlighted).
- **Run** — fires the batch in order with live progress.
- **Scorecard** — precision / recall / accuracy, per-pair verdict table, and the actual
  Person/Channel/Household graph built by the system, mismatches highlighted red.

The generator can also export any composed batch as a YAML scenario, so the CLI
(`python -m engine run scenarios/*.yaml`) and the UI share one fixture format. Reports keep
history so score movement across fixes is visible.

## 8. Isolation & safety

- Everything runs inside a dedicated **QA Ingress Engine practice** on **DEV**
  (`localhost:5432`). Never PROD; never writes to SIM.
- **Practice targeting:** a run can optionally target a DIFFERENT practice
  (`ingress-engine run --practice-slug <slug>`): the JWT is minted for that practice
  (the QA superuser has all-practice access), the DB snapshot scopes to it, and webhook
  lanes auto-discover that practice's existing `IntakeWebhook` row (uuid + plaintext
  secret) via read-only DB; lanes without a usable webhook row for the target practice
  fail with a clear per-driver error. Custom webhooks only store a pbkdf2 hash, so
  targeted mode supports JWT lanes + the legacy webhook lane only.
- **Wipe:** every run's rows are identifiable by the run's email marker
  (`qa+s<seed>-<ns>-`). `ingress-engine wipe --practice-slug <slug> --marker <prefix>`
  (and the web UI's Wipe button) deletes exactly those rows: record rows by marker →
  channels reachable via those persons whose canonical matches the marker OR whose
  person loses its last record → persons (ONLY persons whose every linked record is
  inside the wipe set) → households left empty. Defaults to **dry-run** (prints the
  plan + counts); `--execute` performs the deletion. Wipe outside a `qa*` practice
  additionally requires `WIPE_ANY_PRACTICE=1` in the environment. Deletion runs
  through the QA-gated Django management command (`qa_ingress_wipe`, `QA_SHIM=1`) so
  it obeys the same access constraints as the ingest shim.
- DB assertions use a **read-only** Postgres role where possible (write access is
  needed only by setup/wipe/shim).
- One-time setup command creates the QA practice and a QA staff user + JWT secret
  (`python -m engine setup`).
- Test data is clearly synthetic (marker emails `@qa.example.com`), so it can be spotted
  and swept later.

## 9. Entry-point driver matrix

The driver list is the full entry-point inventory from `docs/ingest-web-diagram.html`
(`ep-*` / `go-*` nodes), so every lane the system actually has is testable:

| Diagram node | Driver | How it fires | V |
|---------------|--------|--------------|---|
| ep-legacy | `legacy_webhook` | POST webhook URL | 1 |
| ep-custom | `custom_webhook` | POST webhook URL | 1 |
| ep-mkt | `form_handoff` | POST form handoff endpoint | 1 |
| ep-ai | `ai_email` | `create_intake_from_email` shim via `manage.py shell` | 1 |
| ep-add | `add_patient` | API with practice JWT | 1 |
| ep-crud | `patients_crud` | API with practice JWT | 1 |
| ep-family | `family_member` | API with practice JWT | 1 |
| ep-move / ep-conv | `move_convert` | API with practice JWT (covers journey conversions) | 1 |
| ep-csv | `csv_import` | upload endpoint / management command shim | 1 |
| ep-dq | `dq_import` | DQ re-import via chokepoint | 1 |
| ep-auto | `automations` | trigger automations action with practice JWT | 1 |
| ep-book | `booking` | booking payment/abandoned-session tasks (task or API trigger) | 1 |
| ep-msg | `message_session` | inbound sender webhook (Twilio/SendGrid-shaped payload) | 1 |
| ep-restore | `archive_restore` | restore endpoint / ORM shim | 1 |
| ep-merge | `merge_unweld` | merge/unweld API | 1 |
| ep-backfill | `backfills` | management commands scoped to QA practice | 1 |
| ep-admin | `admin_edit` | admin save via ORM shim (signals fire, form layer does not) | 1 |
| ep-bot | — (upstream source, not a write path; leads arrive via webhook lanes) | — | — |
| go-sync | `go_sync` | raw-SQL migration execution | 2 |
| go-call | `go_callagent` | GORM intake insert | 2 |
| go-wf | `go_workflows` | delegates to Django REST — reuse other drivers | 2 |

Driver shims (ai_email, csv_import, booking, restore, admin, backfills) that have no clean
HTTP trigger run via `manage.py shell` exec on the DEV server against the QA practice;
drivers that are pure HTTP use signed requests. ep-bot is deliberately excluded: the widget
writes to its own DB and reaches TreatmentPath only through the webhook lanes, which
already have drivers.

**V1 (this project):** generator, all V1 drivers above (the complete Django-lane surface),
scorer, CLI runner, HTML reports, web UI with the three tabs.

**V2 (later):** Go-lane drivers (go-sync, go-call, go-wf), consent-flag archetypes,
history-trend charts in the UI, scenario diffing between runs.

## 10. Success criteria

1. `python -m engine setup` prepares the QA practice; `python -m engine run` executes all
   shipped scenarios and prints a scorecard.
2. A deliberately-broken matcher (e.g. reverting the sibling-adoption fix locally) makes
   the affected archetype's score drop — the harness demonstrably catches regressions.
3. The UI can compose an arbitrary mix (e.g. 3 families + 5 duplicates across 3 entry
   points), run it, and show the graded verdict table.
4. Adversarial families A1–A3 pass on all V1 lanes, and the canary mutation test
   demonstrably lowers the score.

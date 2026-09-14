# End-to-end audit runbook — everything changed since Thu 2026-09-03

**Purpose:** prove that ~95 patient-identity fixes plus a consolidation sweep did not break
the product. Work top to bottom. Every step says what to run, what PASS looks like, and
what to do on failure.

**Scope of change under audit**

| repo | branch | commits since 2026-09-03 | uncommitted |
|---|---|---|---|
| `TreatmentPathBackend` | `dedup-normalization-unification` | 259 | 12 modified + 11 new |
| `EmailServiceGo` | `contact-identity-fix2` | 52 | none |
| `perfect-pixel-playground-project` | `April27` | 36 | none |

**Golden rule for this audit:** a failure is only a finding once you have shown it is NOT
in the known-pre-existing list in §6. Several of these fail for environmental reasons and
have done for weeks.

---

## 0a. Orientation — read this first if you have no prior context

**The system.** TreatmentPath is a multi-tenant dental practice-management platform.
Three codebases share ONE Postgres database:

| repo | what it is | path |
|---|---|---|
| `TreatmentPathBackend` | Django 5.2 REST API — the source of truth | `TreatmentPathBackend/TreatmentPath/` (note the doubled dir; `manage.py` lives in the inner one) |
| `EmailServiceGo` | Go service (Gin): email relay, call agent, Dentally sync — **writes the same tables** | `EmailServiceGo/` |
| `perfect-pixel-playground-project` | React + TypeScript + Vite frontend | `perfect-pixel-playground-project/` |

**The domain model you must understand to audit this.**

- A **Practice** is a tenant. The SAME human legitimately exists once per practice, so
  merging or matching ACROSS practices is always a bug.
- A **Person** is meant to be ONE human. `Person → PersonChannel → ContactChannel`, where a
  ContactChannel is one phone number or one email address.
- `Patient`, `Intake` and `Nurture` records point at a Person.
- **A channel can belong to several People** — families share a phone and a mailbox. That is
  legal and normal, and it is the root of almost everything audited here.

**What went wrong, in one sentence.** Code that already KNEW which human it was dealing
with (it held a `patient_id`) threw that away and RE-RESOLVED identity from a name, phone
or email — and on a shared family channel that picks an arbitrary relative or invents a
duplicate. Internally this is called **CARRY vs DISCOVER**.

**The rule the fixes apply.** Where identity is genuinely ambiguous, the system must
REFUSE to name anyone rather than guess. Helpers named `sole_person_for_channel`,
`sole_patient_for_person` and `sole_record_for_person` return the one candidate, or `None`.
So **a blank name in the UI is often the CORRECT new behaviour**, not a bug — see the
warning below.

### ⚠️ Four ways to file a false report here

1. **"A name is missing."** On a shared family line the system now deliberately shows a
   phone number or a blank instead of guessing a relative's name. Before reporting, check
   whether that channel has more than one Person (query in §5). Only report a blank where
   the channel has exactly ONE owner.
2. **"A test fails."** Check §6 first. Roughly 40 tests fail for environmental reasons
   (missing env var, absent seed data, stale fixtures) and have done for weeks.
3. **"Two test runs disagree."** Never run two Django suites at once — see §0. Re-run the
   failing test ALONE before believing it.
4. **"The frontend has 136 failures."** You included `.worktrees/`. Always pass the
   `--exclude` flags in §3.

### How to report a real finding

For each failure give: **area (B-number or test id) → the code it covers (from §7a) →
what you observed → exact reproduction → evidence (screenshot / network payload / test
output)**. The traceability matrix in §7a turns any failing step into a specific file and
a specific original defect, so state both.

---

## 0. Pre-flight

```bash
cd /home/mannie/Desktop/Projects/treatmentpath
source TreatmentPathBackend/venv/bin/activate        # ALWAYS, for any Django command
```

Rules that will otherwise waste your time:

- Backend tests **must** use `--keepdb`. Never `--noinput` — it destroys the persistent
  test DB and pre-existing duplicate migrations then block a rebuild.
- **Run only ONE Django test process at a time.** Two concurrent runs against the shared
  `--keepdb` database produce `DeadlockDetected`, and a script that calls
  `setup_databases()`/`teardown_databases()` leaves stale `django_content_type` rows that
  surface as `ForeignKeyViolation` in unrelated tests. Both look exactly like real bugs.
- Frontend test/scan commands **must exclude worktrees**: `--exclude "**/.worktrees/**"`.
  There are stale copies under `.worktrees/` AND `.claude/worktrees/`; both are gitignored
  and invisible to git but fully visible to any tool that walks the filesystem.
- `npx tsc --noEmit` at the repo root checks **nothing** and exits 0. Use
  `-p tsconfig.app.json`.

---

## 1. Automated backend suite

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test --keepdb 2>&1 | tail -40
```

Long (~15 min). PASS = no failures outside §6.

### 1a. The round-three fixes, as one fast block (55 tests, ~4s)

```bash
python manage.py test --keepdb \
  messaging.test_sibling_resolvers_refuse_ambiguity \
  activityLog.test_person_matches_target \
  marketingBroadcast.test_consent_records_real_client_ip \
  teamChat.test_display_name_honours_privacy_rule \
  TreatmentPlan.tests.test_str_never_renders_none \
  TreatmentPlan.tests.test_parity_fixtures_are_identical \
  Notes.test_label_stats_scope \
  Appointments.test_antispam_email_is_case_insensitive \
  automations.test_duplicate_conflict_finds_the_patient \
  dentallyIntegration.test_recall_channel_uses_canonical_email
```

**Expected: `Ran 55 tests ... OK`.** Any failure here is a genuine regression in this
week's work — investigate before anything else.

### 1b. The identity audit's own regression tests (~53 files)

```bash
python manage.py test --keepdb \
  TreatmentPlan.tests.test_resolve_callers_pass_dob \
  TreatmentPlan.tests.test_sole_patient_for_person \
  TreatmentPlan.tests.test_phone_lookup_key_parity \
  TreatmentPlan.tests.test_add_family_member_identity \
  TreatmentPlan.tests.test_csv_import_shared_contact \
  TreatmentPlan.tests.test_email_intake_no_placeholder_name \
  messaging.test_inbound_email_caller_auth \
  messaging.test_ws_practice_authorisation \
  messaging.test_template_send_practice_scope \
  Documents.test_send_consent_context_scope \
  onlineBooking.test_verified_prefill_shared_email \
  marketingBroadcast.test_bounce_uses_exact_recipient \
  Notes.tests.test_cross_practice_letter_scoping \
  Notes.tests.test_notes_by_patient_identity
```

`messaging.test_ws_practice_authorisation` fails when run alongside many other modules
(async/event-loop interaction) but passes alone — **8/8 OK**. Re-run it by itself before
reporting it.

### 1c. Static check — the class of bug tests do not catch

```bash
python -m pyflakes $(git -C .. diff --name-only HEAD | sed 's|^TreatmentPath/||' | grep '\.py$') | grep -i "undefined name"
```

**Expected: no output.** This exists because an extracted helper was called with a variable
that did not exist in that scope — every unit test passed (they called the helper
directly) and the endpoint would have 500'd on every request. pyflakes caught it in
milliseconds. Run this after ANY extraction/refactor.

---

## 2. Automated Go suite

```bash
cd ../../EmailServiceGo
go build ./... && echo BUILD-OK
go test -count=1 ./internal/callagent/... ./internal/workflows/actions/... \
                 ./internal/dentally/... ./pkg/phone/... ./pkg/personname/...
```

**Expected:** `BUILD-OK` and `ok` for each package.

**Cross-language parity — do not skip.** Go re-implements the identity keys, pinned by
fixtures that are DUPLICATED files:

```bash
cd /home/mannie/Desktop/Projects/treatmentpath
sha256sum TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/phone_fixtures.json \
          EmailServiceGo/pkg/phone/testdata/phone_fixtures.json
sha256sum TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/contact_key_fixtures.json \
          EmailServiceGo/pkg/personname/testdata/contact_key_fixtures.json
```

**Expected: each pair identical.** (`TreatmentPlan.tests.test_parity_fixtures_are_identical`
also enforces this — it was verified to fail on drift.)

---

## 3. Automated frontend suite

```bash
cd perfect-pixel-playground-project
npx vitest run --exclude "**/.worktrees/**" --exclude "**/.claude/worktrees/**" src/
```

Then the identity-specific ones (38 tests, must be 38/38):

```bash
npx vitest run --exclude "**/.worktrees/**" \
  src/hooks/intakeNameColumns.test.ts \
  src/hooks/practiceScopedCacheRegistry.test.ts \
  src/components/patients/patient-panel/patientNameFields.test.ts \
  src/components/patients/universal-search/phone.test.ts \
  src/hooks/usePatientWorkspace.test.tsx \
  src/hooks/useIntakeCaching.test.ts
```

Typecheck (judge by DELTA — there is a large pre-existing baseline):

```bash
sed '/ignoreDeprecations/d' tsconfig.app.json > /tmp/tsconfig.check.json
npx tsc --noEmit -p /tmp/tsconfig.check.json 2>&1 | wc -l    # baseline ≈ 789
```

---

## 4. BROWSER END-TO-END — the main event

Start the app, log in, then work through each flow. For every step: **open devtools,
watch the Network tab and the console.** A 500, a red console error, or a request that
silently returns `null` where a name should be, is a finding.

Record for each: PASS / FAIL / BLOCKED, plus a screenshot on FAIL.

### Setup

```bash
cd perfect-pixel-playground-project
ss -ltnp | grep -E ':(8080|8081|5173)'      # pick a free port
npm run dev -- --port <PORT>
```

If the port differs from the committed default, add it to `CORS_ALLOWED_ORIGINS` /
`ALLOWED_HOSTS` in the dev-only settings block — otherwise the page loads fine and every
API call fails silently with a CORS error visible only in the console.

Log in and confirm you land on an authenticated page. **A server that boots is not proof
the app works.**

---

### B1 — Patient name editing must not corrupt the name  *(#51, C2/C7)*

The single highest-risk regression area: 1,686 live patients have a name a first-space
split gets wrong (e.g. `Ken (Kenneth) | Judge`).

1. `/patients` → open any patient with a **space inside the first name** or a
   double-barrelled surname. If none is visible, search `(` in the patient search.
2. Open the patient panel → click the **name** field to open the editor.
3. **CHECK:** the two boxes show the STORED split, not a re-split of the display name.
   For `Ken (Kenneth) Judge`, first must read `Ken (Kenneth)` and last `Judge` —
   **not** `Ken` / `(Kenneth) Judge`.
4. Click **Save without changing anything.**
5. **CHECK (Network):** either NO PATCH is sent, or the PATCH body carries the unchanged
   stored values. A PATCH containing a re-split name is a FAIL.
6. Reload. The name must be byte-identical to step 2.
7. Now genuinely edit the surname, save, reload. The edit must persist.

**FAIL looks like:** the name silently changes on open-and-save.

### B2 — Journey tables edit modal  *(#51 remaining sites)*

1. `/journeys/intake` → row menu → **Edit** on a lead with a multi-word first name.
2. **CHECK:** first/last prefilled from the stored columns, as B1.
3. Save unchanged → reload → unchanged.
4. Repeat on a **custom journey stage** table (`/journey/*`).

### B3 — Universal search routes to the right record  *(#71)*

1. Use the universal search for a term matching BOTH a patient and an intake.
2. Click the **intake** result → **CHECK:** lands on `/patients/intake/<id>`, showing that
   intake — not a patient page for a different human.
3. Click the **patient** result → lands on `/patients/<patientId>/...`.
4. Search by **phone number** → result opens the right record.

**FAIL looks like:** an intake result opening some other patient's workspace.

### B4 — Inbox: messages attach to the right human  *(#64, #92)*

1. `/inbox` → open a conversation on a **shared family number** (one channel, several
   people). Find one via the data query in §5.
2. **CHECK the conversation title.** On an ambiguous shared line it must show the PHONE
   NUMBER or a blank — **never** a confidently-wrong relative's name.
3. Send a reply. **CHECK (Network):** the POST body includes `patient_id`.
4. Open that patient's **Activity** tab → the message appears against the right person.
5. Open a **single-owner** conversation → it must still show the person's name normally.

**FAIL looks like:** a conversation labelled "Grant Shimwell" over messages that open
"Good Morning Monique".

### B5 — Call log / task creation from the panel  *(#73, #74, #92)*

1. Patient panel → **Log a call** → save.
2. **CHECK (Network):** the payload carries `patient_id` / `intake_id` / `nurture_id`, and
   first/last name come from the stored columns (no `None`, no re-split).
3. Create a **task** from the inbox → confirm it links to the patient (not `patient_id: null`).

### B6 — Add to Nurture / Open Plan from the workflow panel  *(#65, #72)*

1. Open a patient in `/create/notes` or `/create/letters` (the assistant panel).
2. Workflow panel → **+ Nurture** → add.
3. **CHECK (Network):** payload contains `from_patient_id`.
4. **CHECK:** no duplicate Person is created — the new nurture appears under the SAME
   person, not a second copy. Verify in `/patients`.
5. Repeat for **+ Open** and **+ Active** (`patient_id` in `patient_data`).

### B7 — Practice switching clears every cache  *(#67, #69)*

The highest-severity frontend class: stale data from the PREVIOUS practice.

1. Log in as a user in **two** practices (head/child group, e.g. Mannie/Danbury).
2. Visit, in order: `/day-list`, `/recalls`, `/inbox`, `/journeys/intake`, WhatsApp config.
3. Switch practice with the **PracticeSwitcher** (the soft switch — do NOT reload).
4. Re-visit each page.
5. **CHECK:** every list shows the NEW practice's data. Any row from the previous practice
   is a FAIL — check the patient names against §5's query.
6. Repeat switching back.

### B8 — Day list and recalls  *(#2, #36, #69)*

1. `/day-list` → **CHECK:** each appointment's patient name renders correctly, no literal
   `"None"`, no `"Unnamed patient"`, no double spaces (e.g. `Mary  Jane`).
2. Confirm/cancel an appointment → the row updates live (websocket).
3. `/recalls` → open a recall → **CHECK** it opens the right person's record.

### B9 — Booking OTP prefill refuses a shared mailbox  *(#63)*

Online booking profiles are currently **disabled** for all 7 practices — if you cannot
enable one safely, mark BLOCKED rather than skipping silently.

1. Public booking page → enter an email shared by several patients (§5 query).
2. Complete OTP verification.
3. **CHECK:** name/phone fields come back **BLANK**, not filled with one relative's
   details.
4. Repeat with an email belonging to exactly ONE patient → prefill SHOULD appear.

### B10 — Consent send  *(#57)*

1. Patient panel → **Send consent** with a treatment plan selected.
2. **CHECK:** only that patient's OWN plans/appointments are selectable.
3. If you can craft a request with another patient's `treatment_plan_id` (devtools →
   edit-and-resend), the API must reject it with 400.

### B11 — Team chat shows "System Admin" for superusers  *(G13)*

1. Log in as a **superuser**, post a message in team chat.
2. Log in as an ordinary staff user in the same practice, open that conversation.
3. **CHECK:** the superuser appears as **"System Admin"**, not their real name.
4. **CHECK:** ordinary colleagues still show their real names.

### B12 — Marketing preferences records the real IP  *(C12)*

1. Open a marketing email's **unsubscribe / preferences** link.
2. Change the consent setting.
3. **CHECK (§5 query):** the new `MarketingConsent` row's `ip_address` is a real client
   address, **not** the load balancer's (`10.x`, `172.x`, or the nginx host).

### B13 — Activity history shows only this person's events  *(#17, #95)*

1. Patient panel → **Activity** tab for a patient whose Person is shared/fused.
2. **CHECK:** every entry belongs to THIS human.
3. **CHECK:** no entry names a different person than the record it points at.

### B14 — Notes and letters stay inside the practice  *(#48, #49, C11)*

1. `/create/notes` → **CHECK** the patient picker offers only this practice's patients.
2. Open an existing note → labels/statistics render.
3. As a user in TWO practices, switch practice and re-open the notes list → no note from
   the other practice appears.

### B15 — Smoke: nothing 500s

Click through every top-level nav item with the console open. Any 500, unhandled promise
rejection, or React error boundary is a finding regardless of area.

---

## 5. Data verification queries

Run these to FIND test subjects and to CONFIRM the fixes on real data.

```bash
cd TreatmentPathBackend/TreatmentPath && python manage.py shell
```

```python
from django.db.models import Count
from TreatmentPlan.models import Person, Patient, ContactChannel

# Shared family phone/email — subjects for B4, B9
Person.objects.filter(merged_into__isnull=True, person_channels__channel__kind="phone") \
    .annotate(n=Count("person_channels__channel", distinct=True)).filter(n__gt=1).count()

# Fused persons — subjects for B4, B13   (expect 44 nurture / 497 intake)
Person.objects.filter(merged_into__isnull=True).annotate(n=Count("nurtures", distinct=True)).filter(n__gt=1).count()
Person.objects.filter(merged_into__isnull=True).annotate(n=Count("intakes",  distinct=True)).filter(n__gt=1).count()

# Names a first-space split gets wrong — subjects for B1/B2
Patient.objects.filter(first_name__contains=" ").count()          # ~1,686 live

# No patient may render the literal "None"  (C2/C7)
sum(1 for p in Patient.objects.filter(last_name__isnull=True)[:50] if "None" in str(p))   # MUST be 0

# Activity logs whose person disagrees with their target  (#95) — 9 historical rows
# (the fix stops NEW ones; these 9 need a data decision)
```

Marketing consent IP (B12):

```python
from marketingBroadcast.models import MarketingConsent
MarketingConsent.objects.exclude(ip_address=None).order_by("-id").values("ip_address")[:5]
```

---

## 6. KNOWN PRE-EXISTING FAILURES — do not report these

Verified failing before this week's work, or environmental. **Check here before raising
anything.**

| where | symptom | why |
|---|---|---|
| `dentallyIntegration.test_integration_api_key` | 5 errors | `DENTALLY_ENCRYPTION_KEY` not set in the environment |
| `TreatmentPlan.tests.test_journey_business_workflows` | import error | imports `TreatmentPlan.journey_automation`, which does not exist |
| `TreatmentPlan.tests.test_dedupe_patients_patientdocuments` | 2 errors | fixture collides with the `patient_practice_dentally_uniq` constraint |
| `Notes.tests.test_notes.PhraseSeedTests` | 2 | seed data absent |
| `onlineBooking.tests` Stripe Connect | 3 failures / 5 errors | `PracticeStripeAccount() got unexpected keyword 'stripe_account_id'` |
| `Recall*` tests | ~22 failures | date-sensitive decay, pre-existing |
| `teamChat` avatar precedence | 1 | DEF-36-01 |
| `test_compat03_intake_webhook_baseline` | SHA mismatch | commit `08de331f` changed `intake_views.py` without rebaselining |
| Go `internal/email/api` | build failure | `ensureDefaultPracticeAddresses` signature drift (commit `cbc62c3`) |
| vitest, unscoped | ~136 failures | `.worktrees/` and `.claude/worktrees/` copies — DEF-37-01 |
| `tsc -p tsconfig.app.json` | ~789 errors | pre-existing baseline; judge by delta |

Also expect **`productAnalytics`** to abort the transaction on the first request in a test
that makes 2+ requests in one method — it has no migrations.

---

## 7. Deployment preconditions (verify BEFORE shipping)

Not test steps — ship-blockers.

1. **`WORKFLOW_SERVICE_SECRET` must be set on `email-worker`**, and **Go must deploy before
   Django**. Otherwise finding #52's caller-auth rejects **every inbound email** with 401.
2. `CALL_AGENT_TWILIO_SIGNATURE_MODE` ships as `monitor`. Only flip to `enforce` after the
   logs show sustained 100% signature matches — a mismatch rejects every inbound call.
3. Rotate `JWT_SECRET` (committed in the compose file) and `TWILIO_AUTH_TOKEN` (exposed in
   an earlier session).
4. Pending migrations: `TreatmentPlan 0145`, `medicalHistory 0005`,
   `marketingBroadcast 0038`, and on prod `dentallyIntegration 0168` (**destructive** —
   deletes 146 unattributable AI summaries) + `0169`.
5. `to-run-inprod/2026-09-06-patient-identity-audit-followups.txt` is **stale**: its steps 6
   and 7 cite "#48"/"#49", numbers since reassigned to different findings. Read it against
   the renumbering note in the audit doc before executing.

---

## 7a. Traceability matrix — failure → code → original defect

When a step fails, quote the row. It tells you which files to look at, what the bug
originally was, and what breaks in the real world if the fix has regressed.

| step | covers (files) | original defect | real-world symptom if broken |
|---|---|---|---|
| **B1** | `perfect-pixel-playground-project/src/components/patients/patient-panel/PatientEditableFields.tsx`, `src/utils/nameFields.ts`, `TreatmentPlan/models.py::Patient.__str__` | #51 — the editor received a JOINED display name and re-split it at the first space, so merely OPENING the field rewrote `Ken (Kenneth)\|Judge` as `Ken\|(Kenneth) Judge`. C2/C7 — `__str__` rendered the literal `"John None"` on nullable columns. | Patient names silently corrupted on view. 1,686 live patients are vulnerable. Corrupted names then fail to match on the next lookup, creating duplicates. |
| **B2** | `src/components/compact/IntakeTable.tsx`, `CustomJourneyTable.tsx`, `src/hooks/useIntakeCaching.ts`, `src/components/intake/types.ts` | #51 at the remaining five sites. The transform joined `first_name + last_name` and DISCARDED the columns, so every editor downstream had only the join. | Same corruption as B1, via the journey tables. |
| **B3** | `src/components/patients/universal-search/{phone.ts,types.ts,UniversalPatientSearch.tsx}` | #71 — every search result was routed as if it were a patient, so clicking an intake opened an unrelated patient's page. | Staff open the WRONG patient's record from search. |
| **B4** | `messaging/serializers.py`, `messaging/views/message_views.py`, `src/InboxPage.tsx`, `src/components/patients/patient-panel/PatientInboxTab.tsx` | #64 (sends carried no `patient_id`), #75/#76, #92 (`get_nurture_id` still used `.first()`; `_get_linked_intake_name` labelled a conversation from an arbitrary intake — 497 Persons hold >1 intake, worst holds 44). | A conversation is titled with the wrong family member; replies file against the wrong human; no activity history. |
| **B5** | `src/components/patients/patient-panel/usePatientPanelController.ts`, `messaging/serializers.py` | #73/#74 — a call log dropped the known patient id and re-resolved by phone, so on a family landline it attached to the wrong relative. | Clinical call history on the wrong patient. |
| **B6** | `src/components/WorkflowPanel.tsx`, `src/pages/Conpact3.tsx`, `TreatmentPlan/serializers/nurture.py` | #65/#72 — "Add to Nurture" re-resolved identity instead of carrying `from_patient_id`, minting a duplicate Person. | Duplicate patient records created by ordinary staff actions. |
| **B7** | `src/hooks/useCachedData.ts` (registry), `useDayListData.ts`, `useRecallsData.ts`, `useDormantRecalls.ts`, `useWhatsAppConfig.ts` | #67/#69 — module-level caches were not re-scoped on the soft practice switch. | **Cross-tenant data leak**: practice A's patients shown while practice B is selected. |
| **B8** | `dentallyIntegration/next_appointment.py`, `dentallyIntegration/recall_automation.py` | #2 (recall list opened the wrong person), C6 (a private `_display_name` copy that outlived its bug and drifted — it no longer collapsed whitespace). | Wrong patient opened from the recall list; malformed names on the day list. |
| **B9** | `onlineBooking/views.py::_build_verified_response` | #63 — `.first()` on a shared email returned an arbitrary relative's name, surname and phone to whoever proved control of the mailbox. Practice 21 has NINE Taylors on one address. | One family member is shown another's personal details. |
| **B10** | `Documents/views/signing_views.py::_validate_consent_context` | #57 — `appointment_id` / `treatment_plan_id` were written with no check, so a consent form could cite another patient's treatment. | Medico-legal: a signed consent naming the wrong person's treatment. |
| **B11** | `teamChat/services.py`, `TreatmentPath/utils.py::get_display_name_for_user` | G13 — teamChat had its OWN display-name helper without the canonical one's rule that superusers render as "System Admin". 83 of 86 sites bypassed it. | Superuser real names exposed where they were meant to be anonymous. |
| **B12** | `marketingBroadcast/views/preferences_views.py`, `utils/request_meta.py` | C12 — one of eleven client-IP helpers read `REMOTE_ADDR` only. Behind nginx that is the load balancer, and it fed the consent audit row. | The consent audit trail records the same IP for every patient — worthless as evidence of who opted out. |
| **B13** | `activityLog/serializers.py`, `activityLog/views.py` | #17 (history showed another person's events), #95 (person and target validated against the practice independently, never against each other — 9 live rows already disagree). | A patient's clinical activity timeline contains another human's events. |
| **B14** | `Notes/views/note.py`, `Notes/views/letters.py`, `Notes/services/labels.py`, `utils/practice_mixins.py::scope_clinical_to_practice` | #48/#49 (clinical records gated on the RECORD's practice, not the PATIENT's), C11 (label stats gated on the AUTHOR's practice membership — 22 of 64 rows double-counted). | Clinical notes visible across practices; inflated statistics. |
| **B15** | everything | — | Any 500 is a regression. |
| **§1a** | the 11 round-three fixes | #92, #95, C1, C2/C7, C6, C8, C9, C11, C12, C13, G12, G13 | These are this week's newest changes and the least exercised. |
| **§1c** | every changed `.py` | An extracted helper was called with an out-of-scope variable name — every unit test passed because they called the helper directly; the endpoint would have 500'd on every request. | A whole endpoint down, invisible to unit tests. |
| **§2 parity** | `pkg/phone`, `pkg/personname` vs `TreatmentPlan/utils/` | Go re-implements the identity keys; the pinning fixtures are DUPLICATED files. | Go and Django compute different keys for the same patient → duplicates that no test catches. |


---

## 8. Report template

```
AUDIT — end-to-end sweep, changes since 2026-09-03
Auditor:              Date:              Build/commit:

AUTOMATED
  Django full suite         PASS / FAIL   failures outside §6: ___
  Round-three block (55)    PASS / FAIL
  Identity regressions      PASS / FAIL
  pyflakes undefined names  PASS / FAIL
  Go build + tests          PASS / FAIL
  Parity fixtures identical PASS / FAIL
  vitest (scoped)           PASS / FAIL
  tsc delta vs ~789         ___

BROWSER            result   notes/screenshot
  B1  name editing
  B2  journey edit modals
  B3  universal search routing
  B4  inbox attribution
  B5  call log / task
  B6  nurture / open plan
  B7  practice switch caches
  B8  day list / recalls
  B9  booking OTP prefill
  B10 consent send
  B11 team chat superuser
  B12 marketing consent IP
  B13 activity history
  B14 notes/letters scoping
  B15 smoke — no 500s

DATA CHECKS
  Patients rendering "None"        expect 0 → ___
  Fused-person subjects found      ___
  Consent IP is a client address   YES / NO

FINDINGS (excluding §6)
  #  severity  area  what  repro  evidence

VERDICT:  SAFE TO SHIP  /  SHIP WITH CAVEATS  /  BLOCKED
```

---

## 9. What this audit deliberately does NOT cover

State it in the report rather than implying coverage.

- **#91** — a reply from a shared family line still stops a RELATIVE's recall sequence.
  Not fixed; awaiting a product decision. 3,138 shared phones / 2,884 shared emails.
- **#94** — the duplicate-merge UI can weld two humans (9 live clusters). Not fixed;
  awaiting a UI decision.
- **9 historical activity-log rows** already carry a wrong person. The fix stops new ones;
  these need a data decision.
- **Apps never swept:** `Tasks`, `medicalHistory`, `patientDocuments`, `patient_accounts`,
  `Invoices`, `Labs`, `dedupe_audit`, `practiceConnection`. Regressions there would not
  have been caught by the sweep and may not be caught here either.
- **Consolidation not yet applied:** G1 (65 practice gates), G2 (48 permission classes),
  G9 (websocket group names built by hand at both ends), G10 (10 duplicated serializer
  getters — the structural cause of #92), G11 (18 URL builders).

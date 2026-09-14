# Frontend end-to-end audit — everything our fixes touched

**Goal:** drive the app in a real browser and confirm every change we made since
Thu 2026-09-03 works end to end. **Nothing here is about deployment.** This is purely
"does the product still behave correctly for a user, given what we changed".

**What was changed:** 31 frontend source files across 36 commits, plus ~50 backend changes
whose effects surface in the UI. Every one with a user-visible surface has a flow below.

---

## 0. Your brief — READ FIRST

**The person who commissioned this audit is NOT available.** Nobody will answer questions,
unblock you, or confirm a judgement call. Plan accordingly.

**What that means in practice:**

1. **Run it to the end.** Do not stop at the first failure and wait. Record it, continue,
   finish every flow. A partial audit that stalled on flow 3 is worth far less than a
   complete one with 4 failures documented.
2. **Document as you go, not at the end.** Write each result the moment you have it. If you
   run out of context or time, whatever is written must still stand on its own.
3. **Decide, and say what you decided.** When something is ambiguous, make the most
   reasonable call, act on it, and record BOTH the decision and the reasoning. Do not
   silently skip. "Marked BLOCKED because the booking profiles are disabled and enabling
   one looked risky" is a good outcome; a blank row is not.
4. **You are free to do independent work.** This runbook is a floor, not a ceiling. If you
   notice something off that no flow covers — a slow page, a console error, an odd payload,
   a name that looks wrong — chase it and write it up. Findings outside the script are
   often the valuable ones. Add your own flows if you spot an untested path.
5. **Verify, don't assume.** Every claim in your report needs evidence: a screenshot, a
   network payload, a console line, a query result. "Looks fine" is not a result.
6. **Be honest about what you did not do.** Flows you could not run, areas you ran out of
   time for, things you were unsure about — list them explicitly under "Not covered". An
   audit that overstates its coverage is worse than one that admits gaps.

**Safety is the one hard boundary.** Section 0b is not optional: confirm
`TCP connections attempted: 0` before any send flow. If you cannot get the sandbox running,
mark the three send flows BLOCKED and do the other fifteen. Never send from an unprotected
environment — this database holds 60,537 real patients.

**Deliverable:** a completed report (§5 template) saved as a file, with every flow marked,
every finding evidenced, and your own observations included.


---

## 0a. Orientation — read before testing, or you WILL file false reports

**The domain.** A **Practice** is a tenant. A **Person** is meant to be one human.
`Person → PersonChannel → ContactChannel` (one phone or one email). `Patient`, `Intake` and
`Nurture` records point at a Person. **A channel can belong to several People** — families
share a phone and a mailbox. That is normal, and it is the source of everything below.

**What we fixed, in one sentence.** Code that already knew which human it was dealing with
threw that away and re-resolved identity from a name, phone or email — which on a shared
family channel picks an arbitrary relative or invents a duplicate.

### ⚠️ A BLANK NAME IS USUALLY THE CORRECT NEW BEHAVIOUR

This is the single most important thing to understand before testing.

Where identity is genuinely ambiguous — a phone line with four family members on it — the
system now **refuses to name anyone** rather than guessing. You will see phone numbers
where you used to see names, and empty name fields where one used to be filled.

**That is the fix working, not a bug.**

Only report a blank when the record has exactly ONE owner. §7 tells you how to tell the
difference in the UI without database access.

### Three other ways to file a false report

1. **A double space collapsing.** `Mary  Jane` now renders `Mary Jane` everywhere. Intended.
2. **`Unnamed patient` appearing.** Better than the old literal `"None"` / `"None Smith"`.
   Report only if a patient who HAS a name shows it.
3. **A stale-looking list right after switching practice.** Reload once. If it is correct
   after reload but wrong before, that IS a real finding (F9) — note the difference.

---

## 0b. 🚨 Sandbox mode — REQUIRED before any send flow

**Is the SMS/Twilio sandbox currently active? NO.** Nothing is listening on 8000, 8010,
8080 or 8090, and `OUTBOUND_SANDBOX` is unset. It is not a always-on service — it is a
mode you start deliberately, and this audit needs it.

### Why it matters here

| fact (checked 2026-09-08) | value |
|---|---|
| default local DB `treatmentpath_db` | **60,537 patients, 39,884 with an email address** |
| `OUTBOUND_SANDBOX` in a normal shell | **unset** |
| `TWILIO_ACCOUNT_SID` in a normal shell | the **real** one, not the `ACsandbox…` stub |
| the fail-closed guard at `settings.py:1285` | **does NOT fire for `treatmentpath_db`** |

That guard refuses to boot against restored production data, but it matches on the database
NAME only (`prod_`, `rehearsal`, `pristine`, `control`). `treatmentpath_db` matches none of
them, so **it stays silent even though the data is real**. Run a send flow against it with
the sandbox off and real patients receive real messages.

### The established setup (this is how it was done on 2026-09-06 — reuse it)

Two dedicated ports so nothing collides with a normal dev session on 8000/8080:

**1 — Django sandbox on 8010, against `prod_rehearsal2`** (verified present):

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend
env $(grep -v '^#' .env.sandbox | xargs) \
  python TreatmentPath/manage.py runserver 127.0.0.1:8010 --noreload
```

`.env.sandbox` sets `OUTBOUND_SANDBOX=1`, `DB_NAME=prod_rehearsal2`,
`OUTBOX_PATH=/tmp/treatmentpath-outbox.jsonl` and deliberately INVALID Twilio credentials,
so even a bypassed interceptor fails authentication instead of delivering.

**Confirm it armed** — the startup log must contain:

```
OUTBOUND SANDBOX ACTIVE: twilio, requests
```

If that line is absent, STOP. You are not in sandbox mode.

**2 — Vite frontend on 8090, pointed at 8010:**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vite --mode sandbox --port 8090
```

`--mode sandbox` loads `.env.sandbox`, which sets `VITE_API_BASE_URL` to
`http://127.0.0.1:8010/api/backend`. Django whitelists the 8090 origin **only when
`OUTBOUND_SANDBOX` is on** — so if login fails with a CORS error in the console, the
backend is not in sandbox mode.

Browse to **http://127.0.0.1:8090**.

### The outbox IS your test oracle

Nothing is delivered, so verify sends by reading the outbox rather than an inbox:

```bash
curl -s http://127.0.0.1:8010/api/backend/_sandbox/outbox/ | python3 -m json.tool   # read
curl -s -X DELETE http://127.0.0.1:8010/api/backend/_sandbox/outbox/               # clear
```

Clear it before each send flow, perform the action, then read it back and check the
recipient, body and channel are what you expected. **This is how F6/F14/F16 are graded.**

### How much to trust it

A socket-level tripwire was installed on `socket.socket.connect` and an SMS was sent to a
real mobile number with the body *"SANDBOX PROOF - if you receive this, the sandbox
FAILED."* The Twilio client returned a normal queued response with a SID, and **zero TCP
connections were attempted** — nothing left the machine. Interception happens below the
HTTP layer, so it cannot be bypassed by using a different HTTP library.

### ✅ Stack verified running — 2026-09-08

Booted and checked end to end before this audit began. If you are picking this up later,
re-verify each line rather than assuming:

| check | result |
|---|---|
| Django sandbox `127.0.0.1:8010` | up; log shows `OUTBOUND SANDBOX ACTIVE: twilio, requests` |
| database | `prod_rehearsal2` |
| outbox API `GET /api/backend/_sandbox/outbox/` | HTTP 200 |
| Vite `127.0.0.1:8090` (`--mode sandbox`) | HTTP 200 |
| CORS preflight from origin `:8090` | `access-control-allow-origin: http://127.0.0.1:8090` |
| login `POST /api/backend/auth/login/` | HTTP 200, JWT for user 66 |
| real-number send + socket tripwire | **0 TCP connections attempted** |

**Two setup corrections made — you would otherwise hit both:**

1. **The frontend `.env.sandbox` did not exist** (it is gitignored and was lost since the
   2026-09-06 run). Without it, `--mode sandbox` silently loads nothing and Vite points at
   `:8000` — the NON-sandbox backend, which is not even running. It has been recreated
   with `VITE_API_BASE_URL`, `VITE_API_URL` and `VITE_WEBHOOK_BASE_URL` rewritten from
   `:8000` to `:8010`. **Confirm the file exists before starting Vite.**
2. **The login endpoint is `POST /api/backend/auth/login/`** — not `/UserAuthentication/`
   and not `/user/`. `auth/practice-login/` also exists but requires a `practice_slug`.

Browse to **http://127.0.0.1:8090** and log in with the §0c credentials.

### Testing SMS with REAL patient numbers, safely

You can and should test against real patient numbers — that is the only way to exercise
the real identity paths (shared family lines, mangled country codes, `use_sms` flags).
Nothing leaves the machine. Five independent layers make that true:

| # | layer | what it does |
|---|---|---|
| 1 | **Twilio transport patch** | `TwilioHttpClient.request` is replaced. Production builds Twilio clients in **eight** different places, each a lazy `from twilio.rest import Client` inside a function — patching `Client` would miss any site that already bound the name, and patching call sites would miss the ninth one added next month. Every client instance funnels through this one method, so one patch is airtight. |
| 2 | **HTTP host guard** | `requests` calls to `api.twilio.com`, `api.postmarkapp.com`, `api.sendgrid.com` and `graph.facebook.com` (WhatsApp Cloud) are blocked, plus the Go email-service paths by URL path — so a staging hostname cannot slip past. |
| 3 | **Invalid credentials** | `.env.sandbox` supplies `ACsandbox0000…` / `sandboxtoken`. If a layer were ever bypassed, the call fails **authentication** instead of delivering. |
| 4 | **Fail-closed boot guard** | Django refuses to start against a database whose name looks like restored production unless the sandbox is on. |
| 5 | **The outbox** | Every intercepted message is written to `OUTBOX_PATH` with channel, recipient and body — so you can assert on exactly what *would* have been sent. |

**Verified live on 2026-09-08.** A socket-level tripwire was installed on
`socket.socket.connect`, then an SMS was sent to a real patient pulled straight from the
database:

```
real patient: id=9596 'Johanna Warren'  ->  +447944247585
twilio returned:  SM3c53fbcaeb8740928a3ad49ca224d285  queued
outbound TCP connections attempted:  0  -> NONE
outbox captured:  {'channel': 'sms', 'to': '+447944247585', 'body': 'PROOF - ...'}
```

The Twilio client returned a normal queued SID — production code cannot tell the
difference — while **not a single packet left the machine**. Interception happens below
the HTTP layer, so switching HTTP library would not route around it.

#### Prove it yourself before you trust it

Do this once at the start of the audit. Never take the protection on faith:

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend
source venv/bin/activate
env $(grep -v '^#' .env.sandbox | xargs) python - <<'EOF'
import os, django, socket, sys
os.environ.setdefault('DJANGO_SETTINGS_MODULE','TreatmentPath.settings')
sys.path.insert(0, 'TreatmentPath'); django.setup()
from django.conf import settings
from TreatmentPlan.models import Patient
p = Patient.objects.exclude(phone_number="").filter(country_code="+44").first()
target = f"{p.country_code}{p.phone_number}"
egress = []
real = socket.socket.connect
socket.socket.connect = lambda self, a: (egress.append(a), (_ for _ in ()).throw(AssertionError(a)))[0]
try:
    from twilio.rest import Client
    m = Client(settings.TWILIO_ACCOUNT_SID, settings.TWILIO_AUTH_TOKEN).messages.create(
        to=target, from_="+447700000000",
        body="PROOF - if this arrives, the sandbox FAILED.")
    print("twilio said:", m.sid, m.status)
finally:
    socket.socket.connect = real
print("TCP connections attempted:", len(egress), "<- MUST be 0")
EOF
```

**`TCP connections attempted: 0` is the only acceptable result.** Anything else: stop the
audit and do not send.

#### Which numbers to use for the identity flows

Pick deliberately, not at random — these are the shapes the fixes are about:

```python
from django.db.models import Count
from TreatmentPlan.models import Person, Patient

# a SHARED family line (several Persons on one channel) — the core case
Person.objects.filter(merged_into__isnull=True, person_channels__channel__kind="phone") \
  .annotate(n=Count("person_channels__channel", distinct=True)).filter(n__gt=1)[:5]

# a patient whose name breaks a first-space split
Patient.objects.filter(first_name__contains=" ").exclude(phone_number="")[:5]
```

After each send, read the outbox and check **who** it was addressed to — on a shared line
the interesting question is not "did it send" but "did it pick the right human".

#### Do NOT do this instead

- **Do not edit patient phone numbers to a test number.** That mutates the identity data
  the audit is measuring, and `prod_rehearsal2` is the control copy.
- **Do not rely on `use_sms=False` as protection.** 1,832 patients with invalid phones are
  all flagged `use_sms=True`; the flag is not a safety net.
- **Do not test sends against `treatmentpath_db`.** It has the same real data with none of
  the protection (see the table above).

### If you cannot start it

Mark **F6, F14 and F16 as BLOCKED**. Every other flow (F1–F5, F7–F13, F15, F17, F18) is
read-only or writes only internal records, and is safe against the normal dev server.

> Note: `prod_rehearsal2` is a restored production COPY. It is the right target for
> messaging tests (real-shaped data, senders intercepted). If you audit against
> `treatmentpath_db` instead, you get the same data shape with **no protection** — so only
> do that for the read-only flows.

---

## 0c. Test practice and login credentials

### 🎯 The test practice is **Danbury Dental Care** (practice id 16)

Do all testing in **Danbury Dental Care** unless a flow explicitly says otherwise. It is
the practice the identity audit used throughout, so the known test subjects — shared family
phones, fused Persons, multi-word first names — are the ones documented here.

`manifestkelvin@gmail.com` already starts in Danbury Dental Care, so you should not need to
switch. **If the practice switcher shows something else, switch to Danbury before starting.**

The other two practices on that account exist for ONE purpose: **F9**, the practice-switch
cache test. Switch to `practicedemo` or `Practice Mannie`, then back, and confirm no data
from the other practice survives. Outside F9, stay in Danbury.

> `practicedemo` is a demo practice and `Practice Mannie` is a test practice — records there
> (e.g. a patient literally named `Ethan` with no surname) are seeded, not real. Do not
> report their oddities as findings.

### Login credentials

Two accounts exist with nearly identical addresses. **You need both** — note the `a` vs `q`.

| account | password | type | practices | use it for |
|---|---|---|---|---|
| `manifestkelvin@gmail.com` | `maniZolas1008` | admin | **3** — **Danbury Dental Care (the test practice)**, practicedemo, Practice Mannie | **the main audit account.** Starts in Danbury; the two extra practices exist for F9 only |
| `mqnifestkelvin@gmail.com` (note the **q**) | *(ask — not supplied)* | **superuser** | 2 — Danbury Dental Care, Practice Mannie | **F13 only.** That flow needs a message posted BY a superuser to check it renders as "System Admin" |

`manifestkelvin@` is user id 66 and starts in **Danbury Dental Care** (practice 16). This is the
account the 2026-09-06 sandbox run used, and the password below is already set on
`prod_rehearsal2`. If login fails on that database, reset it:

```bash
env $(grep -v '^#' .env.sandbox | xargs) python TreatmentPath/manage.py shell -c \
  "from django.contrib.auth import get_user_model as G; u=G().objects.get(email='manifestkelvin@gmail.com'); u.set_password('maniZolas1008'); u.save(); print('reset', u.id, u.current_practice)"
```

If you only have the first account, F13 can still be half-tested: log in as the admin and
confirm ordinary colleagues show their real names — but you cannot confirm the superuser
masking without a superuser posting. Mark it PARTIAL.

---

## 1. Setup

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
git rev-parse --abbrev-ref HEAD          # expect: April27
ss -ltnp | grep -E ':(8080|8081|5173)'   # find a free port
npm run dev -- --port <PORT>
```

If your port is not the committed default, add it to `CORS_ALLOWED_ORIGINS` /
`ALLOWED_HOSTS` in the backend's dev-only settings block. Otherwise the page loads, looks
fine, and **every API call fails silently** — visible only as a CORS error in the console.

Log in and confirm you land on an authenticated page.

**Keep devtools open for the whole audit** — Network tab and Console. Half the checks below
are "what was in the request body", not "what is on screen".

### Quick automated pre-check (2 minutes)

```bash
npx vitest run --exclude "**/.worktrees/**" --exclude "**/.claude/worktrees/**" \
  src/hooks/intakeNameColumns.test.ts \
  src/hooks/practiceScopedCacheRegistry.test.ts \
  src/components/patients/patient-panel/patientNameFields.test.ts \
  src/components/patients/universal-search/phone.test.ts \
  src/hooks/usePatientWorkspace.test.tsx \
  src/hooks/useIntakeCaching.test.ts
```

**Expect 38/38.** The `--exclude` flags are mandatory: without them you will see ~136
phantom failures from stale copies in `.worktrees/` and `.claude/worktrees/`.

---

## 2. Finding test subjects

Several flows need a patient whose name breaks a naive split, or a phone shared by a
family. Find them in the UI:

- **Multi-word first name** — universal search for `(` . Real examples in this data:
  `Ken (Kenneth) Judge`, `Christopher (Chris) Mooney`, `Kathy (Kathleen) Coleman`.
  Also try `Mary Jane`, `Sarah Jane`, `James Edward`, `Lyn-Marie`.
- **Shared family phone/email** — in `/inbox`, look for conversations titled with a bare
  PHONE NUMBER instead of a name. That title IS the system telling you the line is shared.
  Also try surnames that repeat: `Taylor`, `Peacock`, `Patchett`, `Harrod-Attfield`.
- **A fused Person** — a patient whose panel shows several intakes or nurtures.

Note the ones you find at the top of your report; later flows reuse them.

---

## 3. BROWSER FLOWS

Each flow: **route → steps → PASS → FAIL signal → code it covers**.
Mark every flow PASS / FAIL / BLOCKED. Screenshot every FAIL.

---

### F1 — Opening the name editor must not corrupt the name
**Route:** `/patients` → any patient with a multi-word first name
**Covers:** `PatientEditableFields.tsx`, `src/utils/nameFields.ts`, `patientPanelTypes.ts`
**Original defect (#51):** the editor got a joined display name and re-split it at the
first space, so `Ken (Kenneth)|Judge` became `Ken|(Kenneth) Judge`. Merely OPENING the
field corrupted the record. 1,686 live patients are vulnerable.

1. Open the patient panel, click the **name** field.
2. **PASS:** first box reads `Ken (Kenneth)`, last box reads `Judge`.
   **FAIL:** first `Ken`, last `(Kenneth) Judge`.
3. Click **Save** without typing anything.
4. **PASS (Network):** either NO PATCH fires, or the PATCH body carries the original
   values. **FAIL:** a PATCH containing the re-split name.
5. Reload. Name byte-identical to step 2.
6. Now really change the surname → save → reload → the edit persisted.

---

### F2 — Journey table edit modals
**Route:** `/journeys/intake`, then a custom stage under `/journey/*`
**Covers:** `IntakeTable.tsx`, `CustomJourneyTable.tsx`, `useIntakeCaching.ts`, `intake/types.ts`
**Original defect (#51):** the data transform joined the name columns and DISCARDED them,
so the edit modal had only the join to work from.

1. Row menu → **Edit** on a lead with a multi-word first name.
2. **PASS:** first/last prefilled from the stored columns (as F1).
3. Save unchanged → reload → unchanged.
4. Repeat on a custom-stage table.

---

### F3 — Add Patient / Add Open Plan dialogs
**Route:** `/journeys/intake` → **Add**; `/journeys/openplans` → **Add**
**Covers:** `AddPatientModal.tsx`, `AddOpenPlanDialog.tsx`
**Original defect (#68):** the UI sent the literal string `"Unknown"` as a first name,
which then became a real identity key — two unrelated nameless people merged because both
were called "Unknown".

1. Create a record leaving the name blank (if the form allows).
2. **PASS:** the created record shows a BLANK name.
   **FAIL:** it shows `Unknown`, `Unknown Lead`, or `(unknown)`.
3. Create a second nameless record with a DIFFERENT email → **PASS:** two separate
   records, not merged into one.

---

### F4 — Universal search opens the right record
**Route:** the global search box
**Covers:** `UniversalPatientSearch.tsx`, `universal-search/phone.ts`, `types.ts`
**Original defect (#71):** every result was routed as if it were a patient, so clicking an
intake opened an unrelated patient's page.

1. Search a term matching both a patient and an intake.
2. Click the **intake** result → **PASS:** `/patients/intake/<id>`, showing that intake.
   **FAIL:** a patient workspace for a different human.
3. Click the **patient** result → `/patients/<patientId>/...`.
4. Search by **phone number** → opens the right record.
5. Search a name with a double space → still finds the patient.

---

### F5 — Inbox conversations are labelled honestly
**Route:** `/inbox`
**Covers:** `InboxPage.tsx`, `PatientInboxTab.tsx`, backend `messaging/serializers.py`
**Original defect (#79, #92):** a conversation on a shared line was titled with an
arbitrary member — a task said "Monique Shimwell" while the inbox said "Grant Shimwell",
over messages opening "Good Morning Monique". `_get_linked_intake_name` picked one of up
to 44 intakes.

1. Open a conversation on a **shared family number**.
2. **PASS:** title shows the PHONE NUMBER or a blank.
   **FAIL:** a confident single name where the line has several owners (verify via §7).
3. Open a **single-owner** conversation → **PASS:** the person's name shows normally. If
   this is blank, that IS a regression.

---

### F6 — Sending a message links it to the right human
**⚠️ SANDBOX REQUIRED (§0b).** Clear the outbox first, then grade the send by reading it back — not by looking for a delivered message.
**Route:** `/inbox` → reply; and the patient panel's **Inbox** tab
**Covers:** `InboxPage.tsx`, `PatientInboxTab.tsx`, `inbox/types.ts`
**Original defect (#64):** sends carried no `patient_id`, so the backend re-resolved by
address and attached replies to whichever relative it found first.

1. Send a reply from `/inbox`.
2. **PASS (Network):** POST body contains `patient_id`.
3. Send from the patient panel's Inbox tab → same check.
4. Open that patient's **Activity** tab → the message is there, against the right person.

---

### F7 — Call logging and clinical plan creation
**Route:** patient panel → **Log call**; patient panel → **Create treatment plan**
**Covers:** `usePatientPanelController.ts`
**Original defect (#73/#74):** the call log dropped the known patient id and re-resolved by
phone, so on a family landline it attached to the wrong relative; names were built by
splitting the display name.

1. Log a call → **PASS (Network):** payload carries `patient_id` / `intake_id` /
   `nurture_id`, and first/last are the stored columns — no `None`, no re-split.
2. **Create treatment plan** from the panel → the plan opens for the RIGHT patient with the
   name intact.

---

### F8 — Workflow panel: Add to Nurture / Open / Active
**Route:** `/create/notes` or `/create/letters` with a patient selected → workflow panel
**Covers:** `WorkflowPanel.tsx`, `Conpact3.tsx`, `NotesEditor.tsx`, `LetterAssistant.tsx`,
`DocumentAssistant.tsx`, `useTreatmentPlanCreation.ts`
**Original defect (#65/#72):** these re-resolved identity instead of carrying the id,
minting a duplicate Person for someone who already existed.

1. Select a patient with a multi-word first name.
2. **+ Nurture** → **PASS (Network):** payload contains `from_patient_id`.
3. **+ Open** and **+ Active** → payload contains `patient_id` inside `patient_data`.
4. **PASS:** the new record appears under the SAME person — no duplicate in `/patients`.
5. **PASS:** the name on the new record is the stored split, not a first-space guess.

---

### F9 — Practice switching clears every cache  ⚠️ highest severity
**Route:** the PracticeSwitcher (sidebar user profile)
**Covers:** `useCachedData.ts` (registry), `useDayListData.ts`, `useRecallsData.ts`,
`useDormantRecalls.ts`, `useWhatsAppConfig.ts`, `usePatientWorkspace.ts`
**Original defect (#67/#69):** module-level caches were not re-scoped on the soft switch,
so one practice's data stayed on screen under another practice.

You need a user in TWO practices (a head/child group such as Mannie / Danbury).

1. Visit in order: `/day-list`, `/recalls`, dormant recalls, `/inbox`, `/journeys/intake`,
   WhatsApp config. Note a few patient names on each.
2. Switch practice with the **PracticeSwitcher** — do NOT reload.
3. Re-visit each page.
4. **PASS:** every list shows the NEW practice's data.
   **FAIL:** any row from the previous practice — a cross-tenant leak. Record the page and
   the leaked name.
5. Switch back and repeat.
6. Note whether a manual reload fixes it (that distinguishes a cache bug from a data bug).

---

### F10 — Day list renders names correctly
**Route:** `/day-list`
**Covers:** backend `dentallyIntegration/next_appointment.py` (C6), `Patient.__str__` (C2/C7)
**Original defect:** a private display-name copy that outlived the bug it was written for
and had drifted — it no longer collapsed internal whitespace.

1. **PASS:** no literal `None` / `None Smith` / `John None` anywhere.
2. **PASS:** no double spaces inside a name (`Mary  Jane`).
3. **PASS:** patients with no surname show just the first name, not `X None`.
4. Confirm/cancel an appointment → the row updates live (websocket) without a reload.

---

### F11 — Recalls open the right person
**Route:** `/recalls`
**Covers:** `useRecallsData.ts`, `useDormantRecalls.ts`
**Original defect (#2):** the recall list opened the wrong person's record.

1. Open several recalls, including one whose phone is shared by a family.
2. **PASS:** each opens the patient named in the row.
3. Check the Dormant tab too.

---

### F12 — Patient workspace
**Route:** `/patients/:patientId/*`
**Covers:** `PatientWorkspacePage.tsx`, `usePatientWorkspace.ts`, `PatientPanelOverviewTab.tsx`
1. Open a patient → every tab (Overview, Notes, Inbox, Activity, Documents) loads.
2. **PASS:** the name in the header matches the record everywhere on the page.
3. **PASS:** the Activity tab shows only THIS person's events (#17, #95).

---

### F13 — Team chat hides superuser identity
**Route:** team chat
**Covers:** backend `teamChat/services.py`, `TreatmentPath/utils.py` (G13)
**Original defect:** teamChat had its own display-name helper missing the rule that
superusers render as "System Admin". 83 of 86 sites bypassed it.

1. As a **superuser**, post a message.
2. As an ordinary staff user, open that conversation.
3. **PASS:** the superuser shows as **"System Admin"**.
   **FAIL:** their real name.
4. **PASS:** ordinary colleagues still show real names.

---

### F14 — Consent sending stays with one patient
**⚠️ REQUIRES `OUTBOUND_SANDBOX=1` — see §0b.**
**Route:** patient panel → **Send consent**
**Covers:** backend `Documents/views/signing_views.py` (#57)
1. **PASS:** only this patient's own treatment plans / appointments are offered.
2. Send one → it appears on this patient's record only.

---

### F15 — Notes and letters stay inside the practice
**Route:** `/create/notes`, `/create/letters`
**Covers:** backend `Notes/views/*`, `Notes/services/labels.py` (#48, #49, C11)
1. **PASS:** the patient picker offers only this practice's patients.
2. Open an existing note → labels and any statistics render.
3. As a user in two practices, switch and re-open → no note from the other practice.

---

### F16 — Online booking (if a profile can be enabled)
**⚠️ REQUIRES `OUTBOUND_SANDBOX=1` — see §0b (booking sends a confirmation).**
**Route:** the public booking page
**Covers:** backend `onlineBooking/views.py` (#63, G12)
All 7 booking profiles are currently **disabled**. If you cannot safely enable one, mark
**BLOCKED** — do not skip silently.

1. Verify by OTP with an email shared by several patients.
2. **PASS:** name/phone come back BLANK. **FAIL:** one relative's details prefilled.
3. Repeat with a single-owner email → prefill SHOULD appear.
4. **PASS:** practitioner names never render as `None Smith`.

---

### F17 — Duplicate-contact review screen  ⚠️ known unfixed risk
**Route:** the data-quality / duplicates screen
**Covers:** backend `dataQuality/views.py` (#94 — NOT FIXED)
**Do not merge anything.** This flow is observation only.

1. Open a duplicate cluster.
2. **Record whether any cluster contains two clearly DIFFERENT people** — e.g. four members
   reading `Joseph Harrold-Attfield`, `Amelie Harrod-Attfield`, `Joseph Harrod-Attfield`,
   `Amelie Harrod-Attfield` (that is two siblings, each duplicated).
3. Note whether the UI warns before merging, or offers one-click "merge all".
4. Report what you see — this informs a pending decision.

---

### F18 — Smoke: nothing 500s
Click every top-level nav item with the console open. Any 500, unhandled promise rejection,
or React error boundary is a finding regardless of area.

---

## 4. Not observable in the browser

State these as untested rather than implying coverage:

| change | why the UI cannot show it |
|---|---|
| C12 — marketing consent records the real client IP | the value lands in a DB audit column; needs a DB read |
| #52 — inbound email webhook auth | server-to-server |
| #82/#85–#89 — Go call-agent fixes | server-side; needs a real inbound call |
| C1 — Go/Django parity fixtures | a build-time guard |
| #91 — recall stop-on-reply | **NOT FIXED**, and invisible: the sequence just silently ends |

---

## 5. Report template

```
FRONTEND E2E AUDIT — changes since 2026-09-03
Auditor:            Date:            Branch/commit:
Port:               Test practice: Danbury Dental Care (16)
Login used:         OUTBOUND_SANDBOX enabled?  YES / NO   (if NO: F6/F14/F16 = BLOCKED)

TEST SUBJECTS FOUND
  multi-word first name:
  shared family phone:
  shared family email:
  fused person:

PRE-CHECK   vitest 38/38   PASS / FAIL

FLOW                                   RESULT    NOTES / SCREENSHOT
F1  name editor
F2  journey edit modals
F3  add patient / open plan
F4  universal search routing
F5  inbox labelling
F6  send links to right human
F7  call log / plan creation
F8  workflow panel journeys
F9  practice switch caches      ← highest severity
F10 day list names
F11 recalls open right person
F12 patient workspace
F13 team chat superuser
F14 consent send
F15 notes/letters scoping
F16 online booking            (likely BLOCKED)
F17 duplicate review          (observation only)
F18 smoke — no 500s

FINDINGS
  #  flow  severity  observed  expected  repro steps  evidence

VERDICT:  WORKS END TO END  /  WORKS WITH ISSUES  /  BROKEN
```

---

## 6. Severity guide

- **CRITICAL** — data from another practice visible (F9), or a record saved against the
  wrong human (F1, F6, F7, F8).
- **HIGH** — the wrong record opens (F4, F11), or a confident wrong name (F5).
- **MEDIUM** — a name renders badly (`None`, double space) but nothing is mis-filed.
- **LOW** — cosmetic.

---

## 7. Telling a CORRECT blank from a BUG, without database access

The fixes deliberately blank a name when identity is ambiguous. To decide which you are
looking at:

1. In `/inbox`, open the conversation. Does the panel show **more than one linked patient**,
   or a family/household indicator? → the blank is CORRECT.
2. Open the patient panel for that number. Are there **several patients sharing it**?
   → CORRECT.
3. Search the phone number in universal search. **More than one result** → CORRECT.
4. **Exactly one owner and still blank → that is a REAL FINDING.** Report it with the
   record id.

Same logic for email addresses and for prefill in F16.

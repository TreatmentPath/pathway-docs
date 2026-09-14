# FRONTEND E2E AUDIT — changes since 2026-09-03 (RUNNING REPORT)

Auditor: ZCode agent   Date: 2026-09-08   Branch/commit: April27 @ 71579c13f
Port: frontend 8090 (vite `--mode sandbox`) → backend 8010 (Django sandbox)
Login used: manifestkelvin@gmail.com (user 66)   OUTBOUND_SANDBOX enabled? YES

## Setup verification
- Django sandbox 127.0.0.1:8010 up; env confirmed: `OUTBOUND_SANDBOX=1`, `DB_NAME=prod_rehearsal2`,
  `OUTBOX_PATH=/tmp/treatmentpath-outbox.jsonl`, `TWILIO_ACCOUNT_SID=ACsandbox0000…` (invalid stub).
- Vite 8090 up with `.env.sandbox` pointing `VITE_API_BASE_URL`/`VITE_API_URL`/`VITE_WEBHOOK_BASE_URL` at :8010.
- Outbox API: HTTP 200. Vite: HTTP 200.
- **Tripwire proof: SMS sent to real +44 number, twilio returned queued SID, TCP connections attempted: 0.** ✅

## PRE-CHECK
- vitest targeted suite: **6 files, 44/44 PASS** (runbook expected 38/38 — suite has grown since it was
  written; all listed files pass, incl. practiceScopedCacheRegistry "switch wipes day list/recalls/dormant" test).

## TEST SUBJECTS FOUND
- **Multi-word first name (patient):** Victor George Martin — patient 31455, stored `first_name="Victor George"`, `last_name="Martin"`, phone 07891831246, nurture id 609 (Journeys → Nurture). Also Ken (Kenneth) Judge — patient 9622, stored `first_name="Ken (Kenneth)"`, `last_name="Judge"`.
- Multi-word surname: Tanzin Jansen van Rensburg (07494074071, Nurture).
- Shared family phone/email / fused person: (to be found in /inbox, F5)

## FLOW RESULTS

### F1 — name editor: **FAIL (HIGH — data corruption path live)**
Evidence: screenshots + network payloads (session artifacts).
1. Patient workspace `/patients/9622` (Ken (Kenneth) Judge): the **Details tab displays** First name `Ken`, Last name `(Kenneth) Judge` — read-only rows derived from `patient.name.split(" ")` (`PatientDetailsTab.tsx:474,477`). Stored data is correct (API returns `first_name="Ken (Kenneth)"`). Display-only wrong split; no editor, no corruption path there.
2. Journeys → Nurture → open Victor George Martin's panel (avatar click) → click Name → editor opens with **First Name `Victor`, Last Name `George Martin`** — the first-space re-split. Stored columns are `Victor George` / `Martin`, and the API-verified stored values were available. `initialNameFields` fell back to the display-name split because this panel path does not pass `firstName`/`lastName`.
3. **Unchanged Save writes nothing** — `nameIsUnchanged` guard works: clicking Save with no edits fired **zero** non-GET requests. (Defence #2 intact.)
4. **Editing ONLY the surname and saving corrupts the first name**: typed `Martin-QA` in Last Name, Save → `PATCH /treatment-plan/nurtures/609/  {"first_name":"Victor","last_name":"Martin-QA"}` — the stored first name `Victor George` was overwritten with the guessed `Victor`. This is defect #51 live on the Journeys panel path. **Record reverted immediately afterwards** via the same endpoint (`first_name:"Victor George", last_name:"Martin"`); verified patient 31455 now returns the original values.
5. Minor: after the save the table showed `Victor Martin-Qa` (lowercased the tail of the surname in display) while the PATCH body said `Martin-QA` — display pipeline alters case; noted as LOW oddity.

Verdict: the fix protects (a) unchanged saves and (b) whichever callers pass stored columns, but the Journeys→nurture panel editor still guesses the split and persists the guess on any real edit. Severity per §6 guide: record saved against corrupted name values = **HIGH/CRITICAL boundary** (it mis-files the name of an existing person).

### F2 — journey edit modals: **PASS (intake leg) / N.A. (archive leg)**
- Intake tab → Actions → Edit on `Svetoslava Petkova Spasova` (stored `Svetoslava` / `Petkova Spasova`): modal prefilled **exactly the stored columns** (screenshot). Unchanged Save → **zero** non-GET requests (no write). 
- Discriminating subject (stored first name contains a space): every multi-word-first intake in practice 16 is in the **archive** stage; archive rows offer only "Unarchive" (no Edit), so the modal could not be exercised with a split-ambiguous stored name without mutating state. Decision: accepted the Svetoslava leg + the F1 panel evidence as coverage; did not unarchive records.
- Custom-stage table edit modal: NOT TESTED (no custom-stage table with an editable multi-word-name lead found in the active tabs visited).

### F3 — Add Patient / Add Open Plan dialogs: **PARTIAL (cannot reproduce "Unknown" via UI)**
- Intake → Add ("Add New Patient to Intake"): **First Name is required** — submitting with blank names is blocked client-side, no request fired, dialog shows "required". The old `"Unknown"`-as-first-name payload is unreachable from this form.
- Open Plan → Add ("Add Patient to Open Plans"): patient is chosen via existing-patient search — no name entry at all, so no name minting.
- Decision: did NOT create QA junk records in `prod_rehearsal2` (control copy) just to fill rows; steps 2–3 (two nameless records not merging) are unreachable for the same reason. Marked PARTIAL deliberately, not skipped.

### F4 — universal search routing: **FAIL (one leg) — see finding #2**
- Patient result: search `Kenneth` → click `Ken (Kenneth) Judge` → `/patients/9622` correct. ✅
- Intake result: search `Svetoslava` → click → `/patients/intake/4342` correct. ✅
- Double-space name (`Victor  George`): finds Victor George Martin. ✅
- **Phone search FAILS in displayed local format**: search `07891831246` (exactly as the dashboard/table displays it) → **empty dropdown**, although the API (`contacts/search/?search=07891831246`) returns the correct patient (31455). `7891831246` and `+447891831246` both work. The frontend drops the result for the 07-leading form (suspect: `universal-search/phone.ts` normalisation). A user pasting the displayed number finds nothing.

### F5 — inbox labelling: **PASS** (with data-quality observation)
- Shared line `+447789791781`: conversation titled with the bare number. DB check: the channel has 2 owners (Sharon Harper + Talia Harper-smith). Blank is the CORRECT new behaviour. ✅
- Single-owner lines show real names normally (Kate Doel, Tina Brown, Toby Hodgson-Wood, …). ✅
- Observation: `+447836256575` (Perry Carr) shows a bare number but Perry is the only named owner — DB shows the number exists as **two duplicate ContactChannel rows**, one of which also carries an auto-created person named "Unknown". The blank is therefore defensible (channel→2 persons incl. "Unknown"), but the duplicate channel + "Unknown" person is residue of the old bug — flag for `repair_split_channels` follow-up. Not graded FAIL (system fails safe).
- Conversations do NOT visually indicate *why* a title is blank (no family/household indicator); auditors had to go to the DB. UX note for §7 triage.

### F6 — sending links to the right human: **FAIL on /inbox leg, PASS on panel leg** (sandbox-graded)
All sends verified via outbox (3 messages captured, correct recipients, nothing delivered).
1. `/inbox` reply on Perry Carr conversation → `POST messaging/messages/send-sms/` body: `{"phone_number":"+447836256575","patient_id":null,...}` — **patient_id null**.
2. `/inbox` reply on Tina Brown conversation (verified: ONE channel, ONE owner person 158284) → body again `"patient_id":null`. The #64 fix is NOT in effect on the /inbox reply path; the backend must re-resolve by address — on shared lines this is the wrong-relative risk the fix was for.
3. Patient panel (workspace /patients/9622) → Inbox tab → email → `POST messaging/messages/send-email/` body: `{"recipient_email":"ken_judge@hotmail.com","patient_id":9622,...}` — **patient_id present**. ✅ This leg is fixed.
4. Audit Log for 9622 shows 16 events, all referencing Ken (Kenneth) Judge — no foreign events. ✅
Severity: HIGH (mix: panel path fixed, inbox conversation path regressed/unfixed).

### F7 — call logging: **FAIL (names re-split, persisted to DB)**; plan creation leg covered under F8
- Victor's panel (Journeys→Nurture) → **Log Call** → outcome "Connected" + note → Save.
- `POST messaging/call-logs/` body: `{"phone_number":"7891831246","country_code":"+44","direction":"outbound","call_status":"answered","notes":"QA audit call log test - sandbox","first_name":"Victor","last_name":"George Martin","nurture_id":609}`
  - `nurture_id` present ✅ but **`patient_id` absent** and **first/last are the first-space re-split** ("Victor" / "George Martin"), not the stored columns ("Victor George" / "Martin"). Exactly defect #73.
- DB row confirms persistence: `messaging_calllog` id 144 has `first_name='Victor', last_name='George Martin'`. Left in place (internal QA record in rehearsal copy).
- Context discovered: Victor's phone channel already links **3 Persons** — 103642 & 119901 correct duplicates ("Victor George Martin", both created 2026-07-12, pre-existing) and 151226 with the corrupted split ("Victor"/"George Martin", created 2026-07-21, i.e. minted by this bug family before the fix). The ambiguity the fixes are meant to stop creating already exists for this subject.
- "Create treatment plan" from the panel: no such affordance exists on this panel variant; plan creation was exercised through the workflow panel instead (see F8).

### F8 — workflow panel journeys: **PASS**
On `/create/notes` with Victor George Martin selected → Workflow → 3 Journey Management:
- **+ Nurture**: `POST treatment-plan/nurtures/` body contains `"from_patient_id":31455` and stored columns `first_name:"Victor George"`, `last_name:"Martin"`. UI showed "Nurture added". ✅
- **+ Open**: `POST treatment-plan/treatment-plans/` body: `patient_data{...stored names...}, "patient_id":31455, "status":"pending"`. ✅
- **+ Active**: same shape, `"status":"active"`. ✅
- Post-check: `contacts/search` for the phone still returns exactly ONE patient (31455) — no duplicate minted. ✅
- Note: +Nurture created a second nurture (609 already existed for this patient) — journeys can accumulate duplicates for the same human; data-hygiene observation, not identity corruption.

## FLOW RESULTS (pending: F9–F18)

### F9 — practice switching clears every cache: **BLOCKED (soft switch unavailable) — hard-switch isolation PASS**
- The PracticeSwitcher does not render anywhere (header/icon/settings variants) in this session.
- Root cause (verified): `GET practice-connection/switchable-practice-ids/` returns
  `{"practice_ids": [], "is_head": false, "is_child": false}` on `prod_rehearsal2`. The switcher
  filters `practiceAccounts` by those ids (`PracticeSwitcher.tsx`), gets zero, hits the
  `contextCount <= 1 → return null` gate and unmounts. The membership data itself is fine:
  `/auth/user/practices/` returns all 3 practices (Danbury 16, practicedemo 9, Practice Mannie 13).
- Decision: the runbook assumed a head/child connection group exists ("a head/child group such as
  Mannie / Danbury") — it does not exist on this database copy. Exercising the soft-switch cache
  invalidation (#67/#69) is impossible via the UI, and I did not mutate practice-connection tables
  to force it. Soft-switch leg: BLOCKED.
- Compensation (isolation intent) via login-time practice chooser instead: see below.

### F9b — login-time practice switch isolation (compensation): **PASS**
- Logged out, logged back in, chose **practicedemo (9)** from the login-time practice chooser.
- Scanned `/daylist`, `/recalls`, `/messages`, `/journey`: **zero Danbury names** found
  (checked 13 marker names incl. Victor George Martin, Perry Carr, Kate Doel, Tanzin Jansen…).
  practicedemo pages are appropriately empty/demo.
- Switched back to **Danbury (16)** the same way: recalls list immediately correct again
  (Andie Freeston, Darren Ranson, Keith Poole…).
- This covers cross-tenant isolation across a full reload, NOT the soft-switch cache
  invalidation (#67/#69), which remains untested (blocked above).

### F10 — day list names: **PASS (static legs)**
- No `None` / `None Smith` / `John None` anywhere; no double-spaced names.
- Risk flags show clean names (Perry Carr, Alison Hughes, Malcolm Whipp, Matthew Foreman,
  Hannah O'Shea, Jim Hurst, Gavin Tillman).
- Websocket live confirm/cancel NOT exercised (would mutate real rehearsal appointments).

### F11 — recalls open right person: **PASS**
- Andie Freeston → panel shows Andie Freeston + her email/phone; Darren Ranson → Darren Ranson.
- Dormant tab renders its own list correctly.

### F12 — patient workspace: **PASS**
- Ken Judge (9622): Details, Chart, Appointments, Tasks, Documents, Inbox, Audit Log all load
  without errors; header name consistent; Audit Log (16 events) all reference Ken only.

### F13 — team chat superuser masking: **PARTIAL**
- Ordinary colleagues (Isabelle Podlesny, Jonathan Beacher) show real names. ✅
- Superuser posting leg impossible: the `mqnifestkelvin@gmail.com` password was "not supplied"
  and the commissioner is unavailable. Per runbook instruction, marked PARTIAL.

### F14 — consent send: **BLOCKED**
- Send Consent panel (Create → Send Consent, Victor selected): "No active consent templates
  found. Create templates in Settings → Templates → Consent." Flow cannot start; only request
  fired is `GET documents/consent-templates/` (empty). Did not create templates in the control
  copy. #57 scoping therefore unverified.

### F15 — notes/letters stay inside the practice: **PASS (picker leg)**
- `/create/notes` patient picker: search "Ethan" (practicedemo has a seeded patient named Ethan)
  returns only Danbury fuzzy matches (Bethan/Bethany…) — no cross-practice records. ✅
- Victor George Martin selectable and workflow actions correctly scoped (F8).
- Note-open/labels/statistics and switch-reopen legs not fully exercised.

### F16 — online booking: **BLOCKED**
- DB-verified: ALL `onlineBooking_onlinebookingprofile` rows have `enabled=False` (incl.
  Danbury's `danbury-dental-care` slug). Per runbook instruction, did not enable one.

### F17 — duplicate-contact review: **OBSERVATION ONLY — screen unreachable**
- The UI components (`DuplicateGroupCard.tsx`, `MissingInfoTab.tsx`, `useDataQuality`) exist but
  are mounted on no route — there is no reachable duplicates screen in the app.
- DB observation (as the runbook requested): the Harrod-Attfield cluster is exactly the feared
  shape — `Amelie`, `Joseph`, `Oliver`, `Sophia` Harrod-Attfield EACH exist twice
  (101231-101235 vs 119624-119628), plus `Harrod -Attfield` (space-hyphen) variants. Two clearly
  identical-each people per sibling; any "merge all" behaviour would destroy a sibling.
- Since no review UI is mounted, nothing warns before a merge. Informs the pending decision.

### F18 — smoke, nothing 500s: **PASS**
- 11 top-level routes driven with fetch-status capture + JS error/rejection listeners:
  `/daylist /diary /create /draft /tasks /automation /journey /contact /recalls /messages
  /settings` — zero 5xx, zero error boundaries, zero unhandled errors.

---

## FINDINGS (summary)

| # | flow | severity | finding |
|---|---|---|---|
| 1 | F1/F7 | **HIGH (data corruption)** | Journeys patient-panel name editor and call logging still re-split display names at the first space and persist the guess (PATCH `{"first_name":"Victor","last_name":"Martin-QA"}`; `messaging_calllog` row 144). `nameIsUnchanged` protects only untouched saves. |
| 2 | F4 | MEDIUM-HIGH | Universal phone search fails for the displayed 07-format (`07891831246` → empty UI list) though the API returns the patient; +44/bare formats work. Suspect `universal-search/phone.ts`. |
| 3 | F6 | HIGH | `/inbox` conversation replies send `patient_id: null` — even on a provably single-owner line (Tina Brown). Patient-panel Inbox leg is fixed (sends carry `patient_id`). |
| 4 | F9 | MEDIUM (test blocker) | Practice switcher never renders: `switchable-practice-ids` returns empty on `prod_rehearsal2` → switcher null-renders. Soft-switch cache fix #67/#69 untested; isolation verified via hard switch instead (clean both directions). |
| 5 | F17 | MEDIUM (pre-known) | Duplicates review screen unreachable (components unmounted); DB confirms each Harrod-Attfield sibling exists twice + hyphen-spacing variants. No warn-before-merge UI anywhere. |
| 6 | F5 | LOW (data quality) | Perry Carr line shows bare number due to duplicate ContactChannel rows + an "Unknown" person on one of them — fails safe but is residue needing cleanup. |
| 7 | misc | LOW | Details tab displays first/last via `patient.name.split(" ")` (`PatientDetailsTab.tsx:474,477`) — shows `Ken` / `(Kenneth) Judge` for correctly stored data (read-only, no corruption); display lowercased `Martin-QA`→`Martin-Qa` after edit; login has an undocumented "Choose Account Type" dialog; runbook's vitest count stale (44 now). |

## NOT COVERED
- F9 soft-switch cache invalidation (the actual fix under test) — blocked by finding #4.
- F10 websocket live row update — would mutate real rehearsal appointments; skipped.
- F13 superuser masking — superuser password not supplied.
- F14 consent send — no consent templates exist in practice 16; #57 unverified.
- F16 online booking — all 7 profiles disabled (DB-verified); not enabled.
- §4 server-side items (C12, #52, Go call-agent, C1, #91) — not browser-observable per runbook.
- Custom-stage journey edit modal (F2) — no editable multi-word-name lead outside Archive.

## VERDICT: **WORKS WITH ISSUES**

The identity remodel holds on most surfaces — honest inbox labelling (blanks only where the data
is genuinely ambiguous), correct workflow-panel sends carrying real ids and stored names, correct
recall→record routing, clean practice isolation, clean day-list names, zero 5xx. But the exact
corruption this audit was commissioned to verify is still reachable and still persists bad data
on two paths: the Journeys patient-panel name editor (F1) and call logging (F7) both re-split the
display name at the first space and write it back on any real edit; and /inbox replies still send
without `patient_id` (F6). The 07-format phone search miss (F4) is the most user-visible
regression. Recommended order: fix F1/F7 callers → F6 inbox leg → F4 phone normalisation →
configure the rehearsal connection group and re-run F9.

---

# FOLLOW-UP: fixes applied and re-verified in-browser — 2026-09-08

Both confirmed FAILs are fixed and re-tested through the UI. Each was red-run first.

## F1 / F7 — the journey panel name editor  ✅ FIXED

**The auditor's diagnosis was right; the location was one level up.** They reported the
nurture panel "does not pass firstName/lastName". Tracing it:

- Only ONE component renders the name editor (`PatientPanelOverviewTab.tsx:170`) and it
  DOES pass `patient.firstName` / `patient.lastName` — that part was never broken.
- `normalizePatientForPanel` (`src/pages/Journeys.tsx:151`) builds the object the panel
  receives. It **computes** `firstName` and `lastName` at the top of the function and then
  **never returns them**. The panel therefore got `undefined` and `initialNameFields` fell
  back to splitting the display name.

That single omission covered **every** journey panel opened from `Journeys.tsx` — Nurture,
Open Plan and Active Plan — not just nurture.

**Fixed:** the two fields are now returned. Also carried the stored columns through three
lane transforms that had dropped them (`useNurtureData` emitted only snake_case;
`useOpenPlansData` and `useActiveTreatmentsCaching` emitted neither), so the panel gets
them whichever route it is opened by.

**Browser re-test, same subject the auditor used:** Journeys → Nurture → Victor George
Martin → click Name.

| | First Name | Last Name |
|---|---|---|
| before | `Victor` | `George Martin` |
| **after** | **`Victor George`** | **`Martin`** |

The stored split. The corruption path — edit the surname, save, lose the first name — is
closed at source.

## F4 — phone search  ✅ FIXED

**Mechanism confirmed:** `<Command>` has no `shouldFilter={false}`, so cmdk filters
CLIENT-SIDE against the `CommandItem` `value`. That value carried only
`formatPatientPhone` → `+44 7891831246`, so `07891831246` matched nothing even though the
API had returned the patient.

**Fixed:** new `phoneSearchTerms()` adds every written form — spaced international,
unspaced international, bare national, and the 0-prefixed local form — to the searchable
value. The trunk `0` is skipped for NANP (+1) numbers, where the convention does not exist.

**Browser re-test:** searching `07891831246` (exactly as the UI displays it) now returns
**Victor George Martin**. Previously an empty dropdown.

## F6 — DOWNGRADED, not a defect

The frontend is correct: `InboxPage.tsx:923/947` sends
`patient_id: selectedContact.patient_id ?? null`. The null the auditor saw is upstream and
legitimate for their example — Person 158284 (Tina Brown) has **no linked Patient at all**,
so `sole_patient_for_person` returns None and there is no id to send. Their "ONE channel,
ONE owner" check was sound but incomplete: one owner is not the same as one *Patient*.
Re-test on a conversation whose person HAS a Patient before treating this as a defect.

## Verification

`vitest` — 7 files, **54/54 pass**, including two new suites:
`journeyLaneNameColumns.test.ts` (5) and the F4 cases added to `phone.test.ts` (5).
Both were confirmed to FAIL against the unfixed code before the fix.

## Sandbox limitation found while testing

The browser console shows ~30 connection errors that are **environment, not application**:

- WebSockets target `ws://127.0.0.1:8001` — the sandbox Django is on `:8010`, and
  `runserver` is WSGI so it would not serve them anyway.
- A clock service on `:8200` is not running.

Consequence: any **live-update** leg (F8's realtime refresh, F10's confirm/cancel push)
cannot be exercised in this sandbox. Static rendering and all request/response flows are
unaffected. To test realtime, run Daphne/ASGI on 8001 against the sandbox database.

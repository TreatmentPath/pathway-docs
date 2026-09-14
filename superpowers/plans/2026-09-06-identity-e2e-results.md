# Patient-Identity E2E Verification — Results Log

Running log for the plan at `2026-09-06-identity-e2e-verification.md`.
Verdicts: **HELD** (fix works), **RE-CREATED** (bug still reproducible), **NOT TESTED**.

## Environment

| Item | Value |
|---|---|
| Django (sandbox) | `http://127.0.0.1:8010` — DB `prod_rehearsal2`, `OUTBOUND_SANDBOX=1` |
| Frontend (sandbox) | `http://127.0.0.1:8090` — `vite --mode sandbox` |
| User's own stack (untouched) | Vite 8080 → Django 8000 → `treatmentpath_db` |
| Login | `manifestkelvin@gmail.com`, user 66, practice 16 (Danbury Dental Care) |
| Outbox | `GET/DELETE http://127.0.0.1:8010/api/backend/_sandbox/outbox/` |

## Task 1 — Outbound sandbox: PASS

Two-sided empirical proof, tripwire on `socket.socket.connect` (the syscall every
outbound TCP connection must traverse):

| Condition | TCP connects | Outcome |
|---|---|---|
| Sandbox ON | **0** | `sid=SMa3f7bb38…`, `status=queued` — app believes it sent |
| Sandbox OFF (control) | **1** → `52.85.27.9:443` | Proves the tripwire genuinely fires |

Target number for the proof was the user's own real mobile `+23467390962`;
nothing was delivered. Unit tests: 6 pass, all shown able to fail by neutering the
Twilio patch (red run produced `AssertionError: network egress attempted`).
Defence in depth: with the Twilio patch disabled, the `requests` adapter guard
still blocked `api.twilio.com`.

## Test subjects — shared-phone families, practice 16

Chosen because these are precisely the rows that mis-address when a channel is
treated as naming one human.

| Phone | People | Names |
|---|---|---|
| `+447508199526` | 7 | Ezra / Finlay / **Finley** / Leanne / Madison / Sam / Tate Wheeler (Finlay vs Finley = likely dup pair) |
| `+441245222528` | 5 | Daniel Tatnell, Ian Porter, June Porter, Suzanne Porter, Tina Rahman — three surnames |
| `+447852268575` | 6 | Simpson ×2 + Bradford ×4 |
| `+447989265614` | 6 | Warn ×5 + Maxine (Janice) Roblin |
| `+447469726576` | 15 | Test/staff data ("TES123", "Rumpelstiltskin the third") — safe for destructive probes |

## Results

_(filled in as each task runs)_

## Frontend routes for the SMS matrix

| Panel | Route |
|---|---|
| Intake journey | `/journey/intake` (also `/journeys/intake`) |
| Nurture journey | `/journey/nurture` |
| Open plans | `/journey/open-plan`, `/journeys/openplans` |
| Active treatments | `/journey/active-plan`, `/journeys/activetreatments` |
| Contacts | `/contact`, `/contact/patients` |
| Patient workspace | `/patients/:patientId/*` |
| Intake / nurture record | `/patients/intake/:recordId`, `/patients/nurture/:recordId` |
| Day List | `/daylist`, `/daylist/administration` |
| Confirmations preview | `/daylist/administration/confirmations/:id/preview` |
| Recalls | `/recalls` |
| Inbox | `/inbox` |
| Marketing campaign | `/marketing/campaigns/:id` |
| Public booking | `/book/:slug` |

## Call-agent webhook facts (Task 5 prep)

- `lookup_patient` requires a **JWT session token** (HS256, issuer `call-agent`,
  30-min expiry, secret from `JWT_SECRET`). Minter written at `/tmp/e2e/mint_token.py`
  and confirmed to produce a token.
- The shared resolver has **three** entry points, so all three need exercising:
  `tools.go:143` (lookup_patient), `handlers.go:347` (incoming call),
  `storage.go:678/771` (call storage → Intake `person_id`, i.e. finding #41).
- The Go `.env` sets `CALL_AGENT_AI_PROVIDER=telnyx`, and `routes.go` only registers
  `/call-agent/post-call-webhook` when the provider is ElevenLabs. Testing that route
  needs the provider overridden, or the Telnyx routes used instead — noted so the
  result is not mistaken for a 404 bug.

---

## NEW DEFECTS FOUND BY RUNNING AGAINST PRODUCTION-SHAPED DATA

Three problems in work that was marked complete this morning. All three were invisible
to the unit tests and to the SQL dry-run the audit relied on; only applying the
migration to a real 8.6 GB production copy exposed them.

### N1 — Migration 0168 was undeployable (BLOCKER)

```
django.db.utils.IntegrityError: could not create unique index
  "uniq_ai_summary_per_dentally_patient"
DETAIL:  Key (practice_id, dentally_patient_id)=(13, 10945) is duplicated.
```

The migration stamps each summary with a Dentally patient id, then adds a UNIQUE
constraint on `(practice, dentally_patient_id)`. But several summaries can resolve to
the **same** patient, so the index cannot be built and the whole migration aborts.
It would have failed identically on production — the deploy would have rolled back at
that step. The audit's "14,432 stamped, 146 deleted" figures came from a dry run of the
*data* step only, which never reached the constraint.

**Fixed:** a dedup pass now collapses duplicates per `(practice, dentally_patient_id)`,
keeping the most recently updated row, before the constraint is added.

### N2 — Root cause: one patient could already own several summaries (NEW, same class as #5)

The constraint being *replaced* was UNIQUE on the raw `(practice, patient_name)`
string, which is case- and whitespace-sensitive. So one human legitimately accumulated
several summary rows:

| Practice | Dentally patient | Name variants held |
|---|---|---|
| 13, 16 | 10945 | `Adam Ribbits` / `ADAM RIBBITS` |
| 16 | 9246 | `Sophie Coward Talbott` / `Sophie  Coward Talbott` (double space) |
| 16 | 9764 | `Sarah Macdonald` / `Sarah  Macdonald` |
| 16 | 10608 | `Alan Graham` / `Alan  Graham` |
| 16 | 11034 | `Shaun Wilson` / `Shaun  Wilson` |
| 16 | 11244 | `Sarah Hayter Ltd` / `Sarah Hayter LTD` |
| 19 | 286 | `George Daley` / `GEORGE DALEY` |
| 21 | 26798 | `Dulcie Sharp` / `Dulcie sharp` |
| 21 | 26804 | `Masie-Ann Mitchell` / `Masie-ann Mitchell` |
| 21 | 27563 | `Dawson Waight` / `Dawson waight` |
| 26 | 63425 | `Madeleine Canty` / `madeleine Canty` |

**12 pairs across 6 practices.** Each of these patients has two AI clinical summaries
today and the day list renders whichever the query returns first — the two can hold
different clinical content. This is finding #5's defect on an axis the audit never
checked: #5 covered *several patients sharing one summary*, this is *one patient
holding several summaries*. Worth tracking as **#45**.

### N3 — Migration 0168 locked the table for 15+ minutes (PERFORMANCE)

The data step was an N+1 loop: one `SELECT DISTINCT` plus one write per summary,
≈44,000 round trips over 14,777 rows. All of it inside the single transaction that
already holds ACCESS EXCLUSIVE on `dentally_patient_ai_summary` from the `AddField`,
so the day list's summary table would have been locked for the entire deploy. The
first attempt was killed by a 900-second timeout without finishing.

**Fixed:** rewritten set-based — one aggregate builds the
`(practice, UPPER(name)) → patient` map, then batched `Case/When` updates and batched
deletes. Data step now completes in **under 90 seconds**. Equivalence with the loop is
exact and documented in the migration's docstring (Django's `iexact` is
`UPPER(a)=UPPER(b)`, and the loop stripped only the summary side, so the appointment
side is deliberately left untrimmed).

**Counts on rehearsal2:** 14,406 stamped / 144 ambiguous / 227 orphans, versus the
audit's 14,432 / 146 / 199 measured on `prod_control`. The two copies differ slightly;
neither number is wrong, but the runbook should quote the figure from the machine it
actually runs on.

---

## Task 3 — SMS from the Contacts / patient panel

**Subject:** Tate Wheeler (patient 32069, person 119734), phone `07508199526`,
a channel shared by **7** Wheelers (Ezra 119732, Finlay 119733, Tate 119734,
Sam 119735, Finley 119736, Madison 119737, Leanne 119738).

### HELD — the shared-channel attribution rule

The patient panel listed exactly the 6 same-practice relatives and no one from
another practice: **#6 holds**. The session was *not* labelled with an arbitrary
family member's name — `sole_person_for_channel` correctly returned `None` on a
7-person channel rather than guessing. **The rule from #38/#41 is working.**

### HELD — the sandbox

Message intercepted, `to=+447508199526`, zero network egress.

### N4 — an SMS sent from a named patient's panel is recorded against NOBODY (NEW)

Sent "intended for TATE Wheeler only" from Tate's own panel. What was stored:

| Record | Field | Value |
|---|---|---|
| `messaging_messagesession` 1989 | `patient_id` | **NULL** |
| | `participant_name` | **`+447508199526`** (the phone number, not a human) |
| | `channel_id` | 105095 (the shared 7-person channel) |
| `messaging_smsmessage` 2426 | `activity_session_id` | **NULL** |
| | `recipient_name` | **empty** |
| `activityLog_activitylog` | rows created | **0** |

Django log: `Entity smsmessage #2426 has no person. Skipping ActivityLog creation.`

**The identity was known and thrown away.** The frontend fetched the thread as
`/api/backend/messaging/contacts/119734/thread/?patient_id=32069` — Tate's person id
*and* patient id were both in the request. But the send path derives the session's
identity purely from the channel via
`_get_person_from_channel` → `sole_person_for_channel`
(`messaging/serializers.py:82-84`), which by design refuses to name anyone on an
ambiguous channel. The user's explicit choice of recipient is never consulted.

This is the audit's Pattern B inverted. The read-side guard is correct — it will not
invent an attribution. The write side discards a free, unambiguous one.

**Consequences on a shared family line:**
- No activity-log entry against Tate, or anyone.
- The inbox thread is titled with the raw phone number instead of a name.
- "Last Contacted" on Tate's record stays "Never" despite a message being sent.
- When a reply arrives it lands in this anonymous thread, and per
  `project_shared_channel_attribution` some downstream path must then pick one of
  7 candidates — the exact mis-attribution risk the audit set out to close.

**Suggested fix (not applied):** the send endpoint should accept the `patient_id` /
`person_id` the caller already supplies and stamp the session and SMS with it,
falling back to `sole_person_for_channel` only when the caller gives nothing.
That keeps the conservative guard for inbound/unknown traffic while honouring an
explicit outbound choice.

Worth tracking as **#46**.

---

## N5 — RENAME + MOVE MINTS A DUPLICATE PERSON (#47) — most serious finding

Reproducible through ordinary UI actions, no API required.

### Steps
1. Journeys → Intake → row Actions → **Edit** → change surname → Save.
2. Same row → Actions → **Move** → Nurture → confirm.

### What happens
The move re-resolves identity by **name**. The renamed name no longer matches the
existing Person, so a **new Person is created** — even though the phone channel
already resolved to exactly one Person in that practice.

| Record | person_id |
|---|---|
| Intake 4365 (Mackalla Williams-Renamed) | 74676 — the original |
| Nurture 679 created by the move | **158648 — brand new** |

Channel `+447576763975` (practice 16) went from **1 Person to 2**:

```
 74676 | Mackalla | Williams            (original)
158648 | Mackalla | Williams-Renamed    (minted by the move)
```

### Control proves the cause is the rename, not the move

Moved **Kate Doel** (intake 4367) to Nurture with **no rename**:
Nurture 680 was created with `person_id = 158291` — **the same Person as the intake**.
No duplicate. So the move path is correct on its own; it is name-driven
re-resolution after a rename that duplicates.

### Proof of harm — the same patient becomes untrackable

Two SMS to Mackalla, same number, 13 minutes apart, across the move:

| SMS | When | `activity_session_id` | Django log |
|---|---|---|---|
| 2427 | 21:41, **before** the move | **110** | attributed |
| 2428 | 21:54, **after** the move | **NULL** | `has no person. Skipping ActivityLog creation.` |

Before the move her thread was named "Mackalla Williams" and her messages appeared in
her history. After it, the channel is ambiguous, `sole_person_for_channel` correctly
refuses to guess, and **every future message to her is invisible in her record.**

### Why this matters against the audit

This is finding **#1's** failure mode (a duplicate Person for a human who already
exists) reachable from the UI in two clicks, and it *manufactures* the precondition
for **#38/#46** (an ambiguous channel nobody can attribute). The fixes shipped this
morning are all working correctly — `sole_person_for_channel` refuses to guess exactly
as designed — but nothing stops the app creating the ambiguity in the first place.

### Suggested fix (not applied)

In the intake→nurture/patient move path, resolve the Person from the **channel** first
when the channel resolves to exactly one Person in that practice, and only fall back to
name matching when it does not. A rename should move the existing Person's name (or
leave it alone), never fork a new identity.

Related: Person 158291 is stored as "Kate **doel**" while the intake says "Kate Doel" —
another case-variance instance, same family as N2.

---

# THE UNIFICATION RULE (answer to "how do we end duplicates for good")

## Your instinct was right, but the fix is not "always use person_id"

Linking by id rather than name is correct — **for the operations where the id is
already known**. It is NOT correct as a blanket rule, and applying it everywhere
would re-introduce a worse bug.

`Person.resolve` deliberately implements: *candidates sharing a channel but a
**different name** -> create a NEW Person in the shared Household, never merge.*
That rule is what keeps "Leanne Wheeler" and "Tate Wheeler" on one family phone as
two humans. If the channel simply won, every family on a shared line would collapse
into a single person — that is finding #3 and defect D2, and it is worse than
duplicates because it is not reversible by a merge.

## The distinction the codebase was missing

| | CARRY | DISCOVER |
|---|---|---|
| Situation | We already know who this is | A record arrived from outside |
| Examples | convert, move, rename, restore, workflow-create-from-record | online booking, call agent, Dentally import, marketing form |
| Identity source | the source record's `person_id` | channel + name + dob heuristics |
| Correct behaviour | **copy the id, never re-derive** | `Person.resolve` (name rule included) |

**Every duplicate found today was a CARRY implemented as a DISCOVER.** The heuristics
are not the problem; calling them when the answer was already in hand is.

## What made this invisible

`_preserve_person_and_link_channels` already guards the exact failure — its own
comment says *"name changed, contact same -> a spelling fix. Re-resolving would find
the existing Person, fail the exact-name match, and split one human into two."*

But it opens with `if not (instance.pk and instance.person)`, so it only runs on
UPDATE. Every CARRY that CREATES a row skipped the guard entirely. That is why the
rename alone was always safe and the move never was.

## The fix, in three layers

1. **Structural (`TreatmentPlan/contact/signals.py`)** — `_assign_person_on_save` now
   honours an explicitly supplied Person on a record with no pk, and still links that
   record's channels to it. Passing `person=` is an assertion of identity, not a
   request for resolution. Without this layer the other two are silently inert —
   proven by a test that failed before it existed.
2. **Call sites** — `ConversionMixin.carried_person()` (`TreatmentPlan/journey/mixins.py`)
   and `automations.actions._carried_person()` supply the known identity at all
   **8** carry sites: 5 journey conversions + 3 workflow handlers.
3. **Guards** — two AST tests fail the build if a new conversion or workflow handler
   forgets. Both include a self-check proving the detector can actually fail.

## Sites audited

| Path | Kind | Verdict |
|---|---|---|
| 5 journey conversion creates (`mixins.py`) | CARRY | **was broken, fixed** |
| 3 workflow handlers (`automations/actions.py`) | CARRY | **was broken, fixed** |
| `TreatmentPlan.objects.create` ×2 | CARRY | exempt — no `person` column, carries via `patient` |
| `onlineBooking/services.py` | DISCOVER | correct (fixed earlier as #33) |
| `marketingBroadcast/form_handoffs.py` | DISCOVER | correct — submission has no person link |
| `TreatmentPlan/views/custom_webhook_views.py` ×2 | DISCOVER | correct |
| `TreatmentPlan/serializers/patient.py` | DISCOVER | correct |
| Go `callagent/storage.go` | DISCOVER | correct (sole-candidate rule, #41) |
| Go `dentally/migration/service.go` | DISCOVER | correct |
| `models.py:2537` | dead | commented-out code inside `convert_to_archive` |
| `ArchiveRecord.restore_from_archive` legacy fallback ×3 | CARRY | **GAP — see below** |

## Remaining gap (not yet fixed)

`_create_archive_record` does not store `person_id` in `original_data`. When a row was
archived by the legacy deleting path, `restore_from_archive` has no identity to carry
and must re-derive by name — so restoring an old archived record can still fork a
Person. The non-legacy path un-archives the row in place and is safe.

Fix: stamp `person_id` into the archive lineage and prefer it on restore. It only
helps rows archived after the change, which is why it is worth doing now.

## Verified end to end in the real UI, on production-shaped data

Same sequence (rename an intake, then move it to Nurture):

| | Before the fix — Mackalla Williams | After the fix — Tina Brown |
|---|---|---|
| Source intake person | 74676 | 158284 |
| Nurture created by the move | **158648 — a new Person** | **158284 — the same** |
| Patient created by the move | — | **158284 — the same** |
| Persons on the phone channel | **1 -> 2** | **still 1** |

---

## #46 FIXED — and which panels it reaches

### The fix uses information that was already arriving

The first attempt added a `known_patient` kwarg threaded through the model layer. The
user pushed back — *"we are adding more things to manage… fix what's already broken
with what we have"* — and was right. The send view **already** reads
`intake_id` / `nurture_id` / `treatment_plan_id`, and its own comment calls them
"the journey record this SMS was sent FROM (the row the user had open)". It used them
to count touch points and discarded them for attribution.

So the fix is:
- `patient_for_journey_record(...)` — takes the **same arguments**
  `increment_touch_points` already receives, because it answers the other half of the
  same question.
- `MessageSession.attribute_to(patient)` — fills a gap only; never overwrites an
  existing attribution or a real name; refuses across a practice boundary.
- `_get_person_from_entity` checks `session.patient` before the channel heuristic, so
  the attribution actually reaches the activity log.

`messaging/models.py` ended up **purely additive: 36 lines, one method**. No new field
on the payload for the journey tables.

### A bug I invented and the red run caught

I claimed a latent defect — that a session created with a name gets `channel = NULL`.
The red run showed the test still passing with my "fix" reverted. Cause:
`TreatmentPlan/contact/signals.py:404` is a `pre_save` receiver that already sets the
channel on every session. **The bug did not exist** and my change duplicated logic that
already had one home. Reverted, and the test asserting it deleted.

### Panel coverage

`getJourneyRecordIds` (frontend) returns ids for `intake`/`custom`, `nurture`,
`treatments`/`activetreatments`, and `{}` for everything else.

| Panel | Sends what | Covered by the backend fix alone? |
|---|---|---|
| Intake / custom-stage table | `intake_id` | **yes, no FE change** |
| Nurture table | `nurture_id` | **yes, no FE change** |
| Open Plan / Active Plan tables | `treatment_plan_id` | **yes, no FE change** |
| Contacts panel | nothing | needed `patient_id` — added |
| Day List panel | nothing | needed `patient_id` — added |
| Recall panel | nothing | needed `patient_id` — added |
| Task panel | nothing | needed `patient_id` — added |
| Global Inbox | nothing | **left alone deliberately** — replying to a thread is not choosing a person, and on a shared line nobody knows who wrote in |

The extension is one more key on the same call, resolved by the same function, checked
last so an open journey row still wins (it identifies the row as well as the human).
Frontend: one line in `PatientInboxTab.sendSmsInline`, using the
`patient.patient_id ?? patient.patient?.patient_id` idiom already used three times in
that file.

## Archive lineage FIXED — third instance of the same shape

`_create_archive_record` never stored `person_id`, so the legacy rebuild path in
`restore_from_archive` could only re-derive identity from the archived NAME. Red run
proved the harm directly: *"the restore left two Persons on one phone."*

Stamped centrally in `_create_archive_record` — all eight conversions funnel through
it, so one stamp rather than eight callers remembering — and consumed by
`Archive._stamped_person()`, which ignores an id belonging to another practice and
falls back to name resolution for lineages written before the stamp existed.

---

## #48 — user-reported: a thread labelled with the WRONG family member, and no activity log

Reported from the UI: *"in task it says Monique Shimwell, in inbox it says Grant
Shimwell, and it has no activity log or anything."* Reproduced exactly.

### What is happening

`messaging_messagesession` **301**:

| field | value |
|---|---|
| `participant_identifier` | `7392136791` — **Monique's** own number |
| `participant_name` | **`Grant Shimwell`** |
| `patient_id` | **NULL** |
| created | 2026-02-23 |

The messages inside it read *"Good Morning Monique…"*. So the content is to Monique,
the number is hers (her patient record 31714 owns it; Grant's row carries a different
number, 7848453566), and the thread is titled Grant.

Channel 104648 (`+447392136791`) links **both** Grant (120218) and Monique (120219).
`sole_person_for_channel` correctly refuses to name either — the #38/#41 guard working.
The contacts serializer then falls through to its next fallback, *"a name captured on
any of this channel's conversations"*, and that stored name is wrong.

`patient_id` NULL is the second half of the report: no patient link means
`_get_person_from_entity` resolves nobody, so no activity row is ever written. The
conversation exists and belongs to no one.

### Scale — not a one-off

Counting only sessions where the label names one member of a shared channel while the
number demonstrably belongs to a **different** member (evidence: that member's own
`Patient.phone_number`, and the labelled person does not own the number):

**71 distinct sessions, 297 messages, across 4 practices — and all 71 have
`patient_id` NULL**, so none of those conversations appear in any patient's history.

A weaker `is_primary`-based query returned 21, but `is_primary` is documented in the
codebase itself as unreliable in production, so the ownership-based count above is the
one to trust.

### Is the live code still doing this?

**No.** A session opened today on a shared channel gets
`participant_name = <the phone number>`, not a guessed human — verified directly:
Tate Wheeler's session 1989 was created during this run with
`participant_name = "+447508199526"`. The 71 are historical damage from the old
arbitrary-`.first()` behaviour, captured before that guard existed.

### Gap this exposes in the #46 fix

`attribute_to` fills `patient_id` but deliberately never overwrites a real
`participant_name`. For these 71 that is too conservative: attributing session 301 to
Monique would leave it **linked to Monique but titled Grant** — an inconsistent state.
When the stored name names a *different member of the same channel* and we are
confidently attributing to one of them, the stored name is contradicted evidence and
should be corrected.

### Proposed (not yet applied)

1. Tighten `attribute_to`: replace the stored name when it names another member of the
   same channel.
2. A repair for the 71, as a runbook review step rather than a blind update — the same
   treatment as #38, because "the number belongs to X" is strong evidence but a family
   member genuinely texting from a relative's phone is a real case too.

---

## #49 — EMAIL: the sandbox had a hole, and the send path had the #46 gap too

Found while sweeping the panels: sending an email from a patient panel left the
outbox **empty**.

### The sandbox hole (safety)

Outbound email does not use Django's `EMAIL_BACKEND` at all. `messaging/email_service.py`
POSTs to the Go email-service (`EMAIL_SERVICE_URL` + `/api/v1/emails/send`), which holds
the provider credentials and does the real sending. Swapping `EMAIL_BACKEND` therefore
intercepted almost nothing. The send only failed here because that service happens to be
down — **with it running against restored production data, real email would have gone to
real patients.**

Closed by blocking the email-service PATHS (`/api/v1/emails/send`, `/api/v1/emails/bulk`)
in the requests-adapter guard, matched on path rather than host so a staging URL cannot
slip past a localhost allow-list. The guard returns the service's own success shape so
the caller proceeds to its bookkeeping instead of raising.

### The attribution gap (same as #46)

`send_plain_email` called `on_record_created` and `increment_touch_points` — reading
`intake_id`/`nurture_id`/`treatment_plan_id` exactly like the SMS path — but never
attributed the session. Fixed identically, plus `patient_id` on the frontend email
sender.

### Verified on the user-reported channel

Sent from Monique Shimwell's panel to `busterandmonique@gmail.com`, the household
address she shares with Grant:

| session 1993 | before | after |
|---|---|---|
| `participant_name` | `busterandmonique@gmail.com` | **Monique Shimwell** |
| `patient_id` | NULL | **31714** |
| activity row | none | **19602 "Email sent"** |
| outbox | **empty — the email escaped the sandbox** | intercepted |

Also fixed the outbox's own fidelity: the Go service sends `to` as a LIST and the body
as `body_html`, so entries were logging `"['a@b.com']"` with an empty body — unusable
for checking who a message actually went to.

---

## #48 repair tooling — and why the first version would have made things worse

`messaging/management/commands/repair_mislabelled_sessions.py`, dry-run by default.

The first dry run returned **49 candidates** and reading them killed the naive fix:
most were not wrong people at all.

| Stored label | Number's owner | Reality |
|---|---|---|
| `Lynn Byatt` | `Lynne Byatt` | same human, one letter |
| `Mrs Tracy Hutchings` | `Tracey Hutchings` | same human, title + spelling |
| `Dawn` | `Dawn Campbell` | same human, first name only |
| `Khumbulani Ndlovu` | `Khumbulani Ndlovu` | **identical** — two Person rows, a duplicate |
| `Abbie Wilson` | `Pauline Wilson` | genuinely different relatives |

Applying that list blindly would have renamed ~20 correct records. The command now
classifies instead:

- **DIFFERENT (29)** — a genuinely different relative; the real mislabels
- **SAME (19)** — nickname/spelling of one human; deliberately untouched
- **DUPLICATE_PERSON** — identical name on two Person rows; wants `dedupe_persons`,
  not a rename

`SAME` is decided with the project's own `canonical_full_name_key`, a token-subset
test, and a small edit distance on the GIVEN name — because the real data is full of
one-character variants (Lynn/Lynne, Tracy/Tracey, George/Geoerge) and two family
members almost never have given names one letter apart, while the same person recorded
twice very often does.

`--apply` touches only DIFFERENT rows, and never a session that already has a patient.
The user's own case is captured:
`session 301 'Grant Shimwell' -> 'Monique Shimwell' (patient 31714)`.

Recorded as step 6 of the prod runbook, as a review queue — the same treatment #38 got,
for the same reason.

## Surfaces confirmed unused in this production copy

Worth recording so nobody mistakes "untested" for "broken":

| Surface | Evidence |
|---|---|
| Online booking | 0 services, 0 practitioners, 0 holds — never launched. Matches the audit's "0 rows today; fires on launch" for #18-20/#33-34. Identity logic covered by `onlineBooking/test_booking_identity.py` (11 tests, passing). |
| WhatsApp | 0 messages, 0 sessions; the send view has no activity-log or attribution call at all. Unused, so not fixed — but it will need the same treatment as SMS/email before launch. |
| `bulk_send_sms` / `bulk_send_email` | Live endpoints, but not called anywhere in the frontend. They take bare phone/email strings with no patient ids, so they cannot attribute per recipient by construction. |
| `send_sms_message` | Dead — its route is commented out in `messaging/urls.py:285`. |

---

## #50 — AUTOMATED sends are unattributed too (found by sweep, FIX PENDING)

The #46/#49 fixes cover the paths a human clicks. They do not cover the paths that run
unattended — which are the bulk of real message volume.

None of these call `attribute_to`, and every one of them knows exactly who it is
messaging:

| Path | Sends | Knows the recipient via |
|---|---|---|
| `dentallyIntegration/recall_automation.py:660` | recall SMS/email | `rec.dentally_patient_id`, `rec.recall_patient` |
| `dentallyIntegration/confirmation_automation.py:334` | appointment confirmations | the appointment's patient |
| `TreatmentPlan/journey/dispatch.py:224` | journey sequence messages | the journey record |
| `dentallyIntegration/views/recall_views.py:1900` | manual recall send | the recall record |
| `Tasks/tasks.py:294` | task SMS | the task's patient |
| `automations/actions.py:1809` | workflow SMS | the source record (see #47) |

So on a shared family line every automated recall, confirmation and journey message is
still filed against nobody: no patient link, no activity history, thread titled with
the raw number. Exactly the symptom reported for Monique Shimwell, but arriving
continuously rather than once.

Resolution is available: `RecallRecord` carries `dentally_patient_id`, and **11,146 of
11,430** patients in practice 16 carry the matching `meta_data->>'id'`.

**Fix shape (same as #46):** a `patient_for_dentally_id(practice_id, dentally_patient_id)`
helper beside `patient_for_journey_record`, then `session.attribute_to(...)` after the
message row is created in each path. Not applied yet — the full regression was in
flight and editing source under it would invalidate the run.

## #50 FIXED — automated sends now name their recipient

| Path | Resolves the recipient via | How |
|---|---|---|
| `recall_automation.send_recall_message` | `rec.dentally_patient_id` | new `_attribute_recall_session` |
| `confirmation_automation` (SMS **and** email) | `appointment.dentally_patient_id` | inside `_log_confirmation_activity`, one edit covers both |
| `journey/dispatch._send_sms` | `enrollment.person` | new `_attribute_journey_session` |

Helpers `patient_for_dentally_id` and `patient_for_person` sit beside
`patient_for_journey_record`, so all four resolution routes live in one module. Every
attribution is wrapped so a labelling failure can never break a send — a recall batch
must not die because one thread could not be named.

### A fourth instance of the same mistake, found on the way

`_log_confirmation_activity` **already** resolved the patient by Dentally id — and used
it only as a guard:

```python
person = _get_contact_for_appointment(appointment)
if not person:
    return
on_record_created(entity=message, user=None)   # re-derives from the CHANNEL
```

`on_record_created` then resolved identity from the message's channel, which on a
family phone deliberately resolves to nobody. So a confirmation sent to a named patient
produced no entry on anyone's record, with the answer sitting in a local variable two
lines above. Same CARRY-implemented-as-DISCOVER shape as #47, #46 and #49.

### A test that could not be written

The first draft asserted "two patients on one Dentally id resolve to nobody". The
database refuses that fixture: `patient_practice_dentally_uniq` on
`(practice_id, meta_data->>'id')`. The ambiguity branch is therefore unreachable by
construction. The test now pins the CONSTRAINT instead, so that if it is ever dropped
the assumption fails loudly rather than silently.

### Not fixed

`Tasks/tasks.py:294` and `automations/actions.py:1809` still do not attribute. Both are
lower volume and neither was exercised in the UI sweep; left for the next pass rather
than changed blind.

### CORRECTION — the "29 to repair" figure was wrong, and the tool would have caused harm

Asked to explain the 29, I read the actual message bodies instead of restating the
count. Three samples killed the heuristic:

| Session | Labelled | Messages open with | Tool proposed | Truth |
|---|---|---|---|---|
| 301 | Grant Shimwell | *"Good Morning **Monique**"* | → Monique | correct |
| 780 | Abbie Wilson | *"Hi **Abbie**…"* | → Pauline Wilson | **would corrupt** |
| 429 | Jane Holmes | *"Hi **Jane**…"* | → Darcy Holmes | **would corrupt** |

Abbie and Jane are correctly labelled people **using a relative's phone**. Number
ownership does not identify who is on the other end of a conversation; it identifies
whose contract the phone is on. Two of three proposals were wrong.

`_greets()` now reads the OUTGOING messages — the ones staff wrote, deliberately
addressing who they believed they were speaking to — and the verdict changed:

| | before | after |
|---|---|---|
| repair | **29** | **2** (CONFIRMED_WRONG) |
| read by hand | — | 5 (DIFFERENT, messages silent) |
| leave alone | 20 | **42** (22 CORROBORATED + 19 SAME + 1 duplicate) |

`--apply` now touches only rows where the messages themselves greet a different person
on the line: session 301 (`Grant` over "Good Morning Monique") and session 485
(`Unknown Hammett` over "Hi Susan"). Both verified by reading them.

**The lesson is the one this whole workstream keeps teaching.** A plausible signal —
"whose number is it" — is not evidence of identity. The conversation itself was the
evidence, and it was one query away the whole time. Had this shipped on the earlier
logic it would have destroyed 22 accurate attributions in order to fix 2.

---

## N6 — migration 0168 was destroying 29 recoverable clinical summaries

A stale background query from earlier in the session surfaced a mismatch: the audit
predicted **14,432 stamped / 146 / 199** from `prod_control`, but the migration
actually produced **14,406 / 144 / 227** on `prod_rehearsal2`. Same 14,777 starting
rows, ~26 more summaries destroyed.

Cause: the audit's SQL applied `btrim` to BOTH sides of the name comparison; the
migration trimmed only the summary side — faithfully reproducing the original loop's
`patient_name__iexact=stripped_name`, because Django's `iexact` does not trim the
column. **952 appointment rows carry leading or trailing whitespace**, identical in
both copies, so it was not an artefact of rehearsal2's repairs.

Measured on `prod_control`, trimming both sides:

| | count |
|---|---|
| summaries deleted as "orphan" but recoverable once trimmed | **29** |
| that become ambiguous once trimmed | 0 |
| currently stamped that become ambiguous once trimmed | **2** |

Those last 2 matter most: they are stamped to ONE patient today, but the trimmed name
maps to TWO different Dentally patients — whitespace was the only thing keeping them
apart. Deleting them as unattributable is correct; keeping them stamped is finding #5's
exact defect, a clinical summary shown against the wrong human.

Fixed by keying the aggregate on `Upper(Trim("patient_name"))`. Whitespace is not
identity, so this can only merge true variants — it can never match two humans the
untrimmed comparison kept apart. The prediction now reproduces the audit's published
numbers exactly (14,432 / 146 / 199, plus 12 dedup deletions → 14,420 rows).

**Note for re-verification:** `prod_rehearsal2` already has 0168 applied under the OLD
untrimmed logic, so its 29 summaries are gone. Confirming the new behaviour end to end
needs a fresh copy; the SQL prediction above was run against untouched `prod_control`.

# Utils consolidation — the fix list

Compiled from the per-app scans in this directory, plus findings derived directly.
Nothing here is applied yet. Each entry needs the same discipline as the audit:
a red run proving the duplication actually matters before it is collapsed.

---

## C1 — the Go↔Django parity FIXTURES are two files with nothing keeping them equal

**Found by:** direct inspection (not an agent), 2026-09-07.

`canonical_phone_e164` (Django) and `phone.CanonicalE164` (Go) are a DELIBERATE
duplication — two languages, one behaviour — and that is fine, because a shared
fixture pins them. The problem is that the "shared" fixture is a COPY:

| fixture | Django | Go |
|---|---|---|
| `phone_fixtures.json` | `TreatmentPlan/tests/` | `EmailServiceGo/pkg/phone/testdata/` |
| `contact_key_fixtures.json` | `TreatmentPlan/tests/` | `EmailServiceGo/pkg/personname/testdata/` |

`TreatmentPlan/utils/phones.py:372` even says so: *"identical copy lives at …"*.

**Verified 2026-09-07:** both pairs are currently byte-identical (sha256 matches;
25 phone cases, 33 name cases). **No drift today.**

**Why it still needs fixing:** nothing enforces it. Add a case to the Django
fixture and not the Go one and BOTH suites stay green while the two
implementations quietly diverge — which is precisely the reader-key ≠ writer-key
failure that produced this whole workstream. The guard is a convention in a
comment, and conventions in comments are what the 90 findings were made of.

**Proposed fix:** one owning copy plus a test that fails when they differ —
cheapest honest version is a test on each side asserting the two files hash the
same, so whichever suite runs first catches it. Do NOT symlink: the Go build
context and the Django test runner do not both follow one reliably.

**Belongs in:** neither `utils/` — this is test infrastructure.

---

## From scan 1 — `TreatmentPlan/` (haiku inventory, then verified by hand)

### C2 — `Patient.__str__` renders the literal string "None"  ✅ REAL, verified

`TreatmentPlan/models.py:1939` — `return f"{self.first_name} {self.last_name}"`, with no
`or ''`. `Patient.last_name` is `null=True` (`models.py:1743`), so a NULL surname renders
**`"John None"`**. `.strip()` does not help: `"John None".strip()` is still `"John None"`.

**Executed, not read:**

```
Patient.__str__ with NULL last_name -> 'John None'
full_name() with NULL last_name     -> 'John'
```

Live rows today (local DB): **Patient 1 of 60,536; Intake 1 of 3,966; Nurture 0 of 619.**
Small, but `__str__` reaches the Django admin, log lines and error messages, so it is the
kind of thing that surfaces in front of a human at the worst moment.

`full_name()` already handles it. The bare-pattern sites need per-model triage: harmless
where both columns are `NOT NULL` (Person is `blank=True, default=""`), wrong where they
are nullable (Patient, Intake, NonRegisteredPatient).

**Belongs in:** `TreatmentPlan/utils/names.py::full_name` — it exists; these sites just
do not call it.

### C3 — 54 inline display joins  ⚠️ REAL but cosmetic, NOT an identity risk

Genuine duplication (matches the known "86 inline builds still open"), worth collapsing
onto `full_name()` for consistency and to fix C2 along the way.

**The agent's stated risk for this is WRONG, and I checked before believing it.** It
reported: *"the same human can resolve to different identity keys depending which code
path writes the name"* because some inline joins skip the whitespace collapse.

That cannot happen. `canonical_full_name_key(first, last)` builds its key from the RAW
COLUMNS and runs `canonical_name_part`, which collapses internal whitespace itself:

```
canonical_full_name_key("Mary  Jane", "Smith") -> 'mary jane smith'
canonical_full_name_key("Mary Jane",  "Smith") -> 'mary jane smith'   # identical
```

Grepping every caller of `canonical_*` confirms none is fed a display join; the only
constructed string is inside `contact_keys.py` itself. So the display joins never touch
identity. Priority drops from "highest" to "tidy-up that also fixes C2".

### C4 — 2 name-split sites  ⚠️ downgrade — one is legitimate

Agent flagged `views/intake_views.py:1192` and
`management/commands/split_collapsed_persons.py:493` as the corruption pattern.

`split_collapsed_persons.py` is the REPAIR TOOL for this very bug — splitting a name is
its job, and it re-runs identity afterwards. Not a defect. `intake_views.py:1192` needs
reading in context before it is called one.

### C5 — 6 email `.lower()` sites  ⚠️ REAL, low risk

`selectors.py:46,82,115,140`, `serializers/intake.py:247`, `serializers/nurture.py:242`
lowercase inline instead of calling `canonical_email()`. All currently agree. The value
is removing the chance of a seventh that forgets `.strip()`.

**Belongs in:** `TreatmentPlan/utils/contact_keys.py::canonical_email` — already exists.

---

**Method note, worth keeping:** the haiku scan was useful for FINDING sites and useless
for RANKING them — it rated a cosmetic issue highest and buried a real one ("None" in a
patient's name) inside a list of 54. Treat every scan as leads to verify, never as
conclusions.

## From scan 2 — `dentallyIntegration/` (haiku inventory, then verified by hand)

### C6 — `_display_name()` is a workaround for a bug that no longer exists, and has since DIVERGED  ✅ REAL

`dentallyIntegration/next_appointment.py:66-74` carries its own name join, and its
docstring explains why:

> *"`Patient.full_name` is an f-string over two nullable columns, so it renders the
> literal "None Smith" when first_name is NULL. Not reused here for that reason."*

**That statement is now false.** `Patient.full_name` (`TreatmentPlan/models.py:1942-1945`)
was changed to delegate to the canonical helper and is None-safe. Measured:

```
Patient.full_name  (NULL first) -> 'Smith'      # safe
_display_name      (NULL first) -> 'Smith'      # same
Patient.full_name  ("  Mary  Jane ", " Smith ") -> 'Mary Jane Smith'
_display_name      ("  Mary  Jane ", " Smith ") -> 'Mary  Jane Smith'   # DIVERGED
```

So the copy is not only redundant, it is now **worse than the original**: it does not
collapse internal whitespace, so a double-spaced stored name reaches the day list with
the double space intact.

This is the whole thesis of the consolidation in one function — a defensive private copy,
written against a real bug, left behind after the bug was fixed, quietly drifting from
the canonical helper it was avoiding. The stale docstring is what kept it alive.

**Fix:** delete `_display_name`, call `patient.full_name`. Delete the docstring's claim
with it, or the next reader re-creates the workaround.

### C7 — `Patient.__str__` is unsafe in BOTH directions  ✅ REAL (extends C2)

`TreatmentPlan/models.py:1939`. C2 recorded `'John None'` for a NULL surname; a NULL
first name is just as bad:

```
Patient.__str__ (NULL first) -> 'None Smith'
Patient.__str__ (NULL last)  -> 'John None'
Patient.full_name            -> correct in both cases
```

The model already HAS the safe join three lines below (`full_name`, line 1942). `__str__`
just does not call it.

### C8 — email canonicalisation hand-rolled in the recall channel builder  ⚠️ REAL, low risk

`dentallyIntegration/recall_automation.py:365` — `canonical = str(raw).strip().lower() or None`
instead of `canonical_email()`. Note the phone branch immediately above it (line 360)
DOES call the canonical `canonical_phone_e164`. So one function canonicalises one channel
kind properly and the other by hand — the inconsistency is inside a single `if/elif`.

### Corrections to the scan's own claims

- **"`full_name()` renders 'None Smith' unsafely"** — FALSE. The agent read the stale
  docstring above and reported it as current fact. Both `full_name()` and
  `Patient.full_name` are None-safe; only `__str__` is not.
- **"B3: the migration command could crash on malformed `meta_data['id']`"** — unverified
  speculation ("could"), left out of this list until someone shows the input that does it.

**Method note:** two scans, two inverted rankings. Scan 1 rated a cosmetic issue highest
and buried a real one; scan 2 reported a fixed bug as live by trusting a code comment.
Both scans still earned their keep by pointing at the right FILES. Verify every claim
against running code before it enters this list.

## From scan 6 — `Appointments/` (verified by hand)

### C9 — the anti-spam counter is case-sensitive; the cancel check is not  ✅ REAL, reachable

Two comparisons of the same column, written differently:

```python
# public_booking_views.py:313 — anti-spam, 3 pending bookings per hour
Appointment.objects.filter(patient_email=data["patient_email"], ...)   # EXACT

# public_booking_views.py:526 — cancel
if appointment.patient_email.lower() != email.lower():                 # case-insensitive
```

`Appointments.models.patient_email` is a plain `EmailField(blank=True, default="")` with
no normalisation on write, so the stored case is whatever the public form submitted.

**Reachable:** booking as `john@x.com`, `John@x.com`, `JOHN@x.com` produces three rows
that the counter treats as three different people, so the "3 pending per hour" limit
never trips. Abuse control, not patient identity — but a real divergence with a trivial
input.

**Fix:** `canonical_email()` on write, or at minimum `__iexact` in the counter.

### C10 — one patient's phone renders two ways in two serializers  ⚠️ REAL, cosmetic

- `Appointments/serializers.py:922` (`ShortNoticePatientSerializer`) canonicalises via
  `canonical_phone_e164`, with a docstring about the ISO-`GB` trap → `+447…`
- `Appointments/serializers.py:125` (`AppointmentListSerializer`) returns
  `patient.phone_number` raw → bare national digits

**Correction to the scan's claim:** it said the diary "would show `GB4473…`". It would
not. The diary path returns `phone_number` ALONE and never concatenates `country_code`,
so the ISO-`GB` corruption cannot appear through it. The real effect is milder: the same
patient's number displays as `+447911123456` in one view and `07911123456` in another.

Worth unifying — one "how do we render a patient's phone" helper — but it is a display
inconsistency, not the identity trap the scan invoked.

## From scan 7 — `Notes/` (verified by hand)

### C11 — label statistics are gated on the AUTHOR's practices, not the note's  ✅ REAL, measured

`Notes/services/labels.py:384`:

```python
elif practice:
    base_queryset = NoteLabel.objects.filter(note__user__practices=practice)
```

`practices` is M2M. So this matches every label on every note by any user who is a member
of that practice — regardless of which practice the note itself belongs to. Every other
note/letter reader in this app gates on `patient__practice` (that is what
`scope_clinical_to_practice()` exists for); this one gates on author membership.

**Measured:** **20 users belong to more than one practice** (one to three), and **22 of 64
NoteLabels** were authored by such a user. Each of those 22 is counted in the statistics
of *every* practice its author belongs to — about a third of all label rows.

Reporting accuracy, not patient identity: no clinical content crosses a boundary, the
counts are just wrong. Fix is to gate on the note's practice (or the patient's), matching
every sibling reader.

### REFUTED — "search_by_patient leaks any practice's patients" (reported CRITICAL, sibling miss of #49)

A good hypothesis — it is exactly the #92 shape, and it would have been my own fix's
sibling — but it does not hold. `search_by_patient` has two branches:

```python
# branch 1 — practice notes
matching_patients = Patient.objects.filter(practice=practice).filter(...)   # SCOPED
practice_notes = Note.objects.filter(patient__in=matching_patients,
                                     practice=practice, ...)                # SCOPED twice
```

The patient set is practice-filtered before it is ever used, so branch 1 cannot return
another practice's patient. Branch 2 (`scope="individual"`) is gated on `user=user` — the
caller's OWN notes — and lacks a `patient__practice` gate, which is a genuine consistency
gap with `notes_by_patient`.

**But it is unreachable today: 0 individual-scope notes have a patient at all.** The
patient-matching `Q` clauses in that branch match nothing.

Worth aligning for consistency (latent, "0 rows today", like several accepted audit
items) — not the cross-practice clinical leak reported.

### Pending — letters image-operation permission checks

Scan reports `add_images` and friends checking `letter.user.practices.filter(...)`
(author membership) while `get_queryset` gates on `patient__practice`, citing #48. Same
shape as C11 and plausible; not yet verified or measured here.

## From scan 8 — `Documents/` (verified by hand)

### C12 — SEVEN copies of "get the client IP", and the ONE that diverges is in the consent audit trail  ✅ REAL

The scan found two identical copies inside `Documents/`. Grepping the whole backend found
**seven**, across six apps:

| site | reads `X-Forwarded-For`? |
|---|---|
| `Documents/views/public_views.py:42` | yes |
| `Documents/views/signing_views.py:145` | yes |
| `compliance/serializers.py:1593` | yes |
| `compliance/views.py:4641` | yes |
| `TreatmentPlan/views/intake_views.py:1053` | yes |
| `patientDocuments/views.py:67` | yes |
| **`marketingBroadcast/views/preferences_views.py:24`** | **NO — `REMOTE_ADDR` only** |

The two in `Documents/` are byte-identical (same normalised sha256). Six of the seven
resolve the forwarded header. One does not:

```python
def _client_ip(request):
    return request.META.get("REMOTE_ADDR")
```

**Why this one matters most.** TLS terminates at nginx (established during the #82 Twilio
signature work), so `REMOTE_ADDR` is the PROXY's address, not the patient's. And this
copy is not incidental — I traced it:

```
preferences_views.py:68   ip_address=_client_ip(request)
  -> consent_ledger.set_marketing_consent(..., ip_address=...)
  -> MarketingConsent.ip_address   (models.py:95, audit row at :845)
```

So the page where a patient changes MARKETING CONSENT records the load balancer's IP in
the audit trail, while every other consent-adjacent path in the codebase records the
real client. That audit row is the evidence of who opted in or out and from where; as
written it is the same address for every patient, which makes it worthless as evidence
rather than merely inaccurate.

**Belongs in:** global `utils/` — `client_ip(request)`. This is the clearest case in the
whole exercise for the user's "build once, use everywhere" goal: seven copies, one
divergent, and the divergent one sits in the compliance-relevant path.

**Also from this scan, lower value:** `_get_practice()` redefined in 4 ViewSets;
`_create_audit_event` vs `_append_audit` with different signatures; 3 inline name joins;
2 implementations of the latest-consent-status mapping. Real duplication, no divergence
found in behaviour.

**Scan quality note:** the first scan to volunteer explicit NEGATIVE findings — "no
author-membership gates, no unordered `.first()`, no cross-practice leaks" — after the
prompt was hardened with the six rules earned by earlier overturned claims. Its two
headline items both held up. Nothing it flagged had to be refuted.

## From scans 9–10 — `activityLog/` and `automations/` (both largely clean)

### C13 — the duplicate-conflict ERROR path looks up a phone differently from the main path  ⚠️ REAL, low severity

`automations/actions.py`:

```python
# main path :589-596 — canonical key OR raw
normalized = canonical_phone_e164(primary_phone, country_code)
phone_q = Q(phone_number=primary_phone)
if normalized:
    phone_q |= Q(person__person_channels__channel__canonical_value=normalized, ...)

# error handler :636-640 — raw exact only
Patient.objects.filter(practice=practice,
                       phone_number=primary_phone,
                       country_code=country_code).first()
```

The error handler can fail to find a patient the main path would have found — a number
stored canonically (`+447911123456`) but submitted locally (`07911123456`) matches the
first and not the second. The consequence is a duplicate-conflict error response whose
`existing_patient_id` is empty even though the duplicate exists: staff are told there IS a
conflict but not which record. Reporting inconsistency in an error path, not a wrong-human
write.

The scan flagged this itself as "reachability unverified" rather than ranking it, which is
the correct call and the first time in this sweep an agent volunteered that label.

### Everything else in these two directories: clean

`automations/` — ingest paths unified on `_normalized_lead_contact_fields()`; #47's
identity carry now centralised in `_carried_person()` and used by all three create
handlers; every query practice-scoped. The `actions.py` phone comments that describe
"+4435699097155" are the RECORD of #13's fix, not a live defect — verified before the scan
ran, and the agent was told so.

`activityLog/` — `get_entity_type_display()` duplicated across 3 serializers,
`get_entity_body()` across 2, and one non-canonical name join in
`PatientAuditEventSerializer`. All cosmetic. Its "whose event is this" logic is a proper
shared helper (`resolve_target_person_ids()`), and `by_contact()` correctly uses
`phones_match()` and `full_name()`.

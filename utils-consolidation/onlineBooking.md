# onlineBooking/ Duplicate Logic Inventory

## B1 — Name Joining (5 sites)

**Canonical helper:** `TreatmentPlan.utils.names::full_name()` (the ONE display join)

**Belongs in:** Global `utils/` — all callers should import and use the single canonical version

**Sites:**
1. `views.py:480` — `f"{p.first_name or ''} {p.last_name or ''}".strip() or p.email`
   - Practitioner display in `OnlineBookingPublicServicePractitionerListView`

2. `views.py:684` — `f"{h.practitioner.first_name} {h.practitioner.last_name}".strip()`
   - Practitioner name in `_build_verified_response()` helper (upcoming bookings payload)

3. `serializers.py:292` — `f"{row.practitioner.first_name or ''} {row.practitioner.last_name or ''}".strip()`
   - `PublicOnlineBookingServiceSerializer.get_practitioners()` method

4. `serializers.py:424` — `f"{first} {last}".strip() or obj.practitioner.email`
   - `OnlineBookingServicePractitionerSerializer.get_name()` method

5. `tasks.py:92` — `session.practitioner.get_full_name() or session.practitioner.email`
   - Using Django's User model method `get_full_name()` instead of canonical helper

**Do they actually agree?** 
IDENTICAL INTENT, but **different implementations**:
- Sites 1, 3, 4: f-string with fallback to `.strip()` or null handling
- Site 2: bare f-string with `.strip()`
- Site 5: Uses User's `get_full_name()` method instead

**Risk:** 
Multiple readers of practitioner name. If one were to change (add a middle initial, handle None-safety differently), the others would diverge. Display-only risk is LOW, but **pattern consolidation risk is HIGH** — any new practitioner display added elsewhere will likely use an f-string instead of the canonical helper.

**Already correct paths:**
- `services.py:368` — Uses `canonical_full_name_key()` for identity matching ✓

---

## B2 — Name Splitting at First Space (2 sites)

**Canonical helper:** NONE — would need creating (see B3 comment)

**Belongs in:** Global `utils/` (new `names.py` helper or `contact_keys.py`)

**Sites:**
1. `services.py:594-596` — In `_provision_ledger_entry_for_booking_payment()`
   ```python
   name_parts = hold.patient_name.strip().split(" ", 1)
   first_name = name_parts[0]
   last_name = name_parts[1] if len(name_parts) > 1 else ""
   ```
   Creates a Patient from booking hold data when no match found.
   **Comment #20 above documents this:** Issue #20 identifies 1,848 live Persons with a space in `first_name`; this split creates duplicates when there's a space in the first name. Example: "Ken (Kenneth) Judge" on Dentally becomes first="Ken (Kenneth)" + last="Judge", while the booking form sends one string "Ken (Kenneth) Judge" split at space 1 → first="Ken (Kenneth)" + last="Judge" ... wait, that's the same. Let me re-read. The comment says split is WRONG: "Ken (Kenneth) Judge" → first="Ken" + last="(Kenneth) Judge" while canonical is first="Ken (Kenneth)" + last="Judge". The comment says "keeping the whole name in first_name" is the fix, yet the code still splits.

2. `tasks.py:115` — In `_create_intake_for_session()`
   ```python
   name_parts = (session.full_name or "").split(" ", 1)
   first_name = name_parts[0] if name_parts[0] else session.email.split("@")[0]
   last_name = name_parts[1] if len(name_parts) > 1 else ""
   ```
   Fallback to email local-part if no name typed. Creates an Intake from abandoned session.

**Do they actually agree?** IDENTICAL LOGIC. Both split at first space, both use fallbacks.

**Risk:** 
**CRITICAL FOR B2.1 (services.py:594)** — Payment-confirmed identity creation. Any name with internal spaces creates a duplicate. The comment documents this was issue #20 but the fix appears incomplete. The comment says "keeping the whole name in first_name when there is no separate surname to be had" — implying the fix should be: if no Last Name was separately gathered, store the whole typed string as first_name and leave last_name empty. **REACHABLE:** 1,848 known Persons have spaces in first_name, the online booking form collects no DOB or surname field — it's all one text box.

**LOWER for B2.2 (tasks.py:115)** — Abandoned session lead creation (Intake, not Patient). Intake names are not identity keys, so divergence is display-only. But the same person creates an Intake at line 115 AND a Patient at services.py:594, so B2.1 is the real blocking issue.

---

## B3 — Patient Identity Matching on Email (2 sites, intentional divergence)

**Canonical helper:** `TreatmentPlan.utils.sole_patient_for_person()` (for existing Patient)

**Belongs in:** Already split correctly — different stages of the booking flow

**Sites:**
1. `services.py:376` — In `_match_patient()` (hold→appointment conversion)
   ```python
   candidates |= set(
       base.filter(email__iexact=canonical_email(hold.patient_email))
   )
   ```
   Then filters named matches only (line 396-402), returns ONE or None. **CORRECT** — documented fix #18.

2. `views.py:646-651` — In `_build_verified_response()` (email verification → prefill)
   ```python
   candidates = list(
       Patient.objects.filter(
           practice=profile.practice,
           email__iexact=email,
       ).order_by("id")[:2]
   )
   patient = candidates[0] if len(candidates) == 1 else None
   ```
   Returns ONE patient or None (ambiguity check). **CORRECT** — documented fix #63.

**Do they actually agree?** NO, and **intentionally different**:
- B3.1: All email matches → filter by booked name → return ONE or None
- B3.2: All email matches → check count ([:2] is enough) → return THE ONE or None

**Risk:** NONE — they're different stages with different semantics. B3.1 filters by name (booking resolves via match). B3.2 refuses to prefill when ambiguous (form forces typing the name). Both use the same tiebreaker rule (one answer only) and both delegate name resolution correctly.

---

## B4 — Phone Normalization (context-reviewed)

**Canonical helper:** `TreatmentPlan.utils.phones::canonical_phone_e164()` + practice region

**Belongs in:** Already correct

**Sites:**
1. `services.py:381-392` — In `_match_patient()`
   ```python
   canonical = ContactChannel.lookup_key(
       practice, ContactChannel.PHONE, hold.patient_phone
   )
   if canonical:
       candidates |= set(
           base.filter(
               person__person_channels__channel__kind=ContactChannel.PHONE,
               person__person_channels__channel__canonical_value=canonical,
           )
       )
   ```
   Uses `ContactChannel.lookup_key()` which applies practice default country. **CORRECT** — documented fix #19.

**Risk:** NONE — the right pattern.

---

## B5 — Email Normalization (context-reviewed)

**Canonical helper:** `TreatmentPlan.utils.contact_keys::canonical_email()`

**Belongs in:** Already correct

**Sites:**
1. `views.py:535` — `OnlineBookingPublicEmailVerifySendView.post()`
2. `views.py:583` — `OnlineBookingPublicEmailVerifyConfirmView.post()`
3. `views.py:712` — `OnlineBookingPublicEmailVerifyDeviceSkipView.post()`

All import and use `canonical_email()` consistently. **CORRECT**.

**Risk:** NONE.

---

## B6 — Practitioner Eligibility Lookup (1 site, minor)

**Canonical helper:** NONE — simple lookup

**Belongs in:** Currently in `services.py`, no consolidation needed

**Sites:**
1. `services.py:119` — In `service_practitioner_is_eligible()`
   ```python
   rule = service.practitioner_rules.filter(
       practitioner=practitioner, is_visible=True
   ).first()
   ```
   No `order_by()`. Assumes one rule per (service, practitioner) pair. **Model unique_together constraint at models.py:166 guarantees this**, so `.first()` is safe (deterministic).

**Risk:** NONE — the constraint makes the unordered `.first()` deterministic.

---

## Summary Table

| Behavior | Sites | Agree? | Consolidated? | Risk | Home |
|----------|-------|--------|----------------|------|------|
| **B1 — Name Joining** | 5 | No, 5 implementations | NO | HIGH pattern risk | Global `utils/names.py` or importer audit |
| **B2 — Name Splitting** | 2 | Yes, identical | NO | **CRITICAL for B2.1** (payment-confirmed identity) | Global `utils/names.py` helper |
| **B3 — Patient Email Match** | 2 | No, intentional | YES ✓ | None — correct divergence | Already correct |
| **B4 — Phone Normalization** | 1 | — | YES ✓ | None — correct helper used | Already correct |
| **B5 — Email Normalization** | 3 | Yes, all use canonical | YES ✓ | None | Already correct |
| **B6 — Practitioner Lookup** | 1 | — | YES ✓ | None — constraint enforces determinism | Already correct |

---

## Recommended Action

### Immediate (blocking payment flow):
- **B2.1 (services.py:594-596):** Verify whether the name split is intentional or an incomplete fix. The comment documents issue #20 but the code still splits. Test with names containing spaces (e.g., "Ken (Kenneth) Judge") to confirm whether duplicates are being created.

### Medium (pattern consolidation):
- **B1 (5 sites, name joining):** Replace all f-string name joins with imports of `full_name()`. This is a straightforward audit + refactor.
  - Import: `from TreatmentPlan.utils.names import full_name`
  - Replace sites 1–4: `full_name(first_name, last_name)` (check param order in canonical)
  - Replace site 5 (tasks.py:92): Decide whether to keep Django's `get_full_name()` for User model (it may be intentional) or consolidate to the custom helper for consistency.

### Low (already correct, document):
- **B3, B4, B5, B6:** These are already using canonical helpers or are deterministic. Document that they are correct and no action needed.

---

## Files Referenced

- `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/onlineBooking/views.py` (B1 sites 1–2)
- `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/onlineBooking/serializers.py` (B1 sites 3–4)
- `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/onlineBooking/services.py` (B2.1, B3.1, B4)
- `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/onlineBooking/tasks.py` (B1 site 5, B2.2)
- `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/onlineBooking/test_booking_identity.py` (audit test suite for B2.1, B3.1)
- `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/onlineBooking/test_verified_prefill_shared_email.py` (audit test suite for B3.2)

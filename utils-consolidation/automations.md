# Automations Module Inventory — Duplicate Logic Scan

**Directory:** `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/automations/`

**Scan date:** 2026-09-07

---

## B1 — Email normalization (3 sites, 1 canonical)

**Canonical helper:** `TreatmentPlan.contact.sync::normalize_email()` (strip + lowercase)

**Belongs in:** Already correct — centralized in `_normalized_lead_contact_fields()`

**Sites:**
- `actions.py:114` — `normalize_email(data.get("email", "") or "")` in `_normalized_lead_contact_fields()`
  - Used by `_create_patient_record` (line 303)
  - Used by `_create_nurture_record` (line 805)
  - Used by `_create_intake_record` (line 911)
- `actions.py:1056` — `canonical_email(email)` in `_create_treatment_plan_record` patient lookup
  - Different context: searching for existing patient by email with channel graph join

**Do they actually agree?** IDENTICAL. Both `normalize_email` and `canonical_email` perform `strip().lower()` and return None for empty values. The lookup at line 1056 uses `canonical_email` explicitly because it has a companion channel-graph query on the same email key.

**Risk:** None — all ingest paths normalize through the same helper; the treatment-plan lookup is correct.

---

## B2 — Phone canonicalization (5 sites, 1 canonical helper, 1 alternate picker)

**Canonical helper:** `TreatmentPlan.utils.phones::canonical_phone_e164()` and `pick_usable_phone()`

**Belongs in:** Already correct — both helpers are imported and used consistently

**Sites:**
- `actions.py:337-349` — `canonical_phone_e164()` + raw phone fallback in `_create_patient_record` duplicate check
  ```python
  normalized = canonical_phone_e164(patient_data["phone_number"], patient_data.get("country_code"))
  phone_q = Q(phone_number=patient_data["phone_number"])  # raw exact
  if normalized:
      phone_q |= Q(person__person_channels__channel__canonical_value=normalized)  # canonical via channels
  ```
- `actions.py:587-595` — Same pattern in dentally patient lookup
  ```python
  normalized = canonical_phone_e164(primary_phone, country_code)
  phone_q = Q(phone_number=primary_phone)
  if normalized: phone_q |= Q(person__person_channels__channel__canonical_value=normalized)
  ```
- `actions.py:958-968` — Same pattern in `_create_intake_record` duplicate check
- `actions.py:449-460` — `pick_usable_phone()` + `_dentally_phone()` for extracting usable phone from Dentally's multi-format fields (mobile_normalized, mobile, mobile_country)
- `actions.py:1074-1085` — `ContactChannel.lookup_key()` in `_create_treatment_plan_record` patient lookup (NOT `canonical_phone_e164` directly, but equivalent)

**Do they actually agree?** 
- Lines 337-349, 587-595, 958-968: IDENTICAL pattern — raw exact match + canonical via channels.
- Line 449-460: Uses `pick_usable_phone()` which is documented (lines 443-450) as the shared rule mirrored in Go. Includes long comment about Dentally's broken normalization ("+4435699097155" for Malta).
- Line 1074-1085: Uses `ContactChannel.lookup_key()` which wraps the same canonicalization logic.

**Risk:** None — all sites use canonical helpers consistently. The long comment at lines 443-450 describing Dentally's normalization bug is the RECORD OF THE FIX, not a live defect.

---

## B3 — Practice scope enforcement (ALL sites verified)

**Canonical pattern:** Every database query filters on `practice=practice`

**Belongs in:** Already correct — practice is passed in at function entry and threaded through

**Sites (sample):**
- `actions.py:94` — `model.objects.get(pk=source_record_id, practice=practice)` in `_carried_person()`
- `actions.py:320` — `Patient.objects.filter(practice=practice)` in `_create_patient_record`
- `actions.py:575` — `Patient.objects.filter(practice=practice, first_name__iexact=..., last_name__iexact=...)` in dentally patient lookup
- `actions.py:1031` — `Patient.objects.get(id=int(patient_id), practice=practice)` in `_create_treatment_plan_record`
- `actions.py:1064` — `Patient.objects.filter(practice=practice)` in treatment plan patient lookup

**Do they actually agree?** UNANIMOUS — every Patient/Intake/Nurture/TreatmentPlan lookup is scoped to `practice=practice`. No cross-practice reads found.

**Risk:** None — practice isolation is enforced across all paths.

---

## B4 — Person identity carry (3 sites, 1 helper)

**Canonical helper:** `automations.actions::_carried_person(practice, source_record_type, source_record_id)` (lines 60-100)

**Belongs in:** Already correct — centralized and used by all create handlers

**Sites:**
- `actions.py:310` — `_create_patient_record` uses `_carried_person(...)`
- `actions.py:812` — `_create_nurture_record` uses `_carried_person(...)`
- `actions.py:918` — `_create_intake_record` uses `_carried_person(...)`
- `test_workflow_record_identity_carry.py:25,52` — Test validates `_carried_person()` recovers the source record's Person

**Do they actually agree?** IDENTICAL — all three handlers call the SAME `_carried_person()` helper, which:
1. Returns None when no source (→ "discover normally")
2. Refuses to carry across practice boundaries
3. Returns the source record's Person if it exists and belongs to the practice

**Risk:** FIXED. Finding #47 (documented in test file) was that these handlers had the signature parameters but never read them. Now they all use `_carried_person()` and pass its result as the `person` field.

---

## B5 — Name joining (SIBLING MISSES — 3 JOIN SITES, ALL INCONSISTENT)

**Canonical helper:** `TreatmentPlan.utils.names::full_name()` — NOT IMPORTED OR USED ANYWHERE

**Belongs in:** Should be used at display/logging sites, but NOT imported

**Sites (inconsistent join patterns):**
- `actions.py:370` — f-string: `f"A patient named {first_name} {last_name} already exists"`
- `actions.py:646` — f-string: `f"{duplicate_patient.first_name} {duplicate_patient.last_name}"`
- `actions.py:1336` — f-string: `f"{patient.first_name} {patient.last_name}" if patient else None`
- `events.py:17` — returns raw `intake.first_name`, `intake.last_name` separately (not joined)
- `events.py:18` — same as above

**Do they actually agree?** IDENTICAL PATTERN — all use f-string join. Never use `full_name()` helper.

**Risk:** VERY LOW — these are logging/response messages, not identity keys. Name splitting/joining for display is cosmetic (no duplicates due to space position). However, these sites are SIBLINGS that could have used the canonical `full_name()` helper for consistency (if that helper were imported).

**Verification:** The canonical helper exists at `TreatmentPlan/utils/names.py` but is not imported in automations. No actual bugs found — just style inconsistency.

---

## B6 — Dentally patient lookup with Dentally IDs (3 SEQUENTIAL FILTERS)

**Canonical pattern:** Try dentally_id, then uuid, then account_id (lines 431-441)

**Belongs in:** Already correct — sequential fallback with practice scope

**Sites:**
- `actions.py:431-433` — `Patient.objects.filter(practice=practice, meta_data__id=dentally_id).first()`
- `actions.py:435-437` — `Patient.objects.filter(practice=practice, meta_data__uuid=dentally_uuid).first()`
- `actions.py:439-441` — `Patient.objects.filter(practice=practice, meta_data__account_id=account_id).first()`

**Do they actually agree?** IDENTICAL PATTERN — practice-scoped JSON field lookups with `.first()`. These are intentional fallback chains (if A, else try B, else try C).

**Risk:** None — Dentally IDs should be unique per practice. Using `.first()` is appropriate here since these are expected to find 0 or 1 match.

---

## B7 — Patient lookup by email/phone in dentally import (2 PARALLEL BRANCHES, INCONSISTENT COMPARISON METHODS)

**Sites:**
- `actions.py:580-582` (INITIAL lookup after name match) — `email__iexact=email_address`
  ```python
  if email_address:
      existing_patient = duplicate_query.filter(email__iexact=email_address).first()
  ```
  where `email_address` = `dentally_patient.get("email_address")` (RAW, NOT NORMALIZED)

- `actions.py:589-596` (SAME BRANCH) — phone lookup uses `canonical_phone_e164()` + raw match
  ```python
  normalized = canonical_phone_e164(primary_phone, country_code)
  phone_q = Q(phone_number=primary_phone) | Q(...canonical_value=normalized...)
  existing_patient = duplicate_query.filter(phone_q).distinct().first()
  ```

- `actions.py:632-633` (ERROR HANDLER AFTER VALIDATION FAIL) — `email__iexact=email_address`
  ```python
  if duplicate_email and email_address:
      duplicate_patient = Patient.objects.filter(practice=practice, email__iexact=email_address).first()
  ```

- `actions.py:637-640` (ERROR HANDLER AFTER VALIDATION FAIL) — raw exact match
  ```python
  elif duplicate_phone and primary_phone:
      duplicate_patient = Patient.objects.filter(
          practice=practice,
          phone_number=primary_phone,
          country_code=country_code,
      ).first()
  ```

**Do they actually agree?** NO — they differ AND they're INTRA-BRANCH MISSES:
- Email: lines 580-582 and 632-633 both use `iexact` on RAW dentally email ✓ consistent
- Phone: line 589-596 uses canonical + raw fallback; line 637-640 uses ONLY raw exact match ✗ **DIVERGENCE**

**Risk:** Line 637-640 (error-handler phone lookup) will miss a match if the stored phone is normalized and the incoming `primary_phone` is raw (or vice versa). This is exactly backwards from lines 589-596 which handle BOTH formats. **However:** this code path only fires AFTER serializer validation fails (lines 609-625), suggesting the patient was already rejected before this point. Executing twice on different criteria is incoherent but may not be reachable in normal flow — **unverified whether this branch executes**.

**Evidence:** 
- `actions.py:637-640` exact-match filter on `phone_number=primary_phone` (NO canonical join)
- `actions.py:589-596` multi-clause `phone_q` with both raw and canonical (CORRECT)

---

## B8 — Email comparison in treatment plan patient lookup (IMPROVED)

**Site:** `actions.py:1049-1068`

**Pattern:** `canonical_email()` + `iexact` + channel graph lookup

```python
normalized_email = canonical_email(email)
email_q = Q(email__iexact=normalized_email)
email_channel = ContactChannel.find_channel(practice=practice, kind=ContactChannel.EMAIL, raw=email)
if email_channel:
    email_q |= Q(person__person_channels__channel_id=email_channel.id)
patient = Patient.objects.filter(practice=practice).filter(email_q).distinct().first()
```

**Comment (lines 1050-1055):** Documents finding #14 — earlier bare `email=email` comparison missed "Ada@Example.com" ↔ "ada@example.com" (1,565 mixed-case patients). Fixed to use `iexact` + channel graph.

**Do they agree with email handling elsewhere?** YES — consistent with the general pattern. BUT this is the ONLY place that:
1. Uses `canonical_email()` explicitly (not `normalize_email`)
2. Checks both the column AND the channel graph
3. Uses `.distinct()` on the result

**Risk:** None — this is the CORRECT and most complete email lookup in the file. It is not a duplicate pattern because it's in a different context (finding existing patient vs. creating new one). The long comment is the RECORD OF FIX #14.

---

## Summary

| Behaviour | Sites | Canonical Helper | Status | Risk |
|-----------|-------|------------------|--------|------|
| Email normalization | 3 (all ingest paths) | `normalize_email()` | UNIFIED | None |
| Phone canonicalization | 5 (all dupe checks) | `canonical_phone_e164()` + `pick_usable_phone()` | UNIFIED | None |
| Practice scope | ALL | Implicit pattern `practice=practice` | UNANIMOUS | None |
| Person identity carry | 3 handlers | `_carried_person()` | UNIFIED + tested | FIXED (#47) |
| Name joining | 5 sites | `full_name()` NOT imported | Consistent f-strings | Very low (cosmetic) |
| Dentally ID lookup | 3 chains | Sequential `.first()` on meta_data | Correct | None |
| Email/phone duplicate lookup | dentally branch | Mixed: `iexact` on raw email, canonical on phone | **DIVERGENCE** | Low — error-handler phone lookup misses canonical match but may not be reachable |

---

## Negative Findings

- ✓ No `sole_patient_for_person()` or `sole_*` helpers used anywhere, but usage patterns show `.first()` is intentional (fallback chain, not expecting exclusivity)
- ✓ No bareword phone comparison (no `phone_number=x` without canonical backup) except in error handler (B7)
- ✓ No unscoped Patient/Intake queries found
- ✓ No identity-carry parameters ignored (finding #47 is FIXED via `_carried_person()`)
- ✓ No case-sensitive email comparison in ingest paths (finding #14 is FIXED via line 1050-1068 lookup)

---

## Unverified

- **B7 error-handler divergence (line 637-640):** Whether the phone-lookup path in the error handler is actually reachable. The code filters `duplicate_query` twice on different phone criteria (lines 589-596 vs. 637-640) after the serializer validates. Requires execution trace or test to confirm if this code path fires.

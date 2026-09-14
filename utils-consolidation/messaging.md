# Duplicate Logic Inventory: messaging/ Directory

## B1 — Name joining via f-string (DIVERGENT pattern)  (18 sites)

**Canonical helper:** `TreatmentPlan.utils.names::full_name`

**Belongs in:** Already imported but inconsistently used; consolidate all sites to call it

**Sites:**
- `messaging/models.py:274` — `name = f"{patient.first_name or ''} {patient.last_name or ''}".strip()`
- `messaging/models.py:313` — `f"{link.person.first_name or ''} {link.person.last_name or ''}".strip().lower()`
- `messaging/models.py:482` — `f"{person.first_name} {person.last_name}".strip() or None`
- `messaging/models.py:560` — `f"{p.first_name} {p.last_name}".strip().lower() == name_lower`
- `messaging/models.py:839` — `patient_name = f"{self.patient.first_name} {self.patient.last_name}"`
- `messaging/models.py:841` — `patient_name = f"{self.non_registered_patient.first_name} {self.non_registered_patient.last_name}"`
- `messaging/models.py:907` — `patient_name = f"{self.patient.first_name} {self.patient.last_name}"`
- `messaging/models.py:909` — `patient_name = f"{self.non_registered_patient.first_name} {self.non_registered_patient.last_name}"`
- `messaging/serializers.py:84` — `name = f"{person.first_name} {person.last_name}".strip()`
- `messaging/serializers.py:138` — `f"{(intake.first_name or '').strip()} {(intake.last_name or '').strip()}".strip()`
- `messaging/serializers.py:1120` — `return f"{obj.sent_by.first_name} {obj.sent_by.last_name}"`
- `messaging/serializers.py:1152` — `return f"{obj.sent_by.first_name} {obj.sent_by.last_name}"`
- `messaging/serializers.py:1680` — `f"{obj.updated_by.first_name} {obj.updated_by.last_name}".strip()`
- `messaging/views/contact_views.py:437` — `f"{patient.first_name or ''} {patient.last_name or ''}".strip()`
- `messaging/views/template_views.py:347` — `else f"{patient.first_name} {patient.last_name}"`
- `messaging/views/template_views.py:375` — `"name": f"{practitioner.first_name} {practitioner.last_name}".strip()`
- `messaging/views/template_views.py:414` — `"full_name": f"{current_user.first_name or ''} {current_user.last_name or ''}".strip()`
- `messaging/views/template_views.py:1281` — `else f"{patient.first_name} {patient.last_name}"`
- `messaging/views/template_views.py:1307` — `"name": f"{practitioner.first_name} {practitioner.last_name}".strip()`
- `messaging/views/template_views.py:1343` — `"full_name": f"{current_user.first_name or ''} {current_user.last_name or ''}".strip()`

**Do they actually agree?** IDENTICAL — all use f-string join + .strip(), but differ in None-handling: some use `or ''` guards, some don't. `serializers.py:1120` and `:1152` never null-guard, risking "None string".

**Risk:** `serializers.py:1120` and `:1152` will render "None Smith" if sent_by has no first_name. Silent data corruption on display. `full_name()` is None-safe; f-strings are not.

---

## B2 — Name splitting (family disambiguation on CallLog save)  (1 site)

**Canonical helper:** NONE — this is a custom split specific to CallLog's need to break `display_name` into first/last

**Belongs in:** `messaging/` (call-log-specific, not shareable)

**Sites:**
- `messaging/models.py:2471` — `parts = live_people[0].display_name.split(" ", 1)` followed by `self.first_name = parts[0]` and `self.last_name = parts[1] if len(parts) > 1 else ""`

**Do they actually agree?** UNIQUE — only one implementation.

**Risk:** Splits at first space only, so "John Michael Smith" becomes first="John", last="Michael Smith". Matches calllog's own join pattern (`f"{first} {last}"` below), so round-trip is stable. Risk is medium: depends on display_name source always being a real name (not phone/email fallback). See context around line 2470: guard checks `live_people[0].display_name` exists before split, so no crash on empty.

---

## B3 — Email normalization and comparison (DIVERGENT SOURCES)  (8 sites + 3 control)

**Canonical helper:** `TreatmentPlan.utils.contact_keys::canonical_email`

**Belongs in:** Global `TreatmentPlan/utils/contact_keys.py` (both routes already use it for create/update)

**Sites:**

Hand-rolled `.lower()` comparisons (identity risk):
- `messaging/serializers.py:468` — `if p.email and p.email.lower() == obj.participant_email.lower()` (finding viewer patient on email session)
- `messaging/serializers.py:1647-1648` — `if email.lower() not in seen: ... seen.add(email.lower())` (dedup in reply-to list)
- `messaging/views/message_views.py:1215` — `sorted([email.from_treatment_path.lower(), email.to_patient.lower()])` (sorting for dedup key)
- `messaging/views/message_views.py:2105` — `to_email = extract_email_address(data.get("to", "")).lower()` (webhook email parse)
- `messaging/views/message_views.py:2136` — `sender_email = extract_email_address(data.get("from", "")).lower()` (webhook email parse)
- `messaging/views/message_views.py:2107` — `to_email.split("@")[0].replace("-", "").lower() if to_email else None` (extracting local part for logic)

Canonical or __iexact (correct):
- `messaging/models.py:513` — `ValueQ(email__iexact=canonical)` (auto-link unlinked records)
- `messaging/signals.py:258` — `records = records.filter(patient_email__iexact=value)` (inbound webhook match)
- `messaging/views/message_views.py:1442-1443` — `models.Q(from_treatment_path__iexact=email) | models.Q(to_patient__iexact=email)` (find session)

**Do they actually agree?** DIFFER — `.lower()` alone vs `canonical_email()` + `__iexact`. Diference is case-sensitivity (both handle it) + whitespace (canonical_email strips; .lower() does not). The `.split("@")[0].replace("-", "")` variant at 2107 is attempting a canonical local-part but diverges from standard practices.

**Risk:** `serializers.py:468` silently fails to find a match if one email has trailing whitespace and the other doesn't. Example: `"alice@example.com "` != `"alice@example.com"` under `.lower()` but both pass `canonical_email()`. Since this is a viewer lookup, the viewer is silently assigned to wrong patient or None (returned at line 490). `messaging/views/message_views.py:2105` stores the result in `to_email` used as a lookup key; whitespace causes key mismatches that split one conversation into two sessions.

---

## B4 — Which human owns this? Using .first() to pick a record  (2 sites, CRITICAL)

**Canonical helper:** `TreatmentPlan.utils.sole_patient::sole_record_for_person`

**Belongs in:** Global `TreatmentPlan/utils/sole_patient.py` (the canonical guard)

**Sites (DIVERGENT IMPLEMENTATIONS):**
- `messaging/serializers.py:133` — `intake = person.intakes.first()` (in _get_linked_intake_name, no sole_* guard)
- `messaging/serializers.py:1782` — `nurture = person.nurtures.first()` (in get_nurture_id, no sole_* guard)

**Correct implementations in same file (for comparison):**
- `messaging/serializers.py:1762-1765` — Uses `sole_patient_for_person(person, ...)` for patient lookup
- `messaging/serializers.py:1770-1776` — Uses `sole_record_for_person(person, "intakes", ...)` for intake lookup (same method as line 133 SHOULD use)

**Do they actually agree?** NO — Line 1782 uses `.first()` while line 1770 (same class, same method family) uses `sole_record_for_person()`. This is the smoking gun: the coder knew the right pattern but applied it inconsistently.

**Risk:** VERY HIGH. On a fused Person (two humans merged incorrectly) or a household link shared by siblings, `intakes.first()` and `nurtures.first()` return arbitrary records, attributing one human's intake/nurture to another. Line 133's result `full_name` becomes `_get_linked_intake_name` which then appears in serializer output and UI. Line 1782 returns a nurture_id that breaks touch-point counts and activity attribution. This is exactly the family-phone bug class (patient_identity_audit #74/#75 in memory).

---

## B5 — Display name construction (duplicates global logic)  (1 site)

**Canonical helper:** `TreatmentPath.utils::get_display_name_for_user`

**Belongs in:** Should import and use the global, not reimplement locally

**Sites:**
- `messaging/views/contact_views.py:100-112` — `_sender_display_name(first, last, user_type, email)` function

**Code:**
```python
def _sender_display_name(first, last, user_type, email):
    if user_type == "superuser":
        return "System Admin"
    full_name = f"{first or ''} {last or ''}".strip()
    return full_name or email or None
```

**Canonical (from comment at line 103):**
```python
# Mirrors TreatmentPath.utils.get_display_name_for_user field-for-field
```

**Do they actually agree?** IDENTICAL in intent, but implemented twice. Comment explicitly says it mirrors the global helper to avoid N+1 (values() dicts can't call model methods). Code path: both check `user_type == "superuser"` → "System Admin", both build `f"{first or ''} {last or ''}"` + fallback to email.

**Risk:** Low for display, medium for maintenance. If `get_display_name_for_user` logic changes (e.g. middle initial), this copy won't follow. The comment justifies the duplication (N+1 avoidance via .values()), so the risk is drift, not a silent bug. Safe to keep if the comment is a permanent guard; at risk if that call site is refactored away.

---

## B6 — Name normalization for conflict detection  (1 site)

**Canonical helper:** NONE — this is a custom comparison for session rename validation

**Belongs in:** `messaging/serializers.py` (rename-conflict-specific)

**Sites:**
- `messaging/serializers.py:115-118` — `_normalize_name_for_compare(value)` method

**Code:**
```python
@staticmethod
def _normalize_name_for_compare(value):
    if value is None:
        return ""
    return " ".join(str(value).strip().lower().split())
```

**Do they actually agree?** UNIQUE — only one implementation. Used in validation to decide if a rename conflict is real (lines 303-305).

**Risk:** Low. Collapses whitespace + lowercases + stringifies. Safe for comparison logic (whether two names match for conflict detection). Does not affect identity or data storage, only decision logic.

---

## B7 — Practice scoping  (Consistent pattern)

**Canonical helper:** `utils.practice_mixins::PracticeAccessMixin`, `utils.ws_practice::current_practice_id`, `TreatmentPath.utils::scope_clinical_to_practice`

**Belongs in:** Already centralized; usage is consistent

**Sites (spot check — all correct):**
- `messaging/views/session_views.py:61-70` — Uses `PracticeAccessMixin` + `get_user_practice_or_none()` + `filter(practice=practice)`
- `messaging/views/call_log_views.py:28` — Uses `PracticeAccessMixin`
- `messaging/views/domain_views.py:31` — Uses `PracticeAccessMixin`

**Do they actually agree?** YES — all sites follow the same pattern. No hand-rolled practice filters found.

**Risk:** None identified. Pattern is correct and consistent.

---

## B8 — Phone normalization  (Consistent)

**Canonical helper:** `TreatmentPlan.utils::canonical_phone_e164`, `parse_phone_number`

**Belongs in:** Already centralized; all sites import and use it correctly

**Sites (spot check):**
- `messaging/models.py:199, 232, 243` — Uses `canonical_phone_e164()`
- `messaging/views/message_views.py:262` — Uses `canonical_phone_e164()`
- `messaging/serializers.py:352-354` — Uses `canonical_phone_e164()`

**Do they actually agree?** YES — all phone normalizations go through canonical_phone_e164. No hand-rolled phone logic identified (except the session split/split logic, which wraps the canonical functions).

**Risk:** None identified.

---

## Summary Table

| Behaviour | Sites | Agree? | Risk Level | Proposed Home |
|-----------|-------|--------|------------|---------------|
| B1 — Name joining (f-string) | 20 | IDENTICAL (impl varies) | MEDIUM | Consolidate to `full_name()` |
| B2 — Name splitting (CallLog) | 1 | UNIQUE | LOW | Keep in `messaging/models.py` |
| B3 — Email comparison | 8+3 | DIFFER (.lower() vs __iexact/canonical) | MEDIUM–HIGH | Use `canonical_email()` + __iexact everywhere |
| B4 — Which human? (.first()) | 2 | DIFFER (.first() vs sole_record_for_person) | **CRITICAL** | Use `sole_record_for_person()` at lines 133, 1782 |
| B5 — Display name (User) | 1 | IDENTICAL (reimplemented) | LOW–MEDIUM | Keep dupe; comment guards it (N+1 reason) |
| B6 — Name normalization (compare) | 1 | UNIQUE | LOW | Keep in `messaging/serializers.py` |
| B7 — Practice scoping | Multiple | IDENTICAL | NONE | No action needed |
| B8 — Phone normalization | Multiple | IDENTICAL | NONE | No action needed |

---

## Critical Issues Summary

1. **B4 is the highest priority:** Lines 133 and 1782 in `messaging/serializers.py` use `.first()` to pick records on families/fused persons, risking arbitrary selection. Line 1770 in the same class already shows the correct pattern (`sole_record_for_person`). This is a family-phone identity bug waiting to happen.

2. **B3 has active divergence:** Email matching uses both `.lower()` and `canonical_email()`, allowing whitespace mismatches to split conversations.

3. **B1 is pervasive:** 20 hand-rolled f-string joins vs one centralized `full_name()` helper. Not immediately dangerous (most include None guards), but creates maintenance drift and None-handling risk at lines 1120, 1152 (no guard).

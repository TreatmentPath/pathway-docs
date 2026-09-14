# Duplicate Logic Inventory: activityLog/

## B1 — Entity Type Display Formatting (3 sites)

**Canonical helper:** NONE — would benefit from extraction to a shared method

**Belongs in:** `activityLog/utils/` (new module)

**Sites:**
- `activityLog/serializers.py:64-71` — `ActivitySerializer.get_entity_type_display()`
- `activityLog/serializers.py:232-239` — `ActivityLogSerializer.get_entity_type_display()`
- `activityLog/serializers.py:319-326` — `ActivityLogTimelineSerializer.get_entity_type_display()`

**Code (all three identical):**
```python
def get_entity_type_display(self, obj):
    if obj.entity_type.lower() == "messagesession":
        session_type = str((obj.metadata or {}).get("session_type") or "").lower()
        if session_type == "sms":
            return "SMS Session"
        if session_type == "email":
            return "Email Session"
    return ENTITY_TYPE_MAP.get(obj.entity_type.lower(), obj.entity_type.title())
```

**Do they actually agree?** IDENTICAL across all three.

**Risk:** Bug fix applied to one copy misses the other two; display logic divergence if maintainers update independently.

---

## B2 — Email Body Extraction for Activity Display (2 sites)

**Canonical helper:** NONE — would benefit from extraction

**Belongs in:** `activityLog/utils/` (new module)

**Sites:**
- `activityLog/serializers.py:73-86` — `ActivitySerializer.get_entity_body()`
- `activityLog/serializers.py:358-371` — `ActivityLogTimelineSerializer.get_entity_body()`

**Code (both identical):**
```python
def get_entity_body(self, obj):
    """Return the email body when the entity is an EmailMessages record"""
    if (
        obj.entity_type.lower() not in ("emailmessage", "emailmessages")
        or not obj.object_id
    ):
        return None
    try:
        from messaging.models import EmailMessages

        email = EmailMessages.objects.get(pk=obj.object_id)
        return email.body
    except Exception:
        return None
```

**Do they actually agree?** IDENTICAL.

**Risk:** Exception handling logic (bare `except`) duplicated; if EmailMessages access changes, both must update.

---

## B3 — Actor Name Formatting (DIVERGENCE across 5 sites)

**Canonical helper:** `TreatmentPath.utils::get_display_name_for_user()` exists and is used in some places. `TreatmentPlan.utils.names::full_name()` is the canonical name join helper.

**Belongs in:** Unified to use canonical helpers; no fragmentation should exist

**Sites using `get_display_name_for_user()` (CORRECT):**
- `activityLog/serializers.py:58-62` — `ActivitySerializer.get_created_by_name()`
- `activityLog/serializers.py:226-230` — `ActivityLogSerializer.get_created_by_name()`
- `activityLog/serializers.py:313-317` — `ActivityLogTimelineSerializer.get_created_by_name()`

**Sites using manual inline name join (DIVERGENT):**
- `activityLog/serializers.py:439-445` — `PatientAuditEventSerializer.get_actor_name()`:
  ```python
  if obj.created_by:
      first = obj.created_by.first_name or ""
      last = obj.created_by.last_name or ""
      full = f"{first} {last}".strip()
      return full or obj.created_by.get_username()
  return "System"
  ```

- `activityLog/models.py:311-316` — `ActivityLog.__str__()`:
  ```python
  user_name = (
      f"{self.created_by.first_name} {self.created_by.last_name}"
      if self.created_by
      else "System"
  )
  ```

- `activityLog/models.py:424-430` — `Activity.__str__()` (identical to above)

**Do they actually agree?** NO — `PatientAuditEventSerializer` matches the pattern, but it's still a divergence from the canonical `get_display_name_for_user()` used elsewhere. The `__str__` methods do NOT fall back to username when name is empty.

**Risk:** MEDIUM — User displays may show different formats (one uses canonical display logic, one uses raw join). If `first_name`/`last_name` are ever required but missing in one code path, results silently diverge.

---

## B4 — Manual Name Joining in Entity Info (1 site, non-canonical)

**Canonical helper:** `TreatmentPlan.utils.names::full_name()` — not used

**Belongs in:** should use `full_name()` from canonical helpers

**Sites:**
- `activityLog/serializers.py:345-346` — `ActivityLogTimelineSerializer.get_entity_info()`:
  ```python
  if hasattr(entity, "first_name") and hasattr(entity, "last_name"):
      info["name"] = f"{entity.first_name} {entity.last_name}".strip()
  ```

**Risk:** Does NOT use `full_name()`, which is None-safe and collapses internal whitespace. Fragile if entity.first_name or last_name is None (will render as bare spaces, not a fallback). Also misses the display-name centralization for future changes.

---

## B5 — "Whose Event Is This?" Person Resolution (WELL-HANDLED)

**Pattern:** Map activities/events to a specific Person

**Canonical helper:** `resolve_target_person_ids()` (lines 33-76) — dedicated helper with excellent documentation

**Sites:**
- `activityLog/views.py:414` — `patient_activities()` uses `resolve_target_person_ids(patient_id, practice)`
- `activityLog/views.py:446` — Filters `person_id__in=target_person_ids` (correct scope)
- `activityLog/views.py:682` — `patient_audit()` re-uses same helper
- `activityLog/views.py:932` — `patient_audit()` again via NoteHistory scoping

**Verdict:** CORRECT — The helper explicitly documents a fixed bug (ID collision between Patient and Person namespaces), uses a FALLBACK pattern (not union), and is reused consistently. The `by_contact()` endpoint also correctly handles ambiguity with HTTP 300 when one email/phone maps to multiple people.

**Evidence:** The helper's docstring (lines 34-57) shows this was a real audit finding with actual data examples (3 cross-practice leaks documented).

---

## B6 — Author-Membership Gates

**Pattern:** Filtering by who created a record rather than whose record it is

**Canonical helper:** `PracticeFilterMixin` / `scope_clinical_to_practice()` 

**Finding:** NONE — No author-membership gates found in activityLog/. All filtering is by practice or person (the subject of the activity), not by `created_by`. Correct.

---

## B7 — Client IP Extraction

**Pattern:** Reading HTTP_X_FORWARDED_FOR vs REMOTE_ADDR

**Finding:** NONE — No IP extraction logic in activityLog/. Not applicable to this module.

---

## B8 — Canonical Helper Usage Audit

**Correct usage found:**
- `full_name()` — used correctly at `views.py:607` for Person display in `by_contact()` ambiguity resolution
- `phones_match()` — used correctly at `views.py:587` for phone number comparison in `by_contact()`

**Email normalization:** `by_contact()` uses `metadata__email__iexact=contact_identifier` (case-insensitive exact on raw metadata field). This bypasses `canonical_email()` but is reasonable here because:
1. The metadata is legacy/raw data from various sources
2. The contact lookup needs to match what was originally stored
3. It's a read-only lookup, not a write that sets canonical form

No issue found.

---

## Summary Table

| Behaviour | Count | Diverges? | Severity | Notes |
|-----------|-------|-----------|----------|-------|
| Entity type display format | 3 | NO (identical) | MEDIUM | Sibling miss: extract to shared module |
| Email body extraction | 2 | NO (identical) | LOW | Sibling miss: extract to shared module |
| Actor/user name formatting | 5 | YES | MEDIUM | Divergence: some use `get_display_name_for_user()`, others inline; `__str__` methods don't fall back to username |
| Entity name join | 1 | YES | MEDIUM | Non-canonical: should use `full_name()` |
| "Whose event" person resolution | 4 sites | NO | CORRECT | Well-documented helper, reused consistently |
| Author-membership gates | — | — | NONE | None found |
| Client IP extraction | — | — | NONE | None found |
| `.first()` without order_by | — | — | NONE | All on FK lookups with natural 0-1 cardinality |

---

## Recommendations

1. **Extract B1 & B2** to `activityLog/utils/formatters.py`:
   - `format_entity_type_display(entity_type, metadata=None)` 
   - `extract_email_body(entity_type, object_id)`

2. **Unify B3**: Deprecate manual name joins in `PatientAuditEventSerializer` and `Activity.__str__()` in favour of `get_display_name_for_user()` + fallback to username pattern.

3. **Fix B4**: Replace inline join with import and call to `full_name(person.first_name, person.last_name)`.

4. **No changes needed** for B5-B8 (all correct/not applicable).

# dentallyIntegration Directory — Duplicate-Logic Inventory

## B1 — Email canonicalization (1 site, DIFFER)

**Canonical helper:** `TreatmentPlan.utils.contact_keys::canonical_email`

**Belongs in:** consolidate into recall_automation; migrate to import and use canonical_email

**Sites:**
- `recall_automation.py:365` — `canonical = str(raw).strip().lower() or None`

**Do they actually agree?** 
IDENTICAL to the canonical helper (which does `str(raw).strip().lower() or None`), but implemented inline. The canonical_email helper already exists and is used elsewhere in this codebase (bridge_dentally_identity.py:109, recall_sequence_views.py:262, recall_views.py:1474, 2115).

**Risk:** Low immediately (logic is correct) but HIGH long-term: if canonical_email ever changes (e.g. to handle unicode normalization), recall_automation.py will silently drift and lookup keys will no longer match channels written by the canonical path.

---

## B2 — Display name joining (2 sites, DIFFER in robustness)

**Canonical helper:** `TreatmentPlan.utils.names::full_name`

**Belongs in:** Merge next_appointment's _display_name logic into full_name or create a dedicated safe variant

**Sites:**
- `next_appointment.py:66-74` — `_display_name()`: `" ".join(p.strip() for p in parts if p and p.strip())` (None-safe, filters empty strings)
- `models.py:1364` — `RecallPatient.full_name`: imports and calls canonical `full_name(self.first_name, self.last_name)`
- `models.py:898` — `Clinician.display_name`: imports and calls canonical `full_name()`, then prepends title

**Do they actually agree?** 
DIFFER in robustness. The canonical full_name does NOT filter None before joining (calls `full_name(first_name, last_name)` which renders f-string directly), so nullable columns produce "None Smith". The next_appointment._display_name() is deliberately more defensive: it only joins parts that exist AND have visible characters. The comment on line 69 of next_appointment.py explicitly flags this divergence.

**Risk:** Medium. The appointment badge uses next_appointment._display_name and will never show "None Smith", but serializers and models elsewhere use canonical full_name. Patient.full_name (on the model) renders f-string unsafely. This is documented as a known issue (next_appointment line 69-71) but the two implementations coexist in different parts of the code, risking inconsistent behavior if one path is updated without the other.

---

## B3 — Dentally patient ID extraction from meta_data (2 sites, IDENTICAL intent, DIFFER in robustness)

**Canonical helper:** NONE — would need creating

**Belongs in:** Create `TreatmentPlan.utils.contact_keys::dentally_patient_id_from_patient()` (or `dentallyIntegration.utils`) and consolidate both paths

**Sites:**
- `next_appointment.py:45-63` — `dentally_patient_id_for(patient)`: reads `meta_data["id"]`, type-coerces to int, tolerant of None/TypeError/ValueError
- `management/commands/migrate_dentally_clinical.py:748-756` — inline logic: reads `meta_data.get("id")`, separate try/except for int() coercion, same tolerance pattern

**Do they actually agree?** 
IDENTICAL intent (extract numeric Dentally id from Patient.meta_data), but:
- next_appointment uses `getattr(patient, "meta_data", None)` first, checks `isinstance(meta, dict)`, then `int(meta.get("id"))`
- migrate_dentally_clinical.py assumes meta_data exists and directly calls `patient.meta_data.get("id")`, then wraps only the int() in try/except
- tasks.py:963 (snapshot logging) reads meta_data.get("id") with NO type coercion

**Risk:** Medium. The comment in next_appointment.py (line 49) explicitly states this is the same lookup as `daylist_reporting_views._resolve_channel_ids` but they are NOT consolidated. If one Patient's meta_data is None or malformed, next_appointment.py tolerates it silently (returns None), while migrate_dentally_clinical.py would crash. This pattern is fragile across two implementation styles.

---

## B4 — Manual name joining in logging/display strings (10+ sites, INCONSISTENT)

**Canonical helper:** `TreatmentPlan.utils.names::full_name`

**Belongs in:** Already using full_name in models; replace all f-string joins with a helper call

**Sites:**
- `tasks.py:158` — `f"{primary_person.first_name} {primary_person.last_name}".strip()`
- `tasks.py:187` — `f"{sibling.first_name} {sibling.last_name}"`
- `tasks.py:804` — `f"{dentally_patient.get('first_name')} {dentally_patient.get('last_name')}"`
- `tasks.py:911` — `f"Dentally patient {dentally_id} ({first_name} {last_name})"`
- `tasks.py:934` — `f"{patient_data['first_name']} {patient_data['last_name']}"`
- `tasks.py:1015` — `f"{patient_data['first_name']} {patient_data['last_name']}"`
- `tasks.py:1047` — `f"{patient_data['first_name']} {patient_data['last_name']}"`
- `tasks.py:1097` — `f"{exact_duplicate.first_name} {exact_duplicate.last_name}"`
- `tasks.py:1194` — `f"{duplicate_matches[0]['first_name']} {duplicate_matches[0]['last_name']}"`
- `tasks.py:1256` — `f"{dentally_patient.get('first_name', '')} {dentally_patient.get('last_name', '')}"`
- `views/dentally_views.py:936` — `f"{first_name} {last_name}"`
- `views/dentally_views.py:957` — `f"{first_name} {last_name}"`
- `views/dentally_views.py:962` — `f"Duplicate patient: {first_name} {last_name} with email {email}"`

**Do they actually agree?** 
All are simple f-string joins, but with VARYING NULL-SAFETY. Most assume fields exist. Line 1256 uses `.get('first_name', '')` with a default. Line 158 calls `.strip()` on the result (can produce "None" if one field is None).

**Risk:** Low for identity (these are logging/error messages, not keys), but cosmetic UX risk. A patient record with NULL first_name will render "None Smith" in error logs in some paths but not others. Not an identity bug, but inconsistent presentation and a sign the code is not using a unified display pipeline.

---

## B5 — Phone stripping before channel creation

**Canonical helper:** `TreatmentPlan.utils::canonical_phone_e164` (used correctly downstream)

**Belongs in:** Already correct; get_or_create_channel handles canonicalization

**Sites:**
- `recall_automation.py:580-581` — `phone = (rec.patient_phone or "").strip()` before calling `get_or_create_channel(practice, ContactChannel.PHONE, phone)`

**Do they actually agree?** 
YES, this is correct. The .strip() is defensive (in case the raw data has whitespace), and get_or_create_channel internally calls canonical_phone_e164 to produce the canonical_value. No duplication here.

**Risk:** None; this is the correct pattern.

---

## Summary

| Behaviour | Sites | Agree? | Proposed Home |
|-----------|-------|--------|---|
| B1: Email canonicalization | 1 | IDENTICAL but duplicate | Consolidate: recall_automation.py import + use `canonical_email()` |
| B2: Display name joining | 2+ | DIFFER (robustness) | Unify: extend `full_name()` to handle None-safe case or document the split |
| B3: Dentally patient ID extraction | 2 | DIFFER (robustness) | Create new canonical: `dentallyIntegration.utils::dentally_patient_id_from_patient()` or migrate to TreatmentPlan.utils |
| B4: Name joining in logging | 10+ | All f-string joins, varying NULL safety | Low risk; cosmetic unification via `full_name()` or logging helper |
| B5: Phone stripping | 1 | Correct | No action needed |

**Key insight:** B1 (email canonicalization) is the highest-risk duplicate — it directly impacts identity key matching. B3 (dentally_patient_id extraction) should be consolidated to prevent silent failures on malformed data. B2 (display name) needs a documented choice: either unify on safe behavior (like _display_name) or accept the split.

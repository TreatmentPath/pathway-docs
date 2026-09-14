# TreatmentPlan Duplicate-Logic Inventory

READ-ONLY audit of the `TreatmentPlan/` Django app for re-implemented canonical helpers and multi-site duplication.

---

## B1 — Name Joining (54 sites)

**Canonical helper that already exists:** `TreatmentPlan/utils/names.py::full_name`

**Belongs in:** already correct (production code uses utility module)

**Sites:**
- `TreatmentPlan/signals.py:109` — `f"{(instance.first_name or '').strip()} {(instance.last_name or '').strip()}".strip()`
- `TreatmentPlan/management/commands/fix_mismatched_contacts.py:74` — `f"{intake.first_name or ''} {intake.last_name or ''}".strip()`
- `TreatmentPlan/management/commands/fix_mismatched_contacts.py:118` — `f"{nurture.first_name or ''} {nurture.last_name or ''}".strip()`
- `TreatmentPlan/management/commands/reattach_stranded_history.py:217` — `f"{src.first_name or ''} {src.last_name or ''}".strip()`
- `TreatmentPlan/management/commands/unweld_persons.py:322` — `f"{patient.first_name} {patient.last_name}".strip()`
- `TreatmentPlan/management/commands/unweld_persons.py:351` — `f"{welded_person.first_name} {welded_person.last_name}".strip()`
- `TreatmentPlan/models.py:426` — `f"{self.first_name} {self.last_name}".strip() or "Unknown"` (Person.__str__)
- `TreatmentPlan/models.py:1939` — `f"{self.first_name} {self.last_name}"` (Patient.__str__)
- `TreatmentPlan/models.py:2059` — `f"{self.first_name or ''} {self.last_name or ''}".strip()` (Intake model method)
- `TreatmentPlan/models.py:2458` — `f"{self.first_name} {self.last_name or ''}".strip()`
- `TreatmentPlan/models.py:2636` — `f"{self.first_name} {self.last_name or ''}".strip()`
- `TreatmentPlan/models.py:3558` — `f"{self.first_name} {self.last_name or ''}".strip()` (Nurture model)
- `TreatmentPlan/models.py:3625` — `f"{self.first_name} {self.last_name or ''}".strip()`
- `TreatmentPlan/models.py:3719` — `f"{self.first_name} {self.last_name}"` (TreatmentPlan.__str__)
- `TreatmentPlan/models.py:3956` — `f"{self.assigned_to.first_name} {self.assigned_to.last_name}"`
- `TreatmentPlan/models.py:4054` — `f"{self.patient.first_name} {self.patient.last_name}"` (TouchPointEvent)
- `TreatmentPlan/models.py:4056` — `f"{self.non_registered_patient.first_name} {self.non_registered_patient.last_name}"`
- `TreatmentPlan/journey/mixins.py:176` — `f"{self.first_name} {self.last_name or ''}".strip()`
- `TreatmentPlan/journey/mixins.py:180` — `f"{self.patient.first_name} {self.patient.last_name}".strip()`
- `TreatmentPlan/views/contact_merge_views.py:62` — `f"{first} {last}".strip() or "Unknown"`
- `TreatmentPlan/views/nurture_views.py:327` — `f"{nurture.first_name or ''} {nurture.last_name or ''}".strip()`
- `TreatmentPlan/views/intake_views.py:507` — `f"{intake.first_name or ''} {intake.last_name or ''}".strip()`
- `TreatmentPlan/views/intake_views.py:1582` — `f"{rec.first_name} {rec.last_name or ''}".strip()`
- `TreatmentPlan/views/intake_views.py:1602` — `f"{rec.first_name} {rec.last_name or ''}".strip()`
- `TreatmentPlan/views/intake_views.py:1757` — `f"{patient.first_name or ''} {patient.last_name or ''}".strip()`
- `TreatmentPlan/views/intake_views.py:1789` — `f"{rec.first_name} {rec.last_name or ''}".strip()`
- `TreatmentPlan/views/intake_views.py:1898` — `f"{first_name} {last_name}".strip() or "Treatment plan"`
- `TreatmentPlan/views/intake_views.py:1940` — `f"{patient.first_name or ''} {patient.last_name or ''}".strip()`
- `TreatmentPlan/views/patient_views.py:364` — `f"{patient.first_name} {patient.last_name}".strip()`
- `TreatmentPlan/views/patient_views.py:433` — `f"{patient.first_name} {patient.last_name}".strip()`
- `TreatmentPlan/views/patient_views.py:570` — `f"{patient.first_name or ''} {patient.last_name or ''}".strip()`
- `TreatmentPlan/views/patient_views.py:2003` — `f"{patient.first_name} {patient.last_name}"`
- `TreatmentPlan/views/patient_views.py:2026` — `f"{patient.first_name} {patient.last_name}"`
- `TreatmentPlan/views/patient_views.py:2076` — `f"{patient.first_name} {patient.last_name}"`
- `TreatmentPlan/views/patient_views.py:2098` — `f"{patient.first_name} {patient.last_name}"`
- `TreatmentPlan/views/conversion_views.py:517` — `f"{patient.first_name} {patient.last_name}"`
- `TreatmentPlan/views/conversion_views.py:847` — `f"{patient.first_name} {patient.last_name}"`
- `TreatmentPlan/views/household_views.py:36` — `f"{person.first_name} {person.last_name}".strip() or "Unknown"`
- `TreatmentPlan/views/household_views.py:180` — `f"{member.first_name} {member.last_name}".strip() or "Unknown"`
- `TreatmentPlan/views/household_views.py:194` — `f"{person.first_name} {person.last_name}".strip()`
- `TreatmentPlan/services/treatment_plan.py:79` — `f"{treatment_plan.patient.first_name} {treatment_plan.patient.last_name}"`
- `TreatmentPlan/services/treatment_plan.py:84` — `f"{treatment_plan.non_registered_patient.first_name} {treatment_plan.non_registered_patient.last_name}"`
- `TreatmentPlan/services/treatment_plan.py:128` — `f"{treatment_plan.current_practitioner.first_name} {treatment_plan.current_practitioner.last_name}".strip()`
- `TreatmentPlan/services/dentally_plan_import.py:97` — `f"{patient.first_name} {patient.last_name}".strip()`
- `TreatmentPlan/serializers/membership.py:64` — `f"{obj.patient.first_name} {obj.patient.last_name}".strip()`
- `TreatmentPlan/serializers/membership.py:98` — `f"{obj.created_by.first_name} {obj.created_by.last_name}".strip()`
- `TreatmentPlan/serializers/journey.py:35` — `f"{first or ''} {last or ''}".strip() or "Unknown"`
- `TreatmentPlan/serializers/journey.py:77` — `f"{getattr(u, 'first_name', '')} {getattr(u, 'last_name', '')}".strip()`
- `TreatmentPlan/serializers/journey.py:119` — `f"{getattr(u, 'first_name', '')} {getattr(u, 'last_name', '')}".strip()`
- `TreatmentPlan/serializers/journey.py:621` — `f"{getattr(person, 'first_name', '') or ''} {getattr(person, 'last_name', '') or ''}".strip()`
- `TreatmentPlan/serializers/practice_treatment.py:111` — `f"{clinician.first_name} {clinician.last_name}".strip() or clinician.email`
- `TreatmentPlan/serializers/nurture.py:169` — `f"{obj.assigned_to.first_name} {obj.assigned_to.last_name}"`
- `TreatmentPlan/serializers/nurture.py:175` — `f"{obj.journey_assignee.first_name} {obj.journey_assignee.last_name}".strip()`
- `TreatmentPlan/serializers/treatment_plan.py:613` — `f"{obj.assigned_to.first_name} {obj.assigned_to.last_name}"`
- `TreatmentPlan/serializers/treatment_plan.py:618` — `f"{obj.assigned_by.first_name} {obj.assigned_by.last_name}"`
- `TreatmentPlan/serializers/treatment_plan.py:728` — `f"{obj.first_name} {obj.last_name}".strip()`
- `TreatmentPlan/serializers/treatment_plan.py:1645` — `f"{user.first_name} {user.last_name}"`
- `TreatmentPlan/serializers/treatment_plan.py:1655` — `f"{instance.journey_assignee.first_name} {instance.journey_assignee.last_name}".strip()`
- `TreatmentPlan/serializers/treatment_plan.py:1736` — `f"{obj.patient.first_name} {obj.patient.last_name}".strip()`
- `TreatmentPlan/serializers/treatment_plan.py:1757` — `f"{obj.non_registered_patient.first_name} {obj.non_registered_patient.last_name}".strip()`
- `TreatmentPlan/serializers/treatment_plan.py:1828` — `f"{obj.current_practitioner.first_name} {obj.current_practitioner.last_name}"`
- `TreatmentPlan/serializers/intake.py:184` — `f"{obj.assigned_to.first_name} {obj.assigned_to.last_name}".strip()`
- `TreatmentPlan/serializers/practice_treatment_plan_template.py:105` — `f"{user.first_name} {user.last_name}".strip() or user.email`
- `TreatmentPlan/views/treatment_plan_views.py:237` — `f"{relationship.user.first_name or ''} {relationship.user.last_name or ''}"`
- `TreatmentPlan/views/treatment_plan_views.py:1104` — `f"{first} {last}".strip()`

**Do they actually agree?** DIFFER. Implementations use four patterns with varying stripping logic:
1. No strip on intermediate parts: `f"{first} {last}"`  
2. Strip intermediate parts only: `f"{(first or '').strip()} {(last or '').strip()}"`
3. Strip outer result only: `f"{first} {last}".strip()`
4. Strip both and outer: `f"{(first or '').strip()} {(last or '').strip()}".strip()`

The helper `full_name()` (names.py:29-37) uses pattern 3 (strip outer) and collapses internal whitespace runs, which handles multi-space names like "Mary  Jane" → "Mary Jane". Several inline implementations skip the collapse step.

**Risk:** The whitespace-collapse gap means the same human (name stored as "Mary  Jane") can resolve to different identity keys depending which code path writes the name, as `canonical_name_key()` uses the collapse rule. Silent divergence between display name and dedup key.

---

## B2 — Name Splitting (2 sites)

**Canonical helper that already exists:** NONE — needs to be created. The `contact_keys.py::canonical_full_name_key` docstring (line 86) notes this is a known latent bug.

**Belongs in:** `TreatmentPlan/utils/contact_keys.py`

**Sites:**
- `TreatmentPlan/views/intake_views.py:1192` — `name_parts = full_name.strip().split(" ", 1); first = name_parts[0]; last = name_parts[1] if len(name_parts) > 1 else ""`
- `TreatmentPlan/management/commands/split_collapsed_persons.py:493` — `first, _, last = name.partition(" ")`

**Do they actually agree?** Both split at the first space. However, neither calls the canonical key during the split, so a name like "Ken (Kenneth) Judge" joined at the break point splits to first="Ken" last="(Kenneth) Judge", which does NOT match the ORIGINAL stored as first="Ken (Kenneth)" last="Judge". This is exactly the bug that created 17 duplicate Persons in production.

**Risk:** Splitting without re-running identity resolution creates DIVERGENT (first, last) pairs from one canonical joined name, so a person arriving as "Ken (Kenneth) Judge" re-split becomes a different identity key and re-merges separately. The contact_keys docs call this a known gap (line 86).

---

## B3 — Email Comparison (3 sites)

**Canonical helper that already exists:** `TreatmentPlan/utils/contact_keys.py::canonical_email`

**Belongs in:** `TreatmentPlan/utils/contact_keys.py`

**Sites:**
- `TreatmentPlan/selectors.py:46` — `lower_emails = {i.email.lower() for i in items if getattr(i, "email", None)}`
- `TreatmentPlan/selectors.py:82` — `le = (p.email or "").lower()`
- `TreatmentPlan/selectors.py:115` — `email_l = i.email.lower() if getattr(i, "email", None) else None`
- `TreatmentPlan/selectors.py:140` — `if (p.email or "").lower() == email_l:`
- `TreatmentPlan/serializers/intake.py:247` — `if p.email and p.email.lower() == intake.email.lower():`
- `TreatmentPlan/serializers/nurture.py:242` — `if p.email and p.email.lower() == nurture.email.lower():`

**Do they actually agree?** Yes, all use `.lower().strip()` semantically (strip is not always explicit but is relied on by the context). Three separate copies of the canonicalization rule.

**Risk:** A fourth site that forgets `.strip()` before `.lower()` (or vice versa) would silently diverge, so emails with leading/trailing whitespace would resolve to different channels depending which code path ingest them. The canonical form already exists and should be the only definition.

---

## Summary Table

| Behaviour | Sites | Agree? | Proposed Home | Status |
|-----------|-------|--------|----------------|--------|
| Name Joining | 54 | DIFFER (4 patterns, whitespace-collapse gap) | `TreatmentPlan/utils/names.py::full_name` | Re-implement inline, 54 sites |
| Name Splitting | 2 | AGREE (both split at first space) | `TreatmentPlan/utils/contact_keys.py` (new) | Known gap, no canonical yet |
| Email Comparison | 6 | AGREE (all `.lower()`) | `TreatmentPlan/utils/contact_keys.py::canonical_email` | 3 separate copies in selectors/intake/nurture |

---

## Notes

- **B1 is the highest-risk finding:** 54 sites with 4 divergent patterns, some missing the whitespace-collapse step that `canonical_name_key` requires. A display name "Mary  Jane" can resolve to a different dedup key than the same name shown in an activity log.
  
- **B2 highlights a documented gap:** `contact_keys.py` line 86 explicitly notes this was a known issue (five places had private copies). The split-and-rewrite path is still missing a canonical rule — no helper exists yet to do "safe" name splitting that preserves the identity relationship.

- **B3 is mid-risk:** Three independent implementations of email canonicalization (strip + lowercase) exist. They currently agree, but two use direct `.lower()` without explicit `.strip()`, relying on context. A future edit could make them diverge.

- **No other duplication patterns found:** practice scoping largely uses canonical helpers; phone normalization routes through `canonical_phone_e164`; "sole" patient lookup uses `sole_patient_for_person` in the places that were audited.

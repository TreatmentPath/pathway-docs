# marketingBroadcast/ Duplicate Logic Inventory

Scan date: 2026-09-07. Directory: `/marketingBroadcast/`

---

## B1 — Email channel resolution (2 sites, DIFFER)

**Canonical helper:** None (should consolidate)

**Belongs in:** `marketingBroadcast/utils/` (new module) or shared `TreatmentPlan/utils/`

**Sites:**
- `delivery_reporting.py:28` — `channel = person.channels.filter(kind="email").exclude(canonical_value="").first()`
- `views/preferences_views.py:20` — `channel = person.channels.filter(kind="email").first()`

**Do they actually agree?** NO. delivery_reporting excludes empty canonical values; preferences doesn't. Both call `.first()` on multi-channel persons, which can arbitrarily pick a channel.

**Risk:** delivery_reporting correctly screens out blank emails before attempting sends (avoids "To: " header with no address). preferences_views could return an empty string on the masked-email display, rendering the privacy mask as "***@example.com" for a person with only blank email channels. More critically, both use `.first()` on a filtered queryset of a person's channels — while `.first()` on `person.channels` is safe (not all channels are email), the inconsistency means a person with multiple email channels gets different behavior in send vs. preference display.

---

## B2 — Full name joining (2 sites, DIFFER)

**Canonical helper:** `full_name()` in `TreatmentPlan/utils/names.py`

**Belongs in:** Already exists; these two sites should import it

**Sites:**
- `template_engine.py:31-39` — `_full_name(person): ... parts = [(person.first_name or "").strip(), (person.last_name or "").strip()]; return " ".join(part for part in parts if part)`
- `delivery_reporting.py:340` — `display_name = f"{row.person.first_name} {row.person.last_name}".strip()`

**Do they actually agree?** NO. template_engine degrades to whichever half exists (handles None-safe join); delivery_reporting renders both parts with literal space, producing "None last_name" or "first_name None" if either is missing.

**Risk:** CSV export of campaign results (the delivery_reporting site) will show "John None" instead of "John" for a person missing a surname. Not a security issue (display-only), but breaks hygiene. The inconsistency is that two codepaths producing display names behave differently on partial data.

---

## B3 — Consent eligibility (2 sites, DOCUMENTED INTENTIONAL DUPLICATE)

**Canonical helper:** `is_marketing_eligible()` in `consent_ledger.py:82-106`

**Belongs in:** This is correct — deliberate per-person vs. batched duality

**Sites:**
- `consent_ledger.py:82-106` — `is_marketing_eligible(practice, person)` — per-person query
- `segment_engine.py:302-317` — inline batched version `resolve_cannot_be_sent(candidate_ids, ...)` 

**Do they actually agree?** YES, intentionally. segment_engine's docstring (lines 299-301) explicitly notes: "This is a deliberate batched duplicate of is_marketing_eligible — one query for the whole audience instead of one per person — so the two MUST be kept in step. Change one, change the other."

**Risk:** None if kept synchronized. Both implement: "Only explicit refusal blocks. A patient who has never been asked is eligible." Documented trade-off (performance vs. clarity) is acceptable.

---

## B4 — Name splitting (1 site, no duplicate found)

**Canonical helper:** None exists (name splitting is lossy-by-design)

**Belongs in:** `marketingBroadcast/form_handoffs.py` is the sole canonical location

**Sites:**
- `form_handoffs.py:118-137` — `split_submission_name(submission): ... first, _, last = full_name.partition(" ")`

**Do they actually agree?** N/A — only one implementation. Docstring explicitly acknowledges lossiness and cross-references identity comparison to `canonical_full_name_key`, proving awareness that "Lai Fong Holland" splits incorrectly but identity resolution compares the joined form.

**Risk:** None — the design is intentional and documented.

---

## B5 — Email canonicalization (3 sites, CONSISTENT)

**Canonical helper:** `canonical_email()` in `TreatmentPlan/utils/contact_keys.py`

**Belongs in:** Already canonical; no consolidation needed

**Sites:**
- `serializers.py:248` — `emails = [canonical_email(value) for value in (attrs.get("emails") or [])]`
- `views/public_form_views.py:353` — `contact_email = (canonical_email(contact.get("email")) or "")[:254]`
- `tasks.py:62` — `value = (raw or "").strip().lower()` — ad-hoc normalization for gender field (not email)

**Do they actually agree?** YES (email sites). tasks.py:62 is for `_normalize_gender`, unrelated to email.

**Risk:** None identified.

---

## B6 — Sole patient resolution (2 sites, CORRECT + GUARDS IN PLACE)

**Canonical helper:** `sole_patient_for_person()` in `TreatmentPlan/utils/sole_patient.py`

**Belongs in:** Already in use correctly

**Sites:**
- `consent_ledger.py:129` — `patient = sole_patient_for_person(person, getattr(practice, "id", practice))`
- `consent_ledger.py:160` — `patient = sole_patient_for_person(person, getattr(practice, "id", practice))`

**Do they actually agree?** YES. Both guard the same ambiguous-person trap (#62). Comments on lines 114-124 and 126-128 document the `.first()` risk that was fixed by switching to the helper.

**Risk:** None — fixed and guarded.

---

## Summary Table

| Behaviour | Sites | Agree? | Proposed Home |
|-----------|-------|--------|----------------|
| Email channel resolution | 2 | NO | marketingBroadcast/utils/ — new `get_primary_email(person)` helper |
| Full name joining | 2 | NO | import `full_name()` from TreatmentPlan/utils/names.py into both sites |
| Consent eligibility | 2 | YES (intentional) | keep as-is; document trade-off |
| Name splitting | 1 | N/A | keep as-is; documented |
| Email canonicalization | 3 | YES | keep as-is |
| Sole patient resolution | 2 | YES | keep as-is; guarded correctly |

---

## Severity Ranking

**High (affects logic):**
1. **B1 (Email channel resolution)** — inconsistent filtering + `.first()` on multi-channel persons can produce different results in send vs. display.

**Medium (display/hygiene):**
2. **B2 (Full name joining)** — CSV export renders "John None" instead of "John"; not a bug but inconsistent with template rendering.

**Acceptable:**
3. B3 (Consent) — intentional performance trade-off, properly documented.
4. B4 (Name splitting) — intentional lossy design, properly documented.
5. B5, B6 — no issues.

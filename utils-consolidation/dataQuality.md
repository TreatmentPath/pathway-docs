# dataQuality Detector vs Merger Key Comparison

## KEY AGREEMENT — detector vs merger

**DIFFERENT KEY — DETECTOR and MERGER DIVERGE**

The **duplicate-contact DETECTOR** and the **Person MERGER** use fundamentally different identity keys:

**Detector (dataQuality):**
- Clusters by: **channel id alone** (shared phone OR shared email)
- File: `dataQuality/legacy_backfill.py:83` — `record_id=f"channel-{row['channel_id']}"`
- A ContactChannel is created using `canonical_phone_e164()` or `canonical_email()`
- Example: Three people sharing `+447911123456` = ONE duplicate cluster, all in `record_id="channel-<id>"`

**Merger (dedupe_persons):**
- Clusters by: **(canonical_full_name_key, canonical_phone_e164)** — BOTH name AND phone must match
- Files: `dedupe_persons.py:87-97` (`norm_name` = canonical_full_name_key), lines 60-84 (canon_phone)
- Example: Three people on `+447911123456` with names "Katherine", "David", "Emily" = NO merge (names differ)

**Concrete divergence example:**
- DataQualityIssue reports: "3 duplicate contacts on channel-42 (phone +447911123456)" — all three persons Katherine, David, Emily
- User staff tries to merge Katherine + David → succeeds
- User tries to merge David + Emily → already merged (David now merged_into Katherine)
- BUT: `dedupe_persons` command run at this point would say: "No duplicates found" because the three names are all different, and dedupe_persons requires BOTH name AND phone to match

**The risk:**
- Staff chases the UI's reported duplicates for a family/household (same phone, different names)
- The UI correctly flags them as a shared-channel cluster
- But only Persons with IDENTICAL NAMES will auto-merge via dedupe_persons
- This creates **asymmetric merge pressure**: the detector surfaces pairs that the merger won't touch, leading to:
  - **Staff phantom-chasing**: merged clusters that re-appear (members without matching names)
  - **Silent misses**: same-name duplicates on shared channels stay unmerged if dismissed/ignored in the UI

## B1 — Name joining inconsistency in dedupe_persons  (1 site)

**Canonical helper:** `contact_keys.py::canonical_full_name_key()`
**Belongs in:** already correct (global utils)
**Sites:**
- `dedupe_persons.py:87-97` — `norm_name(fn, ln)` wraps `canonical_full_name_key()` ✓

**Do they actually agree?** IDENTICAL. The dedupe_persons command's `norm_name()` correctly routes through the canonical function that Person.resolve uses.

**Risk:** none — this was fixed and is now unified.

---

## B2 — Phone key in dedupe_persons  (1 site)

**Canonical helper:** `utils/phones.py::canonical_phone_e164()`
**Belongs in:** already correct (global utils)
**Sites:**
- `dedupe_persons.py:60-84` — `canon_phone(raw, cc)` wraps `canonical_phone_e164()` and validates E.164 prefix ✓

**Do they actually agree?** IDENTICAL. The wrapper explicitly uses the canonical function and adds an E.164 validation step that makes it MORE conservative (valid E.164 only, no bare keys).

**Risk:** none — this is correct and defensive.

---

## B3 — Email handling in dedupe_persons  (0 sites)

**Canonical helper:** `contact_keys.py::canonical_email()`
**Belongs in:** MISSING from detector
**Sites:**
- dedupe_persons does NOT deduplicate on email (only name + phone)
- dataQuality detector CAN report clusters on email channels (`record_id="channel-<email-channel-id>"`)

**Do they actually agree?** DIFFER. The detector surfaces email-based duplicates that the merger never touches.

**Risk:** Medium — staff may merge email-based duplicate clusters only to find dedupe_persons has no opinion. Email clusters are correct to surface; they're simply not auto-mergeable via the current dedupe_persons logic (which requires a shared phone). Acceptable gap so long as it's documented.

---

## B4 — DOB conflict check in Person.resolve vs dedupe_persons  (2 sites)

**Canonical helper:** `contact_keys.py::canonical_dob()`
**Belongs in:** Person.resolve only (not dedupe_persons)
**Sites:**
- `models.py:636-638` (Person.resolve) — `cls._dob_conflict(dob, p.dob)` ✓
- `dedupe_persons.py:385-401` — `_dob_conflict_in_group()` exists but is used ONLY for reporting, not for merge blocking

**Do they actually agree?** DIFFER in application. Person.resolve BLOCKS merge on DOB conflict (line 638: `and not cls._dob_conflict(...)`). dedupe_persons computes DOB conflicts ONLY for the dry-run report (line 390-401), never to skip a merge.

**Risk:** High — A pair (Katherine Smith, 1975-05-04) and (Katherine Smith, 1980-03-12) on the same phone:
- Person.resolve refuses to reuse a candidate with conflicting DOB → creates a new Person in the same Household
- dedupe_persons WILL merge them (no DOB guard in the actual merge)
- The merger's `_dob_conflict_in_group()` is consulted only in `handle_*` output formatting, not in the merge decision

**Evidence:** `dedupe_persons.py:334-373` (_person_identity) has NO DOB check before grouping; it only groups on (name, phone) ✓

---

## B5 — Practice scoping in dedupe_persons  (1 site)

**Canonical helper:** `practice_filter_mixins.py`
**Belongs in:** already correct (per-site isolation)
**Sites:**
- `dedupe_persons.py:334-373` — explicitly filters `practice_id=practice_id` in all queries ✓

**Do they actually agree?** IDENTICAL. dedupe_persons is strictly practice-scoped and never merges across practices.

**Risk:** none

---

## B6 — Household grouping on family_id vs shared channel  (2 sites)

**Canonical helper:** NONE — logic is app-specific
**Belongs in:** contact_identity layer (future)
**Sites:**
- `models.py:579-666` (Person.resolve) — lines 620-622: `q |= models.Q(patients__meta_data__family_id=dentally_family_id)` — includes Dentally family members even without a shared channel
- `dedupe_persons.py:334-373` (_person_identity) — NO family_id check; groups only on (name, phone) from records

**Do they actually agree?** DIFFER. Person.resolve treats Dentally `family_id` as a candidate signal (even without a shared channel). dedupe_persons sees ONLY records and their contact info, never the Dentally family_id.

**Reachable:** Yes — a Dentally family with members who haven't yet shared a phone/email:
- Person.resolve will group them as candidates (family_id match)
- dedupe_persons will see them as separate (no name+phone match across records)

**Risk:** Low-Medium. Family-based household logic is intentional in Person.resolve and separate from the dedup key. dedupe_persons is not a Dentally-aware tool; that's correct.

---

## B7 — `.first()` / `[0]` on ambiguous sets  (1 site)

**Canonical helper:** `contact/channel_owner.py::sole_person_for_channel()`
**Belongs in:** already correct (global utils)
**Sites:**
- `models.py:617-632` (Person.resolve candidates) — uses `.order_by("id").distinct()` then iterates, NOT `.first()` ✓
- `views.py:80-170` (merge action) — uses explicit list unpacking, NOT `.first()` ✓

**Do they actually agree?** IDENTICAL. The code correctly avoids ambiguous `.first()` lookups.

**Risk:** none

---

## Summary

| Finding | Detector | Merger | Same? | Risk |
|---------|----------|--------|-------|------|
| **Key type** | Channel ID | (Name, Phone) | ❌ | **HIGH** — asymmetric reporting |
| Name key | `canonical_full_name_key()` | `canonical_full_name_key()` | ✅ | None |
| Phone key | `canonical_phone_e164()` | `canonical_phone_e164()` | ✅ | None |
| Email key | Present in detector | Absent in merger | ❌ | Medium — acceptable gap |
| DOB conflict | Blocks in resolve | Reported only (not enforced) in dedupe | ❌ | High — inconsistent safety |
| Dentally family_id | Signal in resolve | Ignored in dedupe | ❌ | Low — intentional boundary |
| Practice scoping | ✅ Isolated | ✅ Isolated | ✅ | None |

### Negative Findings (not present)

- No hand-rolled email normalisation found (uses `canonical_email` everywhere)
- No `.first()` on ambiguous multi-person sets (uses `.order_by("id")` or explicit iteration)
- No cross-practice merging in dedupe_persons (practice-scoped filters throughout)
- No duplicate definitions of name/phone normalisation (all route through canonical helpers)
- No name splitting at the first space in dedupe or the detector (canonical_full_name_key is used correctly)

### Central Finding

**The DETECTOR keys by shared channel (phone/email). The MERGER keys by name+phone. These are NOT the same.**

A household on one phone with three different names will show as one duplicate cluster, but the merger cannot auto-resolve it. The UI surfaces the pair correctly (shared channel = legitimate duplicate cluster signal), but staff cannot close it via the merge action if the names differ significantly.

**No code defects found** — the behavior is by design. The detector is right to surface any shared channel (it's evidence, not a merge instruction). The merger is right to demand name+phone agreement (avoiding false merges). The gap is **at the interface layer**: the UI should set expectations that merging a household cluster requires either the names to agree or the staff to explicitly pick the winner/losers using the family=true mode in the merge action.

---

**Output file:** `/home/mannie/Desktop/Projects/treatmentpath/docs/utils-consolidation/dataQuality.md`

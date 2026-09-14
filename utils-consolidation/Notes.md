# Notes/ Duplicate Logic Inventory

## B1 — Practice scoping of clinical records: author-based vs patient-based (3 sites + 1 service)

**Canonical helper:** `scope_clinical_to_practice(queryset, practice, patient_field="patient")` from `utils/practice_mixins.py`

**Belongs in:** Global `utils/` (already canonical), but call pattern varies dangerously

**Sites:**

1. `views/note.py:446-470` — `notes_by_patient()` action
   - **The fix (CORRECT):** Filters by `Q(patient__practice=practice) | Q(patient__isnull=True)` (line 469)
   - Long comment (lines 448-467) documents issue #49: "author who has since moved practices still carries their old notes, so staff in practice 16 could read a clinical note belonging to a practice-13 patient (proven on production: Note 662, patient 20965 Adam Monaghan)."
   - This was **already fixed** and the comment proves it gated on `patient__practice`, not `user` or `Note.practice`.

2. `views/note.py:557-568` — `search_by_patient()` action (DEFECT)
   - **The miss (WRONG):** Lines 557-568 search individual notes without the patient-practice gate
   - Code: `Note.objects.filter(user=user, scope="individual", is_draft=False).filter(Q(patient_name__icontains=query) | Q(patient__first_name__icontains=query) | Q(patient__last_name__icontains=query) | Q(patient__email__icontains=query))`
   - **No filter on `patient__practice`** — an individual note linking to a patient from any practice will be matched
   - This is a SIBLING MISS: notes_by_patient (lines 446-470) was fixed for this exact issue and documented it; search_by_patient (lines 557-568) is the same hole
   - **Risk:** a user who moved practices can still search for (and access) notes about their old practice's patients via individual note search path

3. `views/letters.py:252-270` — `get_queryset()`
   - **CORRECT:** Lines 262-268 filter base_queryset with `Q(patient__isnull=True) | Q(patient__practice=practice)` after first filtering by author membership
   - Comment (lines 254-261) explicitly documents the gate: "A letter is visible whenever its AUTHOR belongs to this practice, even if the PATIENT it's about belongs to a different one (an author with multi-practice membership could otherwise leak a patient's letter to unrelated staff)."
   - **Verified correct:** patient__practice is the gate, not author membership alone

4. `services/labels.py:384` — `get_label_stats()` function
   - **The gate (AUTHOR-BASED, NOT PATIENT-BASED):** Line 384 uses `base_queryset = NoteLabel.objects.filter(note__user__practices=practice)`
   - This counts labels by the NOTE AUTHOR's practice membership, not the PATIENT's practice
   - **Problem:** if a clinician author moves practices, their old practice's notes (and their labels) are still counted under their new practice in stats
   - **Likely intent:** should gate on `note__patient__practice` when a patient exists, or on `note__practice` for patient-less notes
   - **Difference from the rest:** all other places use patient__practice; this one uniquely uses author

**Do they actually agree?**
- **DIFFERS fundamentally:** notes_by_patient and letters.get_queryset gate on `patient__practice`; search_by_patient does NOT gate on patient practice at all; labels.py gates on author membership instead
- Fixed comment in notes_by_patient (issue #49, proven on production) proves the patient__practice gate is correct and was a live bug
- search_by_patient (lines 557-568) and labels.py (line 384) have not been fixed the same way

**Risk:** 
- `search_by_patient()` leaks cross-practice patient notes for individual (non-practice-scoped) searches
- `get_label_stats()` inflates label statistics for practices when authors move
- Notes scoped "individual" are bypassing the patient-practice check that practice-scoped notes enforce

---

## B2 — "Whose letters are these?" resolution: scope_clinical_to_practice call pattern (3 sites)

**Canonical helper:** `scope_clinical_to_practice(queryset, practice, patient_field="patient")` from `utils/practice_mixins.py`

**Belongs in:** Global `utils/` (already canonical)

**Sites:**

1. `views/letters.py:597-600` — `update()` method
   ```python
   letter = scope_clinical_to_practice(
       NotesLetter.objects.filter(user__practices=practice, id=letter_id),
       practice,
   ).first()
   ```
   - Correctly calls the helper; comment `# #48` refers to the author-membership gap
   - `.first()` is unordered (no `order_by()`) but id lookups are unique by nature, so this is safe

2. `views/letters.py:1319-1324` — `retrieve()` method
   ```python
   letter = scope_clinical_to_practice(  # #48
       NotesLetter.objects.prefetch_related("images", "logos").filter(
           Q(Q(user=user) | Q(user__practices=practice)), id=letter_id
       ),
       practice,
   ).first()
   ```
   - Correctly calls the helper
   - `.first()` is unordered but id lookups are unique

3. `views/letters.py:1361-1366` — `destroy()` method
   ```python
   letter = scope_clinical_to_practice(  # #48
       NotesLetter.objects.filter(
           Q(Q(user=user) | Q(user__practices=practice)), id=letter_id
       ),
       practice,
   ).first()
   ```
   - Correctly calls the helper
   - `.first()` is unordered but id lookups are unique

**Do they actually agree?** 
- IDENTICAL pattern in all three; all call `scope_clinical_to_practice()` correctly
- All have comment `#48` referring to the documented gap

**Risk:** None — all three sites agree and use the canonical helper. The pattern is consistent.

---

## B3 — Permission checks: author membership gates (INCONSISTENCY) (3 sites)

**Canonical helper:** None — mixed patterns across the codebase

**Belongs in:** Should consolidate to a helper or single pattern

**Sites:**

1. `views/letters.py:677-681` (in `add_images()`)
   ```python
   if letter.user != request.user:
       practice = self.get_user_practice_or_none()
       if (
           not practice
           or not letter.user.practices.filter(id=practice.id).exists()
       ):
   ```
   - Gates permission on `letter.user.practices.filter(id=practice.id)` — the AUTHOR's practice membership

2. `views/letters.py:779-788` (in `add_images_from_urls()`)
   ```python
   if letter.user != request.user:
       practice = self.get_user_practice_or_none()
       if (
           not practice
           or not letter.user.practices.filter(id=practice.id).exists()
       ):
   ```
   - Identical pattern as add_images

3. `views/letters.py:886-895` (in `update_image_properties()`)
   ```python
   if letter.user != request.user:
       practice = self.get_user_practice_or_none()
       if (
           not practice
           or not letter.user.practices.filter(id=practice.id).exists()
       ):
   ```
   - Identical pattern as add_images and add_images_from_urls

**Do they actually agree?** 
- IDENTICAL across all three (copy-pasted)
- But these check author membership `letter.user.practices.filter()`, NOT the patient's practice like the main queryset does
- **DIVERGENCE:** main `get_queryset()` filters by `patient__practice`, but these image actions check `letter.user.practices`
- Per comment #48 in the code (lines 254-261 of get_queryset), author membership is insufficient

**Risk:**
- An author with multi-practice membership can modify images on a letter about another practice's patient
- The letter is already (correctly) gated by `patient__practice` on read, but modification is gated by author membership
- Violates the pattern established by the main queryset

---

## B4 — Unordered `.first()` in note/letter single-record lookups (2 sites)

**Canonical helper:** None — `.order_by()` missing in some lookups

**Belongs in:** Should add `order_by()` or use `.get()` where id is the lookup key

**Sites:**

1. `views/note.py:718-725` (in `update()`)
   ```python
   note = Note.objects.filter(
       Q(
           Q(user=user, scope="individual")
           | Q(practice=practice, scope="practice")
       ),
       id=note_id,
   ).first()
   ```
   - No `order_by()`, so if the filter somehow matches 2+ rows, which one is returned?
   - **Safety:** id is unique, so filter can return at most 1 row; `.first()` is safe but poor style

2. `views/letters.py:1421` (in `LetterTemplateViewSet.create()`)
   ```python
   existing_template = LetterTemplate.objects.filter(
       user=request.user, name=name.strip()
   ).first()
   ```
   - No `order_by()` on a non-unique filter (user + name)
   - **But:** unique_together constraint on (user, name) in model Meta (line 53), so at most 1 row
   - Safe by database constraint, but not explicit in the code

**Do they actually agree?**
- IDENTICAL pattern: both use unordered `.first()` on lookups that happen to be unique
- Both are safe by uniqueness (one by id, one by constraint), but neither is explicit about the guarantee
- **Better style:** use `.get(id=note_id)` to raise on multiple rows, or add explicit `order_by()` to document intent

**Risk:** Low — both are safe by uniqueness. Correctness risk is minimal; clarity risk is moderate.

---

## B5 — Consent status/alert practice scoping: CORRECT (2 sites)

**Canonical helper:** `patient__practice` direct filter

**Belongs in:** Global pattern (already correct)

**Sites:**

1. `views/consent.py:139` — `ConsentStatusViewSet.get_queryset()`
   ```python
   ConsentStatus.objects.filter(patient__practice=practice)
   ```
   - Correct pattern

2. `views/consent.py:164-165` — `ConsentAlertViewSet.get_queryset()`
   ```python
   ConsentAlert.objects.filter(
       patient__practice=practice, is_active=True, is_dismissed=False
   )
   ```
   - Correct pattern

**Do they actually agree?**
- IDENTICAL and CORRECT: both gate on patient__practice

**Risk:** None — these are correctly implemented.

---

## Summary

| Behaviour | Sites | Agree? | Proposed Home | Severity |
|-----------|-------|--------|---------------|----------|
| Practice scoping: patient vs author | 4 (notes 2, letters 1, labels 1) | NO — author gates in labels.py; mixed in notes | Audit patient__practice in all locations; create `clinical_visible_to_practice()` helper | CRITICAL — proven production leak in notes_by_patient (issue #49) |
| Whose letters are these? (scope_clinical_to_practice) | 3 (letters) | YES — all identical, all correct | Already canonical in utils/ | NONE |
| Permission checks (author membership) | 3 (letters image ops) | YES — identical but DIVERGENT from queryset | Align image op permission checks to patient__practice gate | MEDIUM — can modify another practice's patient's letter |
| Unordered .first() | 2 (note update, letter template create) | YES — both safe by uniqueness | Add explicit `order_by()` or use `.get()` for clarity | LOW — correctness safe, clarity poor |
| Consent scoping | 2 (consent) | YES — both correct | Keep as-is in consent views | NONE |

### Key Findings

1. **SIBLING MISS:** Issue #49 was documented and fixed in `notes_by_patient()` (line 469), proving that author-scoped queries leak cross-practice patients. The identical `search_by_patient()` method (lines 557-568) was NOT fixed and remains vulnerable.

2. **CROSS-FILE DIVERGENCE:** `services/labels.py:384` gates on author practice membership, while all note/letter views gate on patient practice. Labels should align.

3. **PERMISSION CHECK DIVERGENCE:** Letter image operations (add_images, add_images_from_urls, update_image_properties) check author membership, while the main get_queryset checks patient practice. Per comment #48, author membership is insufficient.

4. **Unordered `.first()` calls are safe by uniqueness but poor style** — neither will break, but explicit `order_by()` or `.get()` would clarify intent.

5. **Test evidence:** `test_note_patient_practice_scoping.py` demonstrates that notes/letters should validate patient practice, proving the pattern is correct.

### Recommended Actions

1. **CRITICAL:** Add patient-practice gate to `search_by_patient()` lines 557-568, identical to the fix in `notes_by_patient()` lines 469-470.
2. **HIGH:** Audit and align `services/labels.py:384` to gate on patient practice, not author practice.
3. **MEDIUM:** Align image operation permission checks to use patient__practice, not author membership.
4. **LOW:** Add explicit `order_by()` or use `.get()` in the two unordered `.first()` locations for clarity.

# Consent Template Library Cloning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Separate platform consent templates from practice templates and let authorized practice users clone a platform template into their own environment.

**Architecture:** Follow the existing Email/SMS library pattern. Store clone provenance with a nullable self-FK, keep the practice list local-only, expose active global templates through the existing global endpoint, and add an atomic `add-to-my-environment` action that copies template metadata and PDF field placements. The React Consent view fetches the two collections independently and renders them with the shared template-list primitives.

**Tech Stack:** Django 5.1, Django REST Framework, PostgreSQL, React 18, TypeScript, Vitest, Testing Library.

**Commit policy:** Do not commit. The user will review and commit all changes.

---

### Task 1: Add clone provenance and database uniqueness

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/Documents/models.py`
- Create: `TreatmentPathBackend/TreatmentPath/Documents/migrations/0014_consentdocumenttemplate_cloned_from_global.py`
- Test: `TreatmentPathBackend/TreatmentPath/Documents/test_template_foundation.py`

- [ ] **Step 1: Write a failing model test**

Add a test that creates one global template and one practice clone, asserts `clone.cloned_from_global == global_template`, and asserts a second clone for the same `(practice, cloned_from_global)` raises `IntegrityError` inside `transaction.atomic()`.

- [ ] **Step 2: Run the test and verify RED**

Run:

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend
venv/bin/python TreatmentPath/manage.py test Documents.test_template_foundation.ConsentTemplateFoundationTests.test_practice_cannot_clone_same_global_template_twice --keepdb
```

Expected: FAIL because `ConsentDocumentTemplate` has no `cloned_from_global` field.

- [ ] **Step 3: Add the field and constraint**

Add a nullable self-FK named `cloned_from_global` with `SET_NULL` and `related_name="cloned_instances"`. Add a conditional `UniqueConstraint` over `practice` and `cloned_from_global` where the source is not null. Generate migration `0014` from the current `0013_signingrequest_reuse_key` leaf and inspect it.

- [ ] **Step 4: Run the focused model test and verify GREEN**

Run the command from Step 2. Expected: PASS.

### Task 2: Separate library API and clone global templates

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/Documents/views/template_views.py`
- Modify: `TreatmentPathBackend/TreatmentPath/Documents/serializers.py`
- Modify: `TreatmentPathBackend/TreatmentPath/Documents/urls.py`
- Test: `TreatmentPathBackend/TreatmentPath/Documents/test_template_foundation.py`

- [ ] **Step 1: Write failing API tests**

Add focused tests proving the practice list excludes global and other-practice rows; the library lists active global rows; `already_added` is practice-scoped; clone returns a local row with provenance; content, PDF metadata, and fields are copied; duplicates return 400; non-managers cannot clone; and non-superusers cannot mutate global sources.

- [ ] **Step 2: Run the new tests and verify RED**

Run:

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend
venv/bin/python TreatmentPath/manage.py test Documents.test_template_foundation.ConsentTemplateFoundationTests --keepdb
```

Expected: failures showing globals still appear in the practice list, library GET is forbidden, and the clone route is missing.

- [ ] **Step 3: Implement local-only listing and library metadata**

Filter the practice queryset by `practice=current_practice, is_global=False`. Annotate the global list with an `Exists` query for a clone in the current practice and expose it as `already_added`. Add `cloned_from_global` to list/detail serialization.

- [ ] **Step 4: Implement action-aware permissions and cloning**

Allow authenticated safe list/retrieve access to the global view, require `consent_templates_manage` for `clone_template`, and retain superuser-only global mutations. Implement an atomic clone that copies title, category, source type, body, PDF reference/page count, subject, expiry, active state, and all `ConsentTemplateField` rows. Map:

```text
POST global-consent-templates/<pk>/add-to-my-environment/
```

- [ ] **Step 5: Run backend tests and verify GREEN**

Run the command from Step 2, then broader Documents template/signing tests with `--keepdb`. Expected: all selected tests PASS.

### Task 3: Require practice-owned templates in send/snapshot paths

**Files:**
- Modify: the template-resolution helper in `TreatmentPathBackend/TreatmentPath/Documents/views/signing_views.py`
- Test: `TreatmentPathBackend/TreatmentPath/Documents/test_template_foundation.py`

- [ ] **Step 1: Write the failing ownership test**

Assert the signing-request builder rejects a raw global template ID for a practice, but accepts the cloned practice template and snapshots its content/fields.

- [ ] **Step 2: Run the test and verify RED**

Expected: the raw global template currently succeeds.

- [ ] **Step 3: Restrict template resolution**

Remove the `Q(practice=None, is_global=True)` branch from the signing template lookup. Keep practice scope, active-state validation, and bundle handling unchanged.

- [ ] **Step 4: Run the test and verify GREEN**

Expected: global source rejected; local clone accepted.

### Task 4: Render Consent My Templates and Template Library

**Files:**
- Modify: `perfect-pixel-playground-project/src/config/api.ts`
- Modify: `perfect-pixel-playground-project/src/types/signableDocuments.ts`
- Modify: `perfect-pixel-playground-project/src/pages/settings/components/consent/ConsentTemplatesView.tsx`
- Modify: `perfect-pixel-playground-project/src/pages/settings/components/consent/ConsentTemplatesView.test.tsx`

- [ ] **Step 1: Write failing UI tests**

Mock practice and global lists independently. Assert the two headings, description, correct row separation, preview action, and Add action. Add an interaction test proving Add posts to the clone endpoint, inserts the returned clone into My Templates, and changes the library action to Added. Add a read-only test proving Add is absent.

- [ ] **Step 2: Run the tests and verify RED**

Run:

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vitest run src/pages/settings/components/consent/ConsentTemplatesView.test.tsx
```

Expected: FAIL because Consent has one combined list and no clone action.

- [ ] **Step 3: Add canonical types and endpoints**

Extend the consent API/UI types and transformer with `cloned_from_global`/`clonedFromGlobal` and `already_added`/`alreadyAdded`. Add `API_ENDPOINTS.globalConsentTemplates.addToMyEnvironment(id)`.

- [ ] **Step 4: Implement the two-section UI**

Keep global admin mode unchanged. In practice mode, fetch local and library collections separately, reuse the shared list primitives and existing Consent preview dialog, and follow Email/SMS Add/Added behavior. On success append the returned clone locally and mark the source Added; on failure show the backend message and leave Add retryable.

- [ ] **Step 5: Run focused frontend tests and verify GREEN**

Run the command from Step 2. Expected: all Consent template tests PASS.

### Task 5: Verification and browser walkthrough

- [ ] **Step 1: Run formatting and diff checks**

Run Black on touched Django files, the configured frontend formatter/linter on touched TypeScript files without rewriting unrelated files, and `git diff --check` in both repositories.

- [ ] **Step 2: Run regression suites**

Run focused Django Documents tests with `--keepdb`, focused Consent frontend tests, and inspect full type-check output for errors in touched files. Record unrelated baseline failures separately.

- [ ] **Step 3: Verify serialized values**

Confirm one text and one PDF global template return `already_added=false`, become true only for the cloning practice, and produce local clones with copied field counts and source IDs.

- [ ] **Step 4: Verify in the browser**

On `http://127.0.0.1:8080/settings/templates/consent`, preview a platform template, add it, and verify it appears in My Templates while the source reads Added. Do not send a consent.

- [ ] **Step 5: Final review**

Confirm no hardcoded secrets, no scratch files, no unrelated edits, and no commits. Report exact changed areas and verification evidence.


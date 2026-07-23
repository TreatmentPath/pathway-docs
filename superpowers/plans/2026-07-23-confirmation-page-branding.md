# Confirmation Page Practice Branding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply the current practice's saved brand colours to the public appointment confirmation page only.

**Architecture:** Extend the existing public confirmation API response with the practice's saved palette fields. The confirmation page will normalize valid six-digit hex values, derive missing secondary/accent/background values from the primary colour, and apply the result through inline CSS variables scoped to that page. All authenticated application pages remain unchanged.

**Tech Stack:** Django REST Framework, Django migrations, React, TypeScript, Vitest, Testing Library.

---

### Task 1: Add API coverage for confirmation branding

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests_confirmation_branding.py`.

- [ ] **Step 1: Write the failing API test**

Create `ConfirmationBrandingResponseTests` with a public confirmation test that creates a practice with custom branding, creates a live appointment with confirmation token `brand-test-token`, performs `GET /api/backend/dentally/confirm/?token=brand-test-token`, and asserts the response contains the four brand colour fields plus the practice logo URL/name.

- [ ] **Step 2: Run the focused test and verify it fails**

Run:

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test dentallyIntegration.tests_confirmation_branding --keepdb --verbosity 1
```

Expected: the response does not yet contain the branding keys.

### Task 2: Expose branding through the public confirmation API

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/views/appointment_confirm_views.py:152-178`.
- Modify: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/serializers.py` only if the endpoint is converted to a serializer; otherwise leave unchanged.
- Test: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests_confirmation_branding.py`.

- [ ] **Step 1: Add the branding fields to the response**

Return `brand_primary_colour`, `brand_secondary_colour`, `brand_accent_colour`, and `brand_background_colour` from `AppointmentConfirmViewSet.list()` using the related practice values. Preserve the existing response keys and public token validation.

- [ ] **Step 2: Run the focused API test**

Run:

```bash
python manage.py test dentallyIntegration.tests_confirmation_branding --keepdb --verbosity 1
```

Expected: PASS.

- [ ] **Step 3: Run existing confirmation regression tests**

```bash
python manage.py test dentallyIntegration.tests --keepdb --verbosity 0
```

Expected: existing confirmation behavior remains green.

### Task 3: Add frontend palette normalization and component tests

**Files:**
- Create: `perfect-pixel-playground-project/src/pages/confirmation/confirmationBranding.ts`.
- Create: `perfect-pixel-playground-project/src/pages/confirmation/confirmationBranding.test.ts`.

- [ ] **Step 1: Write failing utility tests**

Cover these exact behaviors:

```ts
expect(deriveConfirmationPalette('#123456', {})).toEqual({
  primary: '#123456',
  secondary: expect.any(String),
  accent: expect.any(String),
  background: expect.any(String),
});
expect(deriveConfirmationPalette('invalid', {})).toEqual(DEFAULT_CONFIRMATION_PALETTE);
expect(deriveConfirmationPalette('#123456', { accent: '#ABCDEF' }).accent).toBe('#ABCDEF');
```

- [ ] **Step 2: Run the utility tests and verify they fail**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vitest run src/pages/confirmation/confirmationBranding.test.ts
```

Expected: FAIL because the utility does not exist yet.

- [ ] **Step 3: Implement the minimal palette utility**

Accept nullable API values, require six-digit hex values, uppercase valid values, preserve valid overrides, and derive missing values by mixing the primary colour with white. Keep the existing purple palette as the default.

- [ ] **Step 4: Run the utility tests**

Expected: PASS.

### Task 4: Apply branding only to the confirmation page

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/ConfirmAppointmentPage.tsx`.
- Modify: `perfect-pixel-playground-project/src/pages/ConfirmAppointmentPage.test.tsx` or create it if no focused test exists.

- [ ] **Step 1: Extend `AppointmentDetails` with optional branding fields**

Keep the fields optional so older backend responses continue to render using the default palette.

- [ ] **Step 2: Write the failing page test**

Mock the confirmation GET response with a custom primary colour and assert the page root receives the expected CSS variables. Add a second test with missing/invalid colours and assert the default palette is used.

- [ ] **Step 3: Apply scoped CSS variables**

Compute the palette after loading the response and set variables on the confirmation page root. Replace only this page's hardcoded purple background, text, border, icon, and button colours with the scoped variables. Do not modify the authenticated app shell or global CSS.

- [ ] **Step 4: Run the page tests**

```bash
npx vitest run src/pages/confirmation/confirmationBranding.test.ts src/pages/ConfirmAppointmentPage.test.tsx
```

Expected: PASS.

### Task 5: Final verification

- [ ] **Step 1: Run focused frontend lint**

```bash
npx eslint src/pages/ConfirmAppointmentPage.tsx src/pages/confirmation/confirmationBranding.ts src/pages/confirmation/confirmationBranding.test.ts
```

- [ ] **Step 2: Run backend pre-commit on touched backend files**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend
./.git/hooks/pre-commit
```

- [ ] **Step 3: Verify migration state and diffs**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py makemigrations --check --dry-run
git diff --check
```

- [ ] **Step 4: Confirm only the confirmation page changed visually**

Open a currently live confirmation URL on `dev.confirm.dental` with a practice using custom colours and confirm the public page reflects them while the main application retains its existing palette.

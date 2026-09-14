# Consent Email Brand Colors Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate the practice’s configured primary, secondary, accent, and background colors into the existing consent email without changing its layout or copy.

**Architecture:** Keep `build_consent_email_body` as the single renderer. Reuse the existing card, logo, CTA, and text structure; only assign palette roles to inline styles so common email clients render them reliably. Extend the existing consent email branding test with independent assertions for each configured color.

**Tech Stack:** Django, Python, inline HTML email styles, Django `APITestCase`.

---

### Task 1: Apply the configured palette to the consent email

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/Documents/utils/notifications.py:180-225`

- [ ] **Step 1: Preserve the existing markup and copy.** Keep the current logo/header, white card, consent label, paragraphs, CTA, copy-link paragraph, and closing exactly in place.

- [ ] **Step 2: Map palette roles in inline styles.** Use `brand_background_colour` for the outer canvas, `brand_primary_colour` for the top rule and CTA, a darkened primary-derived tone for links, and `brand_accent_colour` for the consent label background and card border. Use the existing fallback values when fields are empty.

- [ ] **Step 3: Keep email-safe contrast.** Keep body text dark, use dark text on the light mint CTA, and use a readable dark tone for links rather than placing light text on a pale color.

### Task 2: Extend regression coverage

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/Documents/test_template_foundation.py:527-540`

- [ ] **Step 1: Configure all four palette values in the existing fixture.** Set primary, secondary, accent, and background to distinct known colors before rendering.

- [ ] **Step 2: Assert each configured color appears in the rendered HTML.** Assert the primary appears in the CTA/top rule, secondary-derived link styling is present, accent appears in the label/border, and background appears on the outer body.

- [ ] **Step 3: Run the focused test.**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test Documents.test_template_foundation.ConsentTemplateAccessTests.test_invitation_html_uses_practice_brand_colour --keepdb
```

Expected: PASS.

### Task 3: Verify the backend change

**Files:**
- No additional files.

- [ ] **Step 1: Run the focused consent template suite.**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py test Documents.test_template_foundation --keepdb
```

Expected: all tests PASS.

- [ ] **Step 2: Run Django checks and whitespace validation.**

```bash
python manage.py check
git diff --check
```

Expected: no Django issues and no diff errors.

# Broadcast Template Builder Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add both Basic (Lexical) and Comprehensive (GrapesJS/MJML) email builder support to the admin `BroadcastTemplates` version editor, reusing the existing builder components and preserving the design so it can be re-opened.

**Architecture:** Extract the editor shells from `EmailTemplateBuilder.tsx` (comprehensive) and `BasicEmailBuilder.tsx` (basic) into reusable components that accept initial data and save callbacks. Add a `design_json` JSONField to the `BroadcastTemplateVersion` Django model and expose it in the admin serializer. Integrate both shells into the admin `VersionForm` with a mode toggle, keeping the existing version-specific fields (preview text, disclaimer, default content, editable fields) in a side panel.

**Tech Stack:** React/TypeScript/Vite frontend, Django 5.2/DRF backend, Lexical, GrapesJS/MJML.

---

## File Map

| File | Responsibility |
|------|---------------|
| `TreatmentPathBackend/TreatmentPath/marketingBroadcast/models.py` | Add `design_json` JSONField to `BroadcastTemplateVersion` |
| `TreatmentPathBackend/TreatmentPath/Admin/serializers.py` | Add `design_json` to `AdminBroadcastTemplateVersionSerializer` fields/read-only |
| `TreatmentPathBackend/TreatmentPath/marketingBroadcast/migrations/0022_broadcasttemplateversion_design_json.py` | Migration for the new field |
| `perfect-pixel-playground-project/src/components/email-builder/ComprehensiveEmailBuilder.tsx` | Reusable comprehensive builder shell (extracted from `EmailTemplateBuilder.tsx`) |
| `perfect-pixel-playground-project/src/components/email-builder/BasicEmailBuilderShell.tsx` | Reusable basic builder shell (extracted from `BasicEmailBuilder.tsx`) |
| `perfect-pixel-playground-project/src/pages/EmailTemplateBuilder.tsx` | Refactor to use `ComprehensiveEmailBuilder` |
| `perfect-pixel-playground-project/src/pages/BasicEmailBuilder.tsx` | Refactor to use `BasicEmailBuilderShell` |
| `perfect-pixel-playground-project/src/pages/admin/BroadcastTemplates.tsx` | Add builder modes to `VersionForm` |

---

## Task 1: Add backend `design_json` field to BroadcastTemplateVersion

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/models.py`
- Modify: `TreatmentPathBackend/TreatmentPath/Admin/serializers.py`
- Create: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/migrations/0022_broadcasttemplateversion_design_json.py`

- [ ] **Step 1.1: Add field to model**

Add to `BroadcastTemplateVersion` (after `plain_text_fallback`):

```python
design_json = models.JSONField(default=dict, blank=True)
```

- [ ] **Step 1.2: Add field to admin serializer**

In `AdminBroadcastTemplateVersionSerializer`, add `"design_json"` to `fields` and remove it from `read_only_fields` so superusers can author/save it.

- [ ] **Step 1.3: Create migration**

Run:

```bash
cd TreatmentPathBackend/TreatmentPath
source venv/bin/activate
python manage.py makemigrations marketingBroadcast
```

Expected: creates `0022_broadcasttemplateversion_design_json.py`.

---

## Task 2: Extract reusable comprehensive builder shell

**Files:**
- Create: `perfect-pixel-playground-project/src/components/email-builder/ComprehensiveEmailBuilder.tsx`
- Modify: `perfect-pixel-playground-project/src/pages/EmailTemplateBuilder.tsx`

- [ ] **Step 2.1: Create `ComprehensiveEmailBuilder` component**

Extract the builder UI from `EmailTemplateBuilder.tsx` into a reusable component with this prop interface:

```typescript
interface ComprehensiveEmailBuilderProps {
  initialName: string;
  initialDefaultSubject: string;
  initialPlainTextFallback: string;
  initialDesignJson: unknown;
  titleSlot?: React.ReactNode;
  modeLabel?: string;
  backLabel?: string;
  onBack: () => void;
  onSave: (payload: {
    name: string;
    default_subject: string;
    plain_text_fallback: string;
    layout_html: string;
    personalisation_fields_allowed: string[];
    design_json: GrapesMjmlDocument;
  }) => Promise<void> | void;
}
```

Keep all existing logic: engine ref, export, block library, selected block panel, preview, device toggle, undo/redo, autosave, merge tag validation, dirty unload warning.

- [ ] **Step 2.2: Refactor `EmailTemplateBuilder.tsx`**

Replace the inline builder with `<ComprehensiveEmailBuilder ... />`. It should load the practice template via `fetchWithAuth`, map the template fields to the new props, and call the existing create/update endpoints in `onSave`.

---

## Task 3: Extract reusable basic builder shell

**Files:**
- Create: `perfect-pixel-playground-project/src/components/email-builder/BasicEmailBuilderShell.tsx`
- Modify: `perfect-pixel-playground-project/src/pages/BasicEmailBuilder.tsx`

- [ ] **Step 3.1: Create `BasicEmailBuilderShell` component**

Extract the builder UI from `BasicEmailBuilder.tsx` into a reusable component:

```typescript
interface BasicEmailBuilderShellProps {
  initialName: string;
  initialDefaultSubject: string;
  initialPlainTextFallback: string;
  initialDocument: LexicalEmailDocument;
  titleSlot?: React.ReactNode;
  modeLabel?: string;
  backLabel?: string;
  onBack: () => void;
  onSave: (payload: {
    name: string;
    default_subject: string;
    plain_text_fallback: string;
    layout_html: string;
    personalisation_fields_allowed: string[];
    design_json: LexicalEmailDocument;
  }) => Promise<void> | void;
}
```

- [ ] **Step 3.2: Refactor `BasicEmailBuilder.tsx`**

Replace inline builder with `<BasicEmailBuilderShell ... />`.

---

## Task 4: Integrate builders into `BroadcastTemplates.tsx` version form

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/admin/BroadcastTemplates.tsx`

- [ ] **Step 4.1: Update `BroadcastTemplateVersion` type**

Add `design_json: unknown` to the interface.

- [ ] **Step 4.2: Add builder mode state to `VersionForm`**

Add state for `builderMode: "basic" | "comprehensive"`, defaulting based on the loaded version's `design_json` engine (lexical = basic, grapes/mjml = comprehensive, empty = "comprehensive" as default).

- [ ] **Step 4.3: Replace layout HTML textarea with builder**

When in comprehensive mode, render `<ComprehensiveEmailBuilder>`; in basic mode, render `<BasicEmailBuilderShell>`. Pass version-specific metadata as initial values. Keep the side fields (default preview text, requires clinical disclaimer, default content JSON, editable fields JSON) in a collapsible/scrollable panel.

- [ ] **Step 4.4: Handle save payload**

The builder's `onSave` returns the common fields. Merge those with the side fields and call `onSave` with the full `BroadcastTemplateVersion` payload including `design_json`.

- [ ] **Step 4.5: Toggle UI**

Add a segmented control or select to switch builder mode. Warn if switching would discard unsaved content.

---

## Task 5: Verification

**Files:** All modified files.

- [ ] **Step 5.1: Run Django migration check**

```bash
cd TreatmentPathBackend/TreatmentPath
python manage.py check
python manage.py migrate --check
```

- [ ] **Step 5.2: Run frontend typecheck**

```bash
cd perfect-pixel-playground-project
npm run typecheck
```

- [ ] **Step 5.3: Run frontend lint**

```bash
cd perfect-pixel-playground-project
npm run lint
```

- [ ] **Step 5.4: Run frontend build**

```bash
cd perfect-pixel-playground-project
npm run build
```

---

## Spec Coverage

| Requirement | Task |
|-------------|------|
| Reuse comprehensive builder component | Task 2 + Task 4.3 |
| Reuse basic builder component | Task 3 + Task 4.3 |
| Persist comprehensive design | Task 1 (adds `design_json`) |
| Persist basic design | Task 1 + Task 4.4 |
| Keep admin version metadata fields | Task 4.3 side panel |

## Placeholder Scan

No placeholders. All steps include exact file paths, prop interfaces, and commands.

## Type Consistency

- `GrapesMjmlDocument` used consistently for comprehensive payloads.
- `LexicalEmailDocument` used consistently for basic payloads.
- `design_json: unknown` added to `BroadcastTemplateVersion` interface, then narrowed by builder mode.

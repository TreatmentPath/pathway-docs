# HTML email import → native, no-code editing

**Date:** 2026-08-04  
**Scope:** `perfect-pixel-playground-project` email template builder (`EmailTemplateBuilder`, `GrapesEmailCanvas`, `brevoImport`, `genericHtmlImport`, `newsletter2goNative`).  
**Goal:** Imported HTML email templates must open as fully editable, draggable, GUI-stylable MJML components — not as opaque `mj-raw` blocks. Existing raw templates (e.g. Template 1) must also become editable automatically.

---

## Problem statement

Template "Test 1" (id 18) is currently stored as 10 `mj-raw` blocks containing raw HTML tables. In the builder, clicking any imported element selects a single `Raw` block and shows a "Custom HTML" textarea. Users cannot:

- click and retype a headline or price,
- swap an image through the image panel,
- drag a button or product card to a new location,
- change colours, padding, or alignment through the styling panel.

Diagnostic findings:

1. The current import path uses `convertBrevoHtml`, which rejects the template because it has no `.nl2go-body-table` wrapper.
2. The fallback `convertGenericHtml` collapses the whole email into one `mj-raw` section because the structure is nested: 600px body table → 100% wrapper table → multiple section tables → nested multi-cell content tables.
3. Manually adding the Brevo class still produces only one native section with images; text, buttons, and deeper structure are lost.

## Success criteria

For Template 1 (and structurally similar imports), after the change:

1. Every section renders as `mj-section`.
2. Every column renders as `mj-column`.
3. Text headings, paragraphs, prices, and addresses render as `mj-text`.
4. Product photos, logos, and social icons render as `mj-image`.
5. CTA links render as `mj-button`.
6. Horizontal rules render as `mj-divider`.
7. Spacing bands render as `mj-spacer`.
8. Selecting any element shows the normal styling panel (not "Custom HTML").
9. Elements can be dragged between columns/sections, deleted, and replaced from the block library.
10. No raw HTML is visible to the user during normal editing.

## Implementation approach

### 1. Importer improvements

#### 1.1 Broader body-table detection in `brevoImport.ts`

The Brevo/Newsletter2Go importer currently hard-codes `.nl2go-body-table`. Some exports use a 600px table with class `r0-o` instead. Add a fallback selector chain:

1. `document.querySelector(".nl2go-body-table")` — existing behaviour.
2. If missing, look for the outermost table whose `width` attribute is a numeric value ≥ 400 and ≤ 900 and whose cell contains multiple child tables. This is the canonical email body width.

Keep the existing `descendToSectionCell` and `collectSections` logic; it already handles the nested wrapper structure correctly once the body table is found.

#### 1.2 Recursive wrapper descent in `genericHtmlImport.ts`

The generic fallback must mirror the Brevo importer's strategy:

- `findGenericContainer`: search for the 600px body table, descending through single-cell wrapper tables if needed.
- `collectSections`: recursively unwrap single-cell wrapper tables to reach the real section tables; when a cell contains multiple child tables, each becomes a section.
- `findColumns`: look for columns in the first row of the section table **and** in the first row of an inner layout table if the section table itself is a single wrapper.
- `extractBlocks`: when encountering a multi-cell table inside a column, treat it as a nested layout and decompose its cells into sub-blocks instead of emitting the whole table as `mj-raw`. Leaf content (text, images, links, dividers, spacers) becomes the corresponding MJML component.

#### 1.3 Richer leaf recognition

Both importers should handle:

- Text in `<p>`, `<h1>`–`<h4>`, `<div>` with text, `<span>` with text, and `<td>` bare text.
- Images in `<img>`, including those wrapped in `<a>` (keep the link).
- Buttons: any `<a>` with inline-block/radius/background/padding styling, or any `<a>` inside a table with such styling.
- Dividers: 1px tables with border rules.
- Spacers: empty tables with padding or height.
- Social icon grids: a row of small images inside a multi-cell table should decompose into individual `mj-image` components inside an `mj-group` or single `mj-column` if grouping is not feasible.

### 2. Load-time decomposer for existing raw templates

After `EmailTemplateBuilder` loads a saved template and the engine is ready, scan the GrapesJS model for `mj-raw` components. For each one:

1. Extract its original HTML (`component.getInnerHTML()` or the stored `content`).
2. Run the improved importer on that HTML.
3. Replace the `mj-raw` block with the resulting MJML components.
4. Mark the document dirty so the user must save the converted template.

This is a one-time migration. Once saved, the template is stored as native components.

**Safety guard:** If the decomposer produces fewer recognisable components than a threshold (e.g., 50% of the raw block's images/text nodes are unmapped), leave the raw block in place and surface a warning: "This imported section could not be fully converted. It remains as raw HTML." Never drop content silently.

### 3. UI / side-panel considerations

No new UI is required. Native components already use the existing `SelectedBlockPanel` and `EmailBlockLibrary`. The only visual change is that raw blocks disappear from the canvas model, so the side panel shows normal styling controls instead of the "Custom HTML" textarea.

### 4. Testing strategy

1. **Unit tests** in `genericHtmlImport.test.ts` and `brevoImport.test.ts` using the real Template 1 HTML (stored as a fixture). Assert:
   - zero raw sections,
   - expected counts of `mj-section`, `mj-column`, `mj-image`, `mj-text`, `mj-button`, `mj-divider`, `mj-spacer`,
   - the emitted MJML compiles without errors and contains key text from the template.
2. **Playwright end-to-end test:** open Template 1, click on a product image, a heading, a price, and a button, and confirm the side panel does not show "Custom HTML".

## Files to change

- `perfect-pixel-playground-project/src/lib/emailBuilder/brevoImport.ts`
- `perfect-pixel-playground-project/src/lib/emailBuilder/newsletter2goNative.ts`
- `perfect-pixel-playground-project/src/lib/emailBuilder/genericHtmlImport.ts`
- `perfect-pixel-playground-project/src/pages/EmailTemplateBuilder.tsx`
- `perfect-pixel-playground-project/src/lib/emailBuilder/grapesEngine.ts` (optional: helper to replace raw blocks)
- Test fixtures: `perfect-pixel-playground-project/src/lib/emailBuilder/__fixtures__/template-18.html`

## Out of scope

- Changing the block library or adding new block types.
- Altering the save/export pipeline — it already handles native components correctly.
- Mobile/responsive fidelity changes beyond what the importer naturally preserves.

---

Approved by: [user]  
Implementation plan: `docs/superpowers/plans/2026-08-04-html-import-native-editing-plan.md`

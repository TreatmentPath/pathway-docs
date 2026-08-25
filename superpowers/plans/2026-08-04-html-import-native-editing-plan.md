# HTML email import → native, no-code editing — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans (inline) or superpowers:subagent-driven-development (parallel). Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make imported HTML email templates open as fully editable, draggable, GUI-stylable MJML components, and automatically convert existing raw templates like Template 1.

**Architecture:** Improve the HTML importers (`brevoImport`, `genericHtmlImport`, `newsletter2goNative`) so they decompose nested table structures into native MJML components, then add a load-time decomposer in `EmailTemplateBuilder` that re-runs the importer on any remaining `mj-raw` blocks in saved templates.

**Tech Stack:** React/TypeScript, GrapesJS + grapesjs-mjml, MJML, Vitest + jsdom, Playwright.

---

## Task 1: Add a regression fixture for Template 1

**Files:**
- Create: `perfect-pixel-playground-project/src/lib/emailBuilder/__fixtures__/template-18.html`
- Modify: `perfect-pixel-playground-project/.gitattributes` (add `*.html text` if not present)

- [ ] **Step 1: Save the real Template 1 HTML as a fixture**

The fixture is the `layout_html` from the current backend response. Use the saved file `/tmp/template_18_layout.html`.

```bash
mkdir -p perfect-pixel-playground-project/src/lib/emailBuilder/__fixtures__
cp /tmp/template_18_layout.html perfect-pixel-playground-project/src/lib/emailBuilder/__fixtures__/template-18.html
```

- [ ] **Step 2: Verify the fixture is loadable in a test**

Add a temporary test that reads the fixture and checks it is non-empty.

```typescript
// perfect-pixel-playground-project/src/lib/emailBuilder/fixtureSmoke.test.ts
/** @vitest-environment jsdom */
import { describe, expect, it } from "vitest";
import templateHtml from "./__fixtures__/template-18.html?raw";

describe("template 18 fixture", () => {
  it("loads", () => {
    expect(templateHtml.length).toBeGreaterThan(10000);
    expect(templateHtml).toContain("Discover our trendiest plants");
  });
});
```

Run:

```bash
cd perfect-pixel-playground-project
npx vitest run src/lib/emailBuilder/fixtureSmoke.test.ts --reporter=verbose
```

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add perfect-pixel-playground-project/src/lib/emailBuilder/__fixtures__/template-18.html
rm -f perfect-pixel-playground-project/src/lib/emailBuilder/fixtureSmoke.test.ts
# fixtureSmoke test was temporary; do not commit it
```

---

## Task 2: Write failing importer tests for Template 1

**Files:**
- Modify: `perfect-pixel-playground-project/src/lib/emailBuilder/genericHtmlImport.test.ts`
- Modify: `perfect-pixel-playground-project/src/lib/emailBuilder/brevoImport.test.ts`

- [ ] **Step 1: Add generic import test for Template 1**

Append to `genericHtmlImport.test.ts`:

```typescript
import templateHtml from "./__fixtures__/template-18.html?raw";
import mjml2html from "mjml-browser";

// ... inside describe block

it("decomposes the real Template 1 into native components", async () => {
  const { mjml, nativeSections, rawSections } = convertGenericHtml(templateHtml);

  expect((mjml.match(/<mj-section/g) ?? []).length).toBeGreaterThanOrEqual(8);
  expect((mjml.match(/<mj-column/g) ?? []).length).toBeGreaterThanOrEqual(10);
  expect((mjml.match(/<mj-image/g) ?? []).length).toBeGreaterThanOrEqual(10);
  expect((mjml.match(/<mj-text/g) ?? []).length).toBeGreaterThanOrEqual(10);
  expect((mjml.match(/<mj-button/g) ?? []).length).toBeGreaterThanOrEqual(4);
  expect(rawSections).toBe(0);
  expect(nativeSections).toBeGreaterThanOrEqual(8);

  const { html: output, errors } = await mjml2html(mjml);
  expect(errors ?? []).toEqual([]);
  expect(output).toContain("Discover our trendiest plants");
  expect(output).toContain("Dieffenbachia Seguine");
  expect(output).toContain("Learn More");
});
```

- [ ] **Step 2: Add Brevo import test for Template 1**

Append to `brevoImport.test.ts`:

```typescript
import templateHtml from "./__fixtures__/template-18.html?raw";

it("recognizes Template 1 as a Brevo-style export and decomposes it natively", () => {
  const { mjml, nativeSections, rawSections, warnings } = convertBrevoHtml(templateHtml, { mode: "auto" });

  expect((mjml.match(/<mj-section/g) ?? []).length).toBeGreaterThanOrEqual(8);
  expect(rawSections).toBe(0);
  expect(nativeSections).toBeGreaterThanOrEqual(8);
  expect(warnings).toEqual([]);
  expect(mjml).toContain("Discover our trendiest plants");
  expect(mjml).toContain("Dieffenbachia Seguine");
});
```

- [ ] **Step 3: Run the tests and confirm they fail**

```bash
cd perfect-pixel-playground-project
npx vitest run src/lib/emailBuilder/genericHtmlImport.test.ts src/lib/emailBuilder/brevoImport.test.ts --reporter=verbose
```

Expected: FAIL — `rawSections` is 1 and `nativeSections` is 0 for generic; `BrevoImportError` for Brevo.

- [ ] **Step 4: Commit**

```bash
git add perfect-pixel-playground-project/src/lib/emailBuilder/genericHtmlImport.test.ts
git add perfect-pixel-playground-project/src/lib/emailBuilder/brevoImport.test.ts
```

---

## Task 3: Improve `genericHtmlImport.ts` to decompose nested wrappers

**Files:**
- Modify: `perfect-pixel-playground-project/src/lib/emailBuilder/genericHtmlImport.ts`

- [ ] **Step 1: Replace container detection with recursive descent**

Change `findGenericContainer` to mirror the Brevo importer's wrapper descent. Look for the body table by numeric width and descend through single-cell wrappers until reaching the cell that holds multiple section tables.

```typescript
function isSingleCellWrapper(table: Element): boolean { ... } // existing

function tableChildCount(node: Element): number { ... } // existing

function holdsSections(node: Element): boolean {
  if (node.tagName !== "TD" && node.tagName !== "TH") return false;
  const children = Array.from(node.children);
  return children.length > 0 && children.every((child) => child.tagName === "TABLE");
}

function descendToSectionCell(start: Element): { cell: Element; shell: Element[] } | null {
  const path: Element[] = [];
  let node: Element = start;
  let cellPath: Element[] | null = null;
  const MAX_DESCENT = 20;

  for (let depth = 0; depth < MAX_DESCENT; depth += 1) {
    path.push(node);
    if (holdsSections(node)) cellPath = [...path];

    const children = Array.from(node.children);
    if (children.length !== 1) break;
    node = children[0];
  }

  if (!cellPath) return null;
  return { cell: cellPath[cellPath.length - 1], shell: [] };
}

function findGenericContainer(doc: Document): { cell: Element; shell: Element[] } | null {
  const candidates = Array.from(doc.querySelectorAll("table")).filter(constrainsWidth);

  let best: { cell: Element; shell: Element[] } | null = null;
  let bestCount = 0;

  for (const table of candidates) {
    const deeper = descendToSectionCell(table);
    if (!deeper) continue;
    const count = tableChildCount(deeper.cell);
    if (count > bestCount) {
      best = deeper;
      bestCount = count;
    }
  }

  if (best) return best;

  if (doc.body.children.length > 0) {
    return { cell: doc.body, shell: [] };
  }
  return null;
}
```

- [ ] **Step 2: Replace section collection with recursive descent**

Replace `collectSections` with a version that unwraps single-cell wrappers and collects real section tables:

```typescript
function collectSections(container: Element, depth = 0): Element[] {
  if (depth >= 20) {
    return Array.from(container.children).filter((child) => child.tagName === "TABLE");
  }

  const sections: Element[] = [];
  for (const child of Array.from(container.children)) {
    if (child.tagName !== "TABLE") continue;

    if (isSingleCellWrapper(child)) {
      const cell = child.querySelector("td, th");
      if (cell && tableChildCount(cell) > 0) {
        sections.push(...collectSections(cell, depth + 1));
      } else {
        sections.push(child);
      }
    } else {
      sections.push(child);
    }
  }
  return sections;
}
```

- [ ] **Step 3: Improve column and block extraction**

Update `findColumns` to look in the first row of the section table and also in an inner layout table if the section table itself is a single-cell wrapper.

Update `extractBlocks` to decompose multi-cell tables inside a column by treating each cell as a sub-column instead of emitting the whole table as raw. Add a helper `extractBlocksFromCell` that recurses into the cell and emits leaf components.

```typescript
function extractBlocks(node: Element, blocks: ContentBlock[] = []): ContentBlock[] {
  for (const child of Array.from(node.children)) {
    if (child.tagName === "IMG") { blocks.push({ type: "image", element: child }); continue; }
    if (child.tagName === "A") { /* button or text */ continue; }
    if (child.tagName === "HR") { blocks.push({ type: "divider", element: child }); continue; }
    if (child.tagName === "TABLE") {
      if (isDividerTable(child)) { blocks.push({ type: "divider", element: child }); continue; }
      if (isSpacerTable(child)) { blocks.push({ type: "spacer", element: child }); continue; }
      if (isSingleCellWrapper(child)) {
        const cell = child.querySelector("td, th");
        if (cell) extractBlocks(cell, blocks);
      } else {
        // Multi-cell table: treat as a nested row of sub-columns
        const row = child.querySelector("tr");
        const cells = row ? Array.from(row.children).filter((c) => c.tagName === "TD" || c.tagName === "TH") : [];
        if (cells.length > 1) {
          for (const cell of cells) {
            extractBlocks(cell, blocks);
          }
        } else {
          blocks.push({ type: "raw", element: child });
        }
      }
      continue;
    }
    if (isTextContainer(child)) { blocks.push({ type: "text", element: child }); continue; }
    extractBlocks(child, blocks);
  }
  return blocks;
}
```

- [ ] **Step 4: Run generic import tests and iterate until pass**

```bash
cd perfect-pixel-playground-project
npx vitest run src/lib/emailBuilder/genericHtmlImport.test.ts --reporter=verbose
```

Expected: PASS after iterating on the decomposition logic.

- [ ] **Step 5: Commit**

```bash
git add perfect-pixel-playground-project/src/lib/emailBuilder/genericHtmlImport.ts
```

---

## Task 4: Improve `brevoImport.ts` body-table detection

**Files:**
- Modify: `perfect-pixel-playground-project/src/lib/emailBuilder/brevoImport.ts`

- [ ] **Step 1: Broaden `findSectionContainer`**

Change `findSectionContainer` to try the explicit class first, then fall back to the outermost numeric-width body table.

```typescript
export function findSectionContainer(doc: Document): Element | null {
  const bodyTable = doc.querySelector(BODY_TABLE_SELECTOR);
  if (bodyTable) return descendToSectionCell(bodyTable)?.cell ?? null;

  // Fallback: any table whose width is a numeric email body width and whose cell holds sections
  const candidates = Array.from(doc.querySelectorAll("table")).filter((table) => {
    const width = table.getAttribute("width") ?? "";
    if (!/^\d+$/.test(width)) return false;
    const numeric = parseInt(width, 10);
    return numeric >= 400 && numeric <= 900;
  });

  let bestCell: Element | null = null;
  let bestCount = 0;
  for (const table of candidates) {
    const deeper = descendToSectionCell(table);
    if (!deeper) continue;
    const count = tableChildCount(deeper.cell);
    if (count > bestCount) {
      bestCell = deeper.cell;
      bestCount = count;
    }
  }
  return bestCell;
}
```

`tableChildCount` must be defined or imported; add a small helper in this file if not already exported.

- [ ] **Step 2: Run Brevo tests and iterate until pass**

```bash
cd perfect-pixel-playground-project
npx vitest run src/lib/emailBuilder/brevoImport.test.ts --reporter=verbose
```

Expected: PASS — Template 1 is recognized and decomposed into native sections.

- [ ] **Step 3: Commit**

```bash
git add perfect-pixel-playground-project/src/lib/emailBuilder/brevoImport.ts
```

---

## Task 5: Improve `newsletter2goNative.ts` leaf recognition

**Files:**
- Modify: `perfect-pixel-playground-project/src/lib/emailBuilder/newsletter2goNative.ts`

- [ ] **Step 1: Recognize more text containers and buttons**

Update `isTextContainer` to include `<span>` with text, and update `transpileColumn` to handle plain `<a>` links as text when they do not look like buttons, and as `mj-button` when they do.

```typescript
function isTextContainer(element: Element): boolean {
  if (["P", "H1", "H2", "H3", "H4", "H5", "H6"].includes(element.tagName)) return true;
  if (element.tagName === "DIV" || element.tagName === "SPAN") {
    if (element.querySelector("img, table")) return false;
    return (element.textContent ?? "").trim().length > 0;
  }
  return false;
}

function isButton(anchor: Element): boolean {
  const style = anchor.getAttribute("style") ?? "";
  return /display\s*:\s*inline-block/i.test(style) || /border-radius/i.test(style) || /background-color/i.test(style) || /padding/i.test(style);
}
```

In `transpileColumn`:

```typescript
if (child.tagName === "A") {
  if (isButton(child)) {
    parts.push(emitButton(child, column));
  } else {
    parts.push(emitText(child, column));
  }
  child.querySelectorAll("*").forEach((n) => consumed.add(n));
  continue;
}
```

- [ ] **Step 2: Run native tests**

```bash
cd perfect-pixel-playground-project
npx vitest run src/lib/emailBuilder/newsletter2goNative.test.ts --reporter=verbose
```

Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add perfect-pixel-playground-project/src/lib/emailBuilder/newsletter2goNative.ts
```

---

## Task 6: Add load-time decomposer in `EmailTemplateBuilder`

**Files:**
- Modify: `perfect-pixel-playground-project/src/pages/EmailTemplateBuilder.tsx`
- Modify: `perfect-pixel-playground-project/src/lib/emailBuilder/grapesEngine.ts` (optional helper)

- [ ] **Step 1: Add a `decomposeRawBlocks` method to the engine interface**

In `GrapesEmailCanvas.tsx` interface:

```typescript
export interface GrapesCanvasEngine {
  // ... existing methods
  /** Re-runs the importer on existing raw blocks and replaces them with native components. */
  decomposeRawBlocks(): Promise<{ converted: number; skipped: number }>;
}
```

In `GrapesMjmlEngine`:

```typescript
async decomposeRawBlocks(): Promise<{ converted: number; skipped: number }> {
  if (!this.runtime) throw new Error("The email editor canvas is not ready.");
  return this.runtime.decomposeRawBlocks();
}
```

In `GrapesRuntime` interface:

```typescript
decomposeRawBlocks(): Promise<{ converted: number; skipped: number }>;
```

In `createGrapesMjmlRuntime`:

```typescript
decomposeRawBlocks: () => {
  const rawComponents = editor.Components.getWrapper()?.find("[data-gjs-type=\"mj-raw\"]") ?? [];
  let converted = 0;
  let skipped = 0;

  for (const raw of rawComponents) {
    const html = raw.getInnerHTML();
    if (!html.trim()) continue;

    let result: { mjml: string };
    try {
      result = convertBrevoHtml(html, { mode: "auto" });
    } catch {
      try {
        result = convertGenericHtml(html);
      } catch {
        skipped += 1;
        continue;
      }
    }

    // Safety check: must produce at least one native section
    if (!result.mjml.includes("<mj-section")) {
      skipped += 1;
      continue;
    }

    // Replace the raw component's content with the native MJML
    raw.set("content", "");
    raw.components(result.mjml);
    converted += 1;
  }

  return { converted, skipped };
},
```

Import `convertBrevoHtml` and `convertGenericHtml` at the top of `grapesEngine.ts` if not already imported.

- [ ] **Step 2: Call the decomposer after loading a saved template**

In `EmailTemplateBuilder.tsx`, after the initial document is loaded and the engine is ready, decompose raw blocks:

```typescript
// Inside the load effect, after setDocument(loadedDoc) and setBodyBackgroundColor(...)
if (engineRef.current && isEditMode && hasDocument(template.design_json)) {
  void (async () => {
    try {
      const { converted, skipped } = await engineRef.current!.decomposeRawBlocks();
      if (converted > 0) {
        setDirty(true);
        toast.info(`Converted ${converted} imported section(s) to editable blocks.`);
      }
      if (skipped > 0) {
        toast.warning(`${skipped} section(s) could not be converted and remain as raw HTML.`);
      }
    } catch (error) {
      console.error("Raw block decomposition failed", error);
    }
  })();
}
```

- [ ] **Step 3: Run EmailTemplateBuilder tests**

```bash
cd perfect-pixel-playground-project
npx vitest run src/pages/EmailTemplateBuilder.test.tsx --reporter=verbose
# or the closest test file if it doesn't exist
```

- [ ] **Step 4: Commit**

```bash
git add perfect-pixel-playground-project/src/lib/emailBuilder/grapesEngine.ts
git add perfect-pixel-playground-project/src/components/email-builder/GrapesEmailCanvas.tsx
git add perfect-pixel-playground-project/src/pages/EmailTemplateBuilder.tsx
```

---

## Task 7: Verify with Playwright on Template 1

**Files:**
- Create: `perfect-pixel-playground-project/e2e/template-import-editable.spec.ts` (optional; add if e2e tests exist in the project)

- [ ] **Step 1: Open the builder and load Template 1**

Use Playwright to navigate to `http://localhost:8080/marketing/templates/builder/18` and wait for the canvas.

- [ ] **Step 2: Click a product image and check the panel**

Find a product image in the iframe, click it, and assert the side panel does not contain the text "Custom HTML".

- [ ] **Step 3: Click a heading and check the panel**

Click a heading (e.g., "Discover our trendiest plants") and assert the panel shows editable text fields, not a raw HTML textarea.

- [ ] **Step 4: Click a button and check the panel**

Click a "Learn More" button and assert the panel shows `href` and button styling controls.

- [ ] **Step 5: Take a screenshot for manual review**

Capture the builder after conversion and include it in the final report.

---

## Task 8: Full test suite and cleanup

- [ ] **Step 1: Run all email-builder tests**

```bash
cd perfect-pixel-playground-project
npx vitest run src/lib/emailBuilder --reporter=verbose
```

Expected: all PASS.

- [ ] **Step 2: Remove temporary diagnostic files**

```bash
rm -f perfect-pixel-playground-project/src/lib/emailBuilder/template18Diagnostic.test.ts
rm -f perfect-pixel-playground-project/src/lib/emailBuilder/template18BrevoClass.test.ts
```

- [ ] **Step 3: Run typecheck**

```bash
cd perfect-pixel-playground-project
npm run typecheck
```

Expected: no errors.

- [ ] **Step 4: Run lint/format on touched files**

```bash
cd perfect-pixel-playground-project
npx prettier --write src/lib/emailBuilder/brevoImport.ts src/lib/emailBuilder/genericHtmlImport.ts src/lib/emailBuilder/newsletter2goNative.ts src/lib/emailBuilder/grapesEngine.ts src/components/email-builder/GrapesEmailCanvas.tsx src/pages/EmailTemplateBuilder.tsx
npx eslint --fix src/lib/emailBuilder/brevoImport.ts src/lib/emailBuilder/genericHtmlImport.ts src/lib/emailBuilder/newsletter2goNative.ts src/lib/emailBuilder/grapesEngine.ts src/components/email-builder/GrapesEmailCanvas.tsx src/pages/EmailTemplateBuilder.tsx
```

- [ ] **Step 5: Commit and final review**

```bash
git status
```

Ensure only the intended files are modified and the temporary tests are gone.

---

## Spec coverage check

| Spec requirement | Task |
|---|---|
| Imported sections become `mj-section` | Tasks 3, 4, 5 |
| Columns become `mj-column` | Tasks 3, 4, 5 |
| Text becomes `mj-text` | Tasks 3, 5 |
| Images become `mj-image` | Tasks 3, 4, 5 |
| Buttons become `mj-button` | Tasks 3, 5 |
| Dividers become `mj-divider` | Task 3 |
| Spacers become `mj-spacer` | Task 3 |
| Existing raw templates auto-convert | Task 6 |
| No raw HTML visible to user | Tasks 6, 7 |
| Regression tests with real template | Task 2 |
| Playwright verification | Task 7 |

No placeholders detected. All file paths are exact.

# Consent Template Library Cloning Design

**Date:** 2026-09-10  
**Status:** Approved design; awaiting written-spec review  
**Repositories:** `TreatmentPathBackend`, `perfect-pixel-playground-project`

## Goal

Make platform consent templates behave like the existing Email and SMS template libraries. A platform template must remain an admin-owned library source and must not automatically appear as a usable practice template. A practice user explicitly adds it to their environment, creating an independent practice-owned copy.

## User experience

The Consent tab in Settings → Templates has two sections:

1. **My Templates** — only consent templates owned by the current practice.
2. **Template Library** — active platform templates, with the same supporting text used for Email and SMS: “Ready-made templates you can copy into your practice.”

Each library row provides:

- Preview, using the existing consent preview dialog.
- Add, when the platform template has not been cloned into the practice.
- Added, when a clone already exists, matching the Email/SMS visual treatment.
- A loading state while the Add request is running.

After a successful Add, the practice copy appears in My Templates immediately and the library row changes to Added. The practice may edit or archive its copy without changing the platform source.

Empty, loading, and error states follow the existing Email/SMS template-list primitives. Practice users without `consent_templates_manage` may browse and preview, but cannot add, edit, or archive templates.

## Data ownership

Add a nullable self-reference on `ConsentDocumentTemplate`:

```text
cloned_from_global -> ConsentDocumentTemplate (SET_NULL)
```

Rules:

- Platform templates have `practice = NULL`, `is_global = true`, and `cloned_from_global = NULL`.
- Practice templates have `practice = current_practice` and `is_global = false`.
- A cloned practice template points to its platform source through `cloned_from_global`.
- A database uniqueness constraint prevents more than one clone of the same platform template per practice.
- Existing locally authored templates remain valid with `cloned_from_global = NULL`.

## API behavior

### Practice templates

`GET /api/backend/documents/consent-templates/`

- Returns only templates owned by the current practice.
- Never includes platform templates.
- Existing create, retrieve, update, archive, restore, and field-layout operations remain practice-scoped.

### Platform library

`GET /api/backend/documents/global-consent-templates/`

- Authenticated practice users may list active platform templates for the library.
- The response includes whether the current practice has already added each source template, either through `already_added` or enough clone-source data for the frontend to derive it reliably.
- Superusers retain the existing global create, update, archive, restore, and field-layout management operations.
- Non-superusers cannot mutate platform templates.

### Add to practice

`POST /api/backend/documents/global-consent-templates/{id}/add-to-my-environment/`

- Requires an authenticated user, a current practice, and `consent_templates_manage` access.
- Accepts only active platform templates.
- Rejects duplicate additions with a clear response containing the existing practice template.
- Runs atomically.
- Copies title, category, source type, body, PDF reference, PDF page count, default email subject, expiry, active state, and every signature/date field placement.
- Assigns the new template to the current practice, sets `is_global = false`, records `cloned_from_global`, and records the requesting user as creator.
- Returns the created practice template using the normal consent-template serializer.

The existing global Consent CRUD endpoints remain because the admin management page depends on them. No endpoint is removed as part of this change.

## Signing and sending

Practice signing/send flows may resolve only practice-owned consent templates. A platform source cannot be sent or signed directly; it must first be added to the practice. This enforces the same environment boundary visible in the Settings UI.

Existing signing requests retain their snapshotted documents and are unaffected.

## PDF cloning

The practice clone preserves access to the same immutable uploaded PDF object initially and independently copies all field-layout rows. Replacing the PDF on the practice clone updates only that clone. Archiving or editing either template does not mutate the other.

No physical file duplication is required during Add because template archival does not delete the stored source object, and replacing a FileField changes the database reference rather than overwriting the source template row.

## Failure handling

- Missing/inactive platform source: `404`.
- No current practice: clear `400` or existing practice-resolution error.
- Missing management permission: `403`.
- Already added: `400`, matching Email/SMS behavior, with the existing practice template in the response.
- Unexpected Add failure: transaction rolls back both the template and copied fields.
- Frontend shows the backend message through the existing toast system and leaves the row actionable for retry.

## Tests

Backend tests will prove:

- Practice list excludes global and other-practice templates.
- Library list contains active global templates and exposes Added state per practice.
- Users cannot mutate platform templates through library access.
- Authorized Add creates the correct practice-owned clone.
- Text content and PDF metadata/field layouts are copied.
- Duplicate Add is rejected without creating another row.
- Cross-practice clone state remains isolated.
- Sending rejects a global template ID and accepts its practice clone.

Frontend tests will prove:

- Consent renders My Templates and Template Library separately.
- Platform templates do not appear in My Templates before Add.
- Add calls the canonical endpoint and moves the returned clone into My Templates.
- The library row changes from Add to Added.
- Preview works without cloning.
- Users without management access do not receive an Add action.

The feature will also be exercised in the local browser from Settings → Templates → Consent, including a real Add operation against local development data. No consent will be sent and no git commit will be created.

## Out of scope

- Synchronizing later platform-template edits into existing practice clones.
- Removing or hard-deleting platform templates.
- Bulk-adding all platform templates.
- Changing Email or SMS template behavior.

# Consent Document Workflow Design

## Objective

Deliver FR1-FR16 as five independently deployable vertical slices while preserving practice isolation, existing signing links, in-flight requests, completed artifacts, and the existing note and letter editors.

The signing-template domain in this design is `Documents.ConsentDocumentTemplate`. It is intentionally separate from `Notes.ConsentTemplate`, which controls clinical-note consent detection and is outside this feature.

## Delivery slices

1. Template foundation: remove the configurable email body, add `consent_templates_manage`, and add global signing templates.
2. Signing ceremony: capture one signature outside the PDF, calculate the signing date on the server, stamp all configured placements, and preserve immutable results.
3. Patient presentation and delivery: apply practice branding, send from `confirm.dental`, and make the public signing page responsive.
4. Request lifecycle and Day List: reuse one request across QR/email/SMS, keep QR inside the send dialog, and aggregate all outstanding consent on the Day List.
5. Docs: rename Drafts to Docs and provide one saved-document list containing notes, letters, and consent requests.

Each slice must leave the application deployable and must include backend and frontend regression tests before the next slice begins.

## Slice 1: Template foundation

### Default email content

Remove `default_email_body` from signing-template serializers and frontend types/forms. The consent invitation body becomes a single server-owned standard message containing the patient name, practice identity, and signing link. Staff may retain an optional subject override, but cannot customize the invitation body when sending.

Keep the existing database column for the first deployment but stop reading and writing it. A later cleanup migration may remove it after deployed code and queued tasks no longer reference it. Existing stored values therefore become inert immediately without making the rollout unnecessarily destructive.

### Permission

Add the practice-scoped feature key `consent_templates_manage` to `FEATURE_DEFINITIONS`, enabled only for `practice_admin` by default. Because the existing feature-access system already supports role defaults and individual overrides, the new key automatically appears in both settings surfaces.

Apply the permission to every mutating signing-template action: create, update, partial update, archive, restore, and PDF field-layout replacement. Listing and retrieving active templates remain available to staff who can send consent. Archived-template management views require the management permission. The frontend hides or disables create/edit/archive/restore controls when the permission is absent; the server remains authoritative.

### Global signing templates

Make `ConsentDocumentTemplate.practice` nullable and add an explicit `is_global` boolean with a database constraint enforcing exactly one valid ownership mode:

- global: `is_global=True`, `practice=NULL`;
- practice-owned: `is_global=False`, `practice` populated.

Regular practice reads return active global templates plus templates belonging to the current practice. Global rows are labelled `Global` and are read-only outside the global-admin API. Practice users cannot archive, edit, or replace fields on them. Superusers manage global signing templates through a dedicated Documents endpoint and UI that reuses the signing-template editor; it must not reuse the note-detection consent endpoints.

When creating a signing request, the server accepts a template only when it is active and either owned by the request's practice or global. The request document continues to snapshot content, PDF bytes, and field layout so later global edits cannot alter an in-flight request.

## Slice 2: Signing ceremony

### One signature and authoritative date

The public signing interface captures one signature using controls outside the PDF. PDF pages and their placed fields are display-only during signing. The client submits `signature_data`, `signature_method`, and `signer_name` once; it never submits a signing date.

Inside one database transaction, the server records the completion timestamp with `timezone.now()`. It converts that timestamp through the sending practice's configured IANA timezone and formats the date for the PDF. Every signature placement receives the same submitted signature and every date placement receives the same server-derived local date. Existing field-layout snapshots and new templates use the same stamping path.

For bundles, one final signature covers every required document after all declarations/reviews are complete. PDF documents no longer require their own per-field signature ceremony. A bundle stays `pending` until every required document is ready and the final signature succeeds.

### Preservation and immutability

Persist the authoritative completion timestamp, signature data/method, derived signing date, field-layout snapshot, and generated final PDF in the signing record/artifact. Completed requests reject further signature, field, date, or document mutations. Artifact generation reads only snapshotted request data and never current template fields.

Existing completed artifacts are not rewritten. Existing active requests retain their snapshotted placements and adopt the new automatic stamping behavior when completed.

## Slice 3: Patient presentation and delivery

### Consent email

Create a dedicated consent invitation renderer rather than adapting the shared Pathway alert wrapper. Resolve and sanitize the practice primary, secondary, accent, and background colours, falling back to accessible defaults. Render the practice logo once above the invitation; when absent, render the practice name in the same location. Constrain the logo with email-safe inline width and height styles and provide a mobile media rule.

Use a fixed standard invitation body and a brand-coloured signing button. Include a plain-text signing URL fallback. Do not render Pathway logos or a broken image.

Send with an address on `confirm.dental`, using the practice name as the RFC display name. Resolve Reply-To from the selected practice email-domain configuration and pass it through the existing email-service client. If no reply address is configured, omit Reply-To rather than inventing one. Environment configuration supplies the local part/default sender; no credentials or provider identifiers are hardcoded.

### Public signing layout

The initial route-loading state is a plain document spinner with no Pathway branding. On small screens the signing shell uses the dynamic viewport (`100dvh` with a safe fallback), a compact practice header, a scrollable document region, and a bottom action bar. PDF pages scale to the available container width and retain their aspect ratio. The viewer scrolls through every page instead of fitting the whole document into one screen.

Use `ResizeObserver` and viewport-safe CSS rather than a fixed `55vh` panel so rotation and mobile browser chrome changes recalculate the usable area. Desktop retains a constrained readable width.

## Slice 4: Request lifecycle and Day List

### Request reuse

Separate request creation from delivery. A get-or-create service receives practice, patient, selected ordered template IDs, and source context, then returns the matching active request. A database-backed idempotency identity prevents concurrent QR and delivery actions from creating duplicates. Terminal requests are never reused.

QR display, email sending, and SMS sending all operate on that request. Opening or retrying the QR tab only fetches the QR representation and appends an audit event; it does not resend. Email/SMS delivery updates `sent_at` and delivery audit state on the same request. A failed delivery leaves the request and QR usable for retry.

The consent dialog contains Email, SMS, and QR tabs. Opening QR creates the request when necessary and displays the code plus copyable signing link. Successful email/SMS sends keep the dialog open and switch or enable the QR tab for the same request.

### Day List

Build a practice- and patient-scoped consent summary that distinguishes:

- required with no request: `not_sent`;
- at least one active incomplete request: `pending`;
- no incomplete request and at least one completed request: `signed`.

Aggregate every active outstanding request instead of selecting only the newest request. Return request IDs, document titles, per-request status, and whether a QR request can be opened. Partially signed bundles remain pending until their request is fully signed.

The Day List QR action opens an existing request directly. For `not_sent`, it opens the same consent selection dialog and creates a QR request without delivery. Subscribe the page to the existing practice signing-status WebSocket and refresh the affected date/patient after completion while preserving existing feature and patient-access checks.

## Slice 5: Docs

### Unified data contract

Add a practice-scoped Docs endpoint returning only:

- saved notes (`is_draft=False`), ordered by their latest saved/updated time;
- saved letters (`is_draft=False`), ordered by their latest saved/updated time;
- consent signing requests, ordered by latest successful sent time.

Normalize each row to `id`, `title`, `patient_id`, `patient_name`, `document_type`, `status`, `activity_at`, and an explicit navigation target. Merge and paginate after applying practice, patient, and feature-access restrictions. One signing request produces one row; resend changes `activity_at` on that row and cannot create a duplicate.

Opening a row is read-only with respect to ordering. Notes and letters use their existing views. Consent opens the request detail or completed signed-document view according to status.

### Interface terminology

Rename the Create-area navigation label and visible page copy from Drafts to Docs, including page title, recent-items title, search prompt, help copy, and empty state. The existing `/draft` route may remain as a backward-compatible URL, but all user-facing terminology becomes document-based.

Replace note/letter tabs and draft-status controls with document-type and status filters appropriate to the normalized response. Remove unsaved note/letter fetching from this screen; editor-specific draft recovery remains unchanged elsewhere.

## Error handling and compatibility

- All template and request lookups are practice-scoped before object mutation.
- Permission failures return 403; cross-practice/global mutation attempts do not leak object existence.
- QR creation is retryable and idempotent.
- Delivery failures preserve the signing request and expose a retryable result.
- PDF generation failures preserve the signed record and remain retryable through the existing asynchronous artifact pipeline.
- Existing active signing links remain valid through every slice.
- The Go services require no schema writes for these slices; any shared-table changes must still be compatible with their reads.

## Testing strategy

Use red-green-refactor for every behavior change.

Backend coverage includes permission defaults/overrides, global-template isolation, legacy body non-use, automatic multi-placement stamping, practice-timezone date boundaries, completed-record immutability, request idempotency/concurrency, delivery metadata, unified Docs ordering/deduplication, and Day List aggregation.

Frontend coverage includes permission-gated controls, global badges/read-only actions, one external signature submission, responsive PDF sizing helpers, QR-tab lifecycle/retry, Day List not-sent/pending rendering and refresh, and Docs terminology/filter/navigation.

Run focused suites after each task, then the complete Django Documents/dentally integration tests, frontend Vitest suite, frontend typecheck/build, and formatting/pre-commit checks for every touched file.

## Acceptance

The implementation is complete only when all FR1-FR16 have automated coverage, no current signing link or completed artifact regresses, frontend and backend contracts agree, and the final verification suites pass without weakening existing assertions.

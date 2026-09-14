# TreatmentPath Frontend — Repeated Logic and Utility Map

## Purpose

This document maps logic that is repeated, reimplemented, wrapped, or partially duplicated across the primary React frontend (`perfect-pixel-playground-project`). The goal is to identify code that should become a shared hook, shared component, API-layer helper, validator, or a documented canonical implementation.

This is a read-only source-mapping exercise. It does not modify application code, configuration, or build outputs.

## Areas in scope

Root: `perfect-pixel-playground-project/src/`

1. `pages/` (route components, ~1,178 files)
2. `components/` (~864 files)
3. `hooks/` (~253 files)
4. `lib/` (~144 files, incl. API layer)
5. `utils/` (~35 files)
6. `contexts/` (~18 files)

Direct callers (route-level consumers of components/hooks, cross-area helpers) are included.

## What counts as repeated logic

Map logic when two or more files implement the same semantic operation, including:

- Data fetching, mutations, caching, and query-key construction
- Form validation, normalization, and error display
- Date/time/timezone/currency/number formatting and parsing
- API request handling, auth headers, error handling, retries
- Permission/feature gating and role checks
- Modal/dialog/toast orchestration patterns
- Pagination, filtering, sorting, and search state
- Status/label/color mapping tables and display transforms
- Repeated business rules or workflows

Do not count ordinary imports of the same shared hook/component/util as duplication unless a caller reimplements or bypasses its invariants. Distinguish true shared implementation from copy-pasted, wrapper-specific, or semantically divergent logic. Tests, generated files, and intentionally different implementations are treated separately.

## Sequential review protocol

Use exactly one independent subagent at a time. Never run area agents in parallel. For each area: sweep, record only genuinely new mapping entries, restart that area's clean-pass count on any new entry, and continue until three consecutive clean passes.

## Continuous repeated-logic register

Add new entries at the end; never renumber.

**Current count: 262 mapping entries.**

### Mapping 260 — "Dentally day-list appointment status → display bucket" classifier, 3 divergent live mappings
- `components/dashboard/DaylistStatsWidget.tsx:43–52` — `arrived`/`in_chair` count as **completed** (with `Math.max(0,…)` pending guard); `pages/DayListPage.tsx:215–228` — `arrived`/`in_chair` count as **active** (completed = `{completed}` only); `pages/day-list/lib/fetchDayListAppointments.ts:145–148` normalizer — fourth scheme (`reviewed` = `{confirmed}`, `arrived`/`in_chair` → **pending**). **Live drift:** the same `arrived`/`in_chair` appointment renders as Done on the digest widget, Active in the day-list filter, and Pending in the normalized data model. **Canonical:** `classifyDaylistAppointmentStatus(status)` exported from `pages/day-list/lib/fetchDayListAppointments.ts`. **Extraction:** safe. **Confidence:** high.

### Mapping 261 — M107 ext: `StockKPICards.tsx` is a live 5th member of the KPICard clone family
- `components/spend-reporting/StockKPICards.tsx:18–53` — byte-identical `getAlertStyles` + icon chip as the four mapped siblings; live via `pages/SpendReporting.tsx:19,60`. `SpendKPICards.tsx:24–60` is a 6th identical clone but **dead** (exported via barrel with zero importers).

### Mapping 262 — percentage → SVG circular-progress ring + RAG color ladder re-rolled (live drift on thresholds)
- `pages/compliance/components/dashboard/ComplianceDashboard.tsx:526–537,568–584` and again `:757–768,780–805` (in-file duplicate pair) — hardcoded 75/50 RAG ladders + `r=28` dasharray ring; `pages/compliance/components/audit/AuditScoreDisplay.tsx:55–100` `AuditScoreGauge` (r=45, colors from canonical `getAuditRAGStatus`). **Live drift:** dashboard colors ≥75 green/≥50 amber while the module canonical `getRAGStatus` (`pages/compliance/constants.ts:20–29`) uses **90/70** — a practice at 75–89% shows green ring but amber status.
- **Dead-lane correction:** `pages/compliance/components/dashboard/index.ts` barrel exports (`OverallScoreCard`, `ComplianceScoreToggleCard`, `CategoryBreakdownCards`, `ActionItemsList`, `MiniCalendarWidget`, `LogStatsCard`, `KLOEBarChart`) have **zero importers** (ComplianceDashboard imports four widgets directly) — the byte-identical `RAG_LABELS` pair `OverallScoreCard.tsx:12–19` ↔ `ComplianceScoreToggleCard.tsx:14–19` is dead-alive; deletion candidates.

## Frontend pass-65 clean-verification notes (clean lenses; extensions recorded)
- Email-regex family (see exclusions): additional inline members — `components/{AddPatientModal.tsx:197, compact/AddPatientModal.tsx:913, journeys/AddPatientDialog.tsx:694, inbox/v2/AssignToPatientModal.tsx:194}` and drifted `{2,}` variant `components/marketing/previewTestTypes.ts:51`.
- `truncateLabel` pair `financial-analytics/components/{SankeyTab.tsx:329, CohortReferralTab.tsx:667}` token-identical but CohortReferralTab is dead-lane — excluded.
- `isImageUrl`/`isImage` pair `ChecklistRecordView.tsx:139` ↔ `exportChecklistRecords.ts:29` — inside the mapped XLSX-export clone cluster (M38-adjacent).
- localStorage JSON read scaffolds (NavContext, NavMain, DraggableDock [dead], learning/storage, associatePayUtils, payslipAdminConfig, chromeAlertsSettings) — single-owner per-key codecs; pass-36 clean verdict holds.

### Mapping 258 — "Authenticated fetch → blob → `URL.createObjectURL` → preview state" effect with `cancelled` flag, 6 live implementations
- Preview-render variant (distinct from M17's download scaffold and M225's base64 reader): `pages/settings/components/consent/ConsentTemplatePreview.tsx:50–76` (revoke-on-cleanup, cache hand-off); `pages/compliance/components/documents/{GeneralDocumentPreviewModal.tsx:59–85, GeneralDocumentSigningSection.tsx:66–103}` (byte-identical siblings; revoke-previous via `blobUrlRef`); `PdfFieldPlacementEditor.tsx:69–90` (errors console-only); `components/patients/workspace/documents/unified/DocumentPreviewModal.tsx:211–226` (**never revokes — leak on repeated opens**, silent catch); `unified/RowThumbnail.tsx:26–49` (image thumbnails). **Canonical:** `useAuthenticatedBlobUrl(fetchWithAuth, url, {revoke: 'cleanup'|'replace'})` returning `{url, loading, error}`. **Extraction:** safe. **Confidence:** high.

### Mapping 259 — "AlertDialog confirm whose open state IS the target row object" (`useState<Row | null>`), 8 live sites
- Store-row-as-open-flag machine (`open={!!target}`, `onOpenChange={(o) => !o && setTarget(null)}`, guard → act on `target.id` → null-clear): `pages/MarketingTemplates.tsx:101,206–221,529–548`; `pages/admin/AdminFormTemplates.tsx:80,304–324`; `pages/Stock.tsx:518,1546–1556,5322–5327`; `pages/settings/components/practice/MembershipPlansView.tsx:67,321–343` (richest row-field copy); `pages/admin/LabelTemplates.tsx:78,132–161,449–468`; `pages/Labs.tsx:308,1417–1430,2845–2860`; variants `SigningBatchesTab.tsx:192` (bulk-remind), `finance/PatientBalanceTab.tsx:47,115–127` (void flow). **Canonical:** `useRowActionDialog<Row>(onConfirm)` or `<ConfirmActionDialog target …/>` beside `ui/confirmation-dialog.tsx` (M202's inline-id sibling could fold under the same extraction). **Extraction:** safe. **Confidence:** high.

### Mapping 257 — "Dentally drift status fetch + manual full reconcile + amber drift banner" pair (day-list ↔ settings), byte-identical with field-name drift
- `pages/day-list/components/DaylistReconciliationBanner.tsx:20–58` vs `pages/settings/components/integrations/ReconciliationPanel.tsx:44–53,72–82` — `load` and `reconcileNow` character-identical (same `RECONCILIATION_STATUS`/`RECONCILE({full:true})` endpoints, best-effort catch, `reconciling` finally); amber strip + purple action link classes byte-identical (Banner:56 vs Panel:202); same `{reconciling ? 'Reconciling…' : …}` ternary.
- **Live drift:** banner reads `daylist_needs_reconciliation`/`daylist_mismatch_count` (Banner:7,44) while the panel reads `needs_reconciliation`/`mismatch_count` (Panel:15,84) against the same endpoint — unless the serializer returns both spellings, one surface's drift flag can never fire. **Canonical:** `useDentallyReconciliation()` hook + `<ReconciliationStrip variant>` presentation; reconcile field names first. **Extraction:** safe after field-name check. **Confidence:** high.

### Mapping 254 — Assistant-chat `sendChatMessage` machine (abort single-flight + session reconciliation + AbortError-swallowing catch), verbatim pair
- `components/assistant-toolbar/ChatPanel.tsx:542–670` ↔ `components/chat/ChatAgent.tsx:916–1040` (~120 lines each): `inFlightRequestRef.abort()` → optimistic user push (`${Date.now()}-user`) → textarea height reset (`"auto"`/`"52px"` byte-identical) → `converseWithAssistant(payload, {signal})` → `chat_session`/`session_updated` reconciliation (identical console.log string) → `deriveAssistantMessages` (separately defined per-file :327/:520) → byte-identical AbortError guard + `"I hit a snag: ..."` bubble → guarded finally. Divergence: payload projection (note_sections vs letter/note branch) and tool-side-effect application. **Canonical:** `useAssistantChatSend({buildPayload, onWidgets})` + shared `deriveAssistantMessages`. **Extraction:** safe. **Confidence:** high.

### Mapping 255 — Inline DRF list-envelope unwrap idiom (`Array.isArray(data) ? data : data.results || []`), ~15 live sites
- `hooks/usePathways.ts:196,217,327,437,547` (5× in one file, with `|| []`); `hooks/useStock.ts:390,591,732,836,886` (5×, **no `|| []` fallback — yields undefined on non-array envelopes**); `contexts/{PatientsContext.tsx:104, TreatmentContext.tsx:527, TaskContext.tsx:147–149}`, `hooks/usePatientWorkspace.ts:127`, `useMessaging.ts:512`; `?? []` variant: `useLabsQuery.ts:323`, `useDentalChart.ts:641`, `usePatientConsentHistory.ts:92`, `useStockQuery.ts:1474` (passes the whole envelope through as fallback), `useTaskCommentsQuery.ts:33`; widest ladder `useMarketingReferenceData.ts:23` (adds `raw?.data ?? []`). **Canonical:** `unwrapDrfList<T>(data, {fallback=[]})` in `lib/apiUtils.ts`. **Extraction:** safe, trivial. **Confidence:** high (M127 mapped the type, not this operation).

### Mapping 256 — Marketing paginated-list hook scaffold, 5 hooks
- `hooks/useMarketingSegments.ts:30–80` ↔ `useMarketingCampaigns.ts:50–100` (normalized diff = endpoint + nouns; identical `{results: ??[], count: ??0}` projection, key template `marketing:<x>:${practiceId}:${page}:${pageSize}:${search}`, `useCachedData` config, return shim); `useMarketingArchived.ts:27–70`, `useMarketingCampaignPreview.ts:42–95` (+`total_pages ?? 1`); `useMarketingSegmentPreview.ts:30–80` (POST, `enabled: Boolean(id)`). Return-field naming drift (`segments`/`campaigns`/`items`/`results`); segments/campaigns lack `enabled` gates. **Canonical:** `useMarketingPaginatedList({resource, paginated, method?})` factory. **Extraction:** safe. **Confidence:** high.

### Mapping 250 — fraction → rounded-percent-string helper (`pctText`/`pct`), product-analytics
- Token-identical private pair: `pages/admin/product-analytics/{RetentionView.tsx:26–27, InsightsView.tsx:28}` `pctText`; same-op variants `findings.ts:51 pct` (returns number, 6 uses), `StatTiles.tsx:27` (inline `Math.abs` change), `PracticeFeatures.tsx:96` (inline share-of-total). **Canonical:** export `pctText(fraction)` from the directory. **Extraction:** trivial. **Confidence:** high, minor.

### Mapping 251 — share→percent mini-bar width with visibility floor (`Math.max(share*100, min)`), 4 files
- `pages/admin/product-analytics/{TopUsersChart.tsx:103–108, PracticeUsageChart.tsx:58–63, PracticeFeatures.tsx:83–89, InsightsView.tsx:193}` — identical bar classes `bg-[#7A5CCC] dark:bg-[#9478DF]`; floor drift **1.5 vs 2** (a 0.3%-share route renders twice as wide in InsightsView); share basis differs (grand-total vs max-normalized vs precomputed coverage). **Canonical:** `shareBarWidth(fraction, minPct=1.5)` or `<MiniShareBar>`. **Extraction:** safe, trivial. **Confidence:** high, minor.

### Mapping 252 (thin, minor) — zoom-fraction → percent readout pair
- `pages/automation/editor/components/WorkflowCanvas.tsx:349` vs `marketing/forms/components/LogoCropModal.tsx:220` — identical `Math.round(zoom*100)%`; different zoom mechanics. **Canonical:** `formatZoomPercent(zoom)` only if a third zoom UI appears. **Confidence:** high (duplication), low value.

## Frontend pass-61 excluded (for the record)
- `financial-analytics/components/{SankeyTab.tsx:47–57, CohortReferralTab.tsx:288–295}` `percentage`/`positive` pair — token-identical but CohortReferralTab is dead-lane (re-confirms pass-41 note); becomes an immediate family if DashboardView is revived.

### Mapping 246 — Table bulk-selection action bar ("N selected" + pill buttons + Clear), 10 sites, 9 files, no shared component
- Conditional header-slot bar when `selected.length > 0`: "{n} selected" span + lavender-pill (`bg-[#e8ddf6] hover:bg-[#d7c9ef]`) action buttons + Clear in a `flex items-center gap-2 min-h-[36px]` wrapper.
- **Sites:** `components/compact/{IntakeTable.tsx:1147–1170, ActiveTable.tsx:977–994}` (token-identical; private `filledActionButtonClasses` re-declared byte-identically in 5 files incl. OpenTable:69, NurtureTable:81, CustomJourneyTable:151); `OpenTable.tsx:1020–1054`; `CustomJourneyTable.tsx:845–907` (comment: "Parity with IntakeTable's 'X selected | Move | Clear' toolbar"); `NurtureTable.tsx:111–135` (only props-driven extraction, in-file); `pages/recalls/components/RecallsTable.tsx:497–575` (comment: "chrome lifted verbatim"; inline class re-rolls); `components/ContactsTable.tsx:682–710`; `components/ArchiveTable.tsx:345–361`; `pages/Stock.tsx:3920–3966`; shared-but-orphaned `components/journey-board/BoardMultiSelectBar.tsx:19–28`.
- **Selection-semantics drift under the same bar:** OpenTable clears selection on page change (:790), Intake/Active/Nurture/Recalls retain off-page ids, ContactsTable prunes ids no longer on page on refetch (:489), Stock supports ids beyond the visible page.
- **Canonical:** shared `<BulkSelectionBar selectedCount actions onClear retainAcrossPages>` (NurtureTable's props-driven subcomponent as baseline). **Extraction:** safe; surface the retention divergence as a prop. **Confidence:** high.

### Mapping 242 — "Synchronous blank-tab priming" popup-blocker workaround for async opens, pair
- `pages/compliance/hooks/useCompliance.ts:3118–3150` `downloadSigningEvidence` (`window.open('', '_blank')` → null opener → fetch → navigate retained tab → close+dedicated popup-blocked error on failure) vs `components/modals/treatmentPlan/TreatmentPlanSuccessModal.tsx:74–94` (`window.open("about:blank")` → OTP POST → navigate retained tab; **fallback `window.open(url, "_blank", "noopener")` after the await — the exact sequence compliance's comment documents as reliably blocked**; nulls opener only on success). Same blank-open → retained-ref → close-on-failure state machine. **Canonical:** `openTabForAsync(fetch, {blockedMessage})` in lib/hooks; compliance variant as baseline. **Extraction:** safe. **Confidence:** high.

### Mapping 243 (thin, minor) — `VITE_CURRENT_ENV` webhook-base-URL env read with hardcoded fallback, 3 files
- `components/settings/sms-phone-config/ProvisionPhoneModal.tsx:47` (**live typo'd fallback** `https://app.pathway,dental/...`), `PracticeConfigModal.tsx:151` (correct), `pages/admin/practice-management/components/PhoneConfigSection.tsx:109–110` (correct). None route through `config/environment.ts`. **Canonical:** export `SMS_WEBHOOK_BASE_URL` from `config/environment.ts`. **Extraction:** trivial. **Confidence:** high.

## Frontend pass-58 mapped-family extensions
- **M31 ext:** `pages/settings/EmailService.tsx:375` and `pages/admin/email-service-components/OverviewTab.tsx:81` use `VITE_EMAIL_SERVICE_URL || "http://localhost:9000"` display fallbacks diverging from `config/environment.ts:78` (`https://mail.pathway.dental`).
- **M9 ext:** `components/dialogs/SubscriptionDialog.tsx:22` — third divergent admin predicate variant (`isAdmin || userType === "practice_admin" || "admin"`).

### Mapping 240 (thin, minor) — "Non-negative numeric field commit" clamp idiom
- Empty→sentinel + `Number`/`parseInt` + `Math.max(0,…)` commit idiom, scattered: token-identical pair `pages/journeys/administration/AutomationSection.tsx:869` ↔ `recalls/administration/AutomationSection.tsx:607` (both M15 clone-pair files; `onFocus select()` + zero-hidden scaffold); `components/invoices/FinanceAdminTab.tsx:835,1647` (empty→0, sibling NaN-guard); `pages/settings/components/practice/OnlineBookingServiceDialog.tsx:815,834` (empty→**null**); `pages/hr/Admin.tsx:2556` (parseInt + 120 upper clamp); `ConfirmationSequenceForm.tsx:212` (no empty sentinel); `DentallyConfig.tsx:303` (upper clamp 7). **Canonical:** `parseNonNegativeField(raw, {empty: 'null'|'zero', max?})` beside M146's `parseAmount`. **Confidence:** high (duplication), medium (consolidation wanted?).

### Mapping 241 (thin, minor) — "Discard in-progress work?" confirm implemented 3× with 3 different mechanisms
- `components/journeys/AddPatientDialog.tsx:1769–1791` (inline footer confirm, deliberately avoids stacked dialog; Escape/outside dirty path :1147–1163); `pages/hr/Admin.tsx:3172–3200` (separate Dialog on `pendingUnitSwitch`, + `pendingBasisChange` sibling :3136); `components/invoices/AssociatePayView.tsx:3491–3530` (AlertDialog on `pendingNavAction` with per-dentist draft list + `dirtySaveFailedIds` failure surfacing). Only AssociatePayView offers save-from-dialog. Distinct from M183 (beforeunload) and M202 (row-delete confirms). **Canonical:** `DiscardConfirm` slot or `useDeferredDirtyAction(execute)`; alternatively document as intentional M183 variants. **Confidence:** high (existence), medium (intent).

### Mapping 239 — Training-status → icon/color config re-derived in the same feature, bypassing the module's own exported canonical
- **Canonical (bypassed):** `pages/compliance/constants.ts:80–108` `TRAINING_STATUS_COLORS`/`TRAINING_STATUS_LABELS` (used only by ComplianceDocumentsDialog/TrainingFilters). **Re-rolls:** `pages/compliance/components/training/TrainingMatrixCell.tsx:11–33` (private `STATUS_BG_COLORS` + `STATUS_ICONS`), `TrainingMatrix.tsx:171–177` (inline `statusConfig` — drift: `bg-green-50` vs `bg-green-100`, unknown-status fallback the Cell lacks); structural mirror `pages/hr/components/records/HRRecordsMatrix.tsx:179–185` (`HRRecordStatus` vocabulary, label "Expiring" vs "Expiring Soon").
- **Canonical:** `TRAINING_STATUS_ICONS` in compliance constants + parallel `HR_RECORD_STATUS_DISPLAY` in hr constants. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 240 — Comment "deletable by current user" predicate — invoices/payslips/ledger members (M224-family extension)
- `components/invoices/InvoiceCommentsSection.tsx:65` and `PayslipDraftCommentsSection.tsx:73` (byte-identical predicate + near-identical delete button), `components/patients/workspace/finance/LedgerRow.tsx:102` (raw snake_case `author_id` variant with undefined-guard), plus mapped labs/tasks copies. All strictly author-only except labs (adds `canManageLabs`). **Canonical:** `canDeleteComment(comment, user)` folded into M224's `CommentThreadPanel.canDelete`. **Extraction:** trivial. **Confidence:** high (duplication), medium (LedgerRow normalization).

## Frontend pass-56 doc-hygiene note
- Map line 543 (M105/record-types entry) cites `components/records/HRRecordsMatrix.tsx:117` — stale path; the live file is `pages/hr/components/records/HRRecordsMatrix.tsx:117`.

### Mapping 236 — Date-range picker auto-clamp ("selecting one endpoint pushes/pulls the other"), 4 files, 6 sites
- On start change `if (new > end) setEndDate(new)`; on end change the reciprocal (in 2 of 4): `components/invoices/SupplierPayView.tsx:372–386` (both directions + `setDatePreset('custom')`); `pages/compliance/components/documents/SigningBatchesTab.tsx:567,585` (both directions + `setPage(1)`); `pages/hr/components/reporting/{SicknessAnalyticsReport.tsx:114–118, MaternityPaternityReport.tsx:102–106}` (**start→end clamp only — no reciprocal on end change, so end-before-start is accepted**). Non-clamping validators of the same rule (FilterModal swap+warn :251,563–566; hr/Leave.tsx:1170,1280; BulkLeaveDialog.tsx:273,318) are adjacent, not members. **Canonical:** `useDateRangeState({mode: 'clamp'|'swap'|'validate'})`. **Extraction:** safe. **Confidence:** high.

### Mapping 237 — Hook-side cursor-pagination "load more" append machine, 4 hooks (distinct from M163's UI triggers)
- Map-keyed pair: `hooks/useContactThread.ts:83–88` ↔ `useTeamChat.ts:633–638` (`hasNextRef.get(id)` guard + `pageRef.get(id) ?? 1` + 1). Flat pair: `hooks/useMessaging.ts:166–176 loadMoreConversations` (**has `loadingMoreRef` in-flight guard**) vs `hooks/useContactInbox.ts:141–146 loadMore` (**no ref guard — rapid calls can double-fetch**). **Canonical:** `useCursorPagination(fetchPage)` / `makeLoadMore({pageRef, hasNextRef, fetch})`. **Extraction:** safe. **Confidence:** medium-high.

### Mapping 238 (minor, M3-family) — Whole-pound GBP aggregate formatter, journey-board pair
- `components/journey-board/JourneyBoardColumn.tsx:11–12 formatAggregate` ↔ `JourneyBoardCardItem.tsx:34–35 formatValue` — **character-identical** `` `£${value.toLocaleString("en-GB", {maximumFractionDigits: 0})}` `` (used :90 / :274). Same-output variant `components/patients/patient-panel/PatientPanelRecallSections.tsx:66–67` `gbp`. All bypass canonical `formatCurrencyAbbreviated` (`types/invoice.ts:490–494`, whole-pound branch produces the identical string). **Fix:** import the canonical or export `formatGBPWhole`. **Extraction:** trivial. **Confidence:** high.

### Mapping 230 — "Local useState mirrors of a record prop, re-hydrated via useEffect" derived-state anti-pattern, 6+ files
- Edit dialogs declare per-field `useState` mirrors of a record prop, re-hydrated by a `useEffect` when the record/open changes: `components/journey-board/JourneyBoardEditDialog.tsx:74–89` (6 mirrors, deps `[card]` — **no open guard: mid-edit refetch wipes in-progress edits**); `pages/admin/GlobalPhrases.tsx:111–124` ↔ `GlobalAutoTemplates.tsx:65–73` (byte-identical comments, unguarded); `pages/hr/components/rota/RoomForm.tsx:87–116` (most defensive: null→defaults else-branch + clamping); `pages/settings/components/consent/ConsentTemplateEditor.tsx:90–100`; `pages/stock/StockItemSheet.tsx:1949–1959` (gated on isEditing); variant `pages/admin/ComplianceLearningModules.tsx:377–385` (open-gated rest-spread).
- **Canonical:** `useRecordFormFields(record, buildFields, {gate})` or `<EditDialog key={record.id}>` remount idiom (RoomForm variant as baseline). **Extraction:** safe. **Confidence:** high.

### Mapping 231 — `pendingHighlightId` deep-link consume scaffold, 6 live files (resolves pass-29 watch)
- `useState` pending id → effect: guard → `findIndex` in filtered list → `setCurrentPage(floor(index/pageSize)+1)` → promote to `highlightedId` → clear: `pages/compliance/components/documents/{CoshhTab.tsx:104,200–208, RiskAssessmentsTab.tsx:166,278–296, MeetingMinutesTab.tsx:150,217–224}`, `compliance/components/training/TrainingMatrix.tsx:353,368–375`, `policies/PoliciesLibrary.tsx:460,474–483`, cross-domain `pages/hr/components/records/EmployeeProfileDrawer.tsx:447–455` (comment admits copying PoliciesLibrary/TrainingMatrix). **Canonical:** `useHighlightDeepLink({id, items, pageSize, onPage, onHighlight})` in compliance shared. **Extraction:** safe. **Confidence:** high.

### Mapping 232 (thin, minor) — channel-toggle defaults synced from `hasEmail`/`hasPhone` via useEffect, pair
- `components/modals/treatmentPlan/SendToPatientMock.tsx:46–48,54` vs `components/consent/ConsentSendDialog.tsx:66–67,75` — same `emailOn`/`smsOn` mirrors with different gating deps. **Canonical:** `useChannelToggles({hasEmail, hasPhone, open})` or key-remount. **Confidence:** high, minor.

## Frontend pass-54 extension notes
- **M14/M119 debounce ext:** `pages/admin/MasterCatalogue.tsx:111,161` and `components/compliance/components/calendar/LinkedRecordPicker.tsx:43–44` — private debounced state + setTimeout effect members.

### Mapping 228 — "Creatable" free-text combobox (type-to-filter + free-text fallback + click-outside close), 3 live implementations, one claiming the primitive doesn't exist
- `components/invoices/SupplierCombobox.tsx:26–52,67–79` (cap 10, 150ms blur-close); `pages/compliance/components/shared/DentistTypeaheadField.tsx:46–79,91–112` (no cap, "no matching staff — use typed name" empty state); `pages/compliance/components/shared/CreatableDurationField.tsx:17–19` — header comment claims *"No existing creatable-combobox primitive in this repo"* and builds a fourth on `ui/command`+`ui/popover`. Shared failure mode: none syncs keyboard highlight (Enter submits the form, not the suggestion — gap M217 solved elsewhere). **Canonical:** `CreatableCombobox({options, value, onChange, emptyText?, suggestionCap})` in `components/shared/` beside `StaffCombobox`. **Extraction:** safe. **Confidence:** high.

### Mapping 229 (thin, minor) — Segmented step-progress bar, 3 live implementations beyond the mapped circle-rail family (M22)
- `pages/auth/components/ProgressBar.tsx:6–17` (6 consumers; `i < currentStep` fill; `#6C5ACF`); `pages/PublicMedicalHistoryFormPage.tsx:297–305` `StepDots` (`i <= step` fill — segment 0 purple on load; `#846ce0`); `pages/compliance/components/learning/ComplianceLearning.tsx:449–459` (3-state ladder with distinct current mid-tone). **Canonical:** `StepSegments({current, total, accent?, showCurrent?})` in `components/ui/` beside M22's target. **Confidence:** high, minor.

### Mapping 227 — Parallel "pathway steps editor" stacks in `pages/settings/components/pathway/` (both live, two generations shipped side-by-side)
- Handler clones: `PathwaySteps.tsx:36–140` vs `ImprovedPathwaySteps.tsx:47–130+` (addStep/removeStep/updateStep/addQuestion/removeQuestion/updateQuestion/addAnswer/removeAnswer — incl. identical `fieldOrValue/maybeValue` overload shim). Card pairs: `PathwayStepCard` vs `ImprovedPathwayStepCard`, `PathwayQuestionCard` (154 ln) vs `ImprovedPathwayQuestionCard` (411 ln). `PathwayQuestion`/`PathwayStep` interfaces re-declared in **all six files**. Both stacks live+routed (settings/Notes.tsx:9,315 and NoteTemplateForm/CombinedTemplateForm via admin routes:92,97).
- **Behavioral:** the stacks produce different question payloads (`answerType`, `nestedQuestions`, `defaultAnswerIndex` only in Improved) — data authored in one editor shape isn't editable with the other's affordances. **Canonical:** one `PathwayStepsEditor` with `advanced?: {nested, defaultAnswer}` capability flag + shared types module. **Extraction:** safe. **Confidence:** high.

### Mapping 228 (thin, minor) — Hidden-marker Lexical `DecoratorNode` pair, consumers double-branch everywhere
- `components/notes/{MarkerNode.tsx:11–60, PathwayMarkerNode.tsx:11–60}` — identical hidden-DOM mechanics (`data-marker` vs `data-pathway-marker`); every consumer pays a two-branch tax (`MarkdownPlugin.tsx:29,55–58,426/449,473/478,490,699`; `MarkerGuardPlugin.tsx:4–5`; `InlineLexicalEditor.tsx:23–24,90–91`). **Canonical:** `HiddenMarkerNode({dataAttr, payload})` base. **Confidence:** medium-high, low urgency.

## Frontend pass-52 extensions
- **M3 ext:** private `formatMoney` (`` £${value.toFixed(2)} ``) pair — `pages/hr/PayRates.tsx:50–52`, `hr/components/timesheets/TimesheetWeeklySummary.tsx:28–30`; Intl variant `PracticeTreatmentsView.tsx:154–159`.
- **Dead:** `components/modals/components/ContentEditor_backup.tsx` — zero importers (already noted; confirmed).

### Mapping 225 — `FileReader.readAsDataURL` file→base64 data-URL reader, 7 live copies (two competing promise canonicals)
- **Canonicals:** `lib/patientGuideDocuments.ts:162–168` `fileToDataUrl` (exported, 1 consumer); `lib/imageOptimisation.ts:76–84` `blobToDataUrl` (private, token-equivalent). **Callback re-rolls (all live):** `components/InlineLexicalEditor.tsx:119–125` (CustomEvent dispatch, **no onerror**); `components/modals/components/ContentEditor.tsx:386–400` (CustomEvent + onerror); `components/notes/plugins/ImageUploadPlugin.tsx:221–230` (+CustomEvent consumer :260); `hooks/useImageHandling.ts:63–67,93–97` (in-file pair, no onerror); `marketing/forms/pages/BuilderPage.tsx:1660–1665` (typeof-string guard only). **Risk:** failure silently leaves state unset in 4 of 7. **Canonical:** promote `fileToDataUrl` to utils/fileUtils. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 226 (thin, minor) — M43 extension: `formatFileSize` gains two uncited members
- `components/teamChat/TeamChatComposer.tsx:16–22 formatBytes` (KB rounding drift) and **exported** `lib/patientGuideDocuments.ts:117–125 formatFileSize` (consumed by treatment-verification/treatmentPlan documents tabs) — an exported second canonical beside M43's `utils/fileUtils.ts` target.

## Frontend pass-51 dead-lane annotations (status of families touched by pass-50 corrections)
- **M133 fully dead:** all three cited files (`TaskAddressesTab`, `InvoiceAddressesTab`, `WebFormForwardingTab`) have zero importers; annotate as retired, not re-canonicalized.
- **M64, M93, M31-ext survive with dead members removed:** M64 loses `dialogs.tsx:150`, `InboundParseSettings.tsx:30` + the dead trio's no-toast copies; M93 is now a clean live pair (`RegisterDomainDialog.tsx:30–34` + `AddDomainForm.tsx:31–35`); M31-ext loses `InboundParseSettings.tsx` ×2.
- **M6's local `getErrorMessage` trio is now a pair:** dead `TaskAddressesTab.tsx:46`; live `SupplierMappingsTab.tsx:64`, `EmailAddressesTab.tsx:40`, + `usePatientPanelController.ts:375`.

### Mapping 224 — Entity-comments panel UI clone, labs ↔ tasks (same data shape, both live)
- `components/labs/LabCaseCommentsPanel.tsx:100–140,254–287` vs `components/tasks/TaskCommentsPanel.tsx:96–136,250–283` — normalized diff ~64 of 570 lines; comment-item row token-identical modulo `sourceType`/delete-predicate. Same M178 query-hook family beneath. Divergence: labs gates with `hasFeature("manage_labs")` + manage-level delete; tasks author-only-delete. **Canonical:** `CommentThreadPanel({comments, isLoading, error, canDelete, onCreate, onDelete, sourceType, variant})` — merge with the proposed `CommentThreadSection` so all four comment surfaces (labs, tasks, invoices, payslips) share one renderer. **Extraction:** safe. **Confidence:** high.

## Frontend pass-50 dead-lane corrections (invalidate/annotate map entries)
- **`pages/settings/email-service/` directory is dead** — zero external importers (EmailService.tsx:41 imports from `@/components/email-service`; AdminEmailService imports from `./email-service-components`). Whole-directory clone of live `components/email-service/`. Deletion candidates; also removes the apparent cross-file "No domains registered yet"/"No DNS records available" duplicate pairs (dead-alive). Mappings citing this dir (M31 env reads, M64 clipboards, M93) should be annotated.
- **M133's trio is dead:** `pages/settings/components/integrations/{TaskAddressesTab,InvoiceAddressesTab,WebFormForwardingTab}.tsx` referenced only by an unimported barrel (`integrations/index.ts` — zero importers). Live surface is `EmailAddressesTab.tsx` (via IntegrationsView.tsx:5), which privately consolidated the address-row renderer (`renderAddressRow`, :227–290).

### Mapping 218 — Settings-section header "Cancel/Save" bar gated on dirty-comparison, 6 copies across 4 files
- Extracted components (identical modulo title/subtitle): `pages/settings/components/account/AccountSettingsView.tsx:113–133`, `preferences/PreferencesView.tsx:39–59`, `practice/PracticeSettingsHeader.tsx:26–47`. Inline re-rolls inside `pages/Settings.tsx` bypassing all three: `:1404–1431` (Communication), `:1590–1617` (Diary), `:1757–1784` (Patient Guides). All share the grep-unique class fingerprint `disabled:bg-[#faf9fe]…` (13 hits, all in this family). Dirty-comparison itself is canonical (`hooks/useChangeDetection.ts:41–68`). **Canonical:** one `SettingsSectionHeader({...})`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 219 — Usage-rates table + global/practice override CRUD trio in `sms-phone-config/`
- `components/settings/sms-phone-config/{SmsRatesSection.tsx:75–171, SttRatesSection.tsx:60–173, CallRatesSection.tsx:81–133}` — identical 4-column thead, click-row edit dialog, global save + practice-override save/reset with same toast scaffold (`"Failed to save rate"`). CallRatesSection diverges (different endpoint, no overrides). **Canonical:** config-driven `RatesSection({services, endpoint, supportsOverrides})`. **Extraction:** safe. **Confidence:** high.

### Mapping 220 (thin, minor) — `['Mon','Tue',…]` week-header constant re-declared beside its export
- **Canonical:** `pages/hr/utils/dateUtils.ts:11 WEEK_DAYS`. **Bypasses:** `pages/hr/components/schedules/SchedulePatternsList.tsx:42 DAY_LABELS`, `rota/RotaMonthGrid.tsx:15 DAY_HEADERS`, `pages/hr/Rotas.tsx:4999` inline; `components/calendar/CalendarGrid.tsx:25–26` re-rolls `WEEK_DAYS` + `WEEK_DAYS_SHORT` while `HRCalendarGrid.tsx:13` imports the canonical yet re-rolls its own identical `WEEK_DAYS_SHORT`. **Canonical:** single module exporting both variants. **Extraction:** trivial. **Confidence:** high, minor.

## Frontend pass-49 extension
- **M91 ext:** admin/recalls metrics tables' identical Column/Filter/Metric/Settings header row — another facet of the M91 file pair.

### Mapping 214 — Swipe-to-reveal row-actions gesture state machine, 3 live copies with un-backported fixes
- `components/labs/SwipeableLabCatalogueListItem.tsx:20–82,151–155` (threshold 40/action 80; multi-open; **no direction lock** — vertical scroll drags the row); `components/assets/SwipeableAssetListItem.tsx:13,22–80,~215` (derived threshold); `components/stock/SwipeableStockListItem.tsx:36–126,172–176` (**has direction-lock + parent-controlled single-open** — improvements not back-ported). Same refs trio, clamp/snap/handleRowClick bodies. **Canonical:** `useSwipeActions({actionWidth, threshold, directionLock, open, onOpenChange})` or shared `SwipeableRowListItem` (stock variant as baseline). **Extraction:** safe. **Confidence:** high.

### Mapping 215 — `validateFile(file): string | null` upload guard, 5 copies
- `components/assets/{AssetDocumentManager.tsx:88–100, LogServiceDialog.tsx:100–110, AssetPhotoManager.tsx:53–64, AddAssetDialog.tsx:130–141 (token-identical to AssetPhotoManager), invoices/InvoiceUploadDialog.tsx:42–56}` — extension allowlist + size-cap ladder with the same `File too large. Maximum size is ${n}MB.` template. Three extension-check styles; MIME vs extension-only vs both. **Canonical:** `validateUploadFile(file, {extensions, types?, maxMb})` in `utils/fileUtils.ts` (beside M43). **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 216 (thin, minor) — page-size input commit + Enter-keydown scaffold bypassing `usePersistedPageSize`
- Canonical `hooks/usePersistedPageSize.ts:46–88` (clamp 5–100, persistence). **Bypasses:** `components/stock/ImplantsTable.tsx:139–153` (lower bound `> 0`, no persistence); `components/Library/TemplatesTable.tsx:207–221` (token-identical to ImplantsTable); `pages/recalls/components/RecallsTable.tsx:474–486` + inline onKeyDown :1211 (**no upper bound at all**). **Canonical:** extend `usePersistedPageSize({persist?, onChange?, onResetPage?})`. **Extraction:** safe. **Confidence:** high.

### Mapping 217 (thin, minor) — arrow-key wraparound list-highlight + Enter-commit handler, pair
- `pages/settings/components/PlaceholderAutocomplete.tsx:188–210` (adds Tab-commit, `-1` unselected) vs `components/inbox/v2/TemplatePopover.tsx:205–223` (adds category axis). **Canonical:** `useListKeyboardNav({itemCount, onCommit, onClose})`. **Confidence:** high, low urgency.

### Mapping 208 — Column-sort toggle state machine (`handleSort`), 15+ live copies with live drift
- **Operation:** "same column → flip direction; else set column + default direction" table-sort machine re-declared per table component.
- **Implementations:** `components/stock/{StockTable.tsx:51–58, ImplantsTable.tsx:108–115}`; `pages/Labs.tsx:777–784`; `pages/Stock.tsx:2284–2291`; `pages/callagent/CallAgentView.tsx:97–104` (**default desc**); `pages/hr/leave/PlanningView.tsx:95–102`; `pages/recalls/RecallsPage.tsx:208–216` (**only copy resetting `setPage(1)`** — the rest have a stale-page bug class); `financial-analytics/components/MarginTab.tsx:437–444` (default desc + inverted toggle); `components/assets/AssetsContent.tsx:327–334` (`sortColumn` naming variant); `components/ArchiveTable.tsx:212–219`; `components/compact/{IntakeTable.tsx:1051–1059, NurtureTable.tsx:1011–1019, CustomJourneyTable.tsx:373–380 (functional setState)}`; hook form `hooks/useTeamSorting.ts:24–31`; journeys trio Intake/OpenPlans/ActiveTreatments :141–159 (M55 territory); `pages/Invoices.tsx:392–400` (server `ordering` sync variant).
- **Canonical:** `useSortToggle({defaultDirection})` with optional `onSortChange` (fold into M16's table lib). **Extraction:** safe. **Confidence:** high.

### Mapping 209 — Message-contact archive/restore pair + drifted variant
- `pages/InboxPage.tsx:782–800` ↔ `pages/messages/prototype/FamilyGroupPrototypePage.tsx:786–805` — byte-identical archive/restore handlers, **no error handling** (failed POST leaves contact removed). Drifted variant: `components/patients/patient-panel/PatientInboxTab.tsx:339–366` (`res.ok` check + toasts; flip-flag vs remove-from-list). **Canonical:** `useContactArchiveRestore({onArchived})`; ok-check as baseline. **Extraction:** safe. **Confidence:** high.

### Mapping 210 — `HistoryStatusBadge` medical-history status ladder: exported canonical bypassed in its own feature
- **Canonical (exported):** `components/patients/workspace/medical/MedicalHistoryDetailPanel.tsx:188–198`. **Bypass:** `components/patients/workspace/PatientMedicalTab.tsx:423–433` — identical 4-branch ladder + private `Badge({tone})` helper encoding the same tone→class map, **in a file that already imports MedicalHistoryDetailPanel at :18**. Byte-equivalent output; structure only. **Extraction:** trivial. **Confidence:** high.

### Mapping 211 (minor) — `getStaffAvatar` staff-avatar lookup helper, token-identical compliance trio
- `pages/compliance/components/documents/{RiskAssessmentsTab.tsx:118–125, MeetingMinutesTab.tsx:132–139 (only drift: no fallback param), PoliciesLibrary.tsx:199–206}` — identical comment + `staff.find(s => s.id === id)?.avatar` body; all live. **Canonical:** `getStaffAvatar(staff, id, fallback?)` in `pages/compliance/components/shared/`. **Extraction:** trivial. **Confidence:** high.

### Mapping 212 (minor) — compact-table `PriorityBadge` popover-editor component, 4 copies + 1 read-only clone
- Identical `getPriorityStyle`/`getDotColor` maps (High red / Medium amber / Low green / gray) + popover pill + Low/Medium/High select: `components/compact/{ActiveTable.tsx:885, OpenTable.tsx:935, NurtureTable.tsx:163, JourneyMobileCard.tsx:204 (adds read-only branch, `<button>` trigger)}; read-only clone `IntakeTable.tsx:905 getPriorityBadge`. Distinct from M157 (handler) and the pass-45-corrected TableRow pair. **Canonical:** shared `PriorityBadge({priority, entityId, onPriorityChange?})` in `components/compact/`. **Extraction:** safe. **Confidence:** high.

## Frontend pass-48 clean lenses
- Mentions: canonical `components/shared/mentions/` used everywhere. Avatars: M23/M42 territory. Tooltips: Recharts renderers diverge per chart (intentional). Badge status-map one-offs: excluded M5 category.

### Mapping 208 — "Fetch treatment-categories-with-procedures + flatten" effect, 10+ live sites (0 prior citations)
- Core token-identical pair: `pages/automation/editor/components/action-configs/create-record/{IntakeFields.tsx:27–61, NurtureFields.tsx:29–63}` (normalized diff = 33 lines of identifier renames; identical private `TreatmentCategory` interface). Variants with drifted projections: `TreatmentPlanFields.tsx:51–91` (richer flatten, no dedupe/sort); `components/TreatmentSelection.tsx:44–58`; `components/journeys/{AddPatientDialog.tsx:553–574, MovePatientDialog.tsx:~556, BulkEditDialog.tsx:~126}`; `components/compact/{AddPatientModal.tsx:643, FilterModal.tsx:131}`; `components/intake/ManualIntakeForm.tsx:165`; `contexts/TreatmentContext.tsx:494,498`; `hooks/useTreatmentsCaching.ts:109` (raw-fetch M1-family lane, de-facto hook bypassed by all others). Distinct endpoint from M121. **Canonical:** `useTreatmentCategoryOptions({project})`. **Extraction:** safe. **Confidence:** high.

### Mapping 209 — Workflow-editor API↔editor node/trigger-config codec, token-identical pair
- `pages/automation/editor/WorkflowEditor.tsx:61–78,262–282,398–434` vs `pages/admin/global-workflows/GlobalWorkflowGraphEditor.tsx:58–74,215–236,378` — normalized diff = endpoint namespace only (`workflows.*` vs `globalWorkflows.*`); includes the byte-identical 7-field `trigger_config` payload. **Risk:** a new `trigger_config` field added in one editor silently drops on the other's save; `NodeConfigPanel.tsx:60–145` depends on the emitted format. **Canonical:** `lib/workflowNodeCodec.ts`. **Extraction:** safe. **Confidence:** high.

## Frontend pass-47 watch notes (with a live copy-bug)
- `pages/automation/editor/components/action-configs/TriggerConfig.tsx:273–289` vs `:305–319` — in-file duplicated cron builder; **the second copy's hours branch at :311 emits a 6-field cron `0 */N * * * *`** that `NodeConfigPanel.tsx:74`'s own 5-field parser cannot round-trip (M49 family's type catalog is adjacent). Fix the second branch.
- **M123 ext:** `components/LetterToolbar.tsx:128–151` — third practice-logo fetch effect; fold into `usePracticeLogo()`.
- **M137-family member:** `pages/diary/schedules/utils.ts:25 timeToMinutes` (null-fallback); `:203,382` hand-rolled `yyyy-MM-dd` = M4 family.

### Mapping 206 — Approved-leave range → per-date `Record<string, Set<string>>` expansion, pair
- `pages/DiaryPage.tsx:939–957` (reconciles `lr.staffId` → practitioner id via `practitionerStaffIdByPractitionerId` before bucketing) vs `hooks/useBookingScheduleData.ts:42–56` (stores **raw `lr.staffId`**, comment admits the "same IDs in most setups" id-space assumption — the gap documented in `lib/diaryPractitionerReconciliation.ts:12,182`). Both live (diary booking modal vs patient-workspace booking modal). **Canonical:** `buildPractitionersOnLeaveByDate(leave, resolveStaffId)` beside `diaryPractitionerReconciliation.ts`. **Extraction:** safe. **Confidence:** high.

### Mapping 207 (thin, minor) — hand-rolled week-start (Monday) computation bypassing date-fns idiom
- `pages/booking/hooks/usePublicBooking.ts:57–63` private `startOfWeek` (`(day + 6) % 7`); `marketing/forms/pages/SubmissionsPage.tsx:299–300` inline ms-arithmetic (`startWeek = startToday - ((day+6)%7) * 86_400_000` — **can land off-local-midnight by ±1h on DST weeks**). Meanwhile the established idiom is date-fns `startOfWeek(d, {weekStartsOn: 1})` in 7 sites (CalendarGrid, hr utils/rota/Rotas/useTimesheets, ChecklistComplianceView, CopyRotaPopover). **Canonical:** date-fns call or shared `weekStart(d)` in `utils/dateUtils.ts` (adjacent to M51). **Extraction:** trivial. **Confidence:** medium-high, minor.

### Mapping 204 — Audit-log React-Query hook scaffold (keys factory + filter QS ladder + grouped/raw dual useQuery), triplicate
- `hooks/usePatientAudit.ts:22–74`, `useHRAudit.ts:18–58`, `useFinanceAudit.ts:20–62` — token-identical keys factories modulo namespace; identical 9-condition `URLSearchParams` filter ladder (`entity_type`, `action_family`, `actor_id`, `from`, `to`, `search`, `grouped`, `page`, `page_size`); grouped/raw dual `useQuery` over `Paginated*Response<T>` envelope. All three live (consumers already mapped under M110/M198 for presentational clones only).
- **Divergence:** patient uses `grouped === 0` + `source_system`; others boolean `grouped` + HR adds `practice_id`; only finance forwards React Query `signal` (cancellation). **Canonical:** `useEntityAuditLog({keysNamespace, endpoint, scopeId?, extraFilters, signal?})` or shared `buildAuditQs(filters)`. **Extraction:** safe. **Confidence:** high.

### Mapping 205 (thin, minor) — hand-rolled transient-retry-with-linear-backoff fetch loops, pair
- `lib/assistantApi.ts:89,107–147` (retry on ≥500 + network catch; returns `{error, status}` envelope) vs `hooks/useJourneyBoard.ts:34,322–346` (broader retryable predicate incl. 408/429 + TypeError; throws Error). Private `delay` defs in each (M68). **Canonical:** `withTransientRetry(fn, {attempts, baseDelayMs, isRetryable})` in lib. **Confidence:** medium.

## Frontend pass-44 clean lenses
- aria-live/announcer: inline one-offs only. Cache-key `*Keys` factories: each domain's single owner. Derived-status reductions: single implementations. Retry sweep: only the two above + canonical App.tsx React-Query config.

### Mapping 201 — "Type the name to confirm" permanent-delete AlertDialog, token-near-identical pair in one directory
- `components/assets/AssetsContent.tsx:190–192,968–1021` vs `AssetPanel.tsx:189–191,1140–1195` — identical state trio, identical red-banner copy (byte-identical sentence about service history/faults/photos/documents), identical `font-mono` typed confirm (AssetsContent:987 / AssetPanel:1160), identical disabled predicate, toasts, red button. **Divergence:** entity-in-state vs boolean open; panel closes the whole panel on success. **Canonical:** shared `<PermanentDeleteAssetDialog/>` or `useTypedConfirmDelete(fn)` in `components/assets/`. **Extraction:** safe. **Confidence:** high.

### Mapping 202 (thin, minor) — inline row-level "Remove X?" confirm via pending-id state, pair
- `components/teamChat/TeamChatMembersPanel.tsx:46` (`confirmRemoveId`, string ids) vs `components/patients/patient-panel/PatientPanelOverviewTab.tsx:163` (`clearConfirmPersonId`, numbers). **Canonical:** `useInlineRowConfirm()` / `<InlineConfirm>` primitive. **Confidence:** medium, minor.

### Mapping 203 (thin, minor) — Set-based multi-expand toggle state, 3 copies + single-open siblings
- `components/invoices/SupplierPayView.tsx:545–550`, `components/patients/patient-panel/PatientActivityFeed.tsx:109–117`, `components/chat/NotesList.tsx:77–81` (Set-toggle); single-open siblings: `dashboard/digest/DigestGroup.tsx:16,46`, `PatientMedicalTab.tsx:53,232`, `ActionsWorkflowContent.tsx:28,32`, `DataQualityIssuesScreen.tsx:133,364`. **Canonical:** `useToggleSet({multiple})`. **Confidence:** medium, low urgency (one-line state idiom).

### Mapping 197 — `formatMessageTime` "isToday bubble clock": exported canonical bypassed in its own directory + cross-domain copy
- **Canonical:** `components/inbox/v2/automationMeta.ts:100` (exported; header says the automated row needs the same clock). **Bypass 1:** `components/inbox/v2/MessageBubble.tsx:14` — character-identical re-declaration in the same directory. **Bypass 2:** `pages/admin/email-service-components/TempMailboxMessageBubble.tsx:44` (admin temp-mailbox). **Fourth exported copy:** `pages/messages/prototype/automationMeta.ts` (M42 lane). **Canonical:** TempMailbox imports the v2 export. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 198 — M110 extension: `formatTimestamp(iso)` date-fns audit-log formatter, 4 token-identical copies
- `components/invoices/FinanceAuditView.tsx:137`, `pages/hr/HRAuditLog.tsx:121`, `components/patients/workspace/PatientAuditLogTab.tsx:104`, **new member** `components/stock/InventoryAuditView.tsx:90` (byte-identical; live via pages/Stock.tsx — the M110 audit-UI pattern escaped into a fourth domain). **Canonical:** `formatAuditTimestamp` beside M110's presentational set. **Confidence:** high, minor.

### Mapping 199 (minor) — `formatLastSeen` presence ladder, teamChat clone pair
- `components/teamChat/{TeamChatMembersPanel.tsx:20, TeamChatGroupInfoDialog.tsx:22}` — identical null→"Offline"/minutes/hours/days ladder (last-branch formatting delta only). **Canonical:** `formatLastSeen(lastSeenAt)` export in `components/teamChat/`. **Confidence:** high, minor.

### Mapping 200 (minor) — `LabelCategory`/`Label` API entity types declared twice in hooks
- `hooks/useLabelsHandler.ts:5,13` (private 3-field subsets) vs `hooks/useLabelManagement.ts:7` (exported full 13-field `LabelCategory`); both live (NotesEditor.tsx:1582, admin/LabelTemplates.tsx:64). **Canonical:** import the full type or a `types/labels.ts`. **Confidence:** medium-high, minor.

### Mapping 195 — Journeys stage-table selection-mode state machine + parallel bulk-POST loop, token-identical trio (never-cited files)
- `components/intake/IntakeTable.tsx:26–43`, `components/openplans/OpenPlansTable.tsx:33–50`, `components/activetreatments/ActiveTreatmentsTable.tsx:36–53` — `handleSelectAll`/`handleSelectX`/`allSelected` machine, normalized diff = noun renames; routed at `/journeys/{intake,openplans,activetreatments}` (routes/clinical.routes.tsx:190,195,200).
- **Companion bulk loop (parents):** `pages/journeys/Intake.tsx:183–222`, `OpenPlans.tsx:173–212`, `ActiveTreatments.tsx:171–205` — `Promise.all(selected.map(fetchWithAuth(POST)))` with identical `Failed to ... ${id}: ${statusText}` throw → legacy toast → clear selection. Distinct from M156's sequential PATCH variant. **Latent gap:** no error aggregation — one failed POST throws after siblings succeeded. **Canonical:** `useSelectableRows(...)` + `bulkPostSelected(ids, buildRequest, {noun})` in `pages/journeys/lib/`. **Extraction:** safe. **Confidence:** high.

### Map correction (pass 45)
- Pass-34's dead-code claim is corrected: `components/openplans/OpenPlansTableRow.tsx` and `components/activetreatments/ActiveTreatmentsTableRow.tsx` are **live** (imported by OpenPlansTable.tsx / ActiveTreatmentsTable.tsx). Their `getPriorityBadge` optimistic-cycle clone (ActiveTreatmentsTableRow.tsx:57–160 ↔ OpenPlansTableRow.tsx:59–197) is therefore a live two-file clone, not dead.

### Mapping 192 — `canManageLabs` gate derived with two different feature keys on the same page
- `components/labs/LabCaseCommentsPanel.tsx:36` (`hasFeature("manage_labs")`) vs `LabPartnersView.tsx:144` (`hasFeature('lab_admin')`) — same variable name, same page (`pages/Labs.tsx:75,91`), already-drifted: a practice with one flag but not the other sees management affordances in one labs surface and not the other. **Fix:** derive once; decide the correct key. **Confidence:** high.

### Mapping 193 — error-suppression policy (`shouldSuppressError`) implemented twice with drifted match sets
- `lib/helpers.ts:7–29` (raw message includes '402' **and '403'** + phrase list; 6 importers) vs `utils/errorUtils.ts:7–33` (status-object 402 + phrase list **without 403**; live internally via `extractErrorMessage`, the M6 canonical). Identical JSDoc first line. **Canonical:** one implementation in `utils/errorUtils.ts`; helpers delegate/re-export after reconciling the 403/status branches. **Extraction:** safe. **Confidence:** high.

### Mapping 194 (thin, minor) — create-surface visibility gates hand-derived per component
- `hooks/useDocumentTypeAccess.ts:24–32` (useMemo `canCreateTreatmentPlans`/`canSendConsent`) vs `components/create/EditorHeader.tsx:63–64` (identical plain derivations); both live via NotesEditor.tsx:25,54. **Canonical:** export from `useDocumentTypeAccess`. **Confidence:** medium-high.

## Frontend pass-44 watch notes
- `components/patients/workspace/finance/{protoFormat.ts:50–60, balanceFormat.ts:14–24}` — `balanceTone` token-identical; `balanceFormat`'s exports currently have zero importers (dead-lane duplication; deletion-or-delegate).
- Comment-author delete predicate (`author.id === String(user?.id)`) in labs/tasks/patients panels — candidate `canDeleteComment(comment, user)`; labs copy unmapped.

### Mapping 186 — `openPatientPanel` navigation-state consumer effect, ~90-line near-clone pair
- Producer already consolidated (`pages/day-list/lib/journeyNavigation.ts:1–10` — exists "rather than growing a third copy of this mapping"). **Consumers still hand-duplicated:** `pages/Contacts.tsx:103–196` vs `pages/Journeys.tsx:1908–2013` — read location.state → optional refetch via endpoint ladder → normalize → isolate record → `navigate(pathname, {replace:true, state:null})`. Divergence: Journeys adds `treatment_plan` to the ladder + FR5 stage-mismatch redirect; Contacts handles patient-isolation only; normalization differs. Stray debug logs at Contacts:179, Journeys:1946/1980. **Canonical:** `useOpenPatientPanelFromNavigation({endpoints, normalize, onIsolate})` beside journeyNavigation.ts. **Extraction:** safe. **Confidence:** high.

### Mapping 187 — staff-alert deep-link consume scaffold (`targetRecordType` + `location.key` re-arm), 5 files
- Producer: `components/alerts/staffAlerts/AlertsBell.tsx:20–40` (shape in alertTypes.ts:33). **Consumers re-encode the same init + `[location.key]` re-arm effect + identical boilerplate comment:** `pages/Compact4.tsx:1165–1191`, `pages/hr/Leave.tsx:311–323`, `pages/hr/Payslips.tsx:789–816`, `pages/Invoices.tsx:220–231`; selector-only `pages/Labs.tsx:313–322,361–373` (**no re-arm — re-clicking an alert while on /labs is a latent no-op**); render-inline `pages/hr/Rotas.tsx:4576–4581`. Mount-once sub-variant (cross-page filter handoff): `pages/Journeys.tsx:651–660`, `Compact4.tsx:1155–1163`, `Labs.tsx:643–648`. **Canonical:** `useNavigationStateDeepLink<T>(match)` with re-arm built in. **Extraction:** safe. **Confidence:** high.

### Mapping 188 — optimistic WhatsApp-send machinery, verbatim pair (M36 extension)
- `pages/InboxPage.tsx:152–153,884–970` vs `pages/messages/prototype/FamilyGroupPrototypePage.tsx:175–176,887–975` — token-identical optimistic Map + `whatsapp-optimistic-${Date.now()}` tempId + `_pending`/`_failed` flip + refetch-on-success + mark-as-read rollback (:340 vs :356); rendering twin MessageBubble pair (:158 vs :153,170). **Canonical:** fold into M36's consolidation as `useOptimisticWhatsAppSend()`. **Confidence:** high.

### Mapping 184 — Settings online-booking mutation orchestration scaffold (11 copies, 6 files, one directory)
- `setIsSaving(true)` → try `fetchWithAuth` → `res.ok ? toast.success : toast.error(err.detail || fallback)` → `catch { toast.error("Could not reach server.") }` → `finally { setIsSaving(false) }` — the network-failure string repeated **12×** in `pages/settings/components/practice/` alone: `OnlineBookingTab.tsx:61–107` (three in-file), `OnlineBookingConfigTab.tsx:76–104`, `OnlineBookingServiceDialog.tsx:419–509` (two), `OnlineBookingExceptionsCard.tsx:80–108` (Set-based in-flight), `OnlineBookingServicesCard.tsx:225–289` (incl. optimistic-visibility revert; `handleDropEnd` has no finally), `WhatsAppTab.tsx:30–52`. All use canonical clients — not M1/M7. **Canonical:** `useSettingsMutation(fn, {success, failureFallback})` in `pages/settings/hooks/` (the settings-side analogue of M152's admin scaffold). **Extraction:** safe. **Confidence:** high, minor severity.

### Mapping 185 (thin, minor) — Documents-tab loading-skeleton block token-identical trio
- `pages/compliance/components/documents/{CoshhTab.tsx:289–297, RiskAssessmentsTab.tsx:409–417, MeetingMinutesTab.tsx:314–315}` — byte-identical 5-line `animate-pulse` search-bar + two-card skeleton; the module `pages/compliance/components/skeletons/index.tsx` exists but is bypassed. **Canonical:** add `DocumentsTabSkeleton` to the skeletons module. **Extraction:** safe, trivial. **Confidence:** high, low urgency.

## Frontend pass-42 watch notes
- `reporting/KPICards.tsx` private `KPICard` — divergent variant of M107's spend tile (disabled-reason/subValue vs alert-tone); would be the 5th member if M107 is consolidated.

### Mapping 184 — `browserTimezone()` IANA-default resolver: exported canonical bypassed by two inline re-rolls in the same marketing feature
- **Canonical:** `components/marketing/campaignSchedule.ts:46–51` `browserTimezone()` (doc: last-resort zone when a campaign has no practice_timezone); consumer MarketingCampaignBuilder.tsx:35–38,792,838. **Re-rolls (token-identical, live-routed):** `pages/Marketing.tsx:190`; `pages/MarketingCampaignCreate.tsx:18–21` (used at :56 as `practice_timezone` — exactly the canonical's described payload). **Risk:** fallback-policy change silently diverges campaign creation vs builder. **Fix:** import the canonical. **Extraction:** trivial. **Confidence:** high.

## Frontend pass-41 dead-lane finding (recorded; not a live family)
- `pages/financial-analytics/components/DashboardView.tsx` has **zero production importers** (live page renders only `FinancialAnalyticsPrototype`). Chain DashboardView → PatientJourneyTab:141 → CohortReferralTab renders nothing live, making the V1↔V2 near-clone pairs (`ContributionMatrixTable` incl. identical cellLookup/rowMax loops :1340–1355 vs :1421–1440; `PeerBenchmarkCard`/`RevenueSequenceSection`; `ConvertedMixPanel`) dead-alive pairs — deletion candidates. Income formatting already diverged (`fmtGBPRounded` vs `fmtGBP`).

### Mapping 182 — Tabs-overflow ResizeObserver measurement, 3 token-identical copies
- Hidden offscreen measure element (`scrollWidth > clientWidth + 1`) → `setTabsOverflow`, re-measured via `ResizeObserver` on both refs with a `window.resize` fallback: `pages/Invoices.tsx:193–214` (measure at :193, observer :204–209), `pages/compliance/Compliance.tsx:836–859` (+ second sibling measurement `measureMobileTabTrigger` :861–880), `pages/hr/components/HRSubPageLayout.tsx:130–151`. All render the hidden element and consume `tabsOverflow`. **Canonical:** `useTabsOverflow(containerRef, measureRef)` hook beside the tabs primitive. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 183 (thin, minor) — `beforeunload` dirty-guard re-rolls beside canonical `useUnsavedChangesGuard`
- **Canonical:** `pages/compliance/hooks/useUnsavedChangesGuard.tsx:37–47`. **Bypasses:** `components/invoices/AssociatePayView.tsx:1090–1097` (mirrors it; dialog half documented intentional); `pages/EmailTemplateBuilder.tsx:339–346` — **omits `event.returnValue = ''`** (prompt may not appear on older browsers). **Fix:** accept `isDirty` into the canonical hook. **Extraction:** safe. **Confidence:** medium-high.

### Mapping 180 — Subscription-status → badge-color map: exported canonical bypassed by two token-identical private copies
- **Canonical:** `pages/admin/practice-management/helpers.ts:42–52` `getSubscriptionStatusColor` (imported by SubscriptionSection.tsx:26). **Bypasses:** `pages/admin/Users.tsx:432–441` and `PracticeUsers.tsx:555–564` — character-identical private re-declarations. **Risk:** status vocabulary change (e.g. `paused`) edits 3 places. **Fix:** import the export. **Extraction:** trivial. **Confidence:** high.

### Mapping 181 — "Module + feature → surface" gate table hand-encoded twice: dashboard digest gates vs AppSidebarV2 nav filter
- `components/dashboard/digest/gates.ts:13–32` vs `components/AppSidebarV2.tsx:1513–1636` — identical `hasModule(X) && hasFeature(Y)` pairs for daylist/create/stock/compliance/hr/recalls/inbox/journeys/diary; gates.ts header comment: "visibility rules mirror AppSidebarV2 so the digest never exposes a surface the navigation gated off". **Drift already present:** sidebar gates Docs on `clinical && drafts` and Library on `clinical && library` (:1525–1531) while gates.ts encodes `documents: clinical && (notes || letters)` (:23–25); sidebar's `reporting`/`contacts` gates have no digest counterpart. **Canonical:** one `SURFACE_GATES: Record<Surface, (ctx) => boolean>` in `lib/surfaceGates.ts`. **Extraction:** safe; decide whether digest's looser documents rule is intentional. **Confidence:** high (duplication), medium (intent).

## Frontend pass-39 extensions (already-mapped file pairs, unmapped operations)
- **M78 ext:** `currentSubscription` derivation triplicated — `BillingTab.tsx:195`, `PracticeBillingTab.tsx:196`, `PlansAndBilling.tsx:164` (`find(sub => status === 'active' || 'trialing')`); expose from `useStripeBilling`.
- **M83 ext:** subscription summary card token-identical — `pages/admin/Users.tsx:1500–1530` ↔ `PracticeUsers.tsx:1658–1690`; fold into the same extraction as M83/M128.

### Mapping 178 — Entity-scoped comments React-Query hook trio (list + create + delete wrapper cloned)
- `hooks/useTaskCommentsQuery.ts:8–99`, `useInvoiceCommentsQuery.ts:25–95`, `usePayslipDraftCommentsQuery.ts:29–97` — identical `queryKeys.all(id)` factory, `useQuery` with `enabled: !!id` + Array/`results` unwrap, create/delete mutations with `onSuccess: invalidateQueries`, identical return shim and mapper (`author.name || 'Unknown'`, `mentionsFromPayload`). **Divergence:** task lane uses `useFetchWithAuth` (401-refresh/timeout + ok-check + DRF extraction); invoice/payslip use `useDjangoAPI`; task id `number` vs string. All live (TaskCommentsPanel:7, InvoiceCommentsSection:10, PayslipDraftCommentsSection:10 — pass-15 covered only the component sections). **Canonical:** `useEntityCommentsQuery({endpointFor(id), queryKeyPrefix})` factory. **Extraction:** safe. **Confidence:** high.

### Mapping 179 (thin, minor) — per-domain `STALE_TIMES` tables re-declaring a common freshness policy, 6 files
- `hooks/useAssetsQuery.ts:246–252`, `useStockQuery.ts:464–469`, `pages/hr/hooks/useHR.ts:61–68` (identical banner comments), `hooks/useLabsQuery.ts:37–44` (only exported one), `pages/compliance/hooks/useAudits.ts:99–104`, `useCompliance.ts:618–627` — same tiers (5min static / 30s live / 1min mutable / 2min submission). Same values re-hardcoded inline in CompliancePolicies.tsx:155, ComplianceRiskAssessments.tsx:215,226, useNotesList.ts:60–61, useActiveTemplates.ts:33–34, useWorkingDays.ts:47, useTodayChecklists.tsx:127; two private names in `components/stock/` (PersonSelect.tsx:45 vs AssignedToSelect.tsx:75). **Canonical:** `lib/queryStaleTimes.ts` tiered constants. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 175 — Compliance-calendar `EVENT_ICONS` map triplicated, **already drifted** (live breakage risk)
- `pages/compliance/components/calendar/{CalendarFilters.tsx:30, CalendarEventList.tsx:31, ComplianceCalendarDayDetailPanel.tsx:87}` — identical 16-entry `EVENT_ICONS` declarations in one feature directory. **Drift:** `risk_assessment_review` → `FileWarning` (Filters) vs `AlertTriangle` (EventList) vs `ShieldAlert` (DayDetailPanel); `hr_document_missing` → `FileWarning` vs `User` vs `UserX` — the same event renders a different icon per calendar surface. **Canonical:** one `EVENT_ICONS` export in `calendar/eventTypeMeta.ts`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 176 (thin, minor) — same-named `WEEKDAY_OPTIONS` with opposite index conventions
- `components/AddTaskModal.tsx:142` (Monday=0, sent to backend `repeat_weekday`; matches Compact4.tsx:81 WEEKDAY_LABELS) vs `pages/settings/components/practice/PracticeDiaryTab.tsx:74` (Sunday=0 frontend convention, feeds diary `pattern.dayOfWeek`). Each lane internally consistent; naming trap for future consumers. **Canonical:** shared `WEEKDAYS_MON_FIRST`/`WEEKDAYS_SUN_FIRST` constants tied to M51's `toMondayIndex`. **Confidence:** high (duplication), medium (consolidation wanted?).

### Mapping 177 (thin, minor) — minutes→"Xh Ym" duration formatter escaped the M77 pair
- Token-identical ladder in `pages/admin/ComplianceAuditTemplates.tsx:679` ↔ `ComplianceChecklistTemplates.tsx:639` (both M77's clone pair) **plus a third unmapped copy** `pages/settings/components/practice/AuditTemplatesSettingsView.tsx:164–169`. Near-variants: PracticeTreatmentsView.tsx:166–173, PatientDentalChart.tsx:744,773, RotaAssignmentPanel.tsx:115–121. Fold into M73's `formatDuration` canonical. **Confidence:** medium-high, minor.

## Frontend pass-37 extension/watch notes
- **M35 ext:** `pages/day-list/components/AppointmentCard.tsx:61` ↔ `pages/financial-analytics/components/CohortReferralTab.tsx:297` — token-identical 8-line title-strip `firstName`.
- **Watch:** deep-link param-consume scaffold (`searchParams.get → guard → setSearchParams({}, {replace:true}) → act`) at `compliance/components/audit/AuditLibrary.tsx:298–315` vs `practice/PracticeOverview.tsx:789–805` (comment admits mirroring) — a third occurrence would be reportable (analogue of the pass-29 watch).
- EmployeeProfileDrawer's N/A/force-required props — documented prop contract, wiring not cloned logic.

### Mapping 172 (thin, minor) — Compliance route-path string literals, no constants module
- No `ROUTES`/path-constants module exists anywhere. De-facto routing table: `pages/compliance/Compliance.tsx:751–761` (tab → path if/else) + `:621–656`; `PolicyEditor.tsx:238,302,320,344`; `ChecklistEditor.tsx:251,276,635`; `AuditTemplateEditor.tsx:399,437,1018`; dashboard deep-links `components/dashboard/digest/ClockCardExtras.tsx:22`, `StarredChecklistsWidget.tsx:77,95,106`, `compliance/dashboard/QuickLinksSection.tsx:28,41,50`, `ComplianceDashboard.tsx:744`. Crisp pair: `'/compliance/documents?tab=training'` hand-built identically in Compliance.tsx:621 and QuickLinksSection.tsx:41 (third: reporting/StaffActivityReport.tsx:105). **Risk:** tab rename needs ≥7 edits. **Canonical:** `compliancePath(tab, params?)`/`COMPLIANCE_TAB_PATHS` in `pages/compliance/constants.ts`. **Extraction:** safe. **Confidence:** high (duplication), minor severity.

### Mapping 173 (thin) — post-auth `navigate("/dashboard")` redirect restated per auth screen
- `pages/auth/Onboarding.tsx:339`, `VerifyOTP.tsx:213`, `MagicLoginRequest.tsx:221,331,396` (dead VerifyMagicLogin.tsx:136 excluded per M89). **Canonical:** `POST_AUTH_REDIRECT` constant or post-login routing helper. **Confidence:** medium-low.

### Mapping 174 (thin) — `/tasks` + `/labs` destination pair duplicated
- `components/dashboard/digest/DigestDetails.tsx:84,128` ↔ `components/patients/patient-panel/PatientOutstandingItems.tsx:156,137` — identical "open module" literals for the same two destinations. Fold into M172 if a paths module is created. **Confidence:** high, minor.

## Frontend pass-36 clean lenses
- localStorage UI-preference keys: full inventory shows single-owner modules (heat overlay, starred-checklists M149, group-selection M148, terminal token, journeys-view-mode, practice_slug) — no new family.
- `Intl.NumberFormat`: only 3 constructions, semantically distinct (one already M3) — no pair.

### Mapping 168 — PDF-export toolkit re-rolled beside its own exported canonical (`exportPayslipPdf` kit)
- **Canonical (exported, live):** `components/invoices/exportPayslipPdf.ts` — `loadLogoAsset`:1942 (incl. `resizeLogoForPdf`:1920), `createJsPdfTextWrapper`:264, `blobToDataUrl`:1863, `getImageDimensions`:1882, `renderPayslipPdfDocument`:1824. Consumers: `pages/hr/utils/exportLeaveRequestsPdf.ts:5–12`, `exportSchedulesPdf.ts:5–12`, `components/stock/exportStockQrPdf.ts:3`.
- **Bypass 1:** `components/invoices/exportSupplierInvoiceLetterPdf.ts:40–147` — private near-clones; **omits `resizeLogoForPdf`** (canonical's comment warns un-resized multi-megapixel logos block the main thread for seconds) and mime-sniffs format where canonical forces 'PNG'.
- **Bypass 2:** `pages/hr/utils/exportRotaPdf.ts:797–804` — exports a **competing same-name `createJsPdfTextWrapper`** token-identical to canonical; also mirrors the plan/render API (:772, :995). Sole consumer pages/hr/Rotas.tsx:127.
- **Canonical:** promote kit to `lib/pdfExportKit.ts`; supplier + rota import. **Extraction:** safe. **Confidence:** high.

### Mapping 169 — exported `useIsAdmin()` bypassed by two token-identical inline re-derivations (extends M9)
- **Canonical:** `contexts/AuthContext.tsx:2127–2131` `useIsAdmin()` (`userType === "practice" && isAdmin === true`) — ~17 files use it. **Bypasses:** `routes/guards/AutomationAdminGate.tsx:13` (the actual automation route gate) and `components/AppSidebarV2.tsx:1038` — identical inline line; the sidebar comment asserts "shared between sidebar and route gating" that the code doesn't do. **Risk:** admin-rule change in the hook silently skips the route gate and sidebar. **Fix:** call `useIsAdmin()`. **Extraction:** trivial. **Confidence:** high.

### Mapping 170 (thin, minor) — "who counts as a clinician/dentist" team-member predicate, 3 divergent live copies
- `hooks/usePracticeTreatments.ts:315–318` (`practiceRole === "dentist" || member.isClinician` — comment documents the backend rule and the backfill gap); `pages/Settings.tsx:420–422` (`role === "Clinician"` display string — **carries exactly the un-backfilled failure mode the first copy documents**); `pages/automation/editor/components/action-configs/CreateTaskConfig.tsx:62–65` (snake_case + requires practice-admin). **Canonical:** one `isClinicianMember(member)` predicate with camel/snake adapters, aligned to the backend rule (mirrors backend M21). **Confidence:** medium-high (CreateTaskConfig may be intentional).

### Mapping 171 (thin, minor) — consent artifact client actions: signed-PDF download pair + short-url copy trio
- Download: `components/consent/ConsentSignedActions.tsx:12–22` (date-suffixed filename) ↔ `components/patients/patient-panel/PatientPanelClinicalTab.tsx:191–200` (no suffix) — identical anchor scaffold. Copy: `ConsentQrModal.tsx:72–81` ↔ `SendConsentModal.tsx:344–355` (token-identical `handleCopyLink`) + inline variant `ConsentSendDialog.tsx:44`. **Canonical:** fold into M17's `downloadBlob` and M64's `useCopy()`. **Confidence:** high, minor.

### Mapping 166 — Note-structure parser (`buildNoteStructure` + regex suite), verbatim clone beside its exported canonical
- **Canonical (exported, live):** `utils/notesUtils.ts:11–86` — regex constants, `normalizeLabel`, `buildNoteStructure`; consumers NotesEditor.tsx:73–77, useExternalEdits, applyNoteEditToText. **Clone (private, live):** `components/assistant-toolbar/ChatPanel.tsx:33–108` — token-identical ladder; only adds a deduped `paths` projection. Sole importer ProcedurePathways.tsx. **Risk:** regex/marker tweak diverges the assistant's section targeting from the editor's. **Fix:** import the canonical + small `paths` helper. **Extraction:** safe. **Confidence:** high.

### Mapping 167 — Lexical active-format (B/I/U) + selection tracking hook, canonical `useEditorSelection` bypassed by 5 inline re-rolls
- **Canonical:** `hooks/useEditorSelection.ts:10–63` (sole importer NotesEditor.tsx). **Re-rolls (all live):** `pages/settings/components/consent/ConsentTemplateEditor.tsx:80,158–172`; `components/create/ConsentMarkdownEditor.tsx:46–49,175–196` (adds `selectedText` — parity with canonical); `pages/day-list/components/administration/EmailMessageEditor.tsx:225–270` (mergeRegister + SELECTION_CHANGE superset); `components/email-builder/BasicEmailToolbar.tsx:301–335` (8-format superset + blockType); `components/compliance/PolicyLexicalEditor.tsx:82–125` (plugin form). Divergence: format vocabulary 3 vs 8, editor-acquire strategy. **Canonical:** generalize `useEditorSelection({getEditor, formats})`. **Extraction:** safe. **Confidence:** high.

## Frontend pass-34 dead-code correction/notes
- Correction to pass-34 lead: `components/activetreatments`/`openplans` — the *Table row components (incl. the token-identical `getPriorityBadge` optimistic priority-cycle patch at ActiveTreatmentsTableRow.tsx:57–160 ↔ OpenPlansTableRow.tsx:59–197) are dead, but the dirs' `types.ts` modules remain imported (useOpenPlansData.ts:25, useActiveTreatmentsCaching.ts:27, ActiveTable.tsx:54). Delete the row components, keep types.
- Other zero-citation dirs checked clean: pages/terminal (M1-adjacent raw fetch), pages/guide-link, components/help (self-contained fuzzy search), components/email-builder (clean, shared text helpers), components/marketing/MarketingPagination (M69/M57 extension only).

### Mapping 163 — Infinite-scroll "load more" trigger, 7 live sites, two mechanisms
- **Scroll-ratio (80%):** `components/journeys/AddPatientDialog.tsx:642–647`; `components/compact/AddPatientModal.tsx:846–853` (same patient-search pagination reimplemented in a sibling dialog); `components/inbox/v2/ConversationListPanel.tsx:141–162` (ref-based); `pages/messages/prototype/ConversationListPanelPrototype.tsx:143–158` (M42 twin). **IntersectionObserver sentinel:** `components/teamChat/TeamChatThread.tsx:92–114` (alone preserves scroll position on prepend), `components/patients/patient-panel/PatientPanelAccordion.tsx:356–368`, `components/journey-board/JourneyBoardColumn.tsx:57–69`. Divergence: trigger point, guard style (state vs refs), scroll anchoring. **Canonical:** `useInfiniteLoadMore({canLoad, loading, onLoadMore, threshold, root})`. **Extraction:** safe. **Confidence:** high.

### Mapping 164 (thin, minor) — Template-image `image_data` metadata projection, 3 live copies
- Pending editor images → `{image_id, section, order, alignment, width, height, position_in_content}` feeding `useLetterTemplates().addImagesToTemplate`: `components/Library/CreateTemplateModal.tsx:197–205`, `pages/admin/components/LetterTemplateForm.tsx:467–475` (section hardcoded "body"), `components/letter-library/EditTemplate.tsx:377–386` (only copy omitting empty dims). Same payload, three subtly different encoders. **Canonical:** `buildTemplateImageMetadata(files)` beside `useLetterTemplates`. **Extraction:** safe. **Confidence:** high.

### Mapping 165 (thin, minor) — Audio-transcription FormData POST scaffold, 3 copies
- `hooks/useNotes.ts:492–517` vs `:520–545` (in-file near-twins, differ only in endpoint/`template_content`); `hooks/useMessaging.ts:607–660` (same multipart-POST-without-Content-Type + transcript extraction, divergent return shape). Distinct from M20 (WS/PCM pipeline). **Canonical:** `transcribeAudio(file, endpoint, {extraFields})`. **Extraction:** safe. **Confidence:** high.

### Mapping 160 — Consent "resend" POST handler, 6 live implementations
- `components/consent/JourneyConsentChip.tsx:264–281` (channel hardcoded "email"); `ResendConsentContent.tsx:39–57` (user-selected channel, DRF detail extraction); `ConsentSendDialog.tsx:127–135` (`resendOn`, different error format); `components/patients/patient-panel/PatientPanelClinicalTab.tsx:328–337` (hardcoded email); `components/patients/workspace/documents/unified/UnifiedDocumentsList.tsx:141–152` (token-identical to ClinicalTab modulo fetch fn); `pages/day-list/components/AppointmentCard.tsx:434–447` (partially canonicalized via `lib/sendConfirmation.ts:182–190 resendConsentRequest`, still re-rolls toast wrapper). Divergence: 3 sites do DRF detail extraction, 3 don't; two toast wordings. **Canonical:** generalize `resendConsentRequest(id, channel)` out of day-list. **Extraction:** safe. **Confidence:** high.

### Mapping 161 — Consent QR fetch effect (`qrCode` endpoint + `frontend_origin`), 3 live copies
- `components/consent/ConsentQrModal.tsx:46–58` ↔ `SendConsentModal.tsx:261–270` ↔ `ConsentSendDialog.tsx:117–125` — URL construction incl. the odd `getConsentSigningUrl("placeholder")`-origin trick repeated verbatim; divergences only in loading/retry handling. **Canonical:** `fetchConsentQr(id, fetchWithAuth)`. **Extraction:** safe. **Confidence:** high.

### Mapping 162 (thin, minor) — Consent "cancel request" POST pair
- `components/patients/patient-panel/PatientPanelClinicalTab.tsx:339–349` ↔ `workspace/documents/unified/UnifiedDocumentsList.tsx:154–162` — identical `POST signingRequests.cancel(id)` → toast → refetch scaffold; only toast wording differs. Fold into M160's consent-actions module. **Confidence:** high, minor.

### Mapping 163 — "Assign dentist" auto-select + `transformPractitionerToStaffMember` trio (canonical `hooks/usePractitioners.ts` bypassed by all three)
- `components/create/DentistSelection.tsx:29–43,57–80,91–158` (own fetch effect); `pages/treatment-plans/components/patient-selection/DentistSelection.tsx:40–60` (context-sourced); `TaskAssignment.tsx:36–49` (token-identical transform except name fallback uses `practitioner.id` vs `user_id` — drift). **Canonical:** export the transform once + `useDentistOptions()` (or adopt `usePractitioners.ts`). **Extraction:** safe. **Confidence:** high.

## Frontend pass-32 dead code
- `components/letter-library/CreateLetterFromNoteModal.tsx` is **0 bytes** (dead; unlisted deletion candidate).

### Mapping 155 — Inline quick-notes optimistic update handler `handleNotesUpdate`, 5 live copies
- `components/compact/{OpenTable.tsx:794–820, ActiveTable.tsx:474–496, NurtureTable.tsx:1046–1067}` (always PATCH); `IntakeTable.tsx:1011–1050`, `CustomJourneyTable.tsx:429–465` (variant: skip no-op when text unchanged; CJ comment "copied verbatim from IntakeTable.tsx"; CJ re-rolls fetch). Optimistic `setLocalNotes` → PATCH → `showOmittedMentionsToast` (already shared via `lib/mentionToast.ts`) → catch-revert. **Canonical:** `useQuickNotesOverlay({update, noteField, skipUnchanged})` hook. **Extraction:** safe. **Confidence:** high.

### Mapping 156 — `handleBulkEditConfirm` sequential bulk-PATCH loop, 5 live copies
- `OpenTable.tsx:721–748` ↔ `ActiveTable.tsx:779–805` (token-identical); `NurtureTable.tsx:847–871`, `IntakeTable.tsx:776–801`, `CustomJourneyTable.tsx:603–627` — same empty-guard → per-id PATCH loop with error aggregation → clear selection → refetch → throw joined errors; only payload projection/endpoint differ. **Canonical:** `bulkPatchSelected(ids, endpoint, projectPayload, noun)` or move the confirm into the already-shared `BulkEditDialog`. **Extraction:** safe. **Confidence:** high.

### Mapping 157 — `handlePriorityChange`, 3 live copies
- `OpenTable.tsx:873–886` ↔ `ActiveTable.tsx:497–512` (token-identical incl. toast strings); `NurtureTable.tsx:1068–1088` (same body but legacy `toast({title, variant})` API — M10's documented Nurture drift). **Canonical:** shared field-patch helper; fix Nurture's toast call. **Confidence:** high.

### Mapping 158 — Isolated-record fetch effect, 5 live copies
- Cancelled-guarded `fetchWithAuth(API_ENDPOINTS.<stage>.get(isolatedId))` → transform → `setIsolatedRecord` → catch-silent → finally clear loading: `IntakeTable.tsx:614–635`, `NurtureTable.tsx:717–738`, `CustomJourneyTable.tsx:709–730`, `OpenTable.tsx:366–384`, `ActiveTable.tsx:~231–256`. Fed from `pages/Journeys.tsx` (7 launch points). **Canonical:** `useIsolatedRecord(isolatedId, endpoint, transform)`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 159 — Row exit-animation state machine, 5 live copies
- `type ExitDirection` + `EXIT_ANIMATION_MS = 2000` declared independently in all five compact tables (Intake:221,252; Nurture:424,434; Open:231,241; Active:227; CustomJourney:404–408 — comment "mirrors IntakeTable.exitingLeadDirections"). `getExitClasses`/`animateLeadExit`: Intake:818–847 ↔ Nurture:812–840 token-identical; only the stage-anchor and CJ's fade-only variant genuinely differ. **Canonical:** `useRowExitAnimation(stageAnchor)` hook + shared constant. **Extraction:** safe. **Confidence:** high.

## Frontend pass-31 extension/watch
- **M39 ext:** `pages/Compact4.tsx:264–360` — a third live `Confetti` clone (normalized diff ≈ formatting only) **with the valid `'#846ce0'` palette** where the two compliance copies still carry the invalid `'blue-600'`; reinforces M39's copy-bug note.
- **Watch (below bar):** FR1 operational-column patchers across the five tables — assignee preamble token-identical in 4 files, but persist genuinely diverges per stage; reportable if a fourth persist style appears.
- **Dead:** `components/compact/OpenTable.tsx:543–558` `handleConvertLead` — zero call sites (file routes converts through `openConvertDialog`/`useJourneyConvert`). Deletion candidate.

### Mapping 154 — Compliance "Share with Group" modal — optimistic Set-toggle triplicated
- `pages/compliance/components/audit/AuditTemplateShareModal.tsx:92–139` ↔ `documents/RiskAssessmentShareModal.tsx:92–139` (normalized ~85% identical, incl. full state block) ↔ `policies/PolicyShareModal.tsx:92–139` (same `handleToggle` byte-for-byte; superset adds review-request actions; 305 ln). Identical per-practice loading map + optimistic `Set<practiceId>` update + two-way revert in catch; toast strings `'Access revoked successfully.'`/`'Action failed. Please try again.'` occur exactly 3× — once per file. All live (ManageTrainingTypesModal.tsx:25,636 already reuses the audit modal — evidence a fourth consumer shape exists). **Canonical:** shared `ShareWithGroupModal({entity, onShare, onRevoke, fetchGroupShares, noun})` with Policy's review props as optional slots. **Extraction:** safe. **Confidence:** high.

## Frontend pass-30 exclusions
- Empty-state copy clusters ("No templates available" ×13, "No questions yet" ×8, etc.) — wording-level with divergent markup; per the pass-4 exclusion, genuinely per-domain. Mechanical match: `"No patients found"` div in both M85 PatientSelection copies — thin extension only.

### Mapping 152 — Admin CRUD form submit scaffold: `ok → toast.success → close → resetForm() → refetch` / else DRF field-error toast
- **Implementations (8+ live files):** `pages/admin/CompliancePolicies.tsx:180–236` (+resetForm :289), `ComplianceTrainingTypes.tsx:225–300` (+:203–215), `MasterCatalogue.tsx:304–348` (+:267–280), `Notifications.tsx:136–172` (+:252–260), `SupplierManagement.tsx:206–252` (+:177–184), `UnmappedQueue.tsx:237–334` (**three in-file repeats**), `ComplianceCoshh.tsx:206–210`, `GlobalPhrases.tsx:153–158,409–414` (multi-field error ladder). Same create/update pair per file with `resetForm()` to per-entity defaults.
- **Divergence:** refresh mechanism (invalidate vs named refetches), catch logging, error-field precedence order; some reset only on success. **Canonical:** `useAdminCrudForm({endpoint, defaults, noun, refetch})`; at minimum route error extraction through `utils/errorUtils.extractErrorMessage` (adjacent to M6). **Extraction:** safe. **Confidence:** high.

### Mapping 153 — Tab ↔ URL query-param sync machine
- Token-identical trio (read+write): `pages/compliance/components/logs/LogsLibrary.tsx:207–228` (also resets search/page) ↔ `components/documents/DocumentsLibrary.tsx:152–170`; read-only variant `pages/day-list/components/DayListAdministration.tsx:38–46` (never syncs back). **Canonical:** `useTabQueryParam(validTabs, default)` hook. **Extraction:** safe, trivial. **Confidence:** high.

## Frontend pass-29 extensions (for the record)
- **M78 ext:** `BillingTab.tsx:131–148` ↔ `PracticeBillingTab.tsx:131–149` token-identical `handlePaymentSuccess`/`handlePaymentCancel` URL-clear handlers — fold into M78's `useStripeBilling`.
- **Watch:** compliance tab highlight-param consume effect (`CoshhTab.tsx:118–133` ↔ `RiskAssessmentsTab.tsx:268–279`) — identical URLSearchParams delete-with-replace scaffold, genuinely divergent surrounding resets; a third occurrence would be reportable.

### Mapping 148 — Group-practice selection sessionStorage persistence + cross-instance event sync, duplicated compliance ↔ HR
- `pages/compliance/hooks/useGroupCompliance.ts:32–33,41,46–62,98` vs `pages/hr/hooks/useGroupHR.ts:36–37,50,66–88,134` — token-identical scaffold: sessionKey = `\`${KEY}:${currentPracticeId}\``, lazy init read, SYNC_EVENT CustomEvent listener, persist effect, dispatch-on-select. Keys `compliance_selected_practice` / `hr_selected_practice` also hand-named in `contexts/AuthContext.tsx:1748–1749` and `hooks/usePracticeSwitching.ts:171–172` (4 files, no shared constant). **Divergence (documented):** compliance restores only `'overall'`; HR restores numeric ids too. **Canonical:** `useGroupPracticeSelection({storageKey, syncEvent, parse})` + exported key constants. **Extraction:** safe. **Confidence:** high.

### Mapping 149 (thin) — `starred-checklists` localStorage codec split across reader and writer
- Writer/owner: `pages/compliance/components/checklist/ChecklistTemplateLibrary.tsx:146–230` (init, API sync, optimistic toggle with two revert paths). Independent reader: `components/dashboard/StarredChecklistsWidget.tsx:32` — raw `JSON.parse(localStorage.getItem("starred-checklists") || "[]")` fallback that depends on the Library's serialization by convention only. **Canonical:** `starredChecklistsStorage.load()/save()`. **Extraction:** safe, trivial. **Confidence:** medium-high.

### Mapping 150 (thin, minor) — `campaign_status_updated` WebSocket subscription duplicated
- `pages/Marketing.tsx:407–431` vs `MarketingCampaignPreview.tsx:139–158` — character-identical channel URL `/ws/marketing/campaign-status/${practiceId}/` + event-type guard; divergence (list vs single-campaign refetch) justified. **Canonical:** `useCampaignStatusWebSocket({campaignId?})` or shared URL builder + constant. **Extraction:** trivial. **Confidence:** medium.

### Mapping 151 (minor) — multi-select chip toggle + chip-button scaffold, 4 live copies
- `components/filters/{DentistFilter.tsx:20–26,33–52, SourceFilter.tsx:21–27,34–52, TreatmentFilter.tsx:28–33}` (token-identical toggle bodies + pill-chip JSX, palette variants) + `components/toolbar/LabelsButton.tsx:27–33 handleLabelToggle`. Trio live via `components/filters/FilterModal.tsx:5–8` (journeys pages). **Canonical:** `MultiSelectChipGroup({options, selected, onToggle, tone})`. **Confidence:** medium-high, minor.

## Frontend pass-28 dead-code resolution
- Pass-3 watch-lead resolved: `components/patients/DentallyImportFlow.tsx` is **dead** (zero importers; its ~120-line WS lifecycle cloned DentallyImportWizard.tsx:364–490) — deletion candidate; Wizard alone is live (pages/Contacts.tsx:595). No live duplicate remains.

### Mapping 144 — Compliance "QuickForm" upload-dialog scaffold triplicated
- `pages/compliance/components/documents/{GeneralDocumentQuickForm.tsx:38,70,75–77, PolicyQuickForm.tsx:39,52,66, CoshhQuickForm.tsx:36,70}` — ~90% token-identical after noun/field renames: `canSubmit` derived boolean, `save` useCallback, identical `useImperativeHandle({submit, canSubmit, saving})` + `onCanSubmitChange` effect, "review due in a year" seeded default. All live via `ComplianceDocumentsDialog.tsx:642,656,684` (map cited these files only under M7). Divergence: multi-file vs single file; placeholder body; Coshh seeds reviewer via useAuth. **Canonical:** config-driven `ComplianceQuickForm({fields, toRecord, fileMode})`. **Extraction:** safe. **Confidence:** high.

### Mapping 145 — Sandbox "clear recorded messages" confirm-then-POST handler + toggle + load, generic vs recall copy
- `components/administration/SandboxTab.tsx:93–107` (generic; live in JourneysAdministration + DayListAdministration) vs `pages/admin/practice-management/components/AdminSandboxSection.tsx:66–83` (recall copy; live via PracticeWorkspace.tsx:890) — clear handler + confirm string character-identical; `loadMessages` token-identical (:37–48 vs :74–86). Toggle toast style drifted. Dead commented third copy in `RecallsAdministration.tsx:123–137`. **Canonical:** parameterize recall onto the generic `SandboxTab` (already takes configUrl/messagesUrl/clearUrl); delete AdminSandboxSection + commented block. **Extraction:** safe. **Confidence:** high.

### Mapping 146 — Free-text amount parser `replace(/[^0-9.]/g,'') → Number` pair
- `components/diary/NewAppointmentModal.tsx:112–119` `parseDepositAmount` (non-finite → null) vs `components/journey/CurrencyInput.tsx:48–56` `commit()` (non-finite → **revert to prior value**). Signed variant (plausibly justified): `pages/hr/components/timesheets/InlineTimeAdjustment.tsx:42`. **Canonical:** `parseAmount(value, {onInvalid})` in utils. **Extraction:** safe. **Confidence:** high.

### Mapping 147 (minor) — Medical-history `missingRequiredFollowUp` + `canSubmit` gate duplicated across staff/public lanes
- `components/patients/workspace/medical/MedicalHistoryDraftForm.tsx:81–87` vs `pages/PublicMedicalHistoryFormPage.tsx:81–86,156–157` — `missingRequiredFollowUp` token-identical ("yes"-answer requires detail); completion conditions genuinely differ (custom-question text vs signature/representative). **Canonical:** `requiresFollowUp(q, a)`/`useMedicalHistoryGate(...)`; public page adds signature terms. **Extraction:** safe. **Confidence:** high.

### Mapping 136 — `useLetterPhrases` ↔ `useNotePhrases` whole CRUD-hook clone, normalized diff = 0
- `hooks/useLetterPhrases.ts` vs `hooks/useNotePhrases.ts` (161 lines each) — after token-normalizing Letter/Note, `diff` is **empty**: identical inline API types, `toPhrase`/`toCategory` mappers, isMounted guard, refresh fetch, CRUD/error/toast flow. Only the endpoint namespace differs. Both live (`components/NotePhrases.tsx:701,801` and `:706,812`). **Canonical:** `usePhraseLibrary<TPhrase, TCategory>()` factory. **Extraction:** safe. **Confidence:** high.

### Mapping 137 — `toMinutes` HH:MM→minutes parser, 5 definitions
- Token-identical pair: `hooks/useAutomationWindow.ts:80` ↔ `pages/settings/components/practice/AutomationWindowSection.tsx:54` (the latter imports `normalizeTimeValue` from `components/ui/time-select` yet re-rolls the derived parser). Token-identical pair: `pages/hr/utils/rotaSessions.ts:10` ↔ `exportRotaPdf.ts:185` (`split(':')` variant). Variant: `components/ui/time-select.tsx:72` (null fallback). **Canonical:** export `toMinutes(value, fallback)` from `components/ui/time-select.tsx`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 138 (minor) — Compliance `generateReferenceNumber` pair, drift risk on prefix only
- `pages/compliance/SedationEditor.tsx:57–61` (`SED-`) vs `SafeguardingEditor.tsx:59–63` (`SG-`) — otherwise token-identical (date + random-4 suffix). CoshhEditor likely has a third. **Canonical:** `generateComplianceReference(prefix)`. **Extraction:** trivial. **Confidence:** high.

### Mapping 139 — Day-list local-noon date-label `formatDate`, 4 token-identical copies with misleading comments
- `pages/day-list/components/{AwaitingConsentView.tsx:39, PatientsAtRiskPanel.tsx:16, AttendanceRiskView.tsx:32, ImportantPatientsView.tsx:41}` — identical `new Date(\`${date}T12:00:00\`).toLocaleDateString('en-GB', …)`; per-file comments point at *different* supposed sources, none of which export it. **Canonical:** export from `pages/day-list/lib/dayWindow.ts`. **Extraction:** safe, trivial. **Confidence:** high. (Timezone-safe local-noon idiom; distinct from M4.)

### Mapping 140 — `HtmlOnChangePlugin` Lexical→HTML serialization pair, divergent (one has a bug fix the other lacks)
- `components/admin/AlertEmailEditor.tsx:118` vs `pages/day-list/components/administration/EmailMessageEditor.tsx:175` — same name/signature, both skip first hydration update, both `$generateHtmlFromNodes` on every update. **Divergence:** EmailMessageEditor adds a `lastHtmlRef` change-detector so selection-only updates don't fire `onChange`; AlertEmailEditor lacks it and re-fires onChange on click/blur — looks like a bug fix never back-ported. **Canonical:** shared `useHtmlOnChangePlugin(onChange, {suppressUnchanged})`. **Confidence:** high.

### Mapping 141 (minor) — `GhostMini` icon-button duplicated in marketing form builder pages
- `marketing/forms/pages/BuilderPage.tsx:3095` (12 uses) vs `BlocksPage.tsx:655` (6 uses) — same component; real drift: default icon `text-gray-500` vs `text-gray-400`, `shrink-0` only in Builder. Both routed live. **Canonical:** shared marketing/forms primitive. **Confidence:** high, minor.

### Mapping 142 — ExcelJS style helpers `solidFill`/`thinBorder` token-identical pair (M38 family, styling layer)
- `pages/hr/hooks/exportTimesheetExcel.ts:28,32` vs `pages/financial-analytics/exportPricingModelExcel.ts:17,21` — identical bodies + private `C.borderGray` palettes. **Canonical:** `lib/excelStyles.ts` (fold into M38). **Extraction:** trivial. **Confidence:** high.

### Mapping 143 (thin) — `isLetterImageNode` type-guard pair
- `hooks/useLetterToolbarHandlers.ts:158` vs `components/create/ConsentMarkdownEditor.tsx:26` — token-identical 2-line guard. **Canonical:** export from the Lexical nodes module. **Extraction:** trivial. **Confidence:** high.

## Frontend pass-26 extensions + dead/name-collision resolutions
- **M23 ext:** 6 new `getInitials` defs (journey/AssigneeSelector.tsx:44; stock/stockAssignment.tsx:11 — **exported**, competing with rotaUtils; CoshhEditor.tsx:35; ChecklistComplianceView.tsx:73; MeetingMinutesTab.tsx:83; LogsLibrary.tsx:93). Family ~26 defs.
- **M42 ext:** `pages/messages/prototype/AutomatedMessageRow.tsx:56` ↔ `components/inbox/v2/AutomatedMessageRow.tsx:67` (diff 23 lines).
- **M80 ext:** `components/journeys/{AddPatientDialog.tsx:230, MovePatientDialog.tsx:194}` `flagFor` — MovePatient's comment claims "one source" while re-declaring it.
- **Dead (deletion resolves name collisions):** `hooks/usePatientNotes.ts` (only importer is dead NotesTab; superseded by usePatientWorkspace audit-#70 fix); `hooks/usePatient.ts` (reachable only via zero-importer `hooks/index.ts`).
- **Watch:** `components/Library/LetterTemplateEditor.tsx:22,28` no-op stubs shadowing live `lib/editorContent.ts` — name shadow only.

### Mapping 133 — Settings email-service address tabs: cached-fetch + create + delete CRUD scaffold triplicated
- `pages/settings/components/integrations/{TaskAddressesTab.tsx:52–171, InvoiceAddressesTab.tsx:39–174, WebFormForwardingTab.tsx:50–210}` — token-identical `useCachedData` wiring (3-min TTL), `createXAddress` POST orchestration, `deleteXAddress` DELETE orchestration, `copyToClipboard`, loading JSX. Divergence: empty-description guard only in Invoice/WebForm; error extraction style; window.confirm vs Dialog delete-confirm. Map cites these files only for M6/M64 fragments. **Canonical:** `useAddressTab({endpoints, noun, cacheKey})` factory or config-driven `EmailAddressManagerTab`. **Extraction:** safe. **Confidence:** high.

### Mapping 134 — Day-list "minutes since midnight" appointment-time parser: 3rd inline copy bypassing an exported one
- `pages/day-list/lib/dayWindow.ts:45–63 parseTimeValue` (unparseable → sorts LAST; documented deliberate divergence) and `lib/patientsAtRisk.ts:70–79 parseTimeValue` (unparseable → 0, sorts FIRST) are the known pair. **New bypass:** `pages/DayListPage.tsx:284–289` inline `parseTime` — token-identical to patientsAtRisk's exported parser (same 0 fallback) in a file that already imports `getRiskSortValue` from that module (:50). **Fix:** import `parseTimeValue`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 135 (thin, minor) — Day-list group-rank-then-time list comparator scaffold, 2 token-identical copies
- `pages/day-list/lib/awaitingConsent.ts:74–82` `buildConsentItems` vs `importantPatients.ts:152–161` `buildImportantItems` — identical `filter(qualifies).sort(groupRank diff, then parseTimeValue diff)`; only the rank function varies. **Canonical:** `buildGroupedDayItems(appointments, qualifies, rank)`. **Extraction:** safe. **Confidence:** high.

### Mapping 127 — `PaginatedResponse<T>` DRF envelope declared 9×, bypassing an exported canonical
- **Canonical (exported, bypassed):** `types/labs.ts:167`, importers confined to labs. **Private re-rolls (token-identical):** `hooks/useInvoices.ts:24`, `useStockIntake.ts:6`, `useStock.ts:337`, `useSupplierMappings.ts:12`, `useSupplierRules.ts:9`. **Non-generic variants:** `pages/admin/ComplianceCoshh.tsx:65`, `CompliancePolicies.tsx:68`, `ComplianceRiskAssessments.tsx:70`. **Canonical:** promote to shared `types/api.ts`, import everywhere. **Extraction:** trivial. **Confidence:** high.

### Mapping 128 — `EmailChangeConflictData` + `EmailChangeUserSummary` quartet (admin users domain)
- Token-identical 6-line pairs in `components/admin/users/UserDialogs.tsx:75,85`, `pages/admin/Users.tsx:46,56`, `PracticeUsers.tsx:56,66`, `practice-management/components/PracticeUsersSection.tsx:45,55`; all live. **Canonical:** export from `UserDialogs.tsx` or `types/adminUsers.ts`. **Extraction:** trivial. **Confidence:** high.

### Mapping 129 — labs `PatientSearchResult` trio + same-named divergent export in `useNotes`
- Token-identical loose trio: `components/labs/{AddLabCaseDialog.tsx:54, EditLabCaseDialog.tsx:47}`, `pages/compliance/components/shared/PatientLookupField.tsx:12`. Name-collision hazard: `hooks/useNotes.ts:297` exports a stricter different-contract `PatientSearchResult`. **Canonical:** one `PatientSearchResult` in `types/patient.ts`; rename/re-import the useNotes one. **Confidence:** high (trio), medium (unification).

### Mapping 130 (thin, minor) — `GenerateWelcomeMessageRequest/Response` pair, already drifted
- `contexts/TreatmentContext.tsx:116,124` vs `hooks/useTreatmentApi.ts:6,14` — Requests token-identical; **Response drifted** (hook has `error?: string`, context doesn't). **Canonical:** export from the hook layer. **Confidence:** high.

### Mapping 131 (thin, minor) — `MarketingDomainStatusResponse` verbatim pair
- `pages/settings/components/practice/PracticeCommunicationCard.tsx:185` vs `MarketingDomainConfigView.tsx:28` — identical 3 lines (the `MarketingSendingDomain` type privately duplicated too). **Canonical:** settings email-service types module. **Confidence:** high, low urgency.

### Mapping 132 (minor) — `GlobalTemplate` pair identical + same-name different-domain collision
- `pages/admin/MessageTemplates.tsx:24` vs `pages/settings/components/templates/TemplatesListView.tsx:43` — token-identical 8 lines; separate name collision: `hooks/useNoteLibrary.ts:5` exports a *different* `GlobalTemplate` (notes/pathway_steps). **Canonical:** shared type in `types/`, rename the notes one (`NoteLibraryTemplate`). **Confidence:** medium.

## Frontend pass-24 watch/dead notes
- `NotesApiResponse`/`ApiNote` token-identical pair is inside M82's wholesale clone; third copy `components/PreviousNotesTab.tsx:42,31` is **dead** (deletion candidate).
- `ApiResponse` name collision ×3 (`hooks/useApi.ts` vs TreatmentContext vs AuthContext — divergent shapes, not copies).

### Mapping 126 — Email+SMS template list fetch + projection re-implemented beside its own canonical
- **Canonical:** `hooks/useEmailSmsTemplatesCaching.ts:82–130` `fetchTemplatesData` (raw fetch + own `mapEmailTemplate`/`mapMobileTemplate` :52,64); live via pages/Settings.tsx:558.
- **Re-roll:** `hooks/useRefreshTemplates.ts:26–58` — token-identical fetch/projection against the same two endpoints, imports the canonical mappers but re-rolls the fetch via `fetchWithAuth`; error detail uses `statusText` vs canonical `status` (drifted). Live via Settings.tsx:598→847,858. **Effectively dead third:** `hooks/useFetchTemplates.ts:41–82` re-rolls fetch AND mappers inline; only reference is an unused import (Settings.tsx:62); shims `pages/settings/settingsImports.ts:80` and `pages/settings/hooks/useFetchTemplates.ts` have no importers.
- **Risk:** same templates fetched through two divergent code paths for the same Settings screen. **Canonical:** the caching hook; `useRefreshTemplates` calls its refetch; delete the dead hook + shims. **Extraction:** safe. **Confidence:** high.

## Frontend pass-23 dead-clone notes (deletion candidates, watch)
- `pages/recalls/components/administration/{AtRiskSection, LastVisitSection}.tsx` — **dead** (commented imports only in RecallsAdministration.tsx:25,27,280,286); each token-duplicates its live admin twin (`AdminAtRiskSection.tsx`, `AdminLastVisitSection.tsx`, live via PracticeWorkspace.tsx:858,866) incl. the at-risk `MODIFIERS` catalog and the visit-bucket codec. Same pattern as the pass-16 SegmentsSection note; if revived, these become a new admin↔recalls clone pair extending M54/M90/M91.

### Mapping 107 — `KPICard` stat-tile component, 4 verbatim copies in spend-reporting
- `components/spend-reporting/{InvoiceKPICards.tsx:29, LabKPICards.tsx:29, OverviewKPICards.tsx:29, AssetsKPICards.tsx:31}` — pairwise diffs = 0 (53 lines each: alert-aware border/tone, trend arrow, skeleton). **Canonical:** shared `KPICard` export in the directory. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 108 — COSHH/risk `getRiskLevelConfig` level→badge map, 3 verbatim + 2 near-variants
- Verbatim (diff=0): `pages/compliance/CoshhView.tsx:38`, `components/documents/CoshhTab.tsx:48`, `CoshhPreviewModal.tsx:12` (identical 14-line switch low→emerald/medium→amber/high→orange). Near-variants: `RiskAssessmentView.tsx:49` (+description), `RiskAssessmentEditor.tsx:56` (via `RISK_LEVELS` find). All live. **Canonical:** one `getRiskLevelConfig(level)` in compliance shared module. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 109 — `determineMessageType` assistant chat classifier, verbatim pair
- `components/assistant-toolbar/ChatPanel.tsx:171` vs `components/chat/ChatAgent.tsx:91` — diff = 0 (23 lines). **Canonical:** shared lib export. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 110 — Audit-log `FieldDiffRow` render, 3 copies (2 verbatim)
- `components/invoices/FinanceAuditView.tsx:171` vs `pages/hr/HRAuditLog.tsx:137` (diff = 0); `components/patients/workspace/PatientAuditLogTab.tsx:122` (adds justified `masked` prop). Plus duplicated `SessionCard` scaffolds (type renames only). **Canonical:** shared `AuditLogPresentational` set. **Extraction:** safe. **Confidence:** high.

### Mapping 111 — `isStaffOnLeave` leave-check, 3 copies in HR rota
- `pages/hr/Rotas.tsx:359`, `components/rota/WeeklyRotaAssignmentPanel.tsx:20` (diff = 0), `RotaAssignmentPanel.tsx:322`; all live. **Canonical:** `pages/hr/utils/` export. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 112 — "add_once" invisible-character template-marker codec, verbatim pair (**breakage risk**)
- `components/ProcedurePathways.tsx:71–140` vs `pages/create/notes/components/TreatmentWorkflowSheet.tsx:65–134` — identical `ADD_ONCE_BIT_*` constants, `\u2063/\u2064` markers, `encodeAddOnceKey`/`decode`; both live (encode at :542,598 / :495,508). **Risk:** a scheme change in one silently breaks the other's template recognition. **Canonical:** `lib/addOnceMarker.ts`. **Extraction:** safe. **Confidence:** high.

### Mapping 113 — Lexical rich-text `theme` object, verbatim pair
- `components/modals/components/ContentEditor.tsx:182` vs `pages/create/patient-report/components/ReportTextarea.tsx:396` — diff = 0 (29 lines). **Canonical:** shared `lexicalTheme` export. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 114 — `groupPracticesQuery` duplicated beside a richer canonical, with stale-practice-unsafe key
- `pages/compliance/PolicyEditor.tsx:89` vs `RiskAssessmentEditor.tsx:104` — identical except isHead flags; both use `queryKey: ['practiceConnection','groupPractices']` (no practice-id key) while `hooks/useGroupCompliance.ts:66` is the properly-keyed hook version. **Canonical:** the existing hook. **Extraction:** safe. **Confidence:** high.

### Mapping 115 — `TabButton` compliance-library tab, 3 verbatim copies
- `pages/compliance/components/logs/LogsLibrary.tsx:112`, `components/documents/DocumentsLibrary.tsx:32`, `components/checklist/ChecklistTemplateLibrary.tsx:1295` — diff = 0 (20 lines each), all rendered. **Canonical:** shared export. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 116 — `insertTextAtCursor` Lexical cursor-splice, verbatim pair (third copy dead)
- `components/letter-library/EditTemplate.tsx:488` vs `pages/create/notes/components/NotesTextarea.tsx:201` — diff = 0 (22 lines); consumed via the ref protocol used by `hooks/useAudioRecording.ts:1008`. Dead third copy in `ContentEditor_backup.tsx:478`. **Canonical:** shared helper for the ref protocol. **Extraction:** safe. **Confidence:** high.

### Mapping 117 — `getPageNumbers` pagination-windowing function, 7+ verbatim copies + unmapped sibling handlers
- `components/ContactsTable.tsx:578`, `components/compact/{IntakeTable.tsx:869, NurtureTable.tsx:1223, OpenTable.tsx:899, ActiveTable.tsx:849}`, `ArchiveTable.tsx`, `DataQualityIssuesScreen.tsx:183` (comment: "Mirrors CustomJourneyTable.getPageNumbers exactly"), `CustomJourneyTable.tsx:810`. Same 35-line ellipsis-windowing body. Related unmapped verbatim siblings in the same files: `handleSelectAll` (OpenTable:676 vs IntakeTable:755) and `handleSort` (OpenTable:887 vs ActiveTable:623). **Canonical:** `getPageNumbers(current,total,max=5)` in a shared table lib (fold into M16). **Extraction:** safe. **Confidence:** high.

### Mapping 118 — `useIsMobile` resize-listener re-rolls, 4 new sites with two divergent breakpoints
- `pages/compliance/components/audit/AuditLibrary.tsx:63` and `components/calendar/CalendarGrid.tsx:29` (**640px**); `components/training/TrainingMatrix.tsx:48` and `pages/hr/components/records/HRRecordsMatrix.tsx:46` (**768px**) — pairwise diff = 0; all bypass canonical `hooks/use-mobile.tsx`. Extends M57. Same-named hook, different semantics. **Fix:** parameterize + adopt the shared hook. **Confidence:** high.

### Mapping 119 — Local `useDebounce` hook duplicated in compliance admin pages
- `pages/admin/CompliancePolicies.tsx:95` vs `ComplianceRiskAssessments.tsx:138` — identical 8-line 300ms hook. Extends M14 with a "local hook re-declaration" variant. **Canonical:** `hooks/useDebounce.ts`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 120 — QR-scanner video-stream attach effect, verbatim pair
- `components/UnifiedQRScanner.tsx:139` vs `components/stock/StockQRScannerCard.tsx:104` — identical effect (srcObject attach/play/cleanup), diff = 0. **Canonical:** `useVideoStream(videoRef, stream)`. **Extraction:** safe. **Confidence:** high.

### Mapping 121 — Procedures listAll fetch effect duplicated in compact tables
- `components/compact/IntakeTable.tsx:528` vs `NurtureTable.tsx:635` — identical `loadProcedures` effect (diff = 0); similar Open/Active siblings. **Canonical:** `useProceduresOptions()` hook. **Extraction:** safe. **Confidence:** high.

### Mapping 122 — `PracticeLogo` picture-fallback component duplicated
- `pages/MarketingPreferencesPage.tsx:27` vs `pages/confirmation/ConfirmAppointmentView.tsx:99` — identical fallback asset + `<picture>` logic (formatting only). **Canonical:** shared `PracticeLogo` in `components/branding/`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 123 — Shared-practice fetch effect duplicated in HR
- `pages/hr/Leave.tsx:693` vs `pages/hr/Schedules.tsx:227` — identical cancelled-guarded `PracticeSetting()` → `logo_data_url` effect. **Canonical:** `usePracticeLogo()` hook (also serves M122). **Extraction:** safe. **Confidence:** high.

### Mapping 124 — `_field` JSON-schema factory duplicated across lib
- `lib/autoExam.ts:254` vs `lib/assessmentNote.ts:144` — diff = 0 (value/confidence schema). **Canonical:** export from one lib module. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 125 (thin, minor) — `handleBack` history-fallback navigation duplicated
- `pages/ContactProfilePage.tsx:91` vs `PatientWorkspacePage.tsx:519` — same state.from → navigate / `history.state.idx` back-or-fallback logic. **Canonical:** `useSmartBack(fallback)` hook. **Confidence:** medium.

## Frontend pass-22 mapped-family extensions (member growth only)
- **M69 `PaginationButton`:** ~9 new verbatim members (ContactsTable.tsx:147, Library/TemplatesTable.tsx:86, stock/ImplantsTable.tsx:27, pages/Invoices.tsx:62, pages/Labs.tsx:232, GlobalWorkflowsBrowser.tsx:101, pages/automation/Automation.tsx:107, settings/assets/ImagesDataTable.tsx, ReviewsDataTable.tsx) — family grows to ~18.
- **M20 ext:** `mergeTranscriptSegments` verbatim (useScribeRecorder.ts:60 ↔ useAudioRecording.ts:35).
- **M45/M26 ext:** `timeAgo` verbatim (recalls SyncSection.tsx:41 = integrations/ReconciliationPanel.tsx:20).
- **M104 ext:** StockItemSheet.tsx:1960 click-outside effect verbatim vs AddImplantDialog.tsx:80.

### Mapping 103 — Compliance category-fetch hook cloned verbatim (audit vs checklist `?type=` param)
- `pages/compliance/hooks/useAuditCategories.ts` (280 ln) vs `useChecklistCategories.ts` (269 ln) — normalized diff shows only comments, a color-constant name, and the `?type=audit` vs `?type=checklist` query value (:106 / :101). Token-identical `FALLBACK_COLORS`, `COLOR_ROTATION`, return-shape interface, `categoryMap` useMemo, hash-rotation `getCategoryColors`. Both live, powering the M77 editor-clone pair's callers. **Canonical:** single `useComplianceCategories(type)`. **Extraction:** safe. **Confidence:** high.

### Mapping 104 — Implant-manufacturer autocomplete scaffold duplicated between stock dialogs
- `components/stock/AddImplantDialog.tsx` (filter :55–61, keyboard :110–137, click-outside :80–94, dropdown `implant-manufacturer-option-*` at :249) vs `BulkAddImplantDialog.tsx` (:123–129, :141–170, :83–94, :339 — row-keyed variant). Divergence: suggestion cap 10 vs 5; Escape closes all vs single row. **Canonical:** `useManufacturerAutocomplete(names)` or row-aware `ManufacturerCombobox`. **Extraction:** safe. **Confidence:** high.

### Mapping 105 (minor) — HR record-type ordering array re-declared 3× beside a canonical constants module
- Identical 8-entry `RECORD_TYPES: HRRecordType[]` arrays: `pages/hr/hooks/useHRRecords.ts:58`, `pages/hr/components/records/HRRecordsMatrix.tsx:117`, `search/HRDocumentsDialog.tsx:147` — while `pages/hr/constants.ts:24 HR_RECORD_TYPE_LABELS` is the canonical module these same areas import. Contrast: `pages/documents/prototype/DocumentsUploadPrototypePage.tsx:53` correctly derives via `Object.keys(HR_RECORD_TYPE_LABELS)`. **Risk:** 9th record type added → drift. **Fix:** export `HR_RECORD_TYPES` from constants. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 106 (thin, minor) — Documents search-dialog scaffold pair (`DIALOG_TABS`/`KIND_ICONS`/`STATUS_LABELS`/`STATUS_COLORS`)
- `pages/compliance/components/search/ComplianceDocumentsDialog.tsx:103–129` vs `pages/hr/components/search/HRDocumentsDialog.tsx:103–145` — structurally identical constant blocks (tab list, kind→icon, status→label/color); status vocabularies genuinely divergent (different domains, M5's same-domain rule not met — the scaffold is the duplication). **Canonical:** shared `DocumentSearchDialogFrame` constants. **Confidence:** medium.

### Mapping 101 — Compliance "review due within 30 days" rule: magic number 30 hardcoded in 5 files, bypassing the domain constant
- **Canonical constant exists but unused:** `pages/compliance/constants.ts:226 EXPIRY_WARNING_DAYS = 30` (+ `EXPIRY_CRITICAL_DAYS = 14`, used only by useCalendarEvents). **Re-hardcoded `30`:** `pages/compliance/CoshhView.tsx:53–76` (ReviewBadge), `components/documents/CoshhTab.tsx:67–73` + `:140–141,157–158` (`addDays(new Date(), 30)` filters), `RiskAssessmentView.tsx:111–114`, `components/documents/RiskAssessmentsTab.tsx:176–177,196–197`, `components/policies/PoliciesLibrary.tsx:335,357`.
- **Divergence:** `Compliance.tsx:767–771` computes `documentsNeedingReview` as overdue-only (no 30-day horizon) → top-level tab badge disagrees with the library tab on "due for review". **Canonical:** `isReviewDueSoon(date)`/`REVIEW_DUE_WINDOW_DAYS` in compliance constants; align or document the overdue-only badge. **Extraction:** safe. **Confidence:** high.

### Mapping 102 — Superuser permission matrix (6-claim JWT predicate) third verbatim copy in Login.tsx
- `components/AdminProtectedRoute.tsx:15–45` (full 6-claim matrix, cited in M18 only as the decode pair) vs `pages/admin/Login.tsx:25–45` — **character/token-identical entire block**, drives post-login redirect; `lib/analytics/isSuperuser.ts:15–43` is a documented narrower variant (justified). **Canonical:** `decodeJwtPayload` + `isSuperuserClaimSet(payload, {includeStaff})` in lib (M18 extraction, now 3 consumers). **Extraction:** safe. **Confidence:** high.

## Frontend pass-20 watch notes (below bar)
- Accepted-image MIME allowlist + HEIC extension fallback triplicated (`components/assets/AddAssetDialog.tsx:27`, `AssetPhotoManager.tsx:32`, `PracticeCommunicationCard.tsx:693`) — M92 extension (the allowlist is the policy).
- Payslip/invoice approval derived two ways (AssociatePayView `canViewAll && hasFeature` with justifying comment vs InvoiceForm backend prop) — documented intentional divergence.

### Mapping 95 — Settings snapshot loader scaffold — `useFetch*Data` hook suite cloned 5×
- `hooks/useFetchAccountData.ts:18–65` vs `useFetchTreatmentData.ts:18–71` — after renaming domain tokens the entire skeleton is identical (`data` + `initialData` dirty baseline + `isLoadingX` triple, same `setIsLoading(true)/try/ok-check/finally` flow; only endpoint and field projection differ). Same scaffold: `useFetchPreferencesData.ts` (82 ln), `useFetchTeamData.ts` (80 ln); `useFetchPracticeData.ts` (172 ln) is a half-migrated hybrid. All live via `pages/Settings.tsx:55–59` + compact tables + HR. **Canonical:** `useSettingsSnapshot<T>(endpoint, defaultData, project)`. **Extraction:** safe. **Confidence:** medium-high.

### Mapping 96 (minor) — `formatPayMonthLabel` verbatim duplicate in invoices
- `components/invoices/AssociatePayView.tsx:473–476` vs `AssociatePayPersonalArchive.tsx:46–49` — character-identical (`new Date("${payMonth}-01T00:00:00")` + date-fns "MMMM yyyy"). **Canonical:** single export in `components/invoices/`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 97 (minor) — `formatMonthLabel` (YYYY-MM → "January 2025"), 4 live copies, divergent formatting
- `pages/admin/practice-management/components/UsageSection.tsx:41–48` (en-GB), `pages/settings/components/account/UsageTab.tsx:27–33` and `settings/sms-phone-config/PracticeConfigModal.tsx:446–449` (`toLocaleString(undefined)` — **locale-dependent in a UK app**), `pages/hr/components/rota/CopyMonthPopover.tsx:35–37` (date-fns). **Canonical:** `formatMonthKey(key)` in `utils/dateUtils.ts`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 98 (minor) — Template-type label formatter: canonical export bypassed + token-identical lookup pair
- Canonical `pages/settings/components/templates/TemplateListPrimitives.tsx:7–12 formatTemplateType` bypassed by `pages/admin/MessageTemplates.tsx:299–304` (re-roll, missing `.filter(Boolean)`); token-identical `formatTemplateTypeLabel` pair `MessageTemplates.tsx:821–826` ↔ `pages/settings/components/templates/TemplateEditorDialog.tsx:162–167`. **Canonical:** export `formatTemplateTypeLabel(value, templateTypes)` beside the primitive. **Extraction:** safe. **Confidence:** high.

### Mapping 99 (minor) — Asset "next service due" ladder duplicated
- `components/assets/AssetsContent.tsx:388–406` vs `SwipeableAssetListItem.tsx:103–137` — same regulatory/recommended decision ladder with identical overdue text; divergent Tailwind sizes and decommissioned fallback. **Canonical:** `assetNextService(asset)` in assets directory. **Confidence:** high, minor.

### Mapping 100 (minor) — `formatHours` HH:MM formatter, hr timesheets
- Live in-file duplicate: `pages/hr/components/timesheets/TimesheetWeeklySummary.tsx:72–84` and `:417–425` (two near-identical signed HH:MM formatters). Dead cluster: `TimesheetTable.tsx:28` (zero importers), `TimesheetTableRow.tsx:33–37`, `TimesheetSummaryCards.tsx:18–22` — deletion candidates. **Canonical:** `formatHoursHHMM(hours, {nullFallback})` in `pages/hr/utils/`. **Confidence:** high.

## Frontend pass-19 watch notes
- `pages/settings/components/integrations/ReconciliationPanel.tsx:20–29` private `timeAgo` — new site for the M45/M26 relative-time family.
- `pages/tasks/components/TaskRow.tsx:13` imports toast from legacy `@/hooks/use-toast` — unmapped M7 site.
- Dead: `pages/hr/hooks/index.ts:5` re-exports nonexistent `formatHoursMinutes` from useTimesheets; index has zero importers — deletion candidate.

### Mapping 90 — Opportunity-rules validate + payload-projection cloned admin ↔ day-list (M54 pattern on a new config surface)
- `pages/admin/practice-management/components/AdminOpportunityRulesSection.tsx:224–259` vs `pages/day-list/components/administration/OpportunityRulesSection.tsx:320–360` — token-identical 9-element integer-threshold validation + toast string + ~24-field projection (differs only in `f.`/`form.` prefix). Divergence: admin throws Error; day-list toasts + returns false. **Canonical:** shared `validateOpportunityRules`/`projectOpportunityRules`. **Extraction:** safe. **Confidence:** high.

### Mapping 91 — Metrics-thresholds validate + projection cloned admin ↔ recalls (sibling of M90/M54)
- `pages/admin/practice-management/components/AdminMetricsSection.tsx:199–215` vs `pages/recalls/components/administration/MetricsSection.tsx:219–232` — token-identical guard + toast + payload keys. **Canonical:** `validateMetricThresholds(values)` folded into M54's extraction. **Confidence:** high.

### Mapping 92 — HEIC guard + decode + toast wrapper re-rolled ~24–27× beside the canonical `lib/heicConversion.ts`
- The lib exports primitives (`ensureBrowserRenderableImage`), but every caller re-rolls the guard+decode+toast (`'Could not process that HEIC photo. Please convert it to JPEG/PNG first.'` in 27 files): `components/assets/AddAssetDialog.tsx:189–197`, `pages/create/notes/components/NotesTextarea.tsx:169–176`, `hooks/useImageHandling.ts:58,88`, InlineLexicalEditor, ContentEditor, PracticeLogosUploader, GlobalAssets, ComplianceAudit/ChecklistTemplates, ChecklistEditor, settings/Practice, settings/Assets, GeneralTab, and more. Behavioral divergence: return vs `return []` vs skip-continue. **Canonical:** `importRenderableImage(file): Promise<File | null>` in `lib/heicConversion.ts`. **Extraction:** safe. **Confidence:** high. (Wider than M40, which mapped only the 4-file template-editor scaffold.)

### Mapping 93 (minor) — `validateDomain` regex + message triplicated
- Character-identical `domainRegex` + message `"Please enter a valid domain name (e.g., example.com)"` in `components/email-service/RegisterDomainDialog.tsx:30–34`, `pages/settings/email-service/dialogs.tsx:35–39`, `pages/settings/components/practice/AddDomainForm.tsx:31–35` (setError vs toast.error divergence). **Canonical:** `isValidDomain(value)` in lib. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 94 (minor) — `SendConsentModal.fetchTemplates` bypasses existing canonical hook
- `components/consent/SendConsentModal.tsx:196–210` token-identical to `hooks/useActiveConsentTemplates.ts:31–53` (fetch → results unwrap → transform → isActive filter → same catch toast). The hook exists and siblings use it. **Fix:** call the hook. **Extraction:** safe, trivial. **Confidence:** high.

## Frontend pass-18 watch notes (below bar)
- `"Network error. Please check your connection."` restated ~26× (AuthContext ×10, TreatmentContext ×9, +4 files) — constant restatement; watch only.
- `"Network error. Please try again."` duplicated across recall-cache scaffolds (noted in M63).

### Mapping 69 extension (pass 17) — `PaginationButton` family grows 7 → 9
- `components/data-quality/DataQualityIssuesScreen.tsx:25–27` (header comment: "visual parity with CustomJourneyTable/IntakeTable's PaginationButton") and `pages/hr/components/records/HRRecordsMatrix.tsx:60`.

### Mapping 85 — Task-assignment primitive suite cloned between tasks and treatment-plans (extends M56 from 1 to 5 components)
- `pages/tasks/components/task-creation/{TaskDescription, PrioritySelection, StaffAssignment, PatientSelection}.tsx` vs `pages/treatment-plans/components/{TaskDescription.tsx (diff = one label string), PrioritySelection.tsx (same 3-option structure; shadcn Badge variant vs hardcoded classes), StaffAssignment.tsx, patient-selection/PatientSelection.tsx}` — shared filter/dropdown/select blocks copy-paste; tasks side adds server-side search + self-assign (justified divergence). Both sides live via CreateTaskSheet.tsx:21–24 and TaskAssignment.tsx:3–6. **Canonical:** shared task-primitives module parameterized by search mode + style mechanism. **Extraction:** safe. **Confidence:** high.

### Mapping 86 — HR leave-year `getDefaultDateRange` verbatim duplicate
- `pages/hr/components/reporting/MaternityPaternityReport.tsx:26–42` vs `SicknessAnalyticsReport.tsx:22–38` — 17-line token-identical (fiscal-year rollover via `customYearStart`); related inline logic `pages/hr/Admin.tsx:1336–1348`. **Canonical:** `getDefaultLeaveYearRange(leavePolicy)` in `pages/hr/utils/`. **Extraction:** safe. **Confidence:** high.

### Mapping 87 (minor) — Call-agent call-result badge + duration formatters duplicated in-callagent
- `pages/callagent/CallAgentTable.tsx:37–63` vs `CallDetailsDialog.tsx:127–158` — `getStatusBadge` switch token-identical. Duration formatter 3× with divergent formats: CallAgentTable:33–38 (`m:ss`), CallDetailsDialog:124–127 (`Xm Ys`), CallStatsCards.tsx:14–24 (h/m/s ladder). **Canonical:** `pages/callagent/lib/callFormat.ts`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 88 — Auth password-reset/verify screen suite, 3 live routed clones
- `pages/auth/VerifyOTP.tsx` (546 ln) vs `ResetPasswordVerify.tsx` (328) vs `ResetPasswordSet.tsx` (243) — literal comments "Shared right-panel gradient (mirrored from VerifyOTP.tsx)" in both Reset files; identical password-submit error-mapping blocks (same strings, weak_password/token branches) and 6-digit OTP validation. **Canonical:** shared `AuthShell` + `useSetPassword()` helper. **Extraction:** safe. **Confidence:** high.

### Mapping 89 (minor) — Magic-login request POST block triplicated (2 live in-file + 1 dead)
- `pages/auth/MagicLoginRequest.tsx:52–60` vs `:237–245` (identical raw-fetch POST blocks, both live); third copy `pages/auth/VerifyMagicLogin.tsx:149–170` — **dead (zero importers, not in any routes file; deletion candidate)**. Public token-less lanes (M1-justified) but duplication among themselves merits a `requestMagicLogin(email)` helper. **Confidence:** high.

### Mapping 77 — `ComplianceAuditTemplates` ↔ `ComplianceChecklistTemplates` near-total editor clone (~1,700 lines each, ~95% identical) — largest frontend family
- `pages/admin/ComplianceAuditTemplates.tsx` (1,715 ln) vs `ComplianceChecklistTemplates.tsx` (1,666 ln): normalized diff shows only ~100 differing lines; `newSection` objects, fetch/save/delete effects, "ensure proper structure" submit block, CQC helper text all token-identical modulo endpoints/strings. Divergence: audit adds matrix-question support (388–404). Both live (routes/admin.routes.tsx:268,273); map previously cited these files only for M40's image guard. **Canonical:** config-driven `TemplateEditor({endpoints, categorySource, allowMatrix})`. **Extraction:** safe but large. **Confidence:** high.

### Mapping 78 — Stripe billing panel triplicated (`BillingTab`/`PracticeBillingTab`/`PlansAndBilling`)
- `pages/settings/components/account/BillingTab.tsx:88–420`, `practice/PracticeBillingTab.tsx:89–430`, `pages/settings/PlansAndBilling.tsx:77–300` — `handleCheckoutSession`/`handleOpenCustomerPortal`/`handleCancelSubscription` token-identical (differ only in `page=` URL param and toast wording); plan-card JSX identical incl. hardcoded `bg-[#6C5ACF]...` classes; invoice-row block identical. All three live. **Canonical:** `useStripeBilling({context})` + shared `PlanCard`/`InvoiceRow`/`ManageSubscriptionDialog`. **Extraction:** safe. **Confidence:** high.

### Mapping 79 — Phone-country list builder + flag dial-code Combobox, 4 near-identical copies
- `components/intake/ManualIntakeForm.tsx:87–122,411–448`; `pages/settings/components/account/GeneralTab.tsx:64–105,326–363`; `pages/settings/components/PersonalInformationCard.tsx:52–88,354–403`; `pages/settings/Practice.tsx:58–108,630–679` — ~37-line combobox JSX token-identical in all four. Divergence: only GeneralTab wraps the builder in `useCachedData` (24h); others rebuild per mount. **Canonical:** `buildPhoneCountries()` + `usePhoneCountries()` + `PhoneCountryCombobox`. **Extraction:** safe. **Confidence:** high.

### Mapping 80 (minor) — `getFlagEmoji` re-rolled 8×
- Identical code-point ladder in `components/AddPatientModal.tsx:84`, `compact/AddPatientModal.tsx:588`, `components/intake/ManualIntakeForm.tsx:98`, `patients/ContactInputs.tsx:48`, `patients/patient-panel/PatientEditableFields.tsx:68`, `pages/settings/Practice.tsx:58`, `PersonalInformationCard.tsx:63`, `account/GeneralTab.tsx:64`. **Canonical:** `utils/textUtils.ts`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 81 (minor) — `getProcedureColor` canonical bypass + "Associated Procedures" chip picker cloned 3×
- `pages/settings/utils/procedureUtils.ts:18` exports `getProcedureColor`, yet `pages/settings/Assets.tsx:778` re-declares the same ladder; chip-toggle JSX token-identical across `Assets.tsx:1538–1560`, `EditImageForm.tsx:175–197`, `EditReviewForm.tsx:286–308`. **Canonical:** import from `procedureUtils.ts` + shared `ProcedureTagsPicker`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 82 — `LetterNotesWorkflowSheet` ↔ `PatientReportWorkflowSheet` sibling clone
- `pages/create/notes/components/LetterNotesWorkflowSheet.tsx` (307 ln) vs `pages/create/patient-report/components/PatientReportWorkflowSheet.tsx` (285 ln) — ~114 differing lines; LetterNotes comments admit the copy; already imports shared `SelectableChip`/`SelectedItemsSummary`. Divergence: LetterNotes resets selections on open; different toast strings. **Canonical:** one `WorkflowSheet({source, endpoints, labels})`. **Extraction:** safe. **Confidence:** high.

### Mapping 83 (minor) — Admin "assign subscription plan / custom plan" form cloned
- `pages/admin/Users.tsx:~1380–1460` vs `PracticeUsers.tsx:~1560–1650` — token-identical radio pair + custom-plan field block (same placeholders incl. "0.00 (no charge for access-only)"). Distinct from M25's guest handler in the same files. **Canonical:** shared `PlanAssignmentFields`. **Extraction:** safe. **Confidence:** high.

### Mapping 84 (thin, minor) — Team-chat header + toggleable member-search bar
- `components/teamChat/TeamChatGroupForm.tsx:92–130` vs `TeamChatUpgradeToGroupPanel.tsx:55–93` — token-identical ~20-line header/search bar except placeholder. **Canonical:** shared `MemberSearchBar`. **Confidence:** high, low urgency.

## Frontend pass-16 exclusions/dead code
- Dead: `pages/recalls/components/administration/SegmentsSection.tsx` (commented import only) — but live sibling `AdminSegmentsSection.tsx` token-duplicates its `DEFAULT_CONFIG`/`MAPPING_ORDER`/`derived` useMemo; the live copy becomes canonical if the dead one is deleted.
- Below bar: `CopyMonthPopover` ↔ `CopyRotaPopover` (mirrored state machine, genuinely divergent semantics); `DocumentAssistant` ↔ `LetterAssistant` (~25% scattered identical, not wholesale).

### Mapping 69 — `PaginationButton` component cloned 7× across compact tables
- Token-identical: `components/compact/{IntakeTable.tsx:182, OpenTable.tsx:186, ActiveTable.tsx:183, NurtureTable.tsx:278}`, `components/ArchiveTable.tsx:47`, `components/compact/CustomJourneyTable.tsx:313` (comment: "visual parity with IntakeTable's PaginationButton"). Divergent 7th: `pages/Compact6.tsx:111`. **Canonical:** shared `PaginationButton` in `components/ui/` or `components/compact/`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 70 — `formatDate` Today/Yesterday ladder, 3 copies in compact tables
- `components/compact/IntakeTable.tsx:1062–1098`, `CustomJourneyTable.tsx:220–256` (token-identical ladder + NaN guards), `JourneyMobileCard.tsx:177–200` (variant without tooltip). **Canonical:** `formatCompactDayDate(dateString)` shared lib. **Extraction:** safe. **Confidence:** high.

### Mapping 71 — M13 extension: `DATE_PRESETS`/`getDateRangeFromPreset` now 5 live private copies (3 unmapped)
- Token-identical pair: `components/invoices/SupplierPayView.tsx:78–107` ↔ `AllocationsView.tsx:65–94`; variant: `pages/compliance/components/documents/SigningBatchesTab.tsx:88–104` (+ own `dateRangeToParams`). Fold into M13's `utils/dateUtils.ts` canonical.

### Mapping 72 (minor) — `currencySymbolMap` (`GBP:£, USD:$, EUR:€`) duplicated
- `components/compact/IntakeTable.tsx:143–147` (used :520) vs `NurtureTable.tsx:92–96` (used :627) — identical incl. lookup line. **Canonical:** export beside `formatCurrency` in `types/invoice.ts` or utils. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 73 (minor) — product-analytics duration formatter, 4 copies
- `pages/admin/product-analytics/PracticeDrilldown.tsx:26–30` ↔ `UserDetail.tsx:11–15` (character-identical `formatMinutes`), `StatTiles.tsx:13–17 formatDuration` (same body), `PracticesTable.tsx:12` (truncating variant, no hour escalation — divergent). **Canonical:** `formatDuration(ms)` in the directory's shared module. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 74 — Contexts: `LoadingContext` ↔ `RightOverlayContext` token-identical ref-Set stack scaffold
- `contexts/LoadingContext.tsx:58–86` vs `RightOverlayContext.tsx:23–52` — identical bodies except identifiers; both live. **Canonical:** generic `useIdStack()` hook. **Extraction:** safe. **Confidence:** high.

### Mapping 75 — M1-family extension: named `getAuthHeaders` re-rolled 5×
- `contexts/TreatmentContext.tsx:226–233` ↔ `hooks/useTreatmentApi.ts:33–40` (token-identical useCallbacks); `pages/settings/Assets.tsx:100–103` ↔ `pages/settings/components/ReviewEditor.tsx:35–38` (identical plain functions); `pages/settings/assets/AddReviewSheet.tsx:40–43` (multipart variant). **Canonical:** fold into `useFetchWithAuth`. **Confidence:** high.

### Mapping 76 — Python-list-string `treatment_type` parser, 2 divergent implementations
- `components/ArchiveTable.tsx:61–88` `parseTreatmentType` (bracket/quote strip, per-item `'Not specified'` filter, try/catch) vs `pages/Journeys.tsx:883–892` inline (no filter/try-catch — literal `'Not specified'` survives into filter options). **Canonical:** export ArchiveTable's parser to `utils/listFieldUtils.ts`. **Extraction:** safe. **Confidence:** high.

## Frontend pass-15 observations (below bar)
- financial-analytics `fmtPrice` reimplements `fmtGBPAbbreviated`'s k-branch; `—`-guard repeated 5× — drifting toward a `guarded(formatter)` combinator.
- Near-clone: `components/invoices/InvoiceCommentsSection.tsx` vs `PayslipDraftCommentsSection.tsx` (identifier renames only; separate backends) — candidate `CommentThreadSection({hook, sourceType})`.
- Dead: `contexts/MockOnboardingContext.tsx` (zero external importers) — deletion candidate.

### Mapping 64 — `copyToClipboard` re-rolled ~19× (largest small-utility family)
- Token-identical pairs/triples: `pages/settings/email-service/dialogs.tsx:150` ↔ `DomainCard.tsx:53` ↔ `components/email-service/DNSConfigurationDialog.tsx:30`; `pages/settings/EmailService.tsx:427` ↔ `pages/settings/email-service/InboundParseSettings.tsx:30`; `DomainConfigView.tsx:235` ↔ `MarketingDomainConfigView.tsx:89` ↔ `PracticeCommunicationCard.tsx:605`. Same shape, divergent toast/state: `settings/components/integrations/{EmailAddressesTab:127, TaskAddressesTab:152, InvoiceAddressesTab:150, WebFormForwardingTab:214}` (no toast), `WebhooksTab.tsx:926` (toast only), `pages/admin/APIKeysPage.tsx:248` (legacy shadcn toast API), `PracticeDomainDetail.tsx:181` (no toast). **Copy bug:** `components/settings/sms-phone-config/ProvisionPhoneModal.tsx:47` webhook fallback contains typo'd domain `app.pathway,dental` (PracticeConfigModal:151 has it correct). **Fallback divergence:** `pages/DiaryPage.tsx:1498–1508` and `ConsentSigningLink.tsx:15–24` add `execCommand('copy')`; most others fail silently. `components/toolbar/CopyButton.tsx` exists as UI-level canonical but is bypassed.
- **Canonical:** shared `useCopy()` hook or `lib/clipboard.ts`. **Extraction:** safe. **Confidence:** high.

### Mapping 65 — `titleCase` (snake/slug → Title Case), 5 live copies, divergent separators/casing
- `pages/admin/RolePermissions.tsx:475` (`[_-]` split), `PracticeDomainDetail.tsx:32` (`-` only), `contexts/AuthContext.tsx:465–472` (`_` only), `utils/formatSourceLabel.ts:35–41` (also lowercases remainder — only idempotent variant), `utils/errorUtils.ts:171–173` (inline field names). **Canonical:** `titleCase(value, {separators})` in `utils/textUtils.ts`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 66 (minor) — first-letter `capitalize`, 2 named defs + ~30 inline idioms
- Identical named defs: `components/patients/workspace/PatientDentalChart.tsx:295`, `pages/signing/SigningPage.tsx:90`; inline `charAt(0).toUpperCase()` idiom in ~30 sites. **Canonical:** `capitalize` in `utils/textUtils.ts`. **Confidence:** high (existence), low urgency.

### Mapping 67 (minor) — truncate-with-ellipsis cluster + canonical bypass
- Canonical `utils/textUtils.ts:7 truncateText` bypassed by `components/PreviousNotesTab.tsx:91–94`; inline ternaries in ~12 sites (admin/Prompts.tsx:343, settings/Notes.tsx:354,449, settings/Treatments.tsx:434, Compact4.tsx:151,173, inbox/utils.ts:239, BulkAddStockDialog.tsx:508, …) with thresholds 20–100, some non-null-assuming. **Canonical:** `truncateText(text, max)`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 68 (thin, minor) — `sleep`/`delay` re-rolled
- Named identical: `hooks/useJourneyBoard.ts:40` and `lib/assistantApi.ts:153`; inline `new Promise(r => setTimeout(r, ms))` pervasive. **Canonical:** `sleep(ms)` in `lib/`. **Confidence:** high (existence), borderline for promotion.

## Frontend pass-14 extensions
- **M38 ext (slugify):** 2 new unmapped re-rolls — `pages/admin/practice-management/components/AdminSpendSection.tsx:48` (adds non-empty fallback) and `hooks/usePlaceholders.ts:41`.

## Frontend pass-14 clean checks
- Auth-token reads: all consumers route via `TokenManager.getToken()`; no raw localStorage token reads outside AuthContext. `deepClone/groupBy/formatBytes/pluralize/escapeHtml/isEmpty/chunk/unique` — single implementations or one-line idioms; no families.

### Mapping 62 — Dentally sync-progress WebSocket lifecycle, 3 live copies (2 in one file)
- Connect raw `new WebSocket` to `/ws/dentally/<channel>/${practiceId}/?token=`, parse frames, project progress with `|| 0` defaults, 3s auto-dismiss on completed: `pages/day-list/hooks/useDayListData.ts:116–170` (trigger path) **and** `:325–372` (mount effect — near-identical in-file copy, same shape as M51); `pages/recalls/useRecallSync.ts:57–165` (adds 15s ping/keepalive, resume states). Divergence: token encoding (encodeURIComponent vs raw), `get_status` request, WS close-on-terminal. Dead 4th copy: `components/daylist/AISyncProgress.tsx:56–90` (zero importers). **Canonical:** shared `useDentallySyncProgress(channel, {onCompleted})` on canonical `hooks/useWebSocket.ts`. **Extraction:** safe. **Confidence:** high.

### Mapping 63 — Hand-rolled practice-scoped TTL list cache, 2 token-identical scaffolds + 1 divergent
- `pages/recalls/useRecallsData.ts:44–54` vs `useDormantRecalls.ts:22–30` — identical module-level Map + `registerPracticeScopedCache`/`Reset` + 5-min TTL + silent refresh + JSON dedupe (same error string). Divergent: `pages/day-list/hooks/useDayListOutstandingItems.ts:13–16,53` — 3-min Map cache **without practice-scoped wipe** (survives soft practice switch). Siblings already use canonical `hooks/useCachedData` (useRecallConfig.ts:41–46). **Canonical:** `useCachedData` or thin `useRecallList(params, endpoint)` preserving stale-QS guard. **Extraction:** safe. **Confidence:** high.

### Mapping 58 — `pages/admin/utils/dateFormatter.ts` exists but is bypassed by a token-identical local copy
- Shared export `pages/admin/utils/dateFormatter.ts:1–6` (only 2 importers); `pages/admin/Users.tsx:444–449` re-declares the body **character-identical**; near-variant `pages/admin/SupplierManagement.tsx:389` with **`en-GB`** (locale divergence: "05 Jan 2026" vs "Jan 5, 2026"). **Fix:** import the shared util. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 59 — Consent "send request" dialog implemented twice, divergently, both live
- `components/consent/SendConsentModal.tsx` (568 ln, self-contained, template selection + QR/resend; no shared placeholders lib) — live in DayListPage.tsx:803, AppointmentCard.tsx:997, UnifiedDocumentsList.tsx:501; vs `components/consent/ConsentSendDialog.tsx` (204 ln; `useSendConsent()` hook + `consentPlaceholders` + TokenMessageEditor; shows SMS part-count/tokens) — live in ConsentAssistant.tsx:271 and pages/Conpact3.tsx. **Divergence:** the two UIs differ on what a patient receives for the same action. **Canonical:** ConsentSendDialog's architecture with SendConsentModal's features folded in. **Extraction:** requires design decisions. **Confidence:** high (existence), medium (intent).

### Mapping 60 (M23 ext) — verbatim `getInitials` first-2 cluster in pages/hr, 5 files
- `pages/hr/components/timesheets/{TimesheetWithRotaTable.tsx:87, TimesheetWeeklySummary.tsx:95, TimesheetMobileCard.tsx:27, TimesheetTableRow.tsx:20}` and `calendar/HRCalendarDayCell.tsx:346` — identical body, while `pages/hr/utils/rotaUtils.ts` already exports `getInitials` used by 4 rota files. **Canonical:** rotaUtils export or M23's `utils/textUtils`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 61 (M26 ext) — variant-2 relative-time clone, new site
- `pages/admin/GlobalAssets.tsx:308–317` — token-identical to M26's variant-2 pair; bypasses `formatRelativeTimestamp`. **Confidence:** high.

## Frontend pass-12 minor extensions
- **M3 ext:** inline `£{n.toLocaleString()}` in `components/modals/treatmentPlan/{IssueTreatmentPlanModal.tsx:54,61, TreatmentPlanConfirmationModal.tsx:264, TreatmentPlanSuccessModal.tsx:106}` — bypass `formatCurrency`, no locale arg.
- **M1-adjacent ext:** `getApiUrl("/usage/rates/...")` hardcoded in `components/settings/sms-phone-config/{SmsRatesSection.tsx:79,101,123,131, CallRatesModal.tsx:145,160,196}` while `API_ENDPOINTS.callAgent.rates` defines exactly these; the same directory mixes both styles.

### Mapping 54 — Spend-bucket validation/persist engine duplicated verbatim (recalls vs admin practice-management)
- `pages/recalls/components/administration/SpendSection.tsx:172–225` vs `pages/admin/practice-management/components/AdminSpendSection.tsx:156–205` — token-identical validation (exact toast strings incl. "Only the highest bucket can be open-ended (blank max)"), overlap checks, buckets projection, and identical `toggleVip` (SpendSection:160 vs AdminSpendSection:144). AdminSpendSection refactored into a pure `validate()` (better) + success toast; recalls toasts only on failure. **Canonical:** `validateSpendBuckets(rows, vipKeys)` + shared `SpendBucketsEditor`. **Extraction:** safe. **Confidence:** high.

### Mapping 55 — Journeys patient-table `applyDateRangeFilter` + priority sort comparator, 3 sibling pages token-identical
- `pages/journeys/OpenPlans.tsx:82–107,130–152`, `ActiveTreatments.tsx:68–93,115–135`, `Intake.tsx:80–105,140–163` — OpenPlans↔Intake `applyDateRangeFilter` diff is **empty**; same `priorityOrder = {High:3, Medium:2, Low:1}` dual-mode comparator. All live via `pages/journeys/index.ts:2–4`. **Canonical:** `dateRangePresetFilter` + `sortByPriority` in a journeys lib (or fold into M16's `useTableFilters`). **Extraction:** safe. **Confidence:** high.

### Mapping 56 — `DueDateSelection` component byte-identical clone, live in two features
- `pages/treatment-plans/components/DueDateSelection.tsx` vs `pages/tasks/components/task-creation/DueDateSelection.tsx` — **`diff` reports zero differences** (88 lines each); both live (TaskAssignment.tsx:5 → CreateTreatmentPlanSheet; CreateTaskSheet.tsx:23). **Canonical:** shared component in `components/ui/` or `components/tasks/`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 57 (minor) — `useIsMobile` re-roll bypassing canonical `hooks/use-mobile.tsx`
- `components/treatment-verification/useIsMobile.tsx:3–29` — own resize-listener implementation (768px), sole importer `pages/treatment-verification/Services.tsx:11`; ~25 other files use the canonical `matchMedia` hook. **Fix:** import the shared hook. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 51 — JS(Sun=0)→Backend(Mon=0) day-of-week converter re-rolled ~10 sites, twice inside one file
- `pages/diary/schedules/storage.ts:26–36` defines `frontendDayToBackendDay`/`backendDayToFrontendDay`, then declares them **again at :242–249** as `*2` (token-identical). Inline re-roll: `pages/diary/schedules/utils.ts:350,352`. Same `jsDay === 0 ? 6 : jsDay - 1` expression as "days-since-Monday" offset in `lib/utils.ts:73`, `hooks/useJourneyBoard.ts:138`, `components/compact/IntakeTable.tsx:386`, `CustomJourneyTable.tsx:669`, `pages/hr/utils/dateUtils.ts:67`, `pages/hr/hooks/useTimesheets.ts:432`. **Canonical:** `toMondayIndex`/`fromMondayIndex` in `utils/dateUtils.ts`. **Extraction:** safe. **Confidence:** high.

### Mapping 52 — Notes-template "default marker" (`*` prefix) parsing re-rolled 3–4×
- `pages/Compact5.tsx:68–205` vs `pages/settings/components/preferences/DocsTemplateManager.tsx:120–236` — token-identical `stripDefaultMarker`/`hasDefaultMarker` + whole answer/nested-option preview builder; naive untrimmed variant `components/Library/NotesTemplatePreview.tsx:12–13`; `buildQuestionKey` triplicated (Compact5:157, DocsTemplateManager:218, TemplatePreviewDialog.tsx:105). **Divergence:** `" *Option"` behaves differently across the three UIs. **Canonical:** `notesTemplatePreview.ts` helpers. **Extraction:** safe. **Confidence:** high.

### Mapping 53 — `convertMarkdownToHTML`, 6 live private definitions, one security-hardened divergent (**security**)
- `pages/Compact5.tsx:305`, `pages/settings/components/preferences/DocsTemplateManager.tsx:361`, `components/DocumentViewModal.tsx:38`, `components/modals/components/ContentEditor.tsx:234`, `components/letter-library/LetterBody.tsx:25` — same regex-pipeline skeleton, output fed to `dangerouslySetInnerHTML` **without HTML-escaping**. `pages/compliance/components/policies/PolicyPreviewModal.tsx:23` — the only exported one and the only one that escapes `& < > "` before substitution (comment cites stored-XSS audit C12). **Canonical:** export the PolicyPreviewModal implementation into `lib/markdown.ts`; migrate the five unescaped lanes (real XSS gap for user-typed template content). **Extraction:** safe; fixes a security gap. **Confidence:** high.

### Mapping 54 — Mic `RecordButton` UI scaffold duplicated; second copy bypasses the shared recorder hook
- `pages/create/notes/components/RecordButton.tsx:60–152` (wraps shared `useScribeRecorder`) vs `components/inbox/RecordButton.tsx:270–336` — token-identical `formatTime`, near-identical render/style switches, but the inbox copy hand-rolls a 190-line RecordRTC + `useMessaging().transcribeVoice` pipeline (:81–253) bypassing both `useScribeRecorder` and the AssemblyAI layer. **Canonical:** one `RecordButton` on `useScribeRecorder` with injected transcriber; retire the inbox engine. **Extraction:** safe. **Confidence:** high.

## Frontend pass-10 extensions
- **M1 ext:** `pages/diary/shortNoticeUtils.ts:85–94` + `pages/diary/schedules/storage.ts:104–113` — byte-identical private `fetchAuthHeaders()` feeding ~30 inline diary fetches (new diary-layer raw-fetch cluster).
- **M22 ext:** `pages/booking/components/shared/BookingProgressBar.tsx:7–9,91` — fourth+ StepIndicator variant (CSS-var parameterized).
- **M16 ext:** `pages/Compact6.tsx:111–260` — inline pagination + DataTable "matching the Stock.tsx design exactly".
- **M7 ext:** `components/shared/documents-dialog/DocumentUploadStepper.tsx:6` and `useFileDropTarget.ts` import sonner directly.
- **M4 ext:** local `YYYY-MM-DD` key formatters bypassing `formatLocalISO` in 7 files (diary useAppointments:15–20, DayListPage:70, DiaryPage:260, Labs:862, dayWindow.ts:22, date-picker.tsx:100, datetime-picker.tsx:61–78).

## Frontend dead code (pass 10, deletion candidates)
- `components/sidebar/` — **whole directory zero importers** (prototype leftover, builder.io placeholder URLs).
- `components/kanban/` — zero importers.
- `pages/booking/components/steps/TimeSlotSelectionStep.tsx`, `pages/create/notes/components/NotesInterface_New.tsx` (0 bytes), `NotesTextarea_new.tsx`, `components/modals/components/ContentEditor_backup.tsx` — no importers.

### Mapping 46 — react-dnd drag-to-reorder row scaffold, 4 live implementations
- `useDrag`/`useDrop` pair + `DRAG_TYPE` constant + identical `splice(from,1)/splice(to,0)` move + drop-end persist: `pages/journeys/administration/JourneyStagesTab.tsx` (:81,117,130,359–392 — **only lane with snapshot+revert on failed persist**, D-43/D-49); `pages/settings/components/practice/OnlineBookingServicesCard.tsx` (:28,46,57,66–73,266–274); `pages/day-list/components/administration/ConfirmationTargetingEditor.tsx` (:287,335,344–351,560–573 — settles priorities, never persists directly); `components/TreatmentPlan.tsx` (:258–268 — persists per hover, no drop-end). **Canonical:** `useDragToReorder(list, onPersist)` hook. **Extraction:** safe. **Confidence:** high.

### Mapping 47 (minor) — Payslip status → badge-class map, 2 near-identical copies
- `components/invoices/AssociatePayView.tsx:226–238` vs `AssociatePayPersonalArchive.tsx:40–44` — same three branches/classes. **Canonical:** one export in `components/invoices/`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 48 (minor) — `ConfirmationSequence` status label/style maps duplicated
- `pages/day-list/components/administration/ConfirmationSequenceForm.tsx:32–37` vs `ConfirmationSequencePreviewPage.tsx:40–52` (byte-identical `statusLabel`; preview adds `statusStyles`). **Canonical:** `confirmationSequenceMeta.ts` beside the type. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 49 — Automation workflow type catalog mirrored in two static tables, already drifted
- `pages/automation/editor/types.ts` (`TRIGGER_TYPES`:123–130, `ENTITY_TYPES`:133–145, `ACTION_TYPES`:148–161) vs `hooks/useWorkflowActionTypes.ts` (`FALLBACK_*`:33–75, used by NodePalette). Triggers/entities byte-identical; **`ACTION_TYPES` has drifted** — types.ts has `create_dentally_patient`, the hook has `query_records`/`create_task`/`for_each` + a `category` field — two UIs in the same editor disagree on the action catalog. **Canonical:** single `workflowTypeCatalog.ts`. **Extraction:** safe. **Confidence:** high.

### Mapping 50 (minor) — No-show risk tone ternary, 3 day-list sites
- `pages/day-list/components/NoShowRiskModal.tsx:15–19`, `AppointmentContextPill.tsx:76–78` (collapses Elevated into amber — undocumented), `AppointmentCard.tsx:554–557`. **Canonical:** `riskTone(status)` in day-list lib. **Confidence:** medium (divergences may be intentional).

## Frontend pass-9 extension/exclusion notes
- **M35 ext:** `pages/day-list/components/ClinicianChips.tsx:13–21` `firstName()` — same 6-title list, new site.
- Mostly-dead: `components/patient-view/` — PatientJourneysCard/Table, BroadcastMessagesCard/Table, PatientDetailsCard, PatientHeader, NotesTab, MarketingTab have **no live importers** (dead; deletion candidates). Live members covered by M4/M5.
- Day-list ↔ recalls admin sections already consolidated via `pages/recalls/components/administration/shared.tsx` + `RowPill.tsx`; day-list/recalls sync modals genuinely divergent (excluded).

### Mapping 37 — Compliance log "View" page scaffold, 7 sibling clones
- `pages/compliance/{ComplaintView, IncidentView, SedationView}.tsx` (101–107 lines each; diffs only icon/noun/tab-param/color); `PrescriptionView.tsx` vs `ReferralView.tsx` (148 lines each, identical incl. delete-confirm + toast block); `SafeguardingView.tsx` (107); `RiskAssessmentView.tsx` (331, extended variant). **Canonical:** config-driven `ComplianceRecordView({collectionKey, findFn, detailsComponent, icon, accent, backTab, label})`. **Extraction:** safe. **Confidence:** high.

### Mapping 38 — base64/image-evidence decode helpers + `slugify`, ~10 sites
- XLSX export clones (`pages/compliance/utils/exportAuditRecords.ts:18–84`, `exportChecklistRecords.ts:17–97`, `exportTrainingRecords.ts` — private `isBase64Image`/`getImageExtension`/`base64ToBlob`/`fetchImageAsBlob`/`slugify` each); inline re-rolls `pages/compliance/hooks/useAudits.ts:485–503` (mime default jpeg) vs `useCompliance.ts:1783–1800` (**png** — divergent) vs `ChecklistRecordView.tsx:132–147` (no mime fallback). `slugify` additionally re-rolled in `pages/hr/utils/exportHRRecords.ts:18`, `exportStaffCompliancePack.ts:8`, `lib/customAutoTemplates.ts:132` while `marketing/forms/lib/utils.ts:3` exports one. **Canonical:** `lib/imageEvidence.ts` + `utils/slugify.ts`. **Extraction:** safe. **Confidence:** high.

### Mapping 39 — `Confetti` canvas component cloned in 2 compliance completion modals
- `pages/compliance/components/audit/AuditCompletion.tsx:15` vs `checklist/ChecklistCompletion.tsx:17` — same particle engine near-verbatim; **copy bug: AuditCompletion's colors array starts with `'blue-600'` (invalid canvas color)** where the other has `'#3b82f6'`. **Canonical:** shared `<Confetti/>` in `components/ui/`. **Extraction:** safe. **Confidence:** high.

### Mapping 40 — Image-evidence upload guard scaffold, 4 files
- Token-identical triple `pages/compliance/AuditTemplateEditor.tsx:944–958`, `pages/admin/ComplianceAuditTemplates.tsx:1576–1590`, `ComplianceChecklistTemplates.tsx:1526–1540` (type-check + 10MB cap + HEIC decode toasts); `ChecklistEditor.tsx:475` (HEIC leg only). **Canonical:** `validateEvidenceImage(file)` helper/hook. **Extraction:** safe. **Confidence:** high.

### Mapping 41 — Message threading detection, 3 divergent implementations
- `components/inbox/MessageList.tsx:31–44` (O(1) `threadMap` + self-id guard — canonical) vs `components/inbox/v2/MessageList.tsx:128–141` (O(n²), **no self-id guard**) vs `pages/messages/prototype/MessageListPrototype.tsx:136–149` (token-identical to v2). **Canonical:** extract `useThreadInfo(messages)` from the map version. **Extraction:** safe. **Confidence:** high.

### Mapping 42 — `components/inbox/v2` ↔ `pages/messages/prototype` clone suite (live-routed)
- `channelGlyph.tsx` (diff of 11 lines); `FamilyAvatar.tsx` (139 vs 161 lines, near-clone incl. internal helper); MessageBubble/Header/ListPanel prototypes with private re-rolls of `formatScheduledAt`/`memberSubline`/`joinLabels` (identical bodies, not imported); `automationMeta.ts` — v2 adds a `PURPOSE_META` fallback the prototype lacks (divergent); `toFamilyLabel` exported from `components/inbox/utils.ts:317` yet re-declared identically in the prototype (ConversationHeaderPrototype.tsx:46, FamilyGroupPrototypePage.tsx:69). **Canonical:** prototype imports from v2 or a shared `components/messaging/` layer. **Extraction:** safe. **Confidence:** high.

### Mapping 43 — `formatFileSize`/`getFileIcon`, 7+ private copies
- `components/assets/AssetDocumentManager.tsx`, `components/inbox/v2/{AttachmentPreviewModal, MessageBubble, MessageComposer}.tsx` (three byte-identical), `pages/messages/prototype/MessageBubblePrototype.tsx`, `pages/compliance/MeetingMinutesEditor.tsx`, `pages/hr/Payslips.tsx`, plus inline `components/invoices/InvoiceUploadDialog.tsx:138,145` and `components/inbox/EmailInputFields.tsx:40`. **Canonical:** `utils/fileUtils.ts`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 44 — "Fetch available placeholders" workflow, 5+ sites
- Best-effort fetch of `*AvailablePlaceholders()` → flatten to suggestion items: `pages/admin/MessageTemplates.tsx:757–777`, `pages/settings/components/PlaceholderAutocomplete.tsx:59–85`, `journeys/administration/AutomationSection.tsx:250–260`, `recalls/administration/AutomationSection.tsx:245–255`, `day-list/administration/ConfirmationsSection.tsx:81–92`. **Canonical:** `useAvailablePlaceholders(templateType)` hook. **Confidence:** medium-high.

### Mapping 45 (minor, M26-family extensions) — relative-time "X ago" re-rolls outside pages/admin
- `components/patients/patient-panel/patientPanelUtils.ts:23–50 formatLastContacted` (full ladder), `PatientPanelAccordion.tsx:196–209` (truncated), `components/inbox/utils.ts:105–132` ("Just now"/"Nm ago"/"Yesterday" — casing/granularity diverge). All bypass canonical `utils/dateUtils.ts:43 formatRelativeTimestamp`. Also trivial: email regex duplicated `pages/ThreeMonthsFree.tsx:32` ↔ `pages/individual/DashboardNotes.tsx:69` beside `marketing/forms/lib/utils.ts:32 isValidEmail`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 32 — `generateGradientColors` value→gradient mapper, 9 copies across chart components
- Token-identical purple versions: `components/spend-reporting/SpendByDentistChart.tsx:24`, `LabSpendByDentistChart.tsx:24`, `SpendBySupplierChart.tsx:26`, `SpendByLabChart.tsx:27`, `AssetsOwnedByCategoryChart.tsx:24`, `components/reporting/PipelineDistributionChart.tsx:27`, `InvoiceStatusChart.tsx:27`. Blue variant (same algorithm, different anchors): `pages/compliance/components/dashboard/LogBreakdownChart.tsx:59`. (`SpendByDentistChart` vs `LabSpendByDentistChart` are near-clone components too.) **Canonical:** `generateGradientColors(data, {darkest, lightest})` in a shared `chartColors.ts`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 33 — `insertPlaceholder` textarea cursor-splice, 6 implementations
- `pages/admin/MessageTemplates.tsx:784`; `pages/day-list/components/administration/ConfirmationTargetingEditor.tsx:233`; `pages/journeys/administration/AutomationSection.tsx:287` ↔ `pages/recalls/components/administration/AutomationSection.tsx:264` (token-identical pair, extends M15); `pages/automation/editor/components/action-configs/SendSmsConfig.tsx:52`; `pages/MarketingCampaignBuilder.tsx:753`. Same splice + setTimeout/rAF cursor-restore + append-fallback; divergent setter shapes. Related: accordion `PlaceholderPicker` duplicated (recalls :80 vs day-list `confirmationPlaceholderPicker.tsx:35`, comment admits mirroring).
- **Canonical:** `useTokenInsert({fieldRef})` + shared `PlaceholderPicker`. **Extraction:** safe. **Confidence:** high.

### Mapping 34 — Client-side working-day counting, 4 implementations in HR
- `pages/hr/Leave.tsx:1041` (day-loop), `pages/hr/Admin.tsx:1300` (O(1) arithmetic), `components/hr/BulkLeaveDialog.tsx:53` (date-fns `eachDayOfInterval`), `pages/hr/utils/leaveAmounts.ts:89` `countDaysInRange` (configurable Monday-based `workingDays[]` — the existing exported helper the others bypass). Behavioral: O(1) variant ignores part-time patterns; leave.tsx wisely falls back only when backend `duration_preview` fails. **Canonical:** `countDaysInRange`. **Extraction:** safe. **Confidence:** high.

### Mapping 35 — Practitioner title-strip / given-name extraction fragmented across 9+ files
- Byte-identical pair inside day-list: `pages/day-list/lib/formatPractitionerName.ts:7` vs `pages/day-list/lib/dayListModel.ts:12` (same 6-title array + dot-strip). Same 6-title list re-rolled: `components/patients/workspace/PatientTasksChip.tsx:80–95`, `workspace/documents/unified/documentsModel.tsx:125`. Naive `^Dr\.?\s*`-only sub-family: verbatim-identical `formatDentistName` in `components/compact/ActiveTable.tsx:175`, `OpenTable.tsx:178`, `NurtureTable.tsx:250`, `pages/Labs.tsx:132`; plus `components/journey-board/JourneyBoardCardItem.tsx:44`, `treatment-verification/TherapistProfile.tsx:34`.
- **Canonical:** `stripTitle`/`givenName`/`formatDentistName` in one `utils/nameUtils.ts`. **Extraction:** safe. **Confidence:** high.

### Mapping 36 — `InboxPage` ↔ `FamilyGroupPrototypePage` verbatim clones (`normalizePhone`, `contactAvatar`)
- `InboxPage.tsx:1166` / `:409–416` vs `pages/messages/prototype/FamilyGroupPrototypePage.tsx:1163` / `:426–433` — token-identical (diff empty); prototype live-routed (routes/clinical.routes.tsx:21,85). **Canonical:** export from a shared `lib/contactDisplay.ts`. **Extraction:** safe, trivial. **Confidence:** high.

## Frontend pass-7 exclusion/observation notes
- Phone utilities fragmented: canonical `utils/phoneInput.ts normalizePhoneInput` (libphonenumber) vs `lib/utils.ts:9 formatPhoneNumber` (**US hyphen format in a UK app**) vs `utils/phoneFormatter.ts formatPhoneForDisplay` (UK, ~14 importers) + 5 digit-strip re-rolls; `components/patients/PatientList.tsx:144` local `formatPhoneNumber` is an identity function (dead abstraction). Consolidation candidate adjacent to M3/M35.
- Dead: `pages/settings/components/SignatureCanvas.tsx` (near-duplicate of SigningPage SignaturePad; zero importers) — deletion candidate.
- Related sibling lead (not promoted): raw `new WebSocket` in `components/patients/DentallyImportWizard.tsx:364`/`DentallyImportFlow.tsx:384` — sibling copies, components-layer; watch in later passes.

### Mapping 29 — Patient record normalization / person display-name assembly, 3 near-clone normalizers + lighter re-rolls
- `components/patients/patient-panel/patientPanelUtils.ts:46` `normalizePatientPanelRecord`; `components/patients/workspace/utils.ts:106` `normalizePatientForWorkspace` (strict typeof-guards, `countryCode: undefined`); `pages/Journeys.tsx:199` private `normalizePatientForPanel` — structurally identical field-by-field (alias chains incl. `firstname`, fallback name join, id slug derivation, nested `patient` projection). "Unnamed Patient" fallback in workspace/Journeys, absent in patientPanelUtils. Lighter display re-rolls: `components/patients/universal-search/phone.ts:70 getPatientDisplayName` (canonical for its lane), `components/diary/confirmationActions.ts:53`, `pages/Contacts.tsx:155`.
- **Canonical:** one loose `normalizePatientRecord(record)` + reuse `getPatientDisplayName`. **Extraction:** safe. **Confidence:** high. (Distinct from M11 splitName and M23 initials.)

### Mapping 30 (minor) — Consent-status union re-declared as inline literals beside canonical type
- Canonical `ConsentStatus` at `types/signableDocuments.ts:30–35`. Inline re-declarations: `types/diary.ts:125`, `types/diary.ts:156`, `pages/day-list/types.ts:160`. **Fix:** import the canonical type. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 31 (minor) — `VITE_MAIL_SERVER_DOMAIN`/`VITE_EMAIL_SERVICE_URL` env reads with duplicated fallback strings, 6 sites
- `import.meta.env.VITE_MAIL_SERVER_DOMAIN || "mail.pathway.dental"` restated in `pages/settings/email-service/InboundParseSettings.tsx` (×2), `pages/settings/components/practice/DomainConfigView.tsx` (×2), `pages/admin/PracticeDomainDetail.tsx:117`, `pages/settings/EmailService.tsx:610,616` — bypassing `config/environment.ts`. **Fix:** export constants from `config/environment.ts`. **Extraction:** safe, trivial. **Confidence:** high.

## Frontend pass-6 extension/dead-code notes
- **M20 ext:** ElevenLabs/STT config constants verbatim-triplicated (`hooks/useAudioRecording.ts:20,24–27` ↔ `lib/scribeTranscription.ts:24–27` byte-identical; `useScribeRecorder.ts:15`) — fold into M20 extraction.
- **Dead:** `hooks/usePatientCaching2.ts` — byte-clone of `usePatientCaching.ts` except one debug flag; zero importers. Dead: `pages/daylist/` directory (no importers). Deletion candidates.
- `API_ENDPOINTS` (config/api.ts): 528 importers; only 2 live inline-URL bypasses (already cited in M1). Optimistic-update patterns domain-specific, no family.

### Mapping 25 — "Create guest session → open `/guest?token=` → toast" handler, 7 copies in pages/admin
- `pages/admin/Users.tsx:765–780` (`handleUseGuestToken`); `pages/admin/PracticeUsers.tsx:507–525` and `:852–868` (**two token-identical copies in one file**); `pages/admin/practice-management/components/PracticeUsersSection.tsx:322–338` and `:349–361`; `pages/admin/practice-management/hooks/usePracticeManagement.ts:1126–1152` and `:1155–1175`. Divergence: error extraction, whether token is persisted for re-open vs opened immediately.
- **Canonical:** shared `useGuestSession()` (create/open/revoke). **Extraction:** safe. **Confidence:** high.

### Mapping 26 — local `formatDate` "Xh ago" relative-time helpers, 4 copies in pages/admin, bypassing canonical
- Variant 1 (verbatim clones): `pages/admin/GlobalPhrases.tsx:76–88`, `GlobalAutoTemplates.tsx:156–168` ("Just now"/"Nh ago"/singular day). Variant 2 (verbatim clones of each other): `pages/admin/LetterTemplates.tsx:147–161`, `ClinicalTemplates.tsx:136–148` ("N hours ago"/"1 day ago", no singular). **Canonical exists:** `utils/dateUtils.ts:43 formatRelativeTimestamp` (also 2 more private re-rolls in `hooks/useNotesAndLetters.ts:98,128` for the hooks pass). **Extraction:** safe. **Confidence:** high.

### Mapping 27 (minor) — `ageFromDob` verbatim duplicate in two routed marketing preview pages
- `pages/MarketingSegmentPreview.tsx:28–34` and `MarketingCampaignPreview.tsx:78–84` — byte-identical (365.25-day year, `"—"` fallback); both routed live (routes/marketing.routes.tsx:77,98). **Canonical:** `ageFromDob(dob)` in `utils/dateUtils.ts`. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 28 — "View Full Note" new-tab open with divergent URL prefix (`/note-archive/` vs `/notearchive/`)
- `pages/create/patient-report/components/PatientNoteCard.tsx:27–29` (`/note-archive/${id}`) vs `PatientReportWorkflowSheet.tsx:227` and `pages/individual/DashboardNotes.tsx:104` (`/notearchive/${id}`, which `hooks/useDocumentTitle.ts:12` also maps). **No route for either prefix found in `src/routes/`** — at least one is dead; `/note-archive/` has no other reference and is the likely broken copy. **Fix:** single `noteArchivePath(noteId)` constant + verify the live route. **Extraction:** safe; route verification needed. **Confidence:** high (duplication), medium (which prefix is live).

## Frontend exclusions (pass 5)
- Thin `window.open` flows (sign links, image/PDF opens, PathwayBot admin link) — too thin for a family beyond M25.
- Marketing audience payload built 3× (Marketing.tsx / MarketingCampaignBuilder / MarketingCampaignCreate) — borderline, real divergence (unset state, draft vs committed ids); observation only.
- Detail-page header/back patterns — different navigation targets/unsaved-guards, not copy-paste. Settings sections domain-specific; `DeleteConfirmationModal` twins already M24.

### Mapping 22 — `StepIndicator` wizard step rail copy-pasted 4×
- `components/BulkImportModal.tsx:25–73` (gray-900); `components/patients/DentallyImportWizard.tsx:29–79` (**only the hex color differs** — `#846ce0` — otherwise token-for-token identical incl. comments); `components/patients/DentallyImportFlow.tsx:41–85` `WizardStepIndicator` (same skeleton, drops `isCompleted`); `pages/settings/components/practice/OnlineBookingServiceDialog.tsx` (own copy). **Canonical:** shared `StepIndicator({...,accent})` in `components/ui/`. **Extraction:** safe. **Confidence:** high.

### Mapping 23 — `getInitials` re-rolled 12× beside a shared `PatientAvatar` that already takes `initials`
- Full-initials unguarded: `components/kanban/KanbanCard.tsx:22`, `ContactsTable.tsx:625`, `patient-view/PatientDetailsCard.tsx:14`, `activetreatments/ActiveTreatmentsTableRow.tsx:48` (**no uppercase**), `modals/treatmentPlan/TreatmentPlanConfirmationModal.tsx:140`, `sidebar/UserProfile.tsx:53`. First-2-initials with `'?'` fallback: `compact/IntakeTable.tsx:958`, `ActiveTable.tsx:952`, `OpenTable.tsx:1000`, `ArchiveTable.tsx:308` (identical triple), `compact/JourneyMobileCard.tsx:172` (no empty-guard). Null-safe: `labs/LabCasePatientCell.tsx:13`. `compact/CustomJourneyTable.tsx:259–261` — comment "Mirrors IntakeTable.getInitials".
- **Divergence:** cap 2 vs all initials; uppercase vs not; `'?'` vs `''` fallback; crash on null in unguarded variants. **Canonical:** `getInitials(name, max=2)` in `utils/textUtils.ts`; `PatientAvatar` optionally derives from `name`. **Extraction:** safe, trivial. **Confidence:** high. (Related to but distinct from M11 `splitName`.)

### Mapping 24 — One-off Dialog-based confirm wrappers beside canonical `components/ui/confirmation-dialog.tsx`
- **Canonical:** `components/ui/confirmation-dialog.tsx` (AlertDialog, `variant="destructive"`, `isLoading`) — live in 5 callers (DataQualityIssuesScreen, UnifiedDocumentsList, hr/Payslips, settings/Notes, settings/Treatments).
- **Hand-rolled bypasses:** `components/settings/DeleteConfirmationModal.tsx:24–51`; `pages/settings/components/DeleteConfirmationModal.tsx:30–95` (same name, different file, same scaffold); `components/Library/DeleteConfirmationDialog.tsx:48–75`; inline Dialog confirms at `components/admin/users/UserDialogs.tsx:~180` and `components/assets/AssetPanel.tsx:~1119`. Divergence: Dialog vs AlertDialog, `bg-red-600` classes vs `variant`, only canonical handles `isLoading`.
- **Canonical:** the existing `ConfirmationDialog` (+ icon/warning slot). **Extraction:** safe. **Confidence:** high.

## Frontend exclusions (pass 4)
- Drawer/sheet layouts: `LabsMobileFilterDrawer` vs `compact/MobileFilterDrawer` are ordinary `ui/drawer.tsx` consumers; `DocumentsDialogFrame` imports `FRAME_CLASS` from `journeys/dialogFrame.tsx` (already consolidated).
- DnD: only 8 draggable files; `notes/CustomImageNode` vs `LetterImageNode` are genuinely divergent Lexical nodes; `shared/documents-dialog/useFileDropTarget` is the one shared implementation.
- Empty/loading states: local `EmptyState` defs are genuinely per-domain, not copy-pastes. Tabs all via `ui/tabs.tsx`. Calendar/day-view: no duplicated second implementation.

### Mapping 17 — "Authenticated export → blob → temp anchor → revoke" download scaffold, 5+ copies
- `hooks/useReporting.ts:449–459`, `useSpendReporting.ts:522–531` (reversed remove/revoke cleanup order — divergence), `useStaffActivity.ts:88–99`, `usePatientAudit.ts:96–116` (**also an unlisted raw-fetch+Bearer site extending Mapping 1**), `useInvoices.ts:813–849`. Justified variant: `useTreatmentPlanQr.ts:47–52` (blob kept for `<img>`).
- **CSV-builder bypasses of `lib/csvExport.ts`** (lose UTF-8 BOM + formula-injection sanitization it documents): `components/invoices/SupplierPayView.tsx:411`, `components/invoices/exportAllocationClinicianTotals.ts:123`, `pages/hr/Admin.tsx:880`.
- **Canonical:** shared `downloadBlob(blob, filename)`; CSV builders route through `exportToCSV`/`sanitizeCsvCell`. **Extraction:** safe. **Confidence:** high.

### Mapping 18 — JWT payload decode re-rolled 3×, one divergent (silent fail-open)
- `components/AdminProtectedRoute.tsx:19–26` and `lib/analytics/isSuperuser.ts:20–28` (verbatim copy, header says it "mirrors") — proper base64url normalization + padding. **Divergent:** `contexts/AuthContext.tsx:866` `verifyToken` — naive `JSON.parse(atob(token.split(".")[1]))`, **no base64url/padding; catch returns "valid"** — fails-open on `-`/`_` payloads or unpadded lengths.
- **Canonical:** exported base64url-safe `decodeJwtPayload(token)`. **Extraction:** safe; fix the fail-open while at it. **Confidence:** high.

### Mapping 19 — Parallel stock data layers: `useStock.ts` (1,251 ln) vs `useStockQuery.ts` (1,792 ln), both live
- Disjoint consumers (`pages/Stock.tsx`, `StockItemSheet`, SuppliersManagementView, OrderFromRecommendationDialog vs `pages/stock/*`, useStockIntake). Duplicated types: `StockCategory`/`OrderStatus`/`ConsumptionReason` character-identical (useStock.ts:6–8 vs useStockQuery.ts:30–34); `StockItem` **diverged** (query version adds unit_count/earliest_expiry/allocation_summary/is_active; `'out_of_stock'` vs `'limited'` union differences).
- Related key-convention bypass: literal `['stock','approval-conditions']` ×4 (useStockQuery.ts:1282–1328) and `['implants']`/`['implant-manufacturers']` ×~8 (useImplants.ts:83–241) vs sibling `stockQueryKeys`/`assetsQueryKeys`/`invoiceKeys` factories.
- **Canonical:** `useStockQuery.ts` as the single stock client; shared types module; key factories. **Extraction:** requires design decisions (retire `useStock.ts` consumers). **Confidence:** high (existence), medium (impact).

### Mapping 20 — Audio-recording + AssemblyAI transcription pipeline duplicated across two live hooks
- `hooks/useAudioRecording.ts` (1,245 ln; NotesEditor) vs `hooks/useScribeRecorder.ts` (714 ln; PatientQuickActions/PatientInboxTab/RecordButton/FamilyGroupPrototypePage). **AudioWorklet PCM processor source string copy-pasted verbatim 4×** (useScribeRecorder.ts:340–369,480–506; useAudioRecording.ts:605–635,835–863) — only the registered processor name differs; plus duplicated AssemblyAI WS session lifecycle (:253,415 vs :472,686) and countdown/stop `setInterval` blocks (:636,666 vs :1151,1183). URL builder already shared via `lib/assemblyaiTranscription.ts`.
- **Canonical:** shared `usePcmAudioWorklet(processorName)` + `useAssemblyAiSession()`. **Extraction:** safe. **Confidence:** high.

### Mapping 14 extension (pass 3) — hooks-layer manual debounce
- `hooks/usePatientSearch.ts:58–125` manual 300ms setTimeout debounce beside canonical `hooks/useDebounce.ts` (used by useDocumentSearch.ts:108). Same family as M14's SearchFilter/InvoiceForm sites.

## Frontend exclusions (pass 3)
- Dead: `hooks/usePayslipEditor.ts` (no live consumers; superseded by `usePayslipEditorMap.ts`) — deletion candidate.
- WebSocket layer otherwise clean: all domain hooks wrap canonical `hooks/useWebSocket.ts`. Raw `new WebSocket` remainders: transcription family (M20) + `components/daylist/AISyncProgress.tsx:65` + `components/patients/DentallyImportWizard.tsx:364`/`DentallyImportFlow.tsx:384` (sibling-copy lead for components pass).
- Polling (`usePayments.pollSubscriptionStatus`, FeatureAccessContext interval, chromeAlerts heartbeat), unsaved-changes guards, feature-permission hooks, CSV import — single implementations each; no family.

### Mapping 10 — "Reassign practitioner" confirm flow, 3 copy-pasted handlers
- `components/compact/ActiveTable.tsx:576` (POST `treatmentPlans.reassign`), `OpenTable.tsx:501` (near-identical), `NurtureTable.tsx:961` (divergent: PATCH `nurture.update`, `assigned_to` vs `user_id` payload key, shadcn toast API). Same validate → reassign → refetch → toast scaffold incl. identical `response.text() || Failed to reassign` error shape.
- **Canonical:** shared `useReassignPractitioner({endpoint, method, payloadKey})` or one `ReassignPractitionerDialog` in `components/compact/`. **Extraction:** safe. **Confidence:** high.

### Mapping 11 — Full-name `splitName`, 4 local implementations, no shared export
- `components/compact/ActiveTable.tsx:607`, `NurtureTable.tsx:876` (identical inline), `components/inbox/v2/AssignToPatientModal.tsx:65` (null-safe guard variant), `components/compact/AddPatientModal.tsx:281` (inline `splitFullName`). **Canonical:** exported null-safe `splitFullName()` in utils. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 12 — Large-quantity `window.confirm` guard with triplicated constant
- `const LARGE_QUANTITY_CONFIRM_THRESHOLD = 500` in `components/stock/AddImplantDialog.tsx:33`, `BulkAddImplantDialog.tsx:35`, `AddLotDialog.tsx:36` (used at :170/:217/:74); divergent message copy; native `confirm` vs app's Dialog-based confirmations. **Canonical:** shared stock-domain constant + helper or Dialog confirm. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 13 — `DATE_PRESETS` + `getDateRangeFromPreset` duplicated across filter toolbars
- `components/reporting/ReportingFilters.tsx:36–58` vs `components/spend-reporting/SpendFilters.tsx:28–52` (comment: "matching Journey Reporting style" — explicit copy acknowledgment); overlapping 30d presets (`subDays(today, 29)` both). **Canonical:** shared `getDateRangeFromPreset(presetId)` in `utils/dateUtils.ts` with a superset preset table. **Extraction:** safe. **Confidence:** high.

### Mapping 14 — Manual debounce re-rolls beside existing shared hooks
- `components/patients/SearchFilter.tsx:22–40` manual `debounceTimerRef` although `hooks/useDebounce.ts` exists (used by MarketingTemplates/MarketingSegments); `components/invoices/InvoiceForm.tsx:179,528–543` hand-rolled debounced autosave although `hooks/useDebouncedAutosave.ts` exists (used by EmailTemplateBuilder/BasicEmailBuilder). **Canonical:** the existing hooks (verify InvoiceForm blur-flush maps to hook API). **Extraction:** safe. **Confidence:** high.

### Mapping 15 — Journeys vs Recalls `AutomationSection` (1,143-line sibling copies, diverged)
- `pages/journeys/administration/AutomationSection.tsx` (1,143 lines) vs `pages/recalls/components/administration/AutomationSection.tsx` (871 lines) — same editor skeleton (sequence drafts, drag-handle step list, `PLACEHOLDER_CATEGORIES = ['patient','clinic','current_user']` duplicated); normalized diff still shows ~700 differing lines (journeys excludes last-visit placeholders; recalls adds status options). Both already share `useAutomationWindow`/`getSendTimeWarning`. **Canonical:** shared `SequenceEditor` with per-domain config. **Extraction:** requires design decisions. **Confidence:** high.

### Mapping 16 — Table + search + pagination scaffold, 25–30 live sites
- `useState` searchTerm + `toLowerCase().includes` filter + local `currentPage/itemsPerPage` + inline pagination: compact tables (`components/compact/IntakeTable.tsx:256,266`, `ActiveTable.tsx:242,320`, `OpenTable.tsx:246,257`, `NurtureTable.tsx:461,463`), `pages/admin/Users.tsx:119`, `pages/Contacts.tsx:51`, `pages/admin/MasterCatalogue.tsx:110`, `pages/Journeys.tsx:662`, `pages/hr/HR.tsx:478`, etc.
- Related: three `FilterModal.tsx` coexist — `components/filters/FilterModal.tsx` (`FilterState`) vs `components/compact/FilterModal.tsx` (`FilterCriteria`) are parallel scaffolds; `components/FilterModal.tsx` (532 lines) has **no live imports** (dead — deletion candidate). **Canonical:** `useTableSearch`/`useTableFilters` hooks + one filter model. **Extraction:** safe but large surface. **Confidence:** high.

## Frontend exclusions (pass 2)
- Inline SVGs (31 files) — mostly bespoke one-off graphics; no shared-icon copy family.
- `Loader2` spinners in 284 files — ordinary shared-component usage.
- Dead: `components/FilterModal.tsx` (no imports found).

### Mapping 1 — Raw `fetch` + manual `Authorization: Bearer` bypassing shared clients (~70 sites)
- **Canonical:** `hooks/useFetchWithAuth.ts` / `hooks/useApi.ts fetchWithAuth` (401 token-refresh retry, 15s timeout, in-flight dedup); `hooks/useDjangoAPI.ts` also exists (three parallel shared clients).
- **Reimplementations:** `pages/day-list/hooks/useCredentials.ts:20,42,67,101,132,156` (six inline fetch blocks in one file); `pages/day-list/hooks/useDayListData.ts:161–290`; `pages/day-list/lib/fetchDayListAppointments.ts:223,314,412,448`; `pages/admin/product-analytics/*` (see M2). Each re-rolls `if (!response.ok) throw` with ~30 near-variants.
- **Divergent (arguably justified):** public token-less pages `pages/ConfirmAppointmentPage.tsx:26,46,61`, `MarketingPreferencesPage.tsx:90,114`, `GuestAccess.tsx:28`.
- **Risk:** reimplementations lose refresh/timeout/dedup. **Canonical path:** consolidate day-list and product-analytics onto the shared client; keep public lanes separate. **Extraction:** safe. **Confidence:** high.

### Mapping 2 — `authedGet` defined once, never exported (product-analytics)
- `pages/admin/product-analytics/useProductAnalytics.ts:45–55` defines a private `authedGet<T>()`; sibling files `useInsights.ts:22,55`, `useRetention.ts:42`, `useUserActions.ts:35`, `useTopUsers.ts:28`, `useTimeline.ts:32`, `useFriction.ts:28`, `useDay.ts:71`, `useRebuildRollups.ts:67,116`, `ElementHeatPanel.tsx:35` each re-roll the identical inline pattern. **Fix:** export it (or route through `useFetchWithAuth`). **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 3 — Currency formatting re-rolls (beyond the two documented canonicals)
- **Canonical:** `types/invoice.ts:442` `formatCurrency` / `:490` `formatCurrencyAbbreviated`; `pages/financial-analytics/formatters.ts` `fmtGBP*` (documented intentional GBP-only split).
- **Re-rolls:** `components/invoices/FinanceAdminTab.tsx:410` (third independent formatter in the area that owns the canonical; suppresses pence — behavior differs); `pages/day-list/components/ImportantPatientsView.tsx:52` (inline `£${toLocaleString}`); `pages/recalls/components/RecallsTable.tsx:28` (`Math.round` + **no locale arg** → browser locale) and `:1115`.
- **Canonical path:** single `formatCurrency` import; document pence-suppression as a param if needed. **Extraction:** safe. **Confidence:** high.

### Mapping 4 — Date-time display re-rolls alongside `utils/dateUtils.ts`
- ~90 local `formatDate/formatDateTime/formatTime` definitions across pages/components; most are one-line generic wrappers (excluded), but copy-paste families exist: identical `new Date(x).toLocaleString('en-GB', {...})` blocks at `pages/admin/SecurityLogs.tsx:173`, `pages/booking/components/steps/ReviewStep.tsx:8`, `pages/recalls/components/RecallsTable.tsx:787`; identical `parseISO+format / "Invalid date"` helper at `components/patient-view/PatientJourneysCard.tsx:50–55` vs `PatientJourneysTab.tsx`.
- **Canonical:** extend `utils/dateUtils.ts` with a shared `formatDateTime` (en-GB). **Extraction:** safe. **Confidence:** high.

### Mapping 5 — Status → badge/color map tables, same domain duplicated
- **Asset status (3 copies):** `components/assets/SwipeableAssetListItem.tsx:138`, `AssetPanel.tsx:431`, `AssetsContent.tsx:407` — same colors, differing text sizes/border classes. **Fix:** shared `AssetStatusBadge`.
- **Journey status (4 copies in one feature):** `components/patient-view/PatientJourneysTab.tsx:76` (case-insensitive), `PatientJourneysCard.tsx:58` (**case-sensitive — divergent**), `PatientJourneysTable.tsx:115`, `PatientTreatmentSection.tsx:47`.
- Cross-domain `getStatusBadge`/`getStatusColor` exist in ~25 files — domain-specific maps are excluded; only same-domain clusters above count. **Confidence:** high.

### Mapping 6 — API error-message extraction, 3+ competing implementations
- `utils/errorUtils.ts:46` `extractErrorMessage` (DRF-aware, canonical); `hooks/useStockQuery.ts:17` `extractApiErrorMessage` (overlapping, different shape handling); identical local `getErrorMessage = (e, fallback) => e instanceof Error ? e.message : fallback` copy-pasted at `pages/settings/components/integrations/SupplierMappingsTab.tsx:64`, `EmailAddressesTab.tsx:40`, `TaskAddressesTab.tsx:46`; another local variant `components/patients/patient-panel/usePatientPanelController.ts:375`.
- **Canonical:** `utils/errorUtils.ts` (add Error-instance awareness). **Extraction:** safe. **Confidence:** high.

### Mapping 7 — Toast bypassing the mandated `lib/toast`
- `lib/toast.ts` header: "Import this everywhere; never import from 'sonner'" (test enforces). **23 files still import `{ toast } from "sonner"` directly**, e.g. `pages/compliance/components/documents/SigningSettingsPanel.tsx:3`, `GeneralDocumentQuickForm.tsx:3`, `SigningBatchesTab.tsx:4`, `CoshhQuickForm.tsx:3`, `ComplianceDocumentsDialog.tsx:12`, `pages/hr/components/search/HRDocumentsDialog.tsx:14`, `components/assets/AssetsContent.tsx:37` — losing per-tone durations/tones. Also `lib/toast-shim.ts` (pure re-export shim, legacy) and legacy `hooks/use-toast.ts` coexist. **Fix:** rewrite imports; consider deleting shim. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 8 — Page-size persistence hooks, two near-identical implementations
- `hooks/usePersistedPageSize.ts` (localStorage) vs `hooks/useMarketingTablePageSize.ts` (server preferences) — same MIN 5/MAX 100/DEFAULT 20 and same pageSize+inputValue API surface. **Canonical:** parameterize `usePersistedPageSize` by storage backend. **Extraction:** safe. **Confidence:** medium-high.

### Mapping 9 — "Is this user an admin" re-derived per call site
- No central `useIsAdmin`; `user?.isAdmin` / `is_admin` ~36 sites; `const isAdmin =` re-derived 22×, semantics diverge (some `isAdmin` only, some `Boolean(user?.isAdmin || user?.role === "practice_admin")` e.g. `pages/PatientWorkspacePage.tsx:502`) while routes gate via `AdminProtectedRoute.tsx`. **Canonical:** `useIsAdmin()` hook matching `AdminProtectedRoute`'s rule. **Extraction:** needs decision on the correct rule. **Confidence:** medium.

## Review notes and exclusions (pass 1)
- `pages/hr/utils/dateUtils.ts` — calendar-grid domain helpers, not a re-roll of `utils/dateUtils.ts` (except trivial `formatDateKey` ≈ `formatLocalISO`).
- `pages/financial-analytics/formatters.ts fmtGBP*` — documented intentional GBP-only split; canonical reference only.
- Public-page raw fetches without auth token — plausible justified divergence; revisit only if 401 handling is needed there.
- `hooks/use-toast.ts` (legacy shadcn store) coexisting with sonner — covered under M7; not separately mapped.

## Area status

| Area | Clean pass 1 | Clean pass 2 | Clean pass 3 | Status |
|---|---:|---:|---:|---|
| pages | 0 | 0 | 0 | not started |
| components | 0 | 0 | 0 | not started |
| hooks/lib/utils/contexts | 0 | 0 | 0 | not started |

## Canonical logic candidates

| Logic family | Candidate source of truth | Competing implementations | Decision/status |
|---|---|---|---|
| — | — | — | Not yet mapped |

## Review notes and exclusions

Record safe intentional differences, generated code, dead/quarantined code, and ordinary wrappers here so future agents do not repeatedly report them as new mappings.

## Completion rule

The review is complete only when all areas have three consecutive clean passes, the register count is current, and every cross-area mapping has implementation evidence or is explicitly marked unresolved.

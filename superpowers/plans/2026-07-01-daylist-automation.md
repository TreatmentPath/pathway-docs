# Day List Automation — Design + Execution Plan

**Date:** 2026-07-01
**Status:** Executing autonomously (user stepped away; explicit instruction to research, plan, execute, and verify without waiting for approval)
**Goal:** Add an "Automation" admin tab to Day List (`DayListAdministration.tsx`), styled/structured like Recall's `AutomationSection.tsx`, but backed by the **generic Go Workflow engine** (per explicit user instruction) rather than a new bespoke Django model.

---

## 1. What already exists (research summary)

### 1.1 Day List admin page already has the tab scaffold
`src/pages/day-list/components/DayListAdministration.tsx` already exists, already routed at `/daylist/administration` (gear icon in `DayListHeader.tsx` → `navigate('/daylist/administration')`), with 2 tabs (Rules, Terminology) each a self-contained section component. Adding a 3rd tab is a small, well-precedented change.

### 1.2 Two SEPARATE, unrelated automation systems already exist in this codebase
- **Recall's `AutomationSection.tsx`**: a bespoke, Django-only, flat "N-step sequence" system (`RecallSequence`/`RecallSequenceEnrollment` models, Celery Beat poller every 15 min, Twilio SMS sent directly from Django). Fully self-contained — no navigation to, or use of, the generic Workflow engine.
- **The generic Workflow engine** (`/automation`, `WorkflowEditor.tsx`): a real, fully-built node/edge graph automation builder, **backed by Go** (`EmailServiceGo/internal/workflows/`), not Django. Triggers (manual/scheduled/record_created/record_updated/webhook/consent_*), actions (send_sms, send_email, query_records, condition, wait_delay, http_request, create_task, create_record, create_dentally_patient, for_each, consent reminders), a 30-second trigger poller, full CRUD REST API on the Go service (`/api/v1/workflows/*`).
- These two are **only unified for display** in the generic `/automation` list page (recall rows show up there read-only, clicking them bounces back to Recall Admin) — never for editing or execution.
- **Explicit user instruction overrides codebase precedent here**: the natural "if you want an N-step outreach sequence, copy the Recall pattern" precedent is superseded by the user's direct words: *"we are going to use the workflow as the underlying logic."* This plan builds Day List automation on the generic Workflow engine, not a Recall-style bespoke model.

### 1.3 The Workflow data model (Go, `internal/workflows/models.go`)
- `Workflow` — scoped only by `PracticeID` (int). **No "feature/category" field exists** — a workflow isn't natively tagged "recall" vs "day-list" vs anything else.
- `WorkflowNode` (trigger/action/condition/delay) with a `Config JSONMap` (action-specific).
- `WorkflowEdge` (graph connections, conditional branching).
- `WorkflowTriggerConfig` (trigger type + entity type/filter, or cron+timezone, or webhook path).
- `WorkflowExecution` / `WorkflowNodeExecution` (audit trail).
- `EntityType` includes `appointment` — the natural fit for day-list-relevant triggers (though also usable by non-day-list workflows; there is no exclusive claim on it).

### 1.4 Scoping decision (since there's no native category field)
**Decision:** scope "which workflows show up in the Day List Automation tab" by a **name prefix convention**: every workflow created from this new tab is named `"[Day List] <user's name>"`. The list view filters the generic workflow list to names starting with `"[Day List] "`. This requires **zero backend changes** — pure frontend convention, exactly mirroring how a human would organize a flat list without schema support. Documented here explicitly as a judgment call for the user to revisit — a more robust future fix would be a real `category`/`source_feature` field on `Workflow`, but that's backend schema work out of scope for this pass.

### 1.5 SMS sending — real, no sandbox, at the generic engine layer
`internal/workflows/actions/send_sms.go` → `django_client.go` → Django's `/api/internal/automations/actions/send-sms/` → real Twilio call. **There is no sandbox/dry-run flag anywhere in this path** — every `send_sms` node execution sends a real SMS via Twilio to whatever phone number the node resolves. This is the single most important safety fact for this plan.

**Safety plan for testing (per explicit user instruction — test only to 2348067390962, nothing else):**
- The demo workflow's trigger is **`manual`** only — it is never wired to `record_created`/`record_updated`/`scheduled` on real entity data during this work. A manual trigger only fires when explicitly POSTed to via `/api/v1/workflows/:id/trigger`.
- The `send_sms` node's `to` config is left as the placeholder `{{trigger.phone_number}}` (per `SendSmsConfig.tsx`'s own documented pattern), and the ONLY test trigger call made will supply `trigger_data: {"phone_number": "2348067390962", "first_name": "Test", "practice_id": <real practice id>}` — so the phone number sent to is always the literal value I control in that one API call, never a real patient's number.
- No workflow created during this task is left with `status: "active"` + a live entity/scheduled trigger — it stays `draft` unless the user later chooses to activate it.

---

## 2. What will be built

### 2.1 New file: `src/pages/day-list/components/administration/DayListAutomationSection.tsx`
A self-contained tab component (no `SectionSaveHandle` — mirrors how Recall's `AutomationSection` is mounted bare in `RecallsAdministration.tsx`, not through the batched-save pattern used by the other 2 Day List tabs):

- **List view**: fetches `GET workflows.list()` (`API_ENDPOINTS.workflows.list()`), filters client-side to `name.startsWith('[Day List] ')`, renders each as a card (name with the prefix stripped for display, status badge, node count, updated date, Edit/Delete/Toggle-active buttons) — visually modeled on Recall's `AutomationSection` list rows.
- **"New automation" button**: prompts for a name (simple inline input, matching the weight of Recall's flow), `POST workflows.create()` with `{name: '[Day List] ' + name, description: ''}`, then `navigate('/automation/' + newWorkflow.id)` — handing off immediately to the existing, fully-built generic `WorkflowEditor.tsx` canvas for actual node/edge editing. **No new editor UI is built** — this is the core leverage of "use the workflow as the underlying logic."
- **Edit (pencil) button** on an existing row: `navigate('/automation/' + workflow.id)`.
- **Delete (trash) button**: `DELETE workflows.delete(id)` with a confirm dialog, mirroring Recall's `remove()`.
- **Toggle active/inactive**: `PUT workflows.update(id)` with `{status: 'active'|'draft'}`.

### 2.2 Modify `DayListAdministration.tsx`
- Add `{ id: 'automation', label: 'Automation' }` to the tab list.
- Render `{activeTab === 'automation' && <DayListAutomationSection />}` **bare** (no ref, no dirty-state wiring into the page's header Save button) — exactly matching how `RecallsAdministration.tsx` renders `<AutomationSection />`.

### 2.3 One demonstration workflow, created via direct API calls (not UI-driven) to prove the whole chain end-to-end
Name: `"[Day List] High-Risk Appointment SMS Test"`. Built directly via the Go API (`POST /workflows`, `POST /workflows/:id/nodes`, `POST /workflows/:id/nodes/:node_id/trigger-config`):
- 1 trigger node: `trigger_type: manual`.
- 1 `send_sms` action node: `config: {to: "{{trigger.phone_number}}", message: "Hi {{trigger.first_name}}, this is a Day List automation test message from Pathway."}`.
- 1 edge connecting trigger → send_sms.
- Left in `status: "draft"`.

### 2.4 One real, live test send
`POST /workflows/:id/trigger` with `{trigger_node_id: <id>, data: {phone_number: "2348067390962", first_name: "Test"}}` — a single real SMS to the user's own number, proving the whole engine (Go workflow executor → Django SMS endpoint → Twilio) works for a Day-List-created workflow exactly as it does for any other.

---

## 3. Explicitly out of scope for this pass
- No backend (Go or Django) code changes — the engine already exists and needs none.
- No new `category`/`source_feature` field on `Workflow` (documented as a judgment call above).
- No day-list-specific action node types (e.g. a "query today's high-risk appointments" convenience action) — the existing generic `query_records`/`condition`/`for_each` nodes are sufficient building blocks; a bespoke day-list query action is a possible future enhancement, not required to satisfy "use the workflow as the underlying logic."
- No changes to Recall's automation system.

---

## 4. Execution Checklist

- [ ] 4.1 Read `RecallsAdministration.tsx`'s exact bare-mount pattern for `<AutomationSection />` once more to confirm the no-ref styling before writing `DayListAdministration.tsx`'s edit.
- [ ] 4.2 Write `DayListAutomationSection.tsx` (list view + create + edit-navigate + delete + toggle-active).
- [ ] 4.3 Edit `DayListAdministration.tsx`: add the `automation` tab entry + bare render.
- [ ] 4.4 `npx tsc --noEmit` clean.
- [ ] 4.5 Browser-driven click-through (Playwright, logged in with the saved dev credentials): navigate to `/daylist/administration`, click the Automation tab, confirm the (empty, initially) list renders without error, click "New automation", confirm a workflow gets created and the browser navigates to `/automation/:id` (the real generic editor) successfully.
- [ ] 4.6 Create the demo workflow (§2.3) via direct API calls (curl/script), not through the UI, to keep this step scriptable and reviewable.
- [ ] 4.7 Confirm via `GET /workflows/:id` that the node graph was created correctly (trigger → send_sms edge present, config as specified).
- [ ] 4.8 Fire the ONE real test trigger (§2.4) to `2348067390962` ONLY. Confirm the API response indicates success (Twilio SID returned) — do not fabricate or assume; if it fails, report the real error, don't retry blindly against a different number.
- [ ] 4.9 Confirm via `GET /workflows/:id/executions` that exactly one execution exists, status `completed`, and its trigger data shows the test number — not any other number.
- [ ] 4.10 Delete or leave the demo workflow in `draft` (never `active`) — confirm its status before finishing.
- [ ] 4.11 Dispatch an independent subagent to re-verify: (a) the new frontend files compile and match the plan, (b) the demo workflow's only execution really did target `2348067390962` and nothing else, (c) no other patient/number was touched anywhere in the process. This subagent must re-check by reading actual API responses/DB state, not trust this document's claims.
- [ ] 4.12 Write a final report for the user covering what was built, what was tested, the real SMS confirmation, and the judgment calls made (§1.4 naming-convention scoping, in particular) for their review when they return.

---

## 5. Judgment calls made without user confirmation (flag for review on return)

1. **Scoping by name prefix** (`"[Day List] "`) rather than a backend schema field — lowest-risk, zero-backend-change option; can be upgraded later if this convention proves fragile.
2. **"New automation" hands off to the existing generic canvas editor** rather than building a bespoke simplified editor (like Recall's) — chosen because the user explicitly said to use the workflow engine "as the underlying logic," and reusing the existing, tested editor is both less work and more capable (full node/edge graph vs. a flat list) than reinventing a simplified one.
3. **The demo workflow uses a manual trigger only** — deliberately not wired to any live entity/schedule trigger, so no automatic execution can ever occur against real patient data as a side effect of this build-and-test pass.

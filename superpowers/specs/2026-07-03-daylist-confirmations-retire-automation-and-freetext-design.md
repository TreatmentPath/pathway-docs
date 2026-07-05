# Retire DayListAutomation + Free-Text/Placeholder Override for ConfirmationSequence — Design

## Context

Phase 4 (cohort/bucket targeting engine) built a full admin UI for `ConfirmationSequence` (`ConfirmationsSection.tsx`, wired as a "Confirmations" tab alongside the pre-existing "Automation" tab on the Day List Administration page). Reviewing that alongside the existing `DayListAutomationSection.tsx` ("Automation" tab) surfaced a real question: should these stay as two separate automation concepts, or should one absorb the other?

Investigation found `ConfirmationSequence` is a strict superset of `DayListAutomation`'s capabilities except for three things `DayListAutomation` has that `ConfirmationSequence` doesn't:
1. Per-automation configurable check frequency (`run_every_minutes`) — not relevant to `ConfirmationSequence`'s due-timestamp-based timing model, and out of scope for this change (not being added).
2. A working test-send action — out of scope for this change (not being added).
3. Free-text message override without needing a saved template, plus a placeholder picker — **this is what this design covers.**

Given `DayListAutomation` has zero production data (pre-launch), the decision is: **retire `DayListAutomation` entirely** and bring only the free-text/placeholder capability forward into `ConfirmationSequence`, matching the existing pattern already built for Recall's own sequence editor (`src/pages/recalls/components/administration/AutomationSection.tsx`).

## What gets removed

- Django: `DayListAutomation` model, its migration removal (drop the table), `DayListAutomationSerializer`, `DayListAutomationViewSet`, its URL registration, the `process-daylist-automations` Celery beat entry, and the `dentallyIntegration/daylist_automation.py` module (including `fetch_at_risk_appointments`, `test_send_daylist_automation`, and any other functions solely used by it).
- Frontend: `DAYLIST_AUTOMATIONS`/`DAYLIST_AUTOMATION_DETAIL`/`DAYLIST_AUTOMATION_TEST` entries in `src/config/api.ts`, `src/pages/day-list/components/administration/DayListAutomationSection.tsx`, and the "Automation" tab entry + its `isAutomation` flag in `src/pages/day-list/components/DayListAdministration.tsx` (the existing "Confirmations" tab/flag stays, becoming the only automation tab).

## What gets added to `ConfirmationSequence`

### Backend: free-text step override

Each entry in `ConfirmationSequence.steps` (a `JSONField` list of dicts) gains two new optional keys, matching `RecallSequence.steps`'s exact shape:

```json
{"channel": "sms", "template_id": 3, "offset_days": 7, "send_time": "09:00", "subject": "", "body": ""}
```

- `subject` — optional, email steps only (cosmetic for SMS).
- `body` — optional. **When non-blank, it overrides `template_id` entirely** — the step sends the custom copy instead of resolving any template. This is independent, always-available data (not a UI "mode" toggle) — both `template_id` and `body` can be present in the same step dict; `body`'s non-blank presence is what decides precedence at both validation and render time.

`ConfirmationSequenceSerializer.validate_targeting`... (existing) is untouched; a **new** `validate_steps` method (mirroring `RecallSequenceSerializer.validate_steps`'s per-step checks already established) is added, requiring each step to have `template_id` OR non-blank `body` (not neither) — same rule as Recall's, adapted to `ConfirmationSequence`'s step shape (channel `sms`/`email` only, no `task` channel type — `ConfirmationSequence` has no task-step concept, unlike Recall).

### Backend: send-time precedence

Today (post-Task 13), `advance_confirmation_enrollments` resolves `effective_template_id` as: sub-cohort's `template_override_id` (if the enrollment matched a sub-cohort with one) → else the step's own `template_id`. This design adds ONE more, higher-precedence layer on top:

**Precedence order (highest to lowest):**
1. **Step's own `body`** (custom text) — if non-blank, use it directly (with `step.get("subject") or ""` for email), skip template resolution entirely. This wins outright, even over a sub-cohort's `template_override_id`, since custom text is the most explicit, deliberate choice a staff member made for that specific step — a "custom text" choice and a "swap to a different template" choice aren't sensible to combine.
2. **Sub-cohort's `template_override_id`** (existing, Task 13) — if the step has no custom body, and the enrollment matched a sub-cohort with an override, resolve that template.
3. **Step's own `template_id`** (existing, pre-Task-13 default) — the fallback when neither of the above apply.

### Backend: placeholder endpoint fix

`EmailMessageTemplateViewSet.available_placeholders` (in `messaging/views/template_views.py`) currently returns one static, generic dict of placeholder categories regardless of caller context — confirmed it does NOT include `confirmation_link`, `practice_phone`, or `practitioner_name` (three of the seven real placeholders `confirmation_automation.render_message` actually supports), while including several categories (patient/treatment_plan/clinic/dentist/current_user/recall) irrelevant to confirmation messages and using mismatched names (`dentist_name` instead of `practitioner_name`, no flat `practice_name`/`practice_phone`) that would silently render as empty strings if used in a confirmation template.

Add an optional `template_type` query param to this action: when `template_type=appointment_confirmation`, return a dedicated dict matching `render_message`'s actual 7-key context exactly:

```json
{
  "confirmation": {
    "confirmation_link": "The patient's confirm/cancel link",
    "patient_name": "Patient's name",
    "appointment_date": "Appointment date",
    "appointment_time": "Appointment time",
    "practitioner_name": "Treating practitioner's name",
    "practice_name": "Practice name",
    "practice_phone": "Practice phone number"
  }
}
```

(Exact response shape/nesting to match whatever structure the endpoint already uses for its other categories, so the frontend's existing `StructuredPlaceholders`-consuming code doesn't need a different parsing path — just a new category key.) When `template_type` is absent or any other value, behavior is unchanged (returns the existing generic dict) — this is purely additive, no existing caller (Recall's `AutomationSection.tsx`, which doesn't pass this param) is affected.

### Frontend: step editor UI

In `ConfirmationsSection.tsx`'s step editor (built in the prior "Task 15" work), add — directly below each step's existing Template select, matching Recall's `AutomationSection.tsx` layout exactly — an always-visible "Custom copy (optional — overrides the template)" section:
- Email steps: a `subject` `Input` (optional) + `body` `Textarea`.
- SMS steps: just the `body` `Textarea`.

Add a `PlaceholderPicker` component, defined locally in `ConfirmationsSection.tsx` (confirmed: Recall's own `PlaceholderPicker` is locally-defined in `AutomationSection.tsx`, not a shared/exported component — Day List's version is similarly local-only — so this new one follows the same established precedent rather than extracting a shared component): a `Popover` (ghost icon button, `Hash` icon, "Insert a placeholder") → `ScrollArea` → `Accordion` (all groups open) → `BadgePill` tokens that insert at cursor position into whichever step's subject/body field currently has focus (tracked via a ref, same mechanic as Recall's `activeFieldRef`). Placeholders are fetched once on mount from the now-parameterized endpoint with `template_type=appointment_confirmation`.

Validation in the save flow updates to match the new per-step rule: a step needs `template_id` OR non-blank `body`.

## Out of scope

- `DayListAutomation`'s per-row `run_every_minutes` frequency control and its test-send action — neither is being carried forward; `ConfirmationSequence` continues to run on the existing shared Celery beat cadence with no test-send equivalent.
- Any `trigger_type`/daily-schedule-vs-appointment-anchored unification — explicitly decided against; `ConfirmationSequence` remains purely appointment-anchored.
- Extracting a shared `PlaceholderPicker` component used by both Recall and Confirmations — each stays locally-defined in its own file, matching the existing precedent (Day List's version is also local-only, not shared).

## Testing

- Backend: `validate_steps` accepts a `body`-only step (no `template_id`) and rejects a step with neither; send-time resolution tests for all three precedence levels (custom body wins over sub-cohort override; sub-cohort override wins over step template_id when no custom body; step template_id used when neither of the above); `available_placeholders` returns the confirmation-specific dict when called with `template_type=appointment_confirmation`, and the original generic dict otherwise (regression-proofing Recall's existing usage).
- Frontend: tsc baseline check (no new errors) before/after; no manual browser verification (per standing preference for this session).

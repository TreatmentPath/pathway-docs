# Inbound Reply Activity Logging Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Inbound patient replies — SMS, email, and WhatsApp — appear in the patient activity log, exactly like outbound sends already do.

**Architecture:** Outbound sends call `TreatmentPath.record_events.on_record_created` at three sites; the three inbound ingestion paths (`receive_sms`, `receive_email`/`receive_email_v2`, `whatsapp_incoming_webhook`) create messages without logging. `on_record_created` already derives `sms_received`/`email_received` from `direction`, and `_build_description` already renders "SMS/Email received from …" — so the fix is: (a) call `on_record_created` from the three inbound paths, (b) teach it WhatsApp (new activity types + person resolution via `session.channel`, since `WhatsAppMessage` has no `channel` FK of its own).

**Tech Stack:** Django 5.1 + DRF (TreatmentPathBackend), React (patient panel accordion).

---

### Task 1: `record_events.py` — WhatsApp support + session→person traversal

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPath/record_events.py`
- Test: extend existing `TreatmentPlan`-adjacent record_events tests (locate: `grep -rln on_record_created TreatmentPath.Tests` → add cases)

- [ ] **Step 1: Failing tests** — (a) `WhatsAppMessage(direction="incoming")` → activity_type `whatsapp_received`, description `WhatsApp received from {contact_name}`; (b) an entity with no `channel` but `session.channel` (WhatsApp shape) still resolves a person.
- [ ] **Step 2: Implement**
  - `_get_person_from_entity`: after the `channel` branch, add `session = getattr(entity, "session", None)` → `channel = getattr(session, "channel", None)` → same PersonChannel traversal.
  - `_entity_type_map`: add `"whatsappmessage": "message"`.
  - `on_record_created`: `elif model_name == "whatsappmessage": activity_type = _get_whatsapp_activity_type(entity)` (new helper mirroring the SMS one: `whatsapp_sent`/`whatsapp_received`).
  - `_build_activity_metadata`: `whatsappmessage` branch — direction, contact_name, content snippet.
  - `_build_description`: `whatsappmessage` branch — `WhatsApp sent to {contact_name}` / `WhatsApp received from {contact_name}`.
- [ ] **Step 3: Run tests** — new cases pass.
- [ ] **Step 4: Commit** `feat: whatsapp activity types + session-based person resolution in record_events`.

### Task 2: Activity types + category map + migration

**Files:**
- Modify: `activityLog/models.py` ACTIVITY_TYPE_CHOICES — add `("whatsapp_sent", "WhatsApp Sent")`, `("whatsapp_received", "WhatsApp Received")` in the communication block.
- Modify: `activityLog/views.py` audit-log category map — `whatsapp_sent`/`whatsapp_received` → `"communication"`.
- Generate: choices-only migration via `makemigrations activityLog`.
- [ ] Run `makemigrations`, verify the migration is choices-only (AlterField, no DDL), run `python manage.py test activityLog --keepdb`.
- [ ] Commit `feat: whatsapp activity types in activity log`.

### Task 3: Log the three inbound ingestion paths

**Files:**
- Modify: `messaging/views/message_views.py` — `receive_sms` (~line 507 after `_create_sms_message`), `receive_email` (~line 2090), `receive_email_v2` (~line 2621) after `_create_email_message`.
- Modify: `messaging/views/whatsapp_views.py` — `whatsapp_incoming_webhook` (~line 537 after `WhatsAppMessage.objects.create`).
- Test: extend `messaging/tests.py` webhook tests.

Pattern (identical to the outbound call at `message_views.py:329`):

```python
try:
    from TreatmentPath.record_events import on_record_created

    on_record_created(entity=sms_message)  # user=None → is_system_generated
except Exception as activity_error:
    logger.warning(f"Failed to create activity log for received SMS: {activity_error}")
```

Ingestion must never fail because logging did — the try/except is mandatory.

- [ ] **Step 1: Failing tests** — POST each webhook (existing test fixtures show auth/URL shape) → exactly one Activity exists with `activity_type` = `sms_received` / `email_received` / `whatsapp_received` and a "received" description. Run first: expect FAIL (no Activity created).
- [ ] **Step 2: Implement** the four call sites.
- [ ] **Step 3: Run tests** — pass; confirm the ingest responses are unchanged.
- [ ] **Step 4: Commit** `feat: log inbound SMS/email/WhatsApp replies to the activity log`.

### Task 4: Frontend activity-type mapping

**Files:**
- Modify: `perfect-pixel-playground-project/src/components/patients/patient-panel/PatientPanelAccordion.tsx` — `mapActivityType`: `whatsapp_sent`/`whatsapp_received` → icon/type. Check `ActivityItem['type']` union first; if a `whatsapp` type/icon doesn't exist, map to `'sms'` (communication family) rather than inventing UI.

### Task 5: Verification

- [ ] Backend: `python manage.py test messaging.tests --keepdb` (or the targeted test module) + `activityLog`.
- [ ] Frontend: `npx vitest run` for the touched component suite; `npm run typecheck` clean on touched files.
- [ ] black + isort on touched Python; eslint on touched TS.
- [ ] No scratch files; no secrets.

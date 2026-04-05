# Alerts System — Full Handover Document

## Overview

The alerts system allows each practice to configure automated email notifications for key clinical and HR events. Settings are stored per-practice in the database and toggled via the Settings UI. Sending is handled by Celery tasks — either scheduled (Beat) or signal-triggered.

---

## 1. The Four Alert Types

| Alert | Trigger | Recipients | Configurable |
|---|---|---|---|
| Admin Daily HR & Compliance Digest | Celery Beat every 15 min (send-time window) | HR admin users | Send time (HH:MM) |
| HR Core Rota Alerts | Signal: RoomAssignment created/updated | Assigned staff member | None |
| Payslip Published Alert | Signal: Payslip status → PUBLISHED | Payslip owner | None |
| Lab Work Due Alert | Celery Beat daily at 07:00 UTC | HR admins + assigned dentist | Reminder days (1–365) |

---

## 2. Data Model

**Model:** `PracticeAlertSettings`
**App:** `UserAuthentication`
**File:** `TreatmentPathBackend/TreatmentPath/UserAuthentication/models.py`

```python
class PracticeAlertSettings(models.Model):
    practice = models.OneToOneField(Practice, on_delete=models.CASCADE, related_name="alert_settings")

    admin_daily_digest_enabled    = models.BooleanField(default=False)
    admin_daily_digest_send_time  = models.TimeField(default="08:00:00")

    hr_core_rota_alerts_enabled       = models.BooleanField(default=False)
    payslip_published_alerts_enabled  = models.BooleanField(default=False)

    lab_due_alerts_enabled    = models.BooleanField(default=False)
    lab_due_reminder_days     = models.PositiveSmallIntegerField(default=3)

    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

- One record per practice, auto-created via `post_save` signal on `Practice`.
- Migration: `UserAuthentication/migrations/0030_practice_alert_settings.py`

---

## 3. API Endpoint

**URL:** `/api/backend/settings/alerts/`
**ViewSet:** `PracticeAlertSettingsViewSet`
**File:** `TreatmentPathBackend/TreatmentPath/settings/views.py`
**Permission:** `IsAuthenticated` + `FeatureAccessPermission` (requires `practice_settings` feature)

### GET `/settings/alerts/`
Returns settings for the current user's practice. Uses `get_or_create` so it never 404s.

**Response:**
```json
{
  "admin_daily_digest_enabled": false,
  "admin_daily_digest_send_time": "08:00:00",
  "hr_core_rota_alerts_enabled": false,
  "payslip_published_alerts_enabled": false,
  "lab_due_alerts_enabled": false,
  "lab_due_reminder_days": 3,
  "updated_at": "2026-03-13T22:16:00Z"
}
```

### PATCH `/settings/alerts/`
Partial update — only send fields you want to change.

**Request body:**
```json
{
  "admin_daily_digest_enabled": true,
  "admin_daily_digest_send_time": "09:00:00",
  "hr_core_rota_alerts_enabled": true,
  "payslip_published_alerts_enabled": true,
  "lab_due_alerts_enabled": true,
  "lab_due_reminder_days": 5
}
```

**Validation:**
- `lab_due_reminder_days` must be between 1 and 365 (enforced in serializer)

**Serializer:** `PracticeAlertSettingsSerializer`
**File:** `TreatmentPathBackend/TreatmentPath/settings/serializers.py`

**URL registration:** `TreatmentPathBackend/TreatmentPath/settings/urls.py`
```python
path("alerts/", PracticeAlertSettingsViewSet.as_view({"get": "retrieve", "patch": "partial_update"}), name="practice-alert-settings"),
```

---

## 4. Frontend

**UI Component:** `perfect-pixel-playground-project/src/pages/settings/components/practice/PracticeAlertsTab.tsx`

- Pure controlled component — receives `alertSettings` and `onChange` as props
- Renders 4 rows with toggle switches; Digest and Lab Due have an edit modal for their extra config
- No network calls inside the component

**State lives in:** `src/pages/Settings.tsx`

```typescript
// State
const [alertSettings, setAlertSettings] = useState<PracticeAlertSettings>(DEFAULT_PRACTICE_ALERT_SETTINGS);
const [initialAlertSettings, setInitialAlertSettings] = useState<PracticeAlertSettings>(DEFAULT_PRACTICE_ALERT_SETTINGS);
const alertSettingsLoadedRef = useRef(false); // prevents duplicate fetches on remount
```

**Fetch (on Practice page load):**
```typescript
// ~line 455
if (!filterFetchCache.practiceAlerts) {
  fetchWithAuth(API_ENDPOINTS.practiceAlertSettings.get())
    .then(res => res.json())
    .then(data => {
      setAlertSettings({ ...mapped camelCase fields });
      setInitialAlertSettings({ ...same });
    });
}
```

**Save (on Save button click, alerts tab active):**
```typescript
// ~line 1164
await fetchWithAuth(API_ENDPOINTS.practiceAlertSettings.update(), {
  method: "PATCH",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    admin_daily_digest_enabled: alertSettings.adminDailyDigestEnabled,
    admin_daily_digest_send_time: alertSettings.adminDailyDigestSendTime,
    hr_core_rota_alerts_enabled: alertSettings.hrCoreRotaAlertsEnabled,
    payslip_published_alerts_enabled: alertSettings.payslipPublishedAlertsEnabled,
    lab_due_alerts_enabled: alertSettings.labDueAlertsEnabled,
    lab_due_reminder_days: alertSettings.labDueReminderDays,
  }),
});
```

**Field name mapping (API → Frontend):**

| API (snake_case) | Frontend (camelCase) |
|---|---|
| `admin_daily_digest_enabled` | `adminDailyDigestEnabled` |
| `admin_daily_digest_send_time` | `adminDailyDigestSendTime` |
| `hr_core_rota_alerts_enabled` | `hrCoreRotaAlertsEnabled` |
| `payslip_published_alerts_enabled` | `payslipPublishedAlertsEnabled` |
| `lab_due_alerts_enabled` | `labDueAlertsEnabled` |
| `lab_due_reminder_days` | `labDueReminderDays` |

**TypeScript types:** `src/pages/settings/types.ts`
```typescript
export interface PracticeAlertSettings {
  adminDailyDigestEnabled: boolean;
  adminDailyDigestSendTime: string; // "HH:MM:SS"
  hrCoreRotaAlertsEnabled: boolean;
  payslipPublishedAlertsEnabled: boolean;
  labDueAlertsEnabled: boolean;
  labDueReminderDays: number;
}

export const DEFAULT_PRACTICE_ALERT_SETTINGS: PracticeAlertSettings = {
  adminDailyDigestEnabled: false,
  adminDailyDigestSendTime: "08:00:00",
  hrCoreRotaAlertsEnabled: false,
  payslipPublishedAlertsEnabled: false,
  labDueAlertsEnabled: false,
  labDueReminderDays: 3,
};
```

---

## 5. Celery Tasks

### 5.1 Admin Daily HR & Compliance Digest
**File:** `TreatmentPathBackend/TreatmentPath/HR/tasks.py`
**Task:** `send_daily_hr_digest`
**Schedule:** Every 15 minutes via Celery Beat

**Logic:**
1. Iterates all practices with `admin_daily_digest_enabled=True`
2. Converts current UTC time to the practice's timezone
3. Checks if the current time falls within the 15-minute window around `admin_daily_digest_send_time`
4. Collects digest content:
   - New leave requests (created in last 24h, status=pending)
   - Outstanding leave requests (created >24h ago, status=pending)
   - Missing clock-ins (published shifts whose start time has passed)
   - Active clock-sessions with no clock-out
5. Skips if digest is empty (nothing to report)
6. Sends one email per HR admin user
7. Logs to `AlertDeliveryLog`

**Dedupe key:** `digest:{practice_id}:{YYYY-MM-DD}` — one digest per practice per calendar day

---

### 5.2 HR Core Rota Alerts (Published)
**File:** `TreatmentPathBackend/TreatmentPath/HR/tasks.py`
**Task:** `send_rota_published_alert(assignment_id)`
**Trigger:** Signal — `RoomAssignment` post_save, created with `status='published'`

**Logic:**
1. Fetches the `RoomAssignment`
2. Checks `hr_core_rota_alerts_enabled` on the practice's alert settings
3. Gets the assigned staff member
4. Verifies they have `hr_view` feature access
5. Sends email with room, date, shift times
6. Logs to `AlertDeliveryLog`

**Dedupe key:** `rota_published:{assignment_id}:{recipient_id}`

---

### 5.3 HR Core Rota Alerts (Changed)
**File:** `TreatmentPathBackend/TreatmentPath/HR/tasks.py`
**Task:** `send_rota_changed_alert(assignment_id, before_state)`
**Trigger:** Signal — `RoomAssignment` post_save, meaningful fields changed

**Meaningful fields tracked:** `date`, `shift_type`, `start_time`, `end_time`, `room_id`, `staff_id`

**Logic:** Same as published alert but sends "shift updated" email showing what changed.

**Dedupe key:** `rota_changed:{assignment_id}:{recipient_id}:{updated_at_timestamp}`

---

### 5.4 Lab Work Due Alert
**File:** `TreatmentPathBackend/TreatmentPath/Labs/tasks.py`
**Task:** `send_daily_lab_due_alerts`
**Schedule:** Daily at 07:00 UTC via Celery Beat

**Logic:**
1. Iterates all practices with `lab_due_alerts_enabled=True`
2. Calculates target date: `today (practice timezone) + lab_due_reminder_days`
3. Finds all `LabCase` records where `appointment_date == target_date` and `status != 'arrived'`
4. For each case, sends to: all HR admin users + assigned dentist (deduped)
5. Logs to `AlertDeliveryLog`

**Dedupe key:** `lab_due:{lab_case_id}:{reminder_days}:{recipient_id}`

---

### 5.5 Payslip Published Alert
**File:** `TreatmentPathBackend/TreatmentPath/Invoices/tasks.py`
**Task:** `send_payslip_published_alert(payslip_id)`
**Trigger:** Signal — `Payslip` pre_save, status changes to `PUBLISHED`

**Logic:**
1. Fetches the `Payslip`
2. Checks `payslip_published_alerts_enabled`
3. Gets the payslip owner as recipient
4. Sends email with payslip period info
5. Logs to `AlertDeliveryLog`

**Dedupe key:** `payslip_published:{payslip_id}:{recipient_id}`

---

## 6. Signal Wiring

**HR Signals:** `TreatmentPathBackend/TreatmentPath/HR/signals.py`
```
pre_save  → RoomAssignment  → captures before-state dict
post_save → RoomAssignment  → queues send_rota_published_alert or send_rota_changed_alert
```

**Invoice Signals:** `TreatmentPathBackend/TreatmentPath/Invoices/signals.py`
```
pre_save  → Payslip  → detects status transition to PUBLISHED → queues send_payslip_published_alert
```

All tasks are queued inside `transaction.on_commit()` to guarantee the DB write completes before the task runs.

---

## 7. Email Rendering & Sending

**Template functions:** `TreatmentPathBackend/TreatmentPath/UserAuthentication/alert_templates.py`

| Function | Alert |
|---|---|
| `render_digest_email()` | Admin Daily Digest |
| `render_rota_published_email()` | Rota Published |
| `render_rota_changed_email()` | Rota Changed |
| `render_payslip_email()` | Payslip Published |
| `render_lab_due_email()` | Lab Due |

Each returns `(subject, body_html, body_text)`.

**Email utilities:** `TreatmentPathBackend/TreatmentPath/UserAuthentication/alert_email_utils.py`

| Function | Purpose |
|---|---|
| `get_practice_sender()` | Resolves from-address: preferred domain → practice email → system default |
| `send_alert_email()` | Sends via email service client, returns `{'job_id': ...}` |
| `get_hr_admin_recipients()` | Returns users with `hr_admin` feature access |
| `get_hr_view_recipients()` | Returns users with `hr_view` feature access |

---

## 8. Idempotency — AlertDeliveryLog

**Model:** `AlertDeliveryLog`
**File:** `TreatmentPathBackend/TreatmentPath/messaging/models.py`

Every sent alert is recorded here. Before sending, each task checks if a record with the same `dedupe_key` already has `status='sent'` — if so, it skips.

**Key fields:**

| Field | Purpose |
|---|---|
| `practice` | Which practice |
| `alert_type` | One of: `admin_daily_digest`, `hr_rota_published`, `hr_rota_changed`, `payslip_published`, `lab_due` |
| `entity_type` | e.g. `rota_assignment`, `payslip`, `lab_case`, `digest` |
| `entity_id` | UUID or date string of the entity |
| `recipient` | User who received it |
| `dedupe_key` | Unique string per alert + recipient combo |
| `status` | `pending` → `sent` or `failed` |
| `email_job_id` | Job ID returned by email service |
| `error_message` | Populated on failure |

**DB constraint:** `unique_together = [["practice", "dedupe_key"]]`

---

## 9. Celery Beat Schedule

**File:** `TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py`

```python
CELERY_BEAT_SCHEDULE = {
    "send-daily-hr-digest": {
        "task": "HR.tasks.send_daily_hr_digest",
        "schedule": crontab(minute="*/15"),  # Every 15 minutes
    },
    "send-daily-lab-due-alerts": {
        "task": "Labs.tasks.send_daily_lab_due_alerts",
        "schedule": crontab(hour=7, minute=0),  # 07:00 UTC daily
    },
}
```

---

## 10. Adding a New Alert Type — Checklist

If someone needs to add a fifth alert type, here are the steps:

1. **Model** — Add boolean flag (and any config field) to `PracticeAlertSettings` + migration
2. **Serializer** — Add field to `PracticeAlertSettingsSerializer.Meta.fields`
3. **Frontend types** — Add field to `PracticeAlertSettings` interface and `DEFAULT_PRACTICE_ALERT_SETTINGS` in `src/pages/settings/types.ts`
4. **UI** — Add a row entry in the `rows` array in `PracticeAlertsTab.tsx`
5. **Field mapping** — Add to the GET mapping in `Settings.tsx` (response → state) and the PATCH body (state → request)
6. **Email template** — Add a `render_*_email()` function in `alert_templates.py`
7. **Celery task** — Create the task in the relevant app's `tasks.py` with retry logic and `AlertDeliveryLog` recording
8. **Signal or Beat schedule** — Wire the trigger in `signals.py` or `settings.py`
9. **`AlertDeliveryLog`** — Add the new `alert_type` to `ALERT_TYPE_CHOICES`

---

## 11. Key File Index

| What | Where |
|---|---|
| DB model | `UserAuthentication/models.py` → `PracticeAlertSettings` |
| Serializer | `settings/serializers.py` → `PracticeAlertSettingsSerializer` |
| API view | `settings/views.py` → `PracticeAlertSettingsViewSet` |
| URL route | `settings/urls.py` → `alerts/` |
| Digest task | `HR/tasks.py` → `send_daily_hr_digest` |
| Rota tasks | `HR/tasks.py` → `send_rota_published_alert`, `send_rota_changed_alert` |
| Lab task | `Labs/tasks.py` → `send_daily_lab_due_alerts` |
| Payslip task | `Invoices/tasks.py` → `send_payslip_published_alert` |
| HR signals | `HR/signals.py` |
| Invoice signals | `Invoices/signals.py` |
| Email templates | `UserAuthentication/alert_templates.py` |
| Email utilities | `UserAuthentication/alert_email_utils.py` |
| Delivery log | `messaging/models.py` → `AlertDeliveryLog` |
| Beat schedule | `TreatmentPath/settings.py` → `CELERY_BEAT_SCHEDULE` |
| Frontend UI | `src/pages/settings/components/practice/PracticeAlertsTab.tsx` |
| Frontend types | `src/pages/settings/types.ts` → `PracticeAlertSettings` |
| Frontend state/API calls | `src/pages/Settings.tsx` |
| API endpoint config | `src/config/api.ts` → `practiceAlertSettings` |

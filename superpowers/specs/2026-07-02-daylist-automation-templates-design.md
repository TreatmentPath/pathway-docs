# Daylist Automation — Template Support

## Context

Day List Automation (`DayListAutomation` model, `dentallyIntegration/`) sends
at-risk-appointment outreach on a schedule, but the message content is
free-text only (`message_subject`/`message_body`). Recall Automation
(`RecallSequence`) already supports picking a reusable message/SMS template
per step, with an optional custom-text override. `EmailMessageTemplate` and
`SMSMessageTemplate` already have a `"daylist"` `template_type` option (visible
today in Settings > Templates), but nothing consumes it yet.

Goal: let a Day List automation reference a template, following the same
resolution rule Recall uses, without adding a multi-step engine (Day List
stays single-step, one automation = one message rule).

## Backend

**Model** (`dentallyIntegration/models.py`, `DayListAutomation`): add

```python
template_id = models.IntegerField(null=True, blank=True)
```

Plain int, not a FK — matches Recall's own `template_id` convention, and
avoids needing two nullable FK columns (one per template model) when the
model already knows which table to check via `channel`. New migration
`0145_daylistautomation_template_id.py` (next after `0144_dentallypatientaisummary_raw_notes.py`).

**Serializer** (`DayListAutomationSerializer`): add `template_id` to fields.
`validate()` changes from "require non-blank `message_body`" to "require
non-blank `message_body` OR a `template_id`".

**Resolution rule** (`dentallyIntegration/daylist_automation.py`), mirrors
Recall's `custom_body or _resolve_template(...)`:

```python
def _resolve_daylist_template(practice, channel, template_id):
    """Looks up EmailMessageTemplate/SMSMessageTemplate by channel + pk,
    scoped to practice + is_active=True. Returns (subject, body) or
    (None, None) if missing/inactive — never raises, matches
    fetch_at_risk_appointments' never-crash-a-scheduled-run convention."""

def _resolve_content(automation):
    if automation.message_body:
        return automation.message_subject, automation.message_body
    if automation.template_id:
        return _resolve_daylist_template(
            automation.practice, automation.channel, automation.template_id
        )
    return "", ""
```

`run_daylist_automation` and `test_send_daylist_automation` both call
`_resolve_content(automation)` once per run to get `(subject_str, body_str)`,
then pass those through the existing `render_message(...)` call sites
unchanged (same flat placeholder context: `patient_name`, `appointment_date`,
`appointment_time`, `practitioner_name`, `practice_name`, `confirmation_url`,
`risk_tier`). Templates authored for Day List use this same placeholder set —
no change to `render_message` itself.

If resolution yields an empty body (template missing/inactive and no custom
text), skip sending for that run the same way a missing phone/email is
currently skipped (`failed += 1; continue`), and log a warning.

## Frontend

`DayListAutomationSection.tsx`:

- Fetch active templates on mount/channel-change:
  `fetchActiveEmailTemplates({ templateType: 'daylist' })` /
  `fetchActiveSmsTemplates({ templateType: 'daylist' })`, matching the
  automation's current `channel`, same pattern as
  `AutomationSection.tsx:227-228`.
- Add a "Template" `<Select>` bound to `editing.template_id`, listed above the
  message fields.
- Relabel the existing `message_subject` Input / `message_body` Textarea
  section as "Custom copy (optional — overrides the template)", matching
  Recall's `AutomationSection.tsx` lines ~490-541/544+ copy and layout.
- No auto-fill on template pick — `template_id` is stored as-is. Admins use
  the existing "Send test" action to preview the resolved message
  (`test_send_daylist_automation` already returns `rendered_message`).
- Types: add `template_id: number | null` to the `DayListAutomation`
  interface.

## Out of scope

- No changes to `EmailMessageTemplate`/`SMSMessageTemplate` models or the
  Settings > Templates CRUD UI — `"daylist"` template_type already exists and
  is creatable there today.
- No change to the placeholder/context scheme used by `render_message`.
- No multi-step sequence support for Day List (stays single-step, unlike
  Recall's `steps` JSON array).

# dentallyIntegration identity audit

Scope: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/` only. I read
code only; I did not execute the application, tests, or database queries.

## 1. Critical — a reply from one family member stops another member’s recall sequence

**Evidence:** [recall_automation.py:387](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/recall_automation.py:387)-[429](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/recall_automation.py:429).

```python
phone_channel = _find_channel(practice, ContactChannel.PHONE, phone)
email_channel = _find_channel(practice, ContactChannel.EMAIL, email)
...
EmailMessages.objects.filter(
    practice=practice,
    channel=email_channel,
    direction="incoming",
    received_at__gte=since,
).exists()
...
SMSMessage.objects.filter(
    practice=practice,
    channel=phone_channel,
    direction="incoming",
    created_at__gte=since,
).exists()
```

`process_recall_enrollments` calls this predicate for the enrollment’s exact
`dentally_patient_id` ([recall_automation.py:909](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/recall_automation.py:909)-[927](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/recall_automation.py:927)). The predicate discards that id and asks whether *anyone* has sent an inbound message on the shared channel after the enrollment time.

Concrete failure: Alice (`RecallRecord.dentally_patient_id=100`) and her son Ben (`101`) are distinct Persons but both have `phone_channel=44` for `+447700900123`. Alice has an active recall enrollment at 09:00. Ben replies “Please stop” at 10:00 to his own recall SMS. When Alice’s enrollment is processed, the quoted SMS query finds Ben’s row solely because it has `channel_id=44`, returns `True`, and the caller sets Alice’s enrollment to `status="stopped"`, `stopped_reason="replied"`. Alice is treated as the human who replied and receives no remaining recall contact.

Closest fixed item: **#76** (“the shared-line guard did not cover its own fallthrough”). This is genuinely distinct: it is the recall sequence’s stop-state transition, not that prior guard/path; it has no Person/Patient/Dentally-id predicate after entering `patient_replied_since`.

## 2. High — per-patient recall reporting attributes a relative’s messages and calls to the requested patient

**Evidence:** [views/recall_sequence_views.py:247](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/views/recall_sequence_views.py:247)-[294](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/views/recall_sequence_views.py:294), [397](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/views/recall_sequence_views.py:397)-[407](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/views/recall_sequence_views.py:407), and [440](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/views/recall_sequence_views.py:440)-[476](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/views/recall_sequence_views.py:476).

```python
rec = RecallRecord.objects.filter(
    practice=practice, dentally_patient_id=dentally_patient_id
).first()
...
ch = ContactChannel.objects.filter(
    practice=practice, kind=ContactChannel.EMAIL, canonical_value=email
).values("id").first()
...
email_qs = email_qs.filter(channel_id=contact_id)
sms_qs = sms_qs.filter(channel_id=contact_id)
call_qs = call_qs.filter(channel_id=contact_id)
```

The request starts with a specific Dentally patient id, but `_resolve_contact`
reduces it to a `ContactChannel.id`; the result query retains only that channel
id. A ContactChannel is explicitly allowed to be shared by several Persons, so
the event loops and count mode return every recall event on the household
mailbox/phone as if it belonged to the requested patient.

Concrete failure: Alice (`dentally_patient_id=100`) and Ben (`101`) share
`family@example.test`, backed by email channel `17`. A recall email for Ben is
stored as `EmailMessages(practice=P, channel_id=17, message_purpose="recall")`.
`GET recall-reporting/?dentally_patient_id=100&mode=events` resolves Alice’s
RecallRecord to channel `17`, then the quoted `email_qs.filter(channel_id=17)`
returns Ben’s email. Lines 449–457 render that row as an event in Alice’s
report; count mode includes it in Alice’s `total_contact_attempts` at lines
525–550.

Closest fixed item: **#75** (“call-log patient attribution picked an arbitrary sibling”). This is genuinely distinct: no event is re-attributed or selected by `.first()` among Persons here. Instead this reporting endpoint throws away the caller’s exact patient id and reports the entire shared channel’s event history under that patient.

## 3. High — confirmation task discards the appointment’s exact Patient and files the task against an arbitrary Patient on its Person

**Evidence:** [confirmation_automation.py:235](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/confirmation_automation.py:235)-[266](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/confirmation_automation.py:266) and [857](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/confirmation_automation.py:857)-[908](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/confirmation_automation.py:908).

```python
def _get_patient_for_appointment(appointment):
    return Patient.objects.select_related("person").get(
        practice_id=appointment.practice_id,
        meta_data__id=appointment.dentally_patient_id,
    )
...
person = _get_contact_for_appointment(appointment)
patient = None
if person:
    patient = person.patients.filter(practice=practice).first()
...
Task.objects.create(..., patient=patient, ...)
```

The helper has already resolved the appointment’s own Patient by the scoped
Dentally id, but `create_confirmation_task` carries only its Person forward and
then uses unordered `.first()` over that Person’s Patient rows. The exact
patient is therefore discarded before `Task.patient` is written.

Concrete failure: a legacy fused `Person(id=501)` is linked to both
`Patient(id=20, meta_data={"id": 100}, first_name="Alice")` and
`Patient(id=21, meta_data={"id": 101}, first_name="Ben")` in practice P. A
`DentallyAppointment(dentally_patient_id=100)` reaches this function. The
helper correctly resolves Patient 20, but the quoted queryset can return
Patient 21 first; `Task.objects.create(patient=patient)` then files Alice’s
confirmation task under Ben’s record. Database ordering is not specified
because the queryset has no `order_by`.

Closest fixed item: **#86** (“call agent collapsed a fused Person to its first Patient”). This is genuinely distinct: it is a separate confirmation-automation task creation path, and it uses the same unsafe Person-to-Patient collapse after an exact appointment patient lookup has already succeeded.

# Messaging identity sweep

Read-only audit. I executed no application code, tests, migrations, or requests; findings below are from source inspection only.

## 1. Critical — a family-contact thread accepts an unrelated same-practice patient and returns that patient's messages

**Evidence:** [`messaging/views/contact_views.py:530`](../../TreatmentPathBackend/TreatmentPath/messaging/views/contact_views.py#L530) resolves the contact named in the URL, but [`messaging/views/contact_views.py:542`](../../TreatmentPathBackend/TreatmentPath/messaging/views/contact_views.py#L542) validates the optional `patient_id` only against the practice:

```python
patient = Patient.objects.get(pk=patient_id, practice=practice)
...
patient_channel_ids = list(patient_person.channels.values_list("id", flat=True))
session_filter = {"session__channel_id__in": patient_channel_ids}
```

There is no check that `patient.person` is `channel_person`, that the patient's channels are in `identity_channel_ids`, or even that the patient is a member of the contact's household. The next query consumes that replacement filter directly:

```python
email_qs = EmailMessages.objects.filter(**session_filter)
```

at [`messaging/views/contact_views.py:559`](../../TreatmentPathBackend/TreatmentPath/messaging/views/contact_views.py#L559). The live route is [`messaging/urls.py:458`](../../TreatmentPathBackend/TreatmentPath/messaging/urls.py#L458).

**Concrete wrong-human scenario:** Practice `P` has a family/shared-line contact `C` (so `_is_family(C)` is true) for Alice Jones, whose channel id is `11`. Unrelated Bob Smith in the same practice has `Patient.id=91`, `Person.id=42`, channel id `27`, and email messages on sessions linked to channel `27`. A request for `GET /messaging/contacts/C/thread/?patient_id=91` passes line 542 because Bob is in `P`; lines 545–548 replace Alice's `[11]` filter with Bob's `[27]`; line 560 returns Bob's message history while the URL still identifies Alice's contact. This is a direct wrong-human attribution/disclosure within one practice.

**Already-fixed-index comparison:** It most resembles **#69** (the inbox showing the wrong practice's data after a practice switch) because both are inbox data leaks caused by an insufficient scope check. It is genuinely distinct: this endpoint retains the correct practice predicate but fails the *identity membership* predicate, allowing one human's `patient_id` to override a different human’s contact thread in that same practice.

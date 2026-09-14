# Area 2 — Django identity write / resolution paths

Audit method: **read-only code review only**. I did not execute Django, tests, migrations, or any write operation.

## 1. Critical — Treatment-plan creation arbitrarily assigns a plan to one of two same-named people sharing a channel

**Files:** `TreatmentPathBackend/TreatmentPath/TreatmentPlan/serializers/treatment_plan.py:1005-1039` (also the equivalent phone branch at `1017-1039`).

```python
person = (
    Person.objects.filter(
        practice=practice,
        person_channels__channel__kind=ContactChannel.EMAIL,
        person_channels__channel__canonical_value=normalized_email,
    )
    .distinct()
    .first()
)
...
patient = Patient.objects.filter(
    person=person,
    first_name__iexact=first_name,
    last_name__iexact=last_name,
).first()
if patient:
    validated_data["patient"] = patient
```

`PersonChannel` expressly permits a channel to be linked to more than one Person: its only uniqueness constraint is the pair `(person, channel)` ([`models.py:861-868`](../../TreatmentPathBackend/TreatmentPath/TreatmentPlan/models.py#L861-L868)). The query therefore has multiple valid rows for a shared family email/phone, but calls unordered `.first()` before the name is checked.

**Concrete wrong-human scenario:** In practice `P`, `Person(41, John Smith, dob=1960-01-01)` and `Person(42, John Smith, dob=1990-01-01)` are distinct people and both link to `ContactChannel(email, "smith-family@example.test")`. Their respective Patients are `701` and `702`. Creating a plan with no `patient_id` and `patient_data={"first_name": "John", "last_name": "Smith", "email": "smith-family@example.test"}` takes whichever Person the database returns first, then takes that Person's John Smith Patient and writes it into `validated_data["patient"]`. A plan intended for Patient `702` can therefore be persisted against Patient `701`; the supplied data carries no discriminator and the code neither rejects nor asks for review.

**Already-fixed comparison:** Most resembles **#18** (a booking path selected the first patient on a shared email) and **#72** (a plan path discarded a supplied patient id). It is genuinely distinct: this is the current `TreatmentPlanSerializer.create` fallback used only when no `patient_id` is supplied, and it chooses a **Person** first then a Patient. #72's explicit-id CARRY path is present and correct at lines 981-993; this remaining branch is a separate ambiguous DISCOVER path, including the same-name/different-DOB case that its name test cannot distinguish.

## 2. High — conversion relationship update writes the selected relationship to an arbitrary sibling

**Files:** `TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/conversion_views.py:299-325`; the Nurture equivalent is `623-649`.

```python
email = intake.email
phone = intake.phone_number
lookup = Q()
if email:
    lookup |= Q(email__iexact=email)
if phone and not email:
    lookup |= Q(phone_number=phone)
patient = (
    Patient.objects.filter(lookup, practice=intake.practice).first()
    if lookup
    else None
)
if (
    patient
    and patient.person_id
    and patient.person.household_id
    and patient.person.household.is_family
):
    _set_family_relationship(patient, relationship)
```

The request has already identified the source Intake (or Nurture) through the view's object lookup, yet this branch discards that record and re-discovers a Patient by a shared email — or, when email is absent, a shared phone. `_set_family_relationship` immediately persists the result:

```python
patient.family_relationship = relationship
patient.save(update_fields=["family_relationship"])
```

at [`conversion_views.py:120-141`](../../TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/conversion_views.py#L120-L141), and can also set relationship values on other patients in that household.

**Concrete wrong-human scenario:** Practice `P` has a family email `family@example.test` on Patient `101` / Person Alice and Patient `102` / Person Ben, with both Persons in a family household. An Intake for Ben is addressed by its URL, carries the same `family@example.test`, and the follow-up request posts `{"relationship": "child"}`. The query returns Alice if her row is first; `_set_family_relationship(Alice, "child")` persists `Alice.family_relationship = "child"` and may fill the other household members as `parent`. Ben's source record was known throughout, but the identity write is made to Alice.

**Already-fixed comparison:** Most resembles **#18** and **#73**, both arbitrary-sibling selection from a shared contact. It is genuinely distinct from those and from **#3**: this code neither creates a family member nor merges Persons. It is a later conversion endpoint that writes the `family_relationship` identity attribute to the wrong existing Patient after throwing away the Intake/Nurture identity it had.

## 3. High — a typed shared mailbox is silently mapped to one arbitrary Person for marketing test-send

**Files:** `TreatmentPathBackend/TreatmentPath/marketingBroadcast/views/campaign_views.py:510-524`, with the resulting personalised send at `539-560`.

```python
matched = dict(
    PersonChannel.objects.filter(
        person__practice=campaign.practice,
        channel__kind=ContactChannel.EMAIL,
        channel__canonical_value__in=emails,
    ).values_list("channel__canonical_value", "person_id")
)
lowered = {key.lower(): value for key, value in matched.items()}
for address in emails:
    person_id = lowered.get(address)
    if person_id is None:
        unmatched_addresses.append(address)
    elif person_id not in person_ids:
        person_ids.append(person_id)
```

When several `PersonChannel` rows have the same email-channel value, converting `(email, person_id)` rows to a dict overwrites all but one `person_id`. The queryset has no ordering, so the survivor is not an identity decision. The selected person is then used for eligibility, template rendering and that person's unsubscribe token:

```python
recipient = BroadcastRecipient(campaign=campaign, person=person)
send_broadcast_email(recipient, is_test=True)
```

[`delivery_reporting.py:165-182`](../../TreatmentPathBackend/TreatmentPath/marketingBroadcast/delivery_reporting.py#L165-L182) renders with that Person and builds a per-Person preferences link.

**Concrete wrong-human scenario:** In practice `P`, Alice (`Person 11`) and Ben (`Person 12`) legitimately share the `family@example.test` ContactChannel. A staff member posts `{"emails": ["family@example.test"]}` to the campaign's `test-send` endpoint. `values_list` produces both `("family@example.test", 11)` and `("family@example.test", 12)`; `dict(...)` retains only whichever row is encountered last. If it retains `11`, the email sent to the shared mailbox is rendered with Alice's fields and Alice's unsubscribe token, even if the staff intended the typed address as a neutral test or Ben's contact. The response reports `sent: [11]`, cementing the false attribution.

**Already-fixed comparison:** Most resembles **#61** (a marketing webhook chose a first Person on a shared email). It is genuinely distinct: #61 is the Postmark webhook fallback and now uses `sole_person_for_channel`; this is the staff `test-send` ingress, whose different `dict(email -> person_id)` reduction still picks one shared-channel owner before it calls the send path.

## Non-findings intentionally omitted

I excluded the already documented first-space writers, booking matching, medical-history verification, and known shared-contact serializers from the count, including where the current source retains explanatory comments for their numbered fixes.

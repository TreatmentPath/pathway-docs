# activityLog identity audit

Scope: `TreatmentPathBackend/TreatmentPath/activityLog/` only. I read code and
migrations; I did **not** run Django, tests, migrations, or any requests.

## 1. High — same-practice API writes can attach Alice's patient activity to Bob

**Evidence**

`activityLog/serializers.py:120-126` validates only the supplied Person's
tenant:

```python
def validate_person(self, person):
    if person is None:
        return person
    practice = self._target_practice()
    if practice is None or person.practice_id != practice.id:
        raise serializers.ValidationError("Person is not in this practice.")
    return person
```

`activityLog/serializers.py:128-159` independently validates only that the
generic object exists and is in that same tenant:

```python
obj = model_class.objects.filter(pk=object_id).first()
...
obj_practice_id = getattr(obj, "practice_id", None)
...
if practice is None or obj_practice_id != practice.id:
    raise serializers.ValidationError(
        {"object_id": "That record is not in this practice."}
    )
```

Neither check compares a `Patient` object's `person_id` with the submitted
`person`. The create view saves the validated values unchanged at
`activityLog/views.py:178-190`, and patient history retrieves logs by their
stored `person_id` at `activityLog/views.py:671-683` (and events by it at
`activityLog/views.py:444-448`).

**Concrete wrong-human path**

In practice `P`, let `Patient(id=101, person_id=11)` be Alice and let
`Person(id=12)` be Bob. A client POSTs an ActivityLog with
`content_type=Patient`, `object_id=101`, and `person=12`. Both validation
blocks accept it because Alice and Bob are in `P`; the row is saved as the
activity for Bob. Bob's audit/timeline then selects `person_id=12` and displays
the event whose generic target is Alice's patient record. The patient activity
has therefore been attributed to the wrong human.

**Relation to already-fixed finding #40:** this resembles #40 because both are
ActivityLog write validation failures. It is genuinely distinct: #40's fix
stops a foreign-practice Person/object pair, while the current code still
accepts two *different people in the same practice*. The existing regression
tests cover the foreign cases only (`test_write_serializer_practice_scoping.py:58-149`), not the required entity-person relationship.

## 2. High — migration 0013 assigns each shared-channel email/SMS/call event to an arbitrary Person

**Evidence**

`activityLog/migrations/0013_backfill_activity_person.py:87-105` constructs a
single person per channel by keeping the first row yielded for that channel:

```python
chan_to_person = {}
for chid, pid in PersonChannel.objects.values_list("channel_id", "person_id"):
    chan_to_person.setdefault(chid, pid)
...
for pk, chid in Model.objects.filter(
    pk__in=[int(o) for o in oids], channel_id__isnull=False
).values_list("pk", "channel_id"):
    pid = chan_to_person.get(chid)
    if pid:
        id_to_person[str(pk)] = pid
_update_by_map(Activity, entity_type, id_to_person)
```

The migration's loop applies this mapping to all three entity types:
`emailmessages`, `smsmessage`, and `calllog`
(`activityLog/migrations/0013_backfill_activity_person.py:91-106`).
`setdefault` preserves the first yielded PersonChannel, and the queryset has no
ordering or ambiguity check.

**Concrete wrong-human path**

In practice `P`, a family SMS channel has `channel_id=22` and two links:
`PersonChannel(person_id=11, channel_id=22)` for Alice and
`PersonChannel(person_id=12, channel_id=22)` for Bob. An old inbound
`SMSMessage(id=900, channel_id=22)` has an Activity whose
`entity_type="smsmessage"`, `object_id="900"`, and null `person_id`.
Migration 0013 maps channel 22 to whichever of Alice/Bob its unordered query
returns first and writes that Person id to the Activity. The inbound message's
channel identifies the shared line, not either human, yet the resulting event
is permanently filed under one of them and appears in that person's timeline.

**Relation to already-fixed finding #75:** it most resembles #75 (a call-log
attribution choosing an arbitrary sibling). It is genuinely distinct because
this is a data migration in `activityLog`, not the call-log producer: it writes
the arbitrary attribution into historical Activity rows and does so for email
and SMS as well as calls. It also remains relevant whenever this migration is
applied to an environment carrying pre-Phase-5 events.


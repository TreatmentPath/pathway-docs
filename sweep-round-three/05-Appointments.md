# Appointments identity audit

## Result

**0 new findings.**

## Scope and method

Reviewed only `TreatmentPathBackend/TreatmentPath/Appointments/`, reading the
already-fixed index first. I used read-only shell commands (`rg`, `sed`, `nl`,
and `wc`) to inspect the source; I did **not** execute Django, run tests, call
external services, or modify application code.

## Checked and excluded

- Internal appointment creation/update accepts a `patient` id, but
  `AppointmentCreateUpdateSerializer` inherits
  `PracticeScopedPatientClinicianMixin`; `validate_patient` rejects a Patient
  whose `practice_id` differs from the request/appointment practice
  (`serializers.py:258-285`). This is the in-directory repair for the
  already-fixed #24, not a new instance.
- `ShortNoticePatientSerializer` also inherits that mixin
  (`serializers.py:882-889`), before the view writes the current practice
  (`views.py:1084-1087`). Therefore a request made in Practice A with a
  `patient` id belonging to Practice B fails `validate_patient`; it cannot
  create the cross-practice waitlist row that its model shape alone might
  suggest.
- Public booking persists the submitted name/email/phone as unlinked booking
  fields (`public_booking_views.py:329-343`) and does not resolve a Person or
  select a Patient by shared phone/email. This avoids the CARRY-vs-DISCOVER
  family-channel failure and is not the already-fixed #18/#19/#20/#33 pattern.
- The only relevant `.first()` calls are the consent-summary selection
  (`serializers.py:147-160`), ordered most-recent and keyed by this
  appointment's UUID or its already-linked global Patient id. They do not
  select a Person from a shared phone/email/name set.

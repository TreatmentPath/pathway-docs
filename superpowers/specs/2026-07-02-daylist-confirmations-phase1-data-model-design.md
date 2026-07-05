# Daylist Confirmations — Phase 1: Confirmation Data Model

## Context

`PRD/Daylist/daylist-confirmations-prd.md` describes a large v1 feature (25 functional requirements) letting practices run Pathway-managed appointment-confirmation sequences and see confirmation/cancellation state in Daylist. A gap analysis (`PRD/Daylist/summary2.md`) found the PRD's own claim of "no backend confirmation infrastructure" is stale — a working `confirm.dental` token/link system already exists (`dentallyIntegration/views/appointment_confirm_views.py`, `dentallyIntegration/confirm_utils.py`), backed by a bare `DentallyAppointment.patient_confirmed` boolean — but it has no audit trail (timestamp/source/channel) and no cancellation-request concept.

The gap analysis decomposed the PRD into 5 build phases. This is Phase 1: the confirmation data model, which everything else (the sequence engine, Daylist UI, cohort engine, follow-up tasks) depends on existing first.

**Scope boundary:** Phase 1 covers 5 of the 6 FR11 confirmation-display states (Confirmed, Awaiting confirmation, Cancellation requested, Cancelled, No valid contact). The 6th state, `Suppressed`, means "a confirmation sequence stopped sending to this appointment" — that concept doesn't exist until Phase 2 (the sequence engine) ships, so it's explicitly deferred, not stubbed.

## Go-sync safety (verified before designing further)

`dentally_appointment` is a table the Go service (`EmailServiceGo`) writes to directly via a hand-authored raw-SQL `INSERT ... ON CONFLICT ... DO UPDATE` with an explicit column list (`EmailServiceGo/internal/dentally/daylist/appointment/store.go:571-679`) — no ORM struct, no `SELECT *`. Go only ever touches columns it names literally in that SQL string.

The existing `confirmation_token` and `quick_note` fields are the established precedent for this exact situation: Django-only nullable columns Go's INSERT never references at all, so Django's writes to them survive every Go re-sync untouched. All new fields in this phase follow that same precedent — nullable, never mentioned in `store.go`.

This means:
- Zero changes needed to `EmailServiceGo/internal/dentally/daylist/appointment/store.go` or any Go struct.
- `EmailServiceGo/internal/db/schemaguard.go` needs no change — it only flags NOT-NULL/no-default columns, and these are nullable.
- `EmailServiceGo/internal/db/snapshotguard.go`'s `mirrorExcludedColumns["dentally_appointment"]` map should additively list the new field names (same treatment as the existing `quick_note` entry) so the snapshot-mirror guard test stays green. This is a one-line-per-field additive edit to a Go map literal, not a logic or struct change.

## New fields on `DentallyAppointment`

Django-only, added in one migration. `patient_confirmed_at`/`cancellation_requested_at` are nullable (`null=True` alone is sufficient — Go's INSERT omitting them just leaves `NULL`). The other 4 are NOT-NULL at the DB level, so per this codebase's own established (and previously-enforced-the-hard-way, see migration `0138_boolean_flag_db_defaults.py`) convention for Django-only columns on this Go-shared table, they all need `db_default` alongside `default` — Django's `default=` is Python-only and does nothing for rows Go's raw-SQL INSERT creates without naming these columns:

```python
patient_confirmed_at = models.DateTimeField(null=True, blank=True)
confirmation_source = models.CharField(max_length=30, blank=True, default="", db_default="")
confirmation_channel = models.CharField(max_length=10, blank=True, default="", db_default="")
cancellation_requested = models.BooleanField(default=False, db_default=False)
cancellation_requested_at = models.DateTimeField(null=True, blank=True)
cancellation_requested_source = models.CharField(max_length=30, blank=True, default="", db_default="")
```

- `confirmation_source`: free-form short string identifying the patient-facing mechanism — `"confirm_link"` today (the only one that exists: the public confirm.dental page). Not an enum/choices field, so a future distinct mechanism (e.g. a keyword-reply rule) can be added without a migration. Does not distinguish who triggered the underlying message send — that's `confirmation_channel`'s job.
- `confirmation_channel`: `"sms"` or `"email"` — which channel the confirmation *link* was sent through, stamped at send time (see "Wiring" below), not derived at confirm time.
- `cancellation_requested*`: mirrors the confirmation fields for the "patient wants to cancel" path from the public confirm.dental page (per the PRD's decision plan: v1 records `cancellation requested`, does not instantly cancel in Dentally).

## Display-status derivation

A pure function, not a stored field — computed from data that's already source-of-truth elsewhere, so it can never drift out of sync with the underlying facts:

```python
def compute_confirmation_display_status(appointment: DentallyAppointment) -> str:
    """One of: Cancelled, Cancellation requested, Confirmed, No valid contact,
    Awaiting confirmation. Checked in this priority order so Dentally's own
    cancellation status always wins, per the PRD edge case: "if Dentally says
    cancelled, Daylist should show cancelled even if Pathway previously had
    confirmed."
    """
    if appointment.status == "cancelled":  # existing Dentally-synced field
        return "Cancelled"
    if appointment.cancellation_requested:
        return "Cancellation requested"
    if appointment.patient_confirmed:
        return "Confirmed"
    if not _has_valid_contact(appointment):
        return "No valid contact"
    return "Awaiting confirmation"
```

`_has_valid_contact` checks whether the appointment/patient has a usable phone number or email — reusing whatever existing helper the manual-SMS-send path (`AppointmentCard.tsx` / its backend counterpart) already uses to decide sendability, rather than duplicating that logic. The implementer should locate and reuse that check rather than re-deriving it; if none exists at the right layer, a minimal one is added here.

This function is intentionally backend-only in this phase (a Django method/module function) — no API contract or serializer field is added yet, since no consumer (Daylist UI) reads it until Phase 3. Phase 3 will expose it via the existing day-list appointment serializer.

## FR14 fix — template type mismatch

`EmailMessageTemplate.TEMPLATE_TYPES` and `SMSMessageTemplate.TEMPLATE_TYPES` (`messaging/models.py`) get a new choice: `("appointment_confirmation", "Appointment Confirmation")`. The frontend (`perfect-pixel-playground-project/src/components/diary/confirmationActions.ts:102-106`) already requests `template_type === 'appointment_confirmation'` and silently falls back to name-matching today because the choice doesn't exist — this closes a live bug, independent of the rest of the PRD, with no other code changes required (the frontend already speaks this contract).

## Out of scope for this phase

- The `Suppressed` display state (needs Phase 2's sequence engine).
- Any API/serializer exposure of the new fields or the derivation function (needs Phase 3's Daylist UI work to consume it).
- Any UI changes.
- The sequence/enrollment engine itself, cohort engine, or follow-up tasks (Phases 2, 4, 5).
- Populating these fields from anywhere other than the existing manual-confirm and public-confirm-page code paths — this phase adds the columns and derivation function; wiring the population is in scope for this phase too (see below and Testing), since an unpopulated field is not verifiably correct.

## Wiring the new fields into the existing confirm endpoint

`AppointmentConfirmViewSet.create()` (`dentallyIntegration/views/appointment_confirm_views.py:125-136`) today only supports one action — POST `{"token": ...}` unconditionally sets `patient_confirmed=True`. There is no cancellation-request action at all yet. This phase extends it:

```python
def create(self, request):
    """POST /confirm/ — {"token": ..., "action": "confirm" | "request_cancellation"}.
    `action` defaults to "confirm" for backward compatibility with existing
    outstanding links / the current frontend, which POSTs {"token": ...} alone."""
    token = request.data.get("token")
    action = request.data.get("action", "confirm")
    appointment, err = self._resolve_appointment(token)
    if err:
        return err

    if action == "request_cancellation":
        if not appointment.cancellation_requested:
            appointment.cancellation_requested = True
            appointment.cancellation_requested_at = timezone.now()
            appointment.cancellation_requested_source = "confirm_link"
            appointment.save(update_fields=[
                "cancellation_requested", "cancellation_requested_at", "cancellation_requested_source",
            ])
        return Response({"cancellation_requested": True})

    if not appointment.patient_confirmed:
        appointment.patient_confirmed = True
        appointment.patient_confirmed_at = timezone.now()
        appointment.confirmation_source = "confirm_link"
        appointment.save(update_fields=[
            "patient_confirmed", "patient_confirmed_at", "confirmation_source",
        ])
    return Response({"confirmed": True})
```

`confirmation_source` stays the literal `"confirm_link"` for both confirm and cancellation-request actions — there is only one patient-facing confirmation mechanism today (the public confirm.dental page), so this field's only live value right now is forward-looking infrastructure for a future distinct source (e.g. a keyword-reply rule, if FR18 is ever built). It is not meant to distinguish "who triggered the send," which is what `confirmation_channel` is for.

**`confirmation_channel` is set at SEND time, not confirm time**, by whichever code generates the confirmation link — this is the only way to actually know which channel the link went out on, since the public confirm page itself has no reliable knowledge of that. Traced today: `AppointmentConfirmLinkViewSet.retrieve()` (`appointment_confirm_views.py:13-52`) has exactly one caller in the whole frontend — `AppointmentCard.tsx`'s `handleSendConfirmationSms` (`AppointmentCard.tsx:411-439`), which always sends SMS. There is no manual email-confirmation flow anywhere in the frontend today.

This phase extends `AppointmentConfirmLinkViewSet.retrieve()` to accept an optional `?channel=` query param (defaulting to `"sms"`, matching the one real caller today) and stamp it onto the appointment:

```python
def retrieve(self, request, pk=None):
    ...
    channel = request.query_params.get("channel", "sms")
    appointment.confirmation_channel = channel
    if not appointment.confirmation_token:
        appointment.confirmation_token = generate_short_token()
    appointment.save(update_fields=["confirmation_token", "confirmation_channel"])
    ...
```

`AppointmentConfirmViewSet.create()` (the confirm/cancellation-request endpoint above) does **not** overwrite `confirmation_channel` — it was already stamped at send time and simply persists. No frontend change is required for this phase (the existing call with no query param already defaults correctly to `"sms"`); a future email-confirmation flow or Phase 2's sequence engine would pass `?channel=email` when it exists.

## Testing

- Migration applies cleanly; `patient_confirmed_at`/`cancellation_requested_at` default to `NULL`, `cancellation_requested` to `False`, and the two `source`/`channel` char fields to `""` on existing rows.
- `compute_confirmation_display_status` unit-tested for all 5 branches, including the priority-order case (Dentally-cancelled wins even when `patient_confirmed=True`), and the case where `cancellation_requested=True` and `patient_confirmed=True` both (cancellation-requested wins, since a patient can't meaningfully be both — but the derivation must still pick deterministically).
- `AppointmentConfirmLinkViewSet.retrieve()` stamps `confirmation_channel` from the `?channel=` query param (default `"sms"`) on every call, and still generates/returns `confirmation_token` exactly as before when one doesn't exist yet.
- `AppointmentConfirmViewSet.create()`: default action (no `action` field, or `action="confirm"`) sets `patient_confirmed`/`patient_confirmed_at`/`confirmation_source="confirm_link"`, is idempotent (calling twice doesn't change `patient_confirmed_at` on the second call), and does not touch `confirmation_channel`. `action="request_cancellation"` sets the three `cancellation_requested*` fields, is likewise idempotent, and also doesn't touch `confirmation_channel`.
- Existing `list()` (`GET /confirm/?token=...`) response shape is unchanged — no new fields need to be exposed there in this phase (the public confirm page doesn't need to know the derived display status; that's Daylist/staff-facing).
- `snapshotguard.go`'s `mirrorExcludedColumns["dentally_appointment"]` includes the 6 new field names; the Go test suite for `internal/db` still passes.

# Messaging: next-appointment indicator in the conversation header

**Date:** 2026-08-24
**Status:** Implemented 2026-08-24 (uncommitted). Tests green; see "Testing" below.

## Problem

When a staff member opens a conversation in the v2 inbox, nothing on screen says
whether the person they are replying to is booked in. The information exists —
the Go sync writes `dentally_appointment` rows that the Day List reads — but the
inbox has no view of it, so answering "when am I coming in?" means leaving the
conversation for the Day List or Dentally itself.

## Goal

Show a badge in `ConversationHeader` naming the contact's next upcoming
appointment. It must cost **no additional HTTP request**, and must be computed
only when a conversation is actually opened — never per row of the contact list,
and never on background refetches.

## Non-goals

- Live updates. The badge is a snapshot taken when the session opens. A patient
  booking mid-conversation will not be reflected until the conversation is
  reopened. No websocket work in this scope.
- Past appointments / last-visit context.
- Any write path. This is read-only; nothing books, moves, or cancels.
- Fetching from the live Dentally API. Local table only.

## What already exists (no new infrastructure)

| Need | Existing thing |
|---|---|
| Person → Dentally patient id | `Patient.meta_data["id"]`, practice-scoped. Precedent: `dentallyIntegration/views/daylist_reporting_views.py:42` |
| Appointment rows | `DentallyAppointment`, `db_table = "dentally_appointment"` |
| An index that makes the lookup cheap | `models.Index(fields=["dentally_patient_id", "state", "start_time"])` (migration `0137_dappt_patient_state_start_index`) |
| A once-per-open hook | `GET /messaging/contacts/<pk>/thread/`, called from `handleSelectContact` in `InboxPage.tsx:321` |
| A place to put the payload | the `response.data["person"]` block already assembled at `messaging/views/contact_views.py:648` |

No migration. No new model. No new endpoint.

## Coverage: what the local table actually holds

Coverage is **per-patient, not per-window** — this is the single most important
thing to understand about this feature's accuracy, and it is easy to get
backwards.

The Go scheduler drives sync by *date*: a rolling 7-day window from today
(`slidingWindowDates` → `weekDates`,
`EmailServiceGo/internal/dentally/scheduler/scheduler.go:713`), plus a
today-only auto-sync refresh tick. But the date only decides **which patients
get processed**. For each selected patient, `processPatient` calls
`fetchPatientAppointments` — documented as *"all appointments for a specific
patient (past, present, and future)"*, with no date filter — and the loop headed
*"Store all appointments for this patient"*
(`internal/dentally/daylist/appointment/sync.go:735`) upserts **every one of
them**.

So once a patient has appeared on any synced day, the table holds their
**entire** forward calendar, however far out. A patient booked in three months
is present, provided they were swept in by some date.

### Measured, not assumed

Both halves of the above were verified against `dentally_appointment` on the
local dev database (2026-08-24), because an earlier draft of this spec asserted
a flat 7-day limit and was wrong:

| Measure | Value |
|---|---|
| Future-dated rows | 8,951 |
| …of those, **beyond 7 days** | **8,256 (92%)** |
| …beyond 30 days | 6,672 |
| …beyond 90 days | 4,457 |
| Furthest appointment | **2028-07-03** (~2 years out) |
| Distinct patients with a future appointment | 5,908 |

And the per-patient mechanism, tested directly: of the **4,911** patients
holding an appointment more than 30 days out, **4,911 — every one — also have
past appointment rows.** There is not a single far-future patient without a
past row. Nobody gets into this table except by being swept, and once swept
their whole calendar lands.

(Dev data. The shape and mechanism are what matter here and are driven by
`sync.go`, which is environment-independent; the absolute counts will differ in
production.)

### What this means for the badge

- **A patient seen recently, or booked inside the next 7 days:** their full
  future calendar is local and current. The badge is accurate, and not limited
  to 7 days.
- **A patient who has not been swept recently:** their rows date from whenever
  they were last processed. A booking made *since* then is missing, and a
  booking that has since moved or been cancelled may still be present.

  Verified, not assumed: `MarkRemovedAppointments`
  (`internal/dentally/daylist/appointment/store.go:803`) is scoped
  `WHERE DATE(start_time AT TIME ZONE 'UTC') = ?` — the **synced date only**. A
  future appointment outside a synced date is never marked removed by that
  pass. It is corrected only when that patient is next processed and their
  calendar is re-fetched (with `Cancelled: true`, so a cancellation does come
  back and overwrite the state).

  **This is the feature's real limitation, and it is the dominant case.** Of the
  8,951 future-dated rows on dev, only **647 (7%)** were written in the last 7
  days; **4,518 (50%) have not been touched in over 30 days**. Most future rows
  the badge will show are therefore snapshots of some past sweep, not a live
  read. (Dev sync cadence is irregular; production will be fresher, but the
  mechanism guarantees the effect exists there too.)

Two consequences the implementation must respect:

1. **Absence never means "no appointment."** Render nothing — no "No upcoming
   appointments" empty state anywhere in this feature, because we cannot
   truthfully make that claim.
2. **Presence is a best-known snapshot, not a guarantee.** The badge states what
   the practice's synced record says. It must not be worded as a promise to the
   patient (no "You're booked in on…" phrasing in any staff-facing copy that
   could be pasted into a reply), and staff should confirm in Dentally before
   telling a patient something is definitely happening.

Excluding cancelled states (below) removes the largest category of stale-row
error, but cannot remove it entirely.

## Approach

**Ride the existing thread response** (chosen over a dedicated endpoint, and over
embedding in the contact list).

- A dedicated `GET /dentally/appointments/next/` would be more separable and
  reusable by the patient panel, but adds one request per conversation open and
  makes the badge pop in after the header renders.
- Embedding in `ContactListSerializer` was rejected outright: an appointment
  lookup per row on every list page and scroll is the exact request storm this
  design exists to avoid.

The thread endpoint fires exactly once when a session opens, which is precisely
the trigger condition required.

## Backend

### Family is not a special case

`get_family_members` (`messaging/serializers.py:613`) iterates
`person.patients.all()`. A "family" conversation in this inbox is **one `Person`
carrying several `Patient` rows** — not several `Person`s in a `Household`.

So solo and family reduce to the same query: *the Patient rows attached to this
Person*. Solo yields one, family yields several. The backend does not branch;
only the frontend presentation does.

### New helper

New module `dentallyIntegration/next_appointment.py`:

```python
def next_appointments_for_person(person, practice) -> list[dict]:
    """Soonest future appointment per Patient row on `person`, ascending.

    Practice-scoped on both sides. Returns [] when the person has no linked
    Patient, no Dentally id, or nothing booked in the local record.
    """
```

Behaviour:

1. `Patient.objects.filter(person=person, practice=practice)` — one query.
   Read `id`, `first_name`, `last_name`, `meta_data`.
2. Collect `meta_data["id"]` per patient, skipping rows without one.
   Coerce to `int`; skip anything non-coercible rather than raising.
3. One query:
   ```
   DentallyAppointment.objects
     .filter(practice=practice,
             dentally_patient_id__in=ids,
             start_time__gte=timezone.now())
     .annotate(state_lower=Lower("state"))
     .exclude(state_lower__in=EXCLUDED_STATES)
     .order_by("start_time")
   ```
   `EXCLUDED_STATES` **reuses the existing canonical sets** rather than defining
   a new one:

   ```python
   from dentallyIntegration.confirmation_status import (
       CANCELLED_STATES,   # {"cancelled", "patient_cancelled", "removed_from_schedule"}
       COMPLETED_STATES,   # {"completed", "did_not_attend", "did not attend"}
   )
   EXCLUDED_STATES = {s.lower() for s in CANCELLED_STATES | COMPLETED_STATES}
   ```

   **Compared case-INSENSITIVELY, and that is not optional.** Measured on the
   live table, future rows carry `Pending` (8,381) beside `pending` (106), and
   `Cancelled` (419) beside the lowercase spelling the canonical sets use. A
   plain `state__in` exclusion — which an earlier draft of this spec specified —
   matches none of the capitalised variants, so all 419 cancelled future
   appointments would have rendered as live bookings. Hence `Lower("state")`.

   This matters. `DentallyAppointment.state` is a free-form `CharField` written
   raw by the Go sync — no choices, no validation — and Dentally reports a
   cancellation under several different strings. Matching only the literal
   `"cancelled"` has already caused a live bug once (a `patient_cancelled`
   appointment resolved to "Awaiting confirmation" and the engine kept chasing
   the patient with a confirmation link — see the comment at
   `dentallyIntegration/confirmation_status.py:24`). Reusing the shared sets
   means this feature inherits any future correction to them instead of
   silently drifting.
4. Reduce to the soonest row per `dentally_patient_id` in Python, and return
   ascending by `start_time`.

**Cost: exactly 2 queries, both index-backed, regardless of household size.**
No N+1.

Import direction: `messaging` and `dentallyIntegration` already import each
other via **function-local imports** to dodge circularity
(`messaging/views/template_views.py:548`, `dentallyIntegration/recall_automation.py:114`).
This helper follows that convention — imported inside the `thread` method body,
never at module scope.

### Wiring into `thread`

In `messaging/views/contact_views.py`, after `response.data["person"]` is built:

```python
if request.query_params.get("page") in (None, "1"):
    appts = next_appointments_for_person(channel_person, practice)
    response.data["person"]["next_appointment"] = appts[0] if appts else None
    response.data["person"]["appointments"] = appts
```

The page guard matters: `loadMoreMessages` re-hits this endpoint for older
pages, and the frontend only reads `person` on page 1
(`useContactThread.ts:44`). Recomputing on page 2+ would be pure waste.

`channel_person` is already resolved by `_resolve_identity` at the top of the
action. When it is `None`, both keys are `null` / `[]`.

### Payload

```json
"person": {
  "normalized_email": "...",
  "normalized_phone": "...",
  "country_code": "...",
  "sessions": { "email": 1, "sms": 2, "whatsapp": null },

  "next_appointment": {
    "patient_id": 91,
    "patient_name": "Amy Beacher",
    "start_time": "2026-08-26T14:30:00Z"
  },
  "appointments": [
    { "patient_id": 91, "patient_name": "Amy Beacher", "start_time": "2026-08-26T14:30:00Z" },
    { "patient_id": 92, "patient_name": "Tom Beacher", "start_time": "2026-08-29T09:00:00Z" }
  ]
}
```

`next_appointment` is `appointments[0]`, denormalised so the solo case needs no
array handling. It is `null` and `appointments` is `[]` when there is nothing.

**Dates and names only.** No `reason`, no `treatment_description`, no `notes`,
no practitioner, no risk tier. See PHI below.

### PHI scope

The thread endpoint carries a deliberate cross-member guard (THR-03): when a
family thread is opened with `?patient_id=`, message rows are filtered to that
patient's channels.

The appointment payload intentionally does **not** apply that filter — it lists
every Patient row on the Person. This was an explicit product decision, on the
basis that the header already discloses household membership by name in its
existing family popover, so listing those same names against a date adds no new
identity disclosure.

The mitigation is that the payload is restricted to **name + date/time**.
Clinical fields are excluded so that a conversation opened by a household member
cannot leak what another member is being treated for. This restriction is the
reason the decision is safe and must not be relaxed without revisiting it.

## Frontend

### `useContactThread`

`ApiContactThread['person']` (in `src/types/contactMessaging.ts`) gains:

```ts
next_appointment: PersonAppointment | null;
appointments: PersonAppointment[];
```

```ts
interface PersonAppointment {
  patient_id: number;
  patient_name: string;
  start_time: string;   // ISO 8601
}
```

`threadPersonMeta` captures the `person` block on page 1, but on its own it
outlives the conversation it came from: selecting a new contact leaves the
previous person's metadata in state until the new page-1 response lands, which
would paint the PREVIOUS patient's appointment onto the new conversation for a
beat. The hook therefore also exposes `threadPersonId`, cleared when a page-1
fetch starts and set alongside the metadata, and `InboxPage` only feeds the
badge when it matches the selected contact. `InboxPage` passes the appointment data down to `ConversationHeader` as
a new optional `appointments?: PersonAppointment[]` prop.

### `ConversationHeader`

A new local `AppointmentBadge` component, rendered in the desktop cluster beside
the channel capsule, and hidden on mobile alongside it (the mobile header has no
room and already drops the channel capsule for avatar overlay badges).

Rendered **only** when `appointments.length > 0`. There is no empty state, no
skeleton, and no placeholder — an absent badge is the correct and only
representation of "we have nothing to show".

**Solo** (`appointments.length === 1`, or non-family thread): a static badge.

```
[ Appointment 26 Aug ]
```

Date formatted `d MMM` in `en-GB`, matching the header's existing locale usage.
A tooltip carries the full date and time, using the same white
`TooltipContent` override the action rail uses (the house `#a695eb` tooltip is
unreadable against this header). That tooltip needs its OWN `TooltipProvider`:
`Tooltip` here is a bare Radix Root and the header's only existing provider
wraps the action rail at the far end.

Badge text is `#6941c6`, not the `#846ce0` used elsewhere in this header.
`#846ce0` on `#FAF9FE` measures 3.85:1, under the 4.5:1 WCAG AA floor for text
this size; `#6941c6` is 6.32:1. Both are already in the header's palette.

**Family** (`isFamily` and `appointments.length > 1`): the same badge showing the
**soonest** date, rendered as a `PopoverTrigger` button with a hover underline —
the same interaction the member subline already uses directly below it, so the
two clickable affordances in the header read as one idiom.

The popover is the customary white `PopoverContent`, listing every member with a
booking, ascending by date:

```
┌──────────────────────────────────┐
│ Amy Beacher    Tue 26 Aug, 14:30 │
│ Tom Beacher    Fri 29 Aug, 09:00 │
└──────────────────────────────────┘
```

Household members with no known appointment are simply absent from the list —
again, no "none booked" row, which the coverage caveat above cannot justify.

Visual treatment follows the existing header vocabulary: the same
`#d7ccfb` / `#FAF9FE` outlined-capsule geometry as the action rail, `h-7` to
match both the action rail and the channel capsule, so the header keeps a single
horizontal rhythm rather than gaining a third icon-group height.

## Testing

**Backend** — new tests against `next_appointments_for_person`:

- Person with one Patient and one future appointment → one entry.
- Person with several Patient rows → one entry each, ascending by `start_time`,
  and the *soonest* per patient when a patient has several.
- Past appointments excluded.
- Every string in `CANCELLED_STATES | COMPLETED_STATES` excluded — asserted
  per-literal, `patient_cancelled` included, since that specific variant is the
  one that caused the prior live bug.
- Patient with no `meta_data["id"]`, or a non-integer one → skipped, no raise.
- Cross-practice appointment on a matching `dentally_patient_id` → **excluded**.
  This asserts the practice-scoping rule; the same human exists as separate
  Patient rows per practice.
- Query count pinned with `assertNumQueries(2)` for a multi-patient household,
  so a future refactor cannot reintroduce an N+1.

Plus a view-level test that `GET .../thread/?page=2` does **not** include the
appointment keys.

**Frontend** — extend `ConversationHeader.test.tsx`, which already
characterises this component:

- No `appointments` prop → no badge in the DOM at all.
- One appointment → static badge with the formatted date, not a button.
- Family with several → button that opens a popover listing each member.

Run frontend typecheck as `npx tsc --noEmit -p tsconfig.app.json` — the repo-root
invocation checks nothing and exits 0. Judge by delta against the ~491
pre-existing errors.

Backend tests run with `--keepdb`, never `--noinput`.

## Files touched

**Backend**
- `dentallyIntegration/next_appointment.py` (new)
- `messaging/views/contact_views.py` (`thread` action, ~6 lines)
- test module for the helper (new)

**Frontend**
- `src/types/contactMessaging.ts` (types)
- `src/components/inbox/v2/ConversationHeader.tsx` (badge + popover)
- `src/InboxPage.tsx` (pass the prop through)
- `src/components/inbox/v2/ConversationHeader.test.tsx` (cases)

No migration. No new route. No Go change.

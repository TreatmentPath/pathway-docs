# General consolidation — verified findings

The non-identity half of the sweep: ordinary "defined many times, should be defined once".
Most of this is not a bug. Ranked by number of sites and how mechanical the fix is.

Companion: `../CLONE-CENSUS.md` — an AST census of the whole backend, 180 groups of
byte-identical function bodies (477 definitions), 29 spanning 2+ apps.

---

## G1 — the practice gate is copy-pasted 65 times  ✅ VERIFIED (TreatmentPlan)

```python
practice = self.get_user_practice_or_none()
if not practice:
    return Response({"error": "..."}, status=...)
```

**Counted mechanically, not estimated:**

| measure | count |
|---|---|
| calls to `get_user_practice_or_none()` in `TreatmentPlan/` | **101** |
| …immediately followed by a `None` check — the actual copy-pasted gate | **65**, across **17 files** |

The scan reported "101 sites"; 101 is the call count and 65 is the duplicated gate. Both
worth knowing — the other 36 calls use the practice without gating, which is its own
question.

**Belongs in:** a mixin method on `PracticeAccessMixin` (`utils/practice_mixins.py`)
returning the practice or raising/returning the 403 itself. The mixin is already inherited
by these viewsets, so this is the cheapest large win in the codebase.

**Note:** the ERROR SHAPE varies between copies (`{"error": ...}` vs `{"detail": ...}`,
403 vs 400). Consolidating would also make the API's failure response consistent, which it
currently is not.

## G2 — `permission_classes = [IsAuthenticated, FeatureAccessPermission]` × 48  ✅ VERIFIED

Identical in 48 places in `TreatmentPlan/`. **Belongs in:** a base viewset class. Purely
mechanical.

## G3 — `get_queryset()` boilerplate × ~49  ⚠️ divergent, needs care

Same shape (resolve practice → filter → `select_related`) but the relations differ per
viewset, so this is NOT a copy-paste collapse. Wants a base class with a per-viewset hook,
not a single shared method. Lower priority than G1/G2 despite the similar count.

## G4 — date-param parsing × 10, query-param coercion × 25+

`.get("key", "").strip().lower()` and try/except date parsing repeated. The backend-wide
census shows `_parse_date` alone has **19 identical copies** (Assets + Statistics).
**Belongs in:** global `utils/` — `parse_date_param()`, `get_bool_param()`,
`get_stripped_param()`.

## G5 — `get_created_by_name` × 9 in TreatmentPlan, 13 more in `compliance`  ⚠️ DIVERGENT

Mixes `get_display_name_for_user()` with hand-built strings and inconsistent fallbacks.
Divergent copies are worth more than identical ones: the fallbacks differ, so the same
user renders differently depending which serializer answered.

## G6 — client-IP extraction: 11 definitions in 9 modules  ✅ VERIFIED (backend-wide)

Already recorded as **C12** in `../FIX-LIST.md` because one copy has a genuine behavioural
divergence (`REMOTE_ADDR` only, in the marketing-consent audit path). Repeated here because
it is also the clearest pure-consolidation case: **a shared implementation already exists**
at `utils/secure_logging.py:135`, and nine other modules still define their own.

---

## G7 — 90 hand-written `except → Response` blocks, and the error KEY is inconsistent  ✅ VERIFIED (dentallyIntegration)

The scan reported "42+ sites". Counted mechanically, it is **90 except-blocks returning a
`Response`, across 7 files** — and the response body key is not the same in all of them:

| key returned | count |
|---|---|
| `"error"` | 83 |
| `"detail"` | 3 |
| `"success"` | 2 |

So a client cannot rely on where the message lives: 5 of 90 endpoints answer differently
from the other 85. DRF already has a standard for this (`detail`), and the majority here
have quietly diverged from it.

**Belongs in:** a shared exception handler (`settings.REST_FRAMEWORK["EXCEPTION_HANDLER"]`)
or one helper. 90 hand-written blocks is also 90 chances to swallow an exception that
should have propagated.

## G8 — Redis client constructed 4 times, `_get_service` defined 6 times  ✅ VERIFIED

- 4 independent Redis client constructions in `dentallyIntegration/` — each its own
  connection settings, so a timeout or pool change has to be made four times and any miss
  is silent.
- `_get_service` defined 6 times, all byte-identical (confirmed by the AST census too).

**Belongs in:** one module-level accessor each.

---

## Theme emerging across both general scans: the duplication IS the API inconsistency

G1 (65 practice gates, mixed `{"error"}`/`{"detail"}`, mixed 403/400) and G7 (90 handlers,
83/3/2 key split) are the same story from two directories. Nobody set out to make the API
inconsistent; it drifted because the shape was retyped ~155 times instead of imported once.

That reframes the value of this half of the sweep. Deleting duplicate lines is tidiness.
The reason to do it is that **every copy is an independent opportunity to drift**, and in
these two cases the drift already happened and nobody noticed — exactly as it did with the
client-IP helper (C12), where one of eleven copies quietly stopped reading
`X-Forwarded-For` in the consent audit path.

---

## G9 — websocket group names are hand-built at BOTH ends of every channel  ✅ VERIFIED (messaging)

The group name is the join between a consumer (which subscribes) and a broadcaster (which
publishes). Every one of them is an f-string typed out separately in both places:

```
f"conversations_practice_{practice_id}"    x2   + f"conversations_practice_{self.practice_id}"  x1
f"unread_count_practice_{practice_id}"     x1   + f"unread_count_practice_{self.practice_id}"   x1
f"conversation_messages_{session_id}"      x1   + f"conversation_messages_{self.session_id}"    x1
f"temp_mailbox_{self.mailbox_id}"          f"single_{message.id}"
```

There is no shared constructor. If either end's string is edited, the two silently stop
matching: no error, no failed request — the socket simply never receives anything, and it
looks like "realtime is a bit flaky".

**This has already happened here.** The project record notes a `contact_id`/`person_id`
key mismatch that broke *all* v2 Inbox live updates. Same failure mode, one layer up.

**Belongs in:** `messaging/utils.py` — `group_conversations(practice_id)`,
`group_unread(practice_id)`, `group_messages(session_id)`. Small change, removes an entire
class of silent failure.

## G10 — two serializers carry 10 identical getters each  ✅ VERIFIED (messaging)

`MessageSessionSerializer` and `MessageSessionDetailSerializer` both define byte-identical
copies of: `get_contact_name`, `get_participant_email`, `get_participant_phone_number`,
`get_participant_country_code`, `get_patient_id`, `get_intake_id`, `get_nurture_id`,
`get_activity_log_id`, `get_is_family`, `get_family_relationship`.

Measured by AST over `messaging/serializers.py`: **12 getter bodies defined more than once,
26 definitions in that one file**; the largest cluster is the 10 shared by those two
classes.

This is where **#92 came from** — `get_nurture_id` was fixed in one sibling and not the
other. The duplication is not just untidy here; it is the mechanism that produced a
verified identity finding earlier in this sweep. A `_SessionContactMixin` would have made
that miss impossible.

**Belongs in:** a mixin shared by the two serializers.

---

## G11 — 18 URL builders across 10 apps; a shared mixin exists and 2 modules use it  ✅ VERIFIED (backend-wide)

Triggered by `marketingBroadcast` having two byte-identical `get_url` methods. Grepping
the backend, that is a small instance of a much larger pattern:

| | count |
|---|---|
| `get_url` / `get_image_url` / `get_file_url` / `get_avatar_url` methods | **18**, across 10 apps |
| `build_absolute_uri` call sites | **52** |
| modules importing the existing shared mixin | **2** |

`TreatmentPlan/serializers/mixins.py:33` already implements the canonical version, with the
Spaces-vs-local logic spelled out:

```python
def get_image_url(self, obj):
    """- If USE_SPACES=True: Returns DigitalOcean Spaces URL directly
       - If USE_SPACES=False: Returns local URL with request domain"""
```

**This is the C12 shape again, for the third time:** a correct shared implementation
exists, and the rest of the codebase wrote its own anyway. That matters more than the line
count, because the Spaces-vs-local branch is exactly the kind of environment-dependent
logic where one stale copy produces a broken image in production and nowhere else.

Whether the 18 have already drifted is NOT yet measured — worth doing before consolidating,
since a divergence here would be a live bug rather than tidy-up.

**Belongs in:** global `utils/` (or a serializer mixin in a shared module), not
`TreatmentPlan/`, since 10 apps need it.

## `marketingBroadcast/` is the cleanest app scanned so far

Worth recording as a positive: **9** `except → Response` blocks, against 90 in
`dentallyIntegration` and 26 in `messaging`. Its practice gating already goes through
`PracticeAccessMixin` rather than being retyped. It is the newest of the three, which
suggests the duplication is historical drift rather than current practice — the older the
app, the more copies.

Its own items are small: 2 identical `get_url` methods (part of G11), 4 validation blocks
sharing one shape, 2 pagination parsers, 3 near-identical `perform_create`s.

---

## G12 — an UNGUARDED name join on a public booking page renders "None Smith"  ✅ VERIFIED (onlineBooking)

Three sites build a practitioner name; two guard, one does not:

```python
onlineBooking/serializers.py:292   f"{row.practitioner.first_name or ''} {row.practitioner.last_name or ''}".strip()   # guarded
onlineBooking/serializers.py:422   first = obj.practitioner.first_name or ""                                            # guarded
onlineBooking/views.py:684         f"{h.practitioner.first_name} {h.practitioner.last_name}".strip()                    # NOT guarded
```

`User.first_name` and `User.last_name` are both `null=True`, and **2 users currently have a
NULL first name**. So `views.py:684` renders `"None Smith"` — on the PUBLIC booking page,
where a patient reads it as their practitioner's name.

**Correction to the scan's wording:** it called this a "crash risk". It is not — an
f-string of `None` produces the string `"None"`, it does not raise. The defect is real,
the mechanism is not the one stated.

**Fourth instance of one shape.** C2/C7 recorded `Patient.__str__` producing `'John None'`
and `'None Smith'`; this is the same unguarded f-string on a different nullable column,
found by a different scan in a different app. `full_name()` exists and handles it. The
consolidation and the bug fix are the same edit.

---

## Scan-quality trend

| directory | `except → Response` | error-key drift |
|---|---|---|
| `dentallyIntegration` | 90 | 83 `error` / 3 `detail` / 2 `success` |
| `messaging` | 26 | three-way split |
| `onlineBooking` | 19 | **19/19 `detail` — no drift** |
| `marketingBroadcast` | 9 | minor |

`onlineBooking` is the first app with a perfectly consistent error contract, and
`marketingBroadcast` has the fewest handlers. Both are among the newer apps. The pattern
across the whole sweep is that duplication and drift track AGE, not size — which argues for
consolidating the old apps and leaving the new ones alone.

---

## `Appointments/` — ~30 duplicated definitions, and the LOWEST-VALUE target in the sweep

The scan is accurate: 47 `except → Response` blocks (30 in `views.py`, 17 in
`public_booking_views.py`), 6 practice gates, `get_practitioner_name` ×4 and
`get_clinician_name` ×3 as identical serializer getters. No unguarded name joins — this
app guards its name joins properly, unlike three others.

Two corrections worth recording:

- The reported "appointment active status list ×3" did not reproduce as bare literals.
  This app uses the `Appointment.Status.CONFIRMED` / `.PENDING` enum, which is the CORRECT
  pattern and the thing other apps should copy. Only one bare `status__in=["cancelled"]`
  exists (`serializers.py:149`).
- **The app holds 4 rows.** (Production is reported at 5.) `Appointment` here is the new
  booking model, distinct from the Dentally appointment tables that carry the real volume.

**So this directory should be near-last for consolidation regardless of its count.**

## Prioritisation rule this produced

Rank by **duplication × usage**, not duplication alone. On count, `Appointments` (47
handlers) looks comparable to `dentallyIntegration` (90). On value they are nowhere near:
one serves the entire recall and day-list operation, the other holds four rows.

Current ranking for the general work:

| priority | directory | why |
|---|---|---|
| 1 | `dentallyIntegration/` | 90 handlers WITH key drift; heavy production use |
| 2 | `TreatmentPlan/` | 65 practice gates + 48 permission repeats; the core app |
| 3 | `compliance/` | not yet scanned; census shows `get_created_by_name` ×13 and 2 client-IP copies |
| 4 | `messaging/` | 26 handlers, plus G9/G10 which have already caused real bugs |
| 5 | `marketingBroadcast/`, `onlineBooking/` | already clean — leave alone |
| last | `Appointments/` | 4 rows |

The cross-cutting items (G6 client IP ×11, G11 URL builders ×18, G4 `_parse_date` ×19) sit
above all of these, because they are single small edits that pay out in every app at once.

---

## G13 — a deliberate PRIVACY rule exists in one helper and is bypassed by 83 copies  ✅ VERIFIED BY EXECUTION

The single best argument in this whole sweep for "define once, use everywhere".

`TreatmentPath/utils.py:1` is the canonical display-name helper, and it carries a rule
that is clearly intentional:

```python
def get_display_name_for_user(user):
    """Superusers are shown as 'System Admin' to preserve anonymity in guest sessions."""
    if getattr(user, "user_type", None) == "superuser":
        return "System Admin"
```

There are **three** implementations of "user display name":

| # | where | superuser rule? | `None` returns |
|---|---|---|---|
| 1 | `TreatmentPath/utils.py:1` `get_display_name_for_user` | **yes** | `None` |
| 2 | `compliance/serializers.py:61` `get_user_display_name` — thin wrapper over #1 | inherits it | `None` |
| 3 | `teamChat/services.py:409` `get_user_display_name` — its OWN implementation | **NO** | `""` |

…plus **82 hand-built f-strings** across six apps that reimplement it inline.

**Executed against a real superuser (5 exist in the database):**

```
canonical (TreatmentPath/utils.py) -> 'System Admin'      <- privacy rule applied
teamChat  (its own copy)           -> 'mqnifest kelvin'   <- real name
hand-built f-string (82 sites)     -> 'mqnifest kelvin'   <- real name
null contract:  canonical(None) = None      teamChat(None) = ''
```

**Two separate defects, both caused purely by duplication:**

1. **The privacy rule is bypassed in 83 of 86 places.** Someone decided superusers must
   appear as "System Admin"; that decision holds in exactly one app.
2. **The null contract forked.** Same function name, one returns `None` and one returns
   `""` — so a JSON field is `null` in one app and `""` in another.

**Usage is completely siloed** — `compliance` calls the helper 63 times; every other app
calls it **zero** times and hand-builds instead (`TreatmentPlan` 40, `messaging` 15,
`Tasks` 13, `Notes` 8, `Invoices` 5, `HR` 1).

**Not yet verified:** which of the 83 sites actually render into a guest-visible surface.
The rule names "guest sessions" specifically, so the real exposure is some subset — that
audit is worth doing before deciding how urgent this is. The bypass itself is proven.

**Belongs in:** `TreatmentPath/utils.py` is already the right home. The work is deleting
the teamChat copy, pointing the compliance wrapper's callers at the original, and
replacing the 82 inline joins.

**This is the C12 / G11 pattern for the third and worst time:** a correct shared
implementation exists and almost nobody found it. Client IP (11 copies, one already in
`utils/`), URL builders (18 copies, mixin already exists), and now display names — except
this one carries a privacy decision, so the copies do not merely duplicate it, they
*silently overrule* it.

---

## EmailServiceGo — the answer is YES, but it is a SMALL job (and a correction)

### ⚠️ Correction to a number I published mid-sweep

I first ran the Go clone census without excluding `.worktrees/` and reported **665 groups /
1,384 definitions / 15,821 wasted lines**, concluding Go was "substantially worse" than the
Python backend. **That was wrong.**

`EmailServiceGo/.worktrees/email-to-task` is a registered git worktree containing **103 Go
files** — a near-complete copy of the service, last committed **2026-04-15**. The census
was diffing the repo against itself. The tell was `HandleMediaStream` appearing as two
separate clone groups when only two definitions exist.

I had excluded `.worktrees` from the Python census and failed to carry it across.

**Corrected, main tree only:**

| | groups | definitions | wasted lines |
|---|---|---|---|
| Python backend | 180 | 477 | — |
| **EmailServiceGo** | **25** | **56** | **330** |

**Go is the CLEANEST of the three codebases**, not the worst.

### What the 25 real groups are

One coherent pattern, not scattered mess — `internal/callagent` holds copies of logic that
also lives in its provider sub-packages:

| function | copies | packages |
|---|---|---|
| `handleMessage` | 2 | `callagent`, `callagent/elevenlabs` |
| `Connect` / `Close` / `SendText` | 2 each | `callagent`, `callagent/elevenlabs` |
| `Start` / `handleTwilioMessage` | 2 each | `callagent`, `callagent/twilio` |
| `handleTranscript` | 3 | `callagent`, `callagent/telnyx`, `callagent/twilio` |
| `drainPile` | 2 | `dentally/daylist/appointment`, `dentally/recall` |

This reads as a parent package retaining a legacy copy of provider code that was later
split into `elevenlabs/`, `twilio/`, `telnyx/` sub-packages — worth confirming which copy
is live before touching either, since a stale duplicate of a websocket handler is exactly
where a fix gets applied to the wrong one.

`drainPile` is the odd one out: 50 identical lines in two unrelated domains (day-list
appointments and recall).

### The deliberate cross-language duplication is separate

Go's copies of the identity keys (`phone.CanonicalE164`, `personname.CanonicalFullKey`)
are INTENTIONAL — two languages, one behaviour, pinned by shared fixtures. Do not
"consolidate" those. Their real weakness is already recorded as **C1**: the pinning
fixture is a COPY in two places with nothing enforcing that the copies stay equal.

### Repo hygiene — this worktree has now broken tooling twice

`.worktrees/` is gitignored, so it is invisible to git but fully visible to any tool that
walks the filesystem:

1. **vitest** — the frontend's phantom "136 failures across 310 files", all in `.worktrees/`
   copies (recorded as DEF-37-01), plus a second copy under `.claude/worktrees/`.
2. **this census** — 640 phantom clone groups.

The `email-to-task` worktree is clean (0 uncommitted files) and 5 months stale. Removing it
would eliminate a recurring source of false results. Not done here — `git worktree remove`
is the user's call.

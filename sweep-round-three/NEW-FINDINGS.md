# Round three — NEW findings (beyond the 90 already fixed)

Numbering continues from the main audit. A finding only earns a number here after
I have verified the mechanism myself against running code or the database — a
scan's claim is a lead, not a finding.

---

## #91 — ✅ VERIFIED / HIGH — one relative's reply silently cancels another patient's recall sequence

**Found by:** codex sweep of `dentallyIntegration/`, then verified and measured here.

**Where:** `dentallyIntegration/recall_automation.py:387-429` (`patient_replied_since`),
called at `:923`.

The recall sequence stops when the patient replies (FR29). The predicate resolves the
recall record's phone/email to a `ContactChannel` and then asks:

```python
SMSMessage.objects.filter(
    practice=practice, channel=phone_channel,
    direction="incoming", created_at__gte=since,
).exists()
```

There is **no Person, Patient or `dentally_patient_id` predicate**. The caller passes the
enrollment's exact `dentally_patient_id` to load `rec`, and the predicate then throws that
identity away and asks only "has ANYONE sent an inbound message on this channel?".

**Consequence, confirmed at the call site (`:923-929`):**

```python
if e.sequence.stop_on_reply and patient_replied_since(e.practice, rec, e.enrolled_at):
    e.status = "stopped"
    e.stopped_reason = "replied"
```

Alice and her son Ben share a landline. Ben replies to his own recall SMS; Alice's
enrollment is marked `stopped / replied` and she receives no further recall contact.

**Sharper than the scan stated:** the test is ANY inbound message, not an opt-out. A
relative asking *"what time is my appointment?"* cancels a different patient's recall.
Nobody is told; the sequence simply ends.

**Measured exposure (local DB, 32,165 recall records):**

| shared key | count | worst case |
|---|---|---|
| phone shared by >1 recall patient, same practice | **3,138 numbers** | 8 patients on `+4475389…` (practice 21) |
| email shared by >1 recall patient, same practice | **2,884 addresses** | — |

Not every one has an active enrollment, so this is exposure, not damage — but it is the
population at risk, and shared family numbers are the norm in this data, not the edge.

**Distinct from #76** (which fixed a shared-line guard's fallthrough): this is the recall
sequence's stop transition, which never had a guard at all.

**Fix direction (not yet applied):** the predicate has `rec.dentally_patient_id` in hand.
Either scope the message query to messages attributable to that patient, or — where the
channel genuinely cannot tell the family apart — refuse to infer a reply, the same
"one owner or nobody" rule `sole_person_for_channel` already applies. Which of the two is
a product decision: stopping on a shared line is defensible for an opt-out and wrong for
"what time is my appointment?".

**Verification:** mechanism read end to end (predicate + caller); exposure measured by
query. No test written yet, and no code changed.

---

## #92 — ✅ VERIFIED / MEDIUM-HIGH — fix #75 left one of three sibling methods on `.first()`

**Found by:** haiku scan of `messaging/`, then verified and measured here.

**Where:** `messaging/serializers.py` — the CallLog serializer's three sibling resolvers:

```python
def get_patient_id(self, obj):          # :1758  ✅ FIXED by #75
    chosen = sole_patient_for_person(person, ...)

def get_intake_id(self, obj):           # :1768  ✅ FIXED by #75
    chosen = sole_record_for_person(person, "intakes", ...)

def get_nurture_id(self, obj):          # :1779  ❌ STILL .first()
    nurture = person.nurtures.first()
    return nurture.id if nurture else None
```

Three methods, side by side, doing the same job for three record types. #75 fixed two and
missed the third. The fixed pair even carries the comment *"Same rule as get_patient_id
above (#75)"* — directly above the one that does not follow it.

A second site has the same shape: `messaging/serializers.py:133`
`_get_linked_intake_name()` calls `person.intakes.first()` to LABEL a conversation, then
joins the name by hand.

**Measured (local DB, live Persons only):**

| site | Persons at risk | worst case |
|---|---|---|
| `get_nurture_id` (:1779) | **44** hold >1 nurture | 6 nurtures on one Person |
| `_get_linked_intake_name` (:133) | **497** hold >1 intake | **44 intakes** on one Person |

On a fused Person the call log reports an arbitrary relative's nurture id, and a
conversation is labelled with an arbitrary one of up to 44 intakes.

**Distinct from #75/#76:** same bug class, sites those fixes did not reach. Recording it
separately because the interesting part is not the line — it is that a fix applied to
N-1 of N siblings looks complete in review and in tests.

**Fix direction:** `sole_record_for_person(person, "nurtures", practice_id)` already
exists and is the exact helper for :1779. :133 wants the same refusal plus `full_name()`
instead of the hand-rolled join.

**Method note for the remaining scans:** grep for the *siblings* of every helper call,
not just for the anti-pattern. The tell here was a correct call and an incorrect one in
the same class.

---

## #93 — ⚠️ VERIFIED MECHANISM / MEDIUM — the address a campaign sends to and the address the unsubscribe page acts on are both arbitrary

**Found by:** haiku scan of `marketingBroadcast/` (which reported a different, inert
divergence); the real issue surfaced on verification.

**Where:** two functions that must agree and are written independently:

```python
# delivery_reporting.py:27  — the address the campaign SENDS to
person.channels.filter(kind="email").exclude(canonical_value="").first()

# views/preferences_views.py:19 — the address the unsubscribe page SHOWS and ACTS ON
person.channels.filter(kind="email").first()
```

`ContactChannel._meta.ordering` is `[]` — **no ordering at all** — so each `.first()`
returns whatever Postgres hands back. Nothing makes the two agree.

**Measured:** **3,008 live Persons hold more than one email channel** (worst case: 18).

**Honest status: they agree TODAY.** I ran both against a real 4-address Person (76516)
and both returned `layna@smilehqs.com`. The queries have the same shape, so the planner
returns the same row. That is a coincidence of the current plan, not a guarantee — a new
index, a plan change, a concurrent write or a VACUUM can separate them.

**Why it matters if they ever separate:** a recipient unsubscribes on the preferences
page, which suppresses address B, while the campaign keeps sending to address A. The
person keeps receiving marketing they explicitly opted out of. That is a
consent/compliance failure, not a cosmetic one.

**Fix direction:** ONE helper answering "which address do we use for this person", with a
deterministic tie-break (primary flag, then oldest id). Both call sites use it. This is a
genuine "build once, use everywhere" case — the bug is not either implementation, it is
that there are two.

**Side observation, not part of this finding:** Person 76516's four addresses are
`manager@church-view-dental.com`, `reception@hactondentalcare.com`, `layna@smilehqs.com`,
`melikag21@outlook.com` — three different organisations on one Person. That looks like a
fused Person of the kind `split_collapsed_persons` exists for, and is worth a separate
look.

### Corrections to scan 4's claims

- **"`delivery_reporting.py:340` renders 'John None', CSV export broken"** — FALSE.
  `Person.last_name` is `null=False, default=''`; **0 Persons have a NULL last_name**, so
  `f"{p.first_name} {p.last_name}".strip()` cannot produce "None" for a Person. It yields
  `"John"`. Only a whitespace-collapse difference from `full_name()` remains, which is
  cosmetic.
  **Third scan in a row to assume a None-safety bug without checking the model's
  `null=`.** Nullability is per-model: it is real on `Patient`/`Intake`, impossible on
  `Person`.
- **"B1: the two email resolvers DIFFER because one excludes empty `canonical_value`"** —
  inert. **0 email channels have an empty `canonical_value`**, so the `.exclude()` changes
  nothing today. The agent found the right pair of functions for the wrong reason; the
  actual defect is the missing ordering, which it did not mention.

---

## Pending verification (reported by the sweep, not yet confirmed by me)

- `dentallyIntegration` #2 — per-patient recall reporting attributes a relative's
  messages/calls to the requested patient (`recall reporting`, HIGH as reported).
- `dentallyIntegration` #3 — confirmation task discards the appointment's exact Patient
  and files against an arbitrary Patient on its Person (HIGH as reported).
- `TreatmentPlan` #1 — `TreatmentPlanSerializer.create` fallback picks `.first()` Person
  on a shared channel before the name is checked (CRITICAL as reported).
- `TreatmentPlan` #2 — conversion relationship update writes to an arbitrary sibling.

These look plausible and well-cited, but none is a finding until the mechanism is
confirmed and the exposure measured.

---

## Scan 5 (`onlineBooking/`) — CRITICAL claim REFUTED, one minor item stands

### REFUTED: "the payment flow still splits names and creates duplicates" (reported CRITICAL)

`onlineBooking/services.py:594-596` does split a typed name at the first space:

```python
name_parts = hold.patient_name.strip().split(" ", 1)
first_name = name_parts[0]
last_name  = name_parts[1] if len(name_parts) > 1 else ""
```

The agent cited the comment above it — which describes finding #20's original damage
(persons 74615/74616, money attaching to a duplicate on a confirmed Stripe payment) — and
reported it as live and CRITICAL.

**It is not.** The same comment's next sentence states the mitigation, and I confirmed it:

```
booking split   ("Ken", "(Kenneth) Judge")  -> 'ken (kenneth) judge'
dentally store  ("Ken (Kenneth)", "Judge")  -> 'ken (kenneth) judge'
SAME identity key? True   -> duplicate Person? False
```

Finding #1 changed `Person.resolve` to compare on the JOINED name, so where the boundary
falls no longer affects resolution. `("Mary Jane","Watson")` and `("Mary","Jane Watson")`
also collapse to one key. The booking lane and the Dentally lane land on the same Person.

**What remains is real but minor:** the STORED boundary is still wrong — `first="Ken"`,
`last="(Kenneth) Judge"` rather than `first="Ken (Kenneth)"`, `last="Judge"`. The comment
says so itself: *"splitting at all is still wrong for storage."* Data-quality wart, not a
duplicate, not a payment problem. Logged as a consolidation item, not a finding.

### Method note — the failure mode that keeps recurring

**Two of five scans have now reported a FIXED bug as live by reading a code comment's
description of the historical defect and stopping before the sentence describing the
mitigation.** The explicit instruction "never trust a comment as evidence of current
behaviour" did not prevent it in scan 5.

These comments are long because the fixes were subtle, and that length is exactly what
trips a scanner: the first half reads like a live incident report. Remaining scans are
told to treat a comment that describes a bug as a prompt to check whether the SAME comment
describes a mitigation, and to mark any comment-derived claim "unverified — needs
execution" rather than ranking it.

This is an argument for keeping the scans as site-finders and doing the ranking here.

---

## #94 — ✅ VERIFIED / MEDIUM — the duplicate-merge UI can weld two humans, with no name guard

**Found by:** the `dataQuality/` scan's "detector vs merger use DIFFERENT keys" verdict —
though not for the reason the scan gave, and not before my own first hypothesis was
refuted by measurement (see below).

**Where:** `dataQuality/views.py:83-150`, the `merge` action on a `duplicate_contact`
cluster.

It validates that every id belongs to the issue's cluster and that all Persons are in the
caller's practice — then calls `Person.merge(winner, loser)` for each loser with **no name
check and no DOB check**. Its docstring actively encourages merging everything:

> *"Merges EVERY listed loser, not just one. The previous version merged a single pair and
> then marked the whole issue resolved, so a 3-person cluster silently kept its third
> duplicate…"*

Clusters are keyed on `channel_id` (`detail` holds `contacts`, `channel_id`, `match_type`,
`member_person_ids`) — i.e. on a SHARED PHONE OR EMAIL, which a family shares by
definition.

**Measured on 252 live `duplicate_contact` issues:**

| | count |
|---|---|
| clusters whose members are all one human (incl. typos/nicknames) | 243 |
| **clusters containing genuinely different first names** | **9** |

The 9, by first-name set:

```
9365, 9023, 9037, 8139:  ['amelie', 'joseph']                                  (4 members: each sibling duplicated)
9075:                    ['gemma', 'maia']                                     (4 members)
9024, 8088:              ['alex', 'alexander oliver', 'james', 'james alexander']
8932, 8074:              ['harry', 'harry & jack', 'laura']                    ("Harry & Jack" is a family record)
```

Issue 9365 is the clearest: four members, two humans — Joseph and Amelie Harrod-Attfield,
each with a spelling variant of their own name. The correct action is TWO merges. The
endpoint's design (pick one winner, merge all losers) produces ONE Person holding both
siblings — exactly the fused Person that `split_collapsed_persons` exists to undo, and
that the 90-finding audit spent its length repairing.

**Fix direction:** the merge action should apply the same rule the automatic path already
applies — refuse, or require explicit confirmation, when the losers do not share the
winner's `canonical_full_name_key`. `dedupe_persons --only-joined-name` already refuses
these; only the manual path lacks the guard.

### I was wrong first, and the data corrected me

My initial hypothesis was that the detector wholesale over-reports family clusters —
shared phone, different names — and that the UI therefore invites welding at scale. That
is **not** what the data shows. The detector is well-tuned: 243 of 252 clusters are one
human recorded twice, and it catches variants the canonical key cannot —
`Dan/Daniel`, `Mcgrath/Mc Grath`, `Stimpson/Stipmson` (a transposition),
`Deboroh/Deborah`. Those are real duplicates that `canonical_full_name_key` will never
match, which is precisely why a human-reviewed merge queue exists.

Two crude heuristics of mine also over-counted before I tightened them: a prefix test
flagged 53 clusters, and a similarity test flagged 45, both dominated by nicknames and
middle names (`Joanne/Jo`, `Lisa Louise/Lisa`, `William/William Alex Jack`). The defensible
number is **9**.

So the finding is narrow and specific: not "the detector is wrong", but "the merge action
has no name guard, and 9 live clusters would weld two humans if an operator accepted them
as presented."

---

## #95 — ✅ VERIFIED / HIGH — an activity log's `person` and its target record can be two different humans

**Found by:** codex sweep of `activityLog/`; mechanism and live rows verified here.

**Where:** `activityLog/serializers.py:120-159`. Two validators, run independently:

```python
def validate_person(self, person):            # :120
    if practice is None or person.practice_id != practice.id:
        raise ValidationError("Person is not in this practice.")

def _validate_content_object(self, data):     # :128
    ...
    if practice is None or obj_practice_id != practice.id:
        raise ValidationError({"object_id": "That record is not in this practice."})
```

The practice is checked TWICE — once for the supplied `person`, once for the generic FK
target. **Their relationship to each other is never checked.** Both can be valid members
of practice 16 and refer to two different humans.

This is the same shape as #57 (consent citing another patient's plan), which was fixed by
requiring practice AND patient. Here only the practice half exists.

**Live rows, measured:** of **685** ActivityLogs that point at a `Patient` and also carry a
`person`, **9 disagree about the human**:

```
log 2562: person=138608 'Ashton Scott'      but patient 113591 belongs to person 138609 'Amanda Vokes'
log 2561: person=139373 'Harriet Kitchener' but patient 113590 belongs to person 139374 'Theo Cully'
log 2505: person=118966 'Bethany Patchett'  but patient 113568 belongs to person 118967 'Phillip Patchett'
```

**Two wrong guesses of mine, recorded because both nearly ended the investigation early:**

1. The Person ids are CONSECUTIVE (138608/138609, 139373/139374), so I assumed these were
   duplicate-Person pairs — the same human split in two — which would have made the
   mismatch harmless. Checking the names refuted it: Ashton Scott and Amanda Vokes are not
   one person. The consecutive ids are just creation order.
2. I expected the missing cross-check to be theoretical. It is not; 9 rows already carry it.

**Probable origin — codex's second `activityLog` finding.** It reports that *migration
0013 assigns each shared-channel email/SMS/call event to an arbitrary Person*. `Bethany
Patchett` / `Phillip Patchett` sharing a surname is exactly what a shared family channel
produces. That would make these 9 rows historical damage from the migration rather than
API writes — plausible and unproven; the missing validator is real either way, and is
what would let new ones in.

**Fix direction:** when both a `person` and a practice-scoped `content_object` are
supplied, require that they agree — the object's own person/patient must resolve to the
supplied Person, or the write is refused. Plus a decision on repairing the 9.

---

# IMPLEMENTATION LOG — 2026-09-08

Each fix red-run first: a failing test against the unfixed code, then the change.

| finding | fix | tests |
|---|---|---|
| **#92** sibling miss | `get_nurture_id` → `sole_record_for_person`; `_get_linked_intake_name` → `sole_record_for_person` + `full_name()` | 7/7 |
| **#95** activity person/target | cross-check added to `_validate_content_object` — refuses when both name a human and they differ | 4/4 |
| **C12** client IP | new `utils/request_meta.py::client_ip`; marketing consent path now resolves `X-Forwarded-For` | 6/6 |
| **G13** privacy bypass | `teamChat.get_user_display_name` delegates to the canonical helper; canonical now joins via `full_name()` | 6/6 |
| **C2/C7** `"None"` in names | `Patient.__str__` → `self.full_name or "Unnamed patient"` | 9/9 |
| **C6** stale workaround | `_display_name` deleted, delegates to `patient.full_name` | (same file) |
| **C11** label stats scope | `note__user__practices` → `note__practice` | 4/4 |

Regression: `activityLog` + `messaging` **217 tests OK**.

## Two things learned while implementing, worth keeping

### `Note.save()` overwrites the practice you pass it

`Notes/models/note.py:137` sets the practice from `user.current_practice` on creation —
commented "SINGLE SOURCE OF TRUTH". `Note.objects.create(practice=B, user=u)` where `u`'s
current practice is A silently stores **A**.

Found because the C11 test could not build its own fixture. It makes the C11 fix *more*
clearly correct: `note__practice` is where the note actually lives, whereas
`note__user__practices` is every practice its author has ever belonged to. The test now
uses `.update()` to place the note, with a comment explaining why.

### A claim of mine, corrected mid-implementation

While writing the #95 test I found `ActivityLogSerializer` does not expose `content_type`,
and concluded the target's practice check was DEAD. Then I checked which serializer the API
actually uses: `create` → `ActivityLogCreateSerializer`, which DOES expose it, and `update`
falls back to `self.instance`. **The practice check is not dead; my direct-instantiation
test was not a reachable path.** The real gap was the missing cross-check, exactly as
codex reported it. The test now drives the real create serializer, and one of its cases
pins that the practice half still refuses a foreign target.

## Not implemented, and why

- **#91** (recall stop-on-reply) — needs a product decision: stopping on a shared line is
  defensible for a genuine opt-out and wrong for "what time is my appointment?".
- **#93, #94, C1, C8, C9, C13, G1–G12** — remain in the list, unstarted.
- **#94** in particular wants a UI decision as well as a guard: whether to refuse a
  mixed-name merge outright or require explicit confirmation.

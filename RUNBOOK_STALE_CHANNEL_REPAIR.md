# Runbook — defect C: contact channels linked to the wrong person

**Status:** cause fixed in BOTH languages and mutation-proven; data repaired to
**C = 0** on DEV and on a restored production copy (2026-09-05). NOT yet run on
production itself. See §7 for how the last 13 were cleared.

---

## 1. What the defect is

A `PersonChannel` row says "this Person can be reached at this email/phone".
Defect C is a Person linked to a value **none of their records carry**, while
someone outside their household does carry it.

Concretely: Reggie Durrant's Person was reachable at Terri Parish's email address.

## 2. The cause — additive-only linking, in BOTH languages

Channel linking only ever INSERTed. Nothing ever removed a link.

| | Code | Behaviour |
|---|---|---|
| Django | `Person.link_channels()` via the save signals | idempotent insert |
| Go | `INSERT ... ON CONFLICT (person_id, channel_id) DO NOTHING` | insert only |

So when a record was created with the wrong contact detail and then **corrected**,
the new value was linked and **the wrong one stayed attached forever**.

**Proven live, not inferred.** A test drove the real save path: create Reggie on
Terri's address, correct it, then read the links back.

```
linked before correction: ['+447700000222', 'terri@example.com']
linked AFTER  correction: ['+447700000222', 'reggie@example.com', 'terri@example.com']
```

## 3. The code fix — both sides, mirrored

| | Function |
|---|---|
| Django | `TreatmentPlan/contact/signals.py` → `_release_stale_channels` |
| Go | `internal/dentally/migration/service.go` → `releaseStaleChannels` |

Both carry `!! CROSS-LANGUAGE PARITY !!` markers. The rule is deliberately narrow —
a link is released only when BOTH hold:

1. this save actually dropped that exact value from **this** record, and
2. **no other** Patient/Intake/Nurture of the same Person still carries it.

So a family sharing a phone keeps it while any of their records has it. Only the
`PersonChannel` link is removed; the shared `ContactChannel` is never touched.

**Mutation-proven on both sides** (disable the call, the tests go red):

| Suite | Without the fix |
|---|---|
| `TreatmentPlan/tests/test_stale_channel_release.py` (5 tests) | 2 fail |
| `internal/dentally/migration/stale_channel_release_test.go` (2 tests) | 1 fails: *"STALE LINK: Person 159765 still carries terri.sentinel@..."* |

The tests that pass either way are the **over-removal guards** — a sibling record
still using the value, an unrelated save, the shared ContactChannel surviving for
its real owner. Those exist so the cure cannot become worse than the disease.

## 4. The data repair

`TreatmentPlan/management/commands/repair_stale_person_channels.py`

Deletes `PersonChannel` LINK rows only. Never a ContactChannel, Person, Patient,
Intake or Nurture.

**Hard guard — messaging traffic.** A channel referenced by any MessageSession,
SMSMessage, EmailMessages or CallLog is **never** released: the inbox resolves a
thread's person through `PersonChannel`, so removing the link would orphan the
conversation. Reported as `KEEP_MESSAGING` for human review (18 on DEV).

### DEV result, 2026-09-05

| | |
|---|---|
| C links assessed | 3,175 |
| RELEASE_ORPHAN_PERSON (Person has no records at all) | 2,363 |
| RELEASE_STALE (Person has records, none carry the value) | 744 |
| KEEP_MESSAGING (protected) | 18 |

| Defect | before | after |
|---|---|---|
| **C_WRONG_PERSON** | **2,715** | **13** |
| CONTROL_HEALTHY broken | — | **0** |
| D / D2 | unchanged | unchanged |

### Verified individually, not by totals

- all **3,157** planned releases confirmed gone; 0 still present
- all **2,702** distinct ContactChannel rows still alive — 0 destroyed
- all **18** messaging-protected links still intact
- 0 channels left with no person link
- 0 orphaned PersonChannel FKs (person or channel)
- record counts unchanged: 60,514 / 3,966 / 619

**The decisive independent check:** defects A and B (a record's OWN value not
linked to its Person) were 0 before and remained **0** after. If the repair had
released a channel a record actually carries, A/B would have risen. They did not.

## 5. The repair damaged something, and that is now fixed too

The first DEV run left **232 person/kind combinations with channels but no
`is_primary`** — every one on a Person with live patient records. Deleting a link
that happened to carry the primary flag silently demoted them.

That is user-visible: `primary_email`, `primary_phone` and the patient
serializer's `email_channel_id` / `sms_channel_id` all filter on `is_primary`, so
those people would show as **"no email address on file"** in the inbox.

The command now re-promotes a primary in the same transaction
(`_restore_primaries`), preferring a channel the Person's records actually carry,
then lowest id for determinism. It never demotes an existing primary.

After restoration: touched persons missing a primary **0**; global count 6,620 —
*below* the 6,765 that pre-dated this work. Proven idempotent: a second run
promotes 0 and leaves the multi-primary count unchanged at 305 (those 305 are a
separate pre-existing issue, not caused here).

## 6. Production procedure

```bash
# 0. deploy BOTH code fixes first, or the repair competes with a live defect
# 1. baseline
python manage.py contact_identity_audit --snapshot prod-C-before.csv

# 2. review the plan — writes nothing
python manage.py repair_stale_person_channels --csv prod-C-plan.csv

# 3. a small batch, then verify per link
python manage.py repair_stale_person_channels --limit 50 --apply

# 4. the rest
python manage.py repair_stale_person_channels --apply

# 5. prove it
python manage.py contact_identity_audit --snapshot prod-C-after.csv
python manage.py contact_identity_audit --compare prod-C-before.csv prod-C-after.csv
```

**Acceptance — totals are not acceptance:**

- C falls; **CONTROL breakage 0**; **NEW 0** on every defect
- **A and B must still be 0** — the tripwire for over-removal
- every planned release confirmed gone, every ContactChannel still alive
- **0 touched persons left without a primary** per kind
- messaging-protected links untouched


---

## 7. Clearing the last 13 — the messaging guard was too blunt

After the main repair, 13 channels (18 links) survived as `KEEP_MESSAGING`: the
rule refused to release **any** channel carrying messages, because the inbox
resolves a thread's person through `PersonChannel` and unlinking the last owner
would orphan the conversation.

### What the evidence actually showed

Each of the 13 was inspected individually — who genuinely carries the value, who
is linked to it, and how many messages it holds:

| channel | value | wrongly linked | genuine owner still linked |
|---|---|---|---|
| 104620 | +447383839690 | Scott Brenton, Laura Skudra | Charlie Sherridan Edwards (+2) |
| 104862 | +447469726576 | Test1, JB TEST, HELLO | Jonathan Beacher (+15 test rows) |
| 105115 | +447511235091 | Stephen Huntley, Sophie Huntley | Scott Huntley (+1) |
| 105263 | +447534281696 | Lisa Gowers, Katherine Slater | Theo Saggs (+1) |
| 107057 | +447790816140 | Louise Jones | Finlay / Mackenzie / David Jones |
| 109555 | +447976797825 | Kerry Davies | Kevin Samuels, Kerry Davis |
| 110041 | +447488387392 | Vanessa Taundry (no records) | Vanessa Taundry (+1) |
| 110166 | +447882918085 | Somir Ali (no records) | Somir Ali |
| 111585 | +447454756956 | Scott Grant | Jessica Grant, Don Ray Dare |
| 113403 | +447719011412 | John Winkles | Jennie Winkles (+1) |
| 113633 | +447734709439 | Joan Harrison | Joan K Harrison, Frank Thompson |
| 155329 | andersadlem@msn.com | Anders Adlem (other Person) | Anders Adlem |
| 190505 | +447864538288 | **Alex Cooper** | **Jacqui Rogan** |

**In all 18 of 18 links, a Person who genuinely carries the value stayed linked.**
So releasing the wrong link could not orphan anything — the thread keeps resolving
through its real owner. The blanket guard was holding wrong links in place for no
benefit at all.

### The corrected rule

A channel with messaging traffic is protected **only when releasing would leave it
with no linked owner at all** — verdict `KEEP_MESSAGING_WOULD_ORPHAN`. Otherwise
it is released like any other stale link.

This is not a manual data edit. It is the same repeatable command, with a rule
that asks the right question: *does the conversation still have a home?*

### Verified individually — all 13

For every channel, after applying:

- exactly the flagged links were removed, and nothing else
- **a genuine owner is still linked** (named in the table above)
- **message counts unchanged** — MessageSession, SMSMessage, EmailMessages, CallLog
- the ContactChannel row itself still exists

**0 problems across all 13.**

### The Alex Cooper / Jacqui Rogan case

`MessageSession` 1364 carries no name of its own — `participant_name` is the bare
number `+447864538288`. The thread takes its identity from the channel's
`PersonChannel` links, which held **both** Jacqui (the real owner) and Alex.

That is why the conversation could display as Alex Cooper: with two links, name
resolution picked whichever came back first. Removing Alex's link leaves exactly
one owner, so the thread now resolves to **Jacqui Rogan** — the person whose
number it actually is. The 9 SMS and 1 session are untouched.

### Result

| | before | after |
|---|---|---|
| C_WRONG_PERSON (production copy) | 3,528 | **0** |
| C_WRONG_PERSON (DEV) | 2,715 | **0** |
| control rows broken | — | **0** |

All five defects — A, B, C, D, D2 — now measure **0** on both.

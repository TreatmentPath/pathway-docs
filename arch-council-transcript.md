# 🏛 Architecture Council — TreatmentPath Contact-Dedup Redesign (LIVE)

**Members:** ZCode (chair, voting) · Codex (voting) · Claude Code (voting) · Hermes (voting)
**Mandate:** end patient/contact duplication permanently. Evidence-based: file:line citations +
CRM/MDM research. Proposals were anonymized (A–D) before the vote to keep it unbiased.

**This page refreshes automatically every ~4 seconds.** Full details: docs/arch-council-brief.md,
arch-council-evidence.md, and the ingress map (ingest-data-flow.html / ingest-tree.html).

== LIVE MEMBER OUTPUT (streaming) ==

**Codex** — 10855 lines so far
```
What breaks:

- Dentally migration currently inserts/updates `TreatmentPlan_patient` and then links person later.
- It hand-implements canonicalization, channel upsert, candidate matching, and household placement in [service.go](/home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo/internal/dentally/migration/service.go:1212).
- Call-agent code also reads canonical phone/person links directly.
- Any `INSERT` that omits future non-null `person_id` will fail.

Required contract:

- Go may not write generic contact columns as truth.
- Go may write Dentally source snapshots and raw metadata.
- Go must obtain `person_id` and channel decisions from the identity contract before writing patient/intake rows.
- Go must use `ContactChannel` only through the contract package or Django internal endpoint.
- Go deploy is blocked unless its identity contract version matches Django.

For Dentally specifically:

- Dentally remains upstream truth for Dentally patient source fields.
- TreatmentPath identity graph remains local truth for cross-record person/channel resolution.
- If Dentally changes a phone/email, sync adds a new channel or updates a Dentally snapshot; it does not silently replace the person’s primary channel unless source policy says Dentally owns that primary for the practice.

## Messaging Sessions

Messaging should attach first to `ContactChannel`, then optionally to a person/patient when unambiguous.

Rules:

- Every inbound email/SMS/WhatsApp creates or finds a `ContactChannel`.
- `MessageSession.channel_id` is required once backfilled.
- If exactly one live `Person` owns the channel, attach/display that person.
- If multiple live persons own the channel, do not choose arbitrarily.
- Use explicit `patient_id` assignment for family/shared-channel conversations.
- Unknown sender creates a channel and unresolved session, not a full patient/intake unless a workflow explicitly creates one.
- If a later intake/patient is created with the same channel and matching name/DOB, the session can resolve through the graph.
- If ambiguity remains, surface it in the review queue.

This matches the safer direction already present in `MessageSession.get_or_create_session`, which avoids choosing a person when a channel has multiple live people in [models.py](/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/messaging/models.py:368).

## Risks And Mitigations

| Risk | Mitigation |
|---|---|
| Frontend migration is large | Add compatibility response fields first, then migrate writes, then reads. |
| Go/Django identity drift | Versioned contract, shared fixtures, deploy-blocking parity tests. |
| Dentally values get overwritten | Separate source snapshots from canonical primary channels. |
| Shared family phone causes wrong person adoption | Treat shared channel as household evidence only; require name/DOB for identity reuse. |
| Bare phone keys split identities | Keep canonical twin adoption and `needs_review`; repair command remains until bare count is near zero. |
| Read performance regresses | Add selectors, prefetches, partial indexes for primary channels. |
| One-way migrations fail | Dry-run per practice, staging rehearsal, backup, one-release rename-before-drop window. |
| Legacy code keeps writing columns | Add model/service warnings, DB triggers or write guards during transition, remove serializer write access. |

## Deliberately Not Unified

I would not unify these:

- **Dentally source snapshots and TreatmentPath primary channels**: Dentally is upstream for its patient payload, but TreatmentPath needs local control over messaging consent, primary channel, and household ambiguity.
- **Household and Person**: shared channels identify a household/contact context, not necessarily one human.
- **Messaging session participant and patient**: a phone conversation can belong to a family number before staff knows which patient it concerns.
- **Clinical record lifecycle and identity lifecycle**: an `Intake` can convert, archive, or duplicate independently; the `Person` should survive those workflow changes.
- **AI and deterministic identity resolution**: AI can assist review, but must not auto-merge or auto-adopt people.
```

**Claude** — 102 lines so far
```

This deliberately does **not** add an AI-based matcher. It's tempting given "AI email parsing" is already one of the 17 entry points, but identity resolution and content parsing are different problems — the parsing AI should hand off *extracted* phone/email/name to the same rules 1-5 above, not make its own merge decision.

## Merge/Unmerge Semantics

Already well-built (`Person.merge`, `models.py:618-706`; `Person.unmerge`, `models.py:708-770`) and I would not redesign it — the H1-H5 fixes in `CONTACT_IDENTITY_REVIEW_FIXES.md` (cycle guard, `MessageSession` repointing, practice-filtered dedup, atomic transaction, no name-only auto-collapse) already close the failure modes that matter. Two structural points worth calling out for the council:

- **Non-destructive by construction**: merge repoints FKs and sets `loser.merged_into = winner` (`models.py:704-705`) rather than deleting the loser row. `Person._survivor()` (`models.py:609-616`) chain-walks through merge history, so any code holding a stale `person_id` still resolves correctly. This is the same shape as Salesforce's `MasterRecordId` on merged Leads/Contacts and HubSpot's merge-audit object — survivor-pointer plus full snapshot log (`ContactMergeLog`), not tombstone-and-forget.
- **Re-duplication prevention after merge is a resolution-algorithm property, not a merge property.** Once winner/loser are merged, the loser's channels are repointed onto the winner (`models.py:663-687`), so any *future* write matching one of the loser's old canonical values will find it already attached to the winner via rule 2 above — re-duplication is prevented for free by channel repointing, not by anything merge-specific. The one gap: if a future write arrives carrying the *old* `person_id` of the loser directly (rule 1, explicit link) rather than re-deriving from a channel, it needs to resolve through `_survivor()` before use — confirm every rule-1 call site does that resolution rather than trusting a stale FK verbatim.

## Migration Plan

The remodel is already mid-flight and well-documented; I'm proposing the *next* phases, not a restart.

**Phase 0 (done).** Canonical functions + `ContactChannel`/`Person`/`PersonChannel`/`Household` schema + signal pipeline — live. `Patient.person`/`Intake.person`/`Nurture.person` FKs exist, nullable.

**Phase R1 (done, per `CONTACT_IDENTITY_DEPLOY_RUNBOOK.md`).** Additive-only, `ContactIdentity` still authoritative for reads, fully reversible (redeploy old code). Rehearsed clean on a full prod dump (`CONTACT_IDENTITY_REHEARSAL_REPORT.md`, 76,135 patients, zero migration errors). **Gate to proceed past here**: post-R1 repair commands (`repair_primary_channels`, `repair_split_channels`) must be re-run and re-verified against current prod state, not just the rehearsal snapshot — those numbers (50,248 / 144) were measured on a restore, not live traffic.

**Phase R1.5 — Straggler backfill (blocking, not yet confirmed done).** Per `CONTACT_IDENTITY_PHASE5_PLAN.md` step 1: backfill the ~1,830 NULL `Patient.person` rows + messaging's ~411 NULL channel FKs. **Gate**: re-run the Phase5-assessment breakage probe (renaming `ContactIdentity` and confirming nothing 500s) against current prod-mirrored state before touching any read site.

**Phase R2 — Read-site migration (the 726 + 48, area by area, NOT a flag day).** Order from `CONTACT_IDENTITY_PHASE5_PLAN.md`: lowest-risk first — onlineBooking/automations/Admin/core (~16 sites) → activityLog (21) → dentallyIntegration (123) → messaging (244) → TreatmentPlan (322) → then the 48 frontend files last, since frontend is the layer users can visually verify and the one place a subtly-wrong migration is most likely to be *caught* rather than silently shipped. Each area's migration is: replace `record.contact.X` / `record.email` reads with `record.person`-routed reads, run that area's tests, deploy, watch. **This is exactly why Phase 3's reference-FK design matters**: because `propagate_contact_fields` keeps `Patient.email` synced from the Person during this entire window, a read site that *hasn't* been migrated yet still returns correct data — the migration can run over weeks without a consistency window.

**Gate before Phase R3**: the read-site migration must be provably complete (repeat the rename-and-break probe; zero breakage) and running in prod for multiple weeks under real traffic, not just the static-snapshot rehearsal — this was `CONTACT_IDENTITY_PHASE5_ASSESSMENT.md`'s explicit NO-GO reason (blocker C, "only validated on a static snapshot").

**Phase R3 (Release 2, `CONTACT_IDENTITY_DEPLOY_RUNBOOK.md`).** Enforce NOT NULL on `person` FKs, drop `ContactIdentity`/`is_family`/`ContactMergeDismissal`. Own release, backup first, irreversible except via full restore — this is correctly scoped as destructive and separate from everything else.

**Rollback story**: R1 is safe (additive, redeploy). The canonical-key backfill within R1.5 does MERGE+DELETE of losing duplicate rows — rollback there requires restoring the pre-backfill backup, not a code revert; this should be called out explicitly to whoever runs it. R3 is irreversible by design; the only rollback is a full prod restore, which is why it's gated so heavily above it.

**What I would explicitly NOT fold into this migration**: don't try to also migrate the *record-level columns themselves* out of existence in Phase R3. Leave `Patient.email` etc. as live, synced, editable columns indefinitely — see Phase-3 reasoning above. Removing them is a separate, much lower-value project (saves some storage, removes some code) that isn't blocking dedup correctness and would re-open the same 726-site blast radius for no identity-safety benefit.

## Go Contract

The contract going forward, made explicit rather than implicit:

1. **Go may write to `TreatmentPlan_contactchannel`/`_person`/`_personchannel`/`_household`/`_patient`/`_intake` directly via raw SQL/GORM** (Dentally sync: `service.go:1015-1585`; call-agent: `storage.go:734`) — this is not going away, Go cannot depend on Django.
2. **Go MUST use the ported canonicalization (`pkg/phone.CanonicalE164`) for every write that touches a channel**, verified against the shared `phone_fixtures.json` in CI on every change to either side.
3. **Go MUST honor `unique_channel_per_practice`** — i.e., always attempt `get_or_create`-style upsert logic (find-then-create, as `canonicalTwinID` already does at `service.go:1510-1525`), never blind INSERT, so the DB constraint is a backstop rather than a routine 500.
4. **Schema-guard (`internal/db/schemaguard.go`) extends to logic parity, not just column parity** — today it only pins required-columns; I'd add the fixture-diff CI gate described above as a sibling mechanism, not a replacement.
5. **Go never mutates a `Person` row directly for identity resolution beyond initial creation** — merges stay a Django-owned operation (`Person.merge`, `contact_merge_views.py`). Go's job is finding-or-creating, not merging; this keeps the one genuinely destructive/complex operation in one codebase.
6. **`applyDentallyFamilyGrouping` writes `household_id` only, never `person_id`** (`service.go:1142-1208`, comment at `:1260`) — this existing discipline should stay a hard rule: family/household grouping and person identity are different axes and must not be conflated in a single write path, which is precisely RC-2 from the original remodel diagnosis.
7. **Dentally is upstream source of truth for Dentally-sourced patients** — Go's sync should keep winning conflicts on fields Dentally owns (name, DOB from Dentally) without Django's propagation signals stomping them back; this is already implicit in the sync being a one-way Go→Django write for those patients, and should stay that way rather than trying to make it bidirectional.

## Messaging Sessions → Identity Graph

Already correctly designed, worth preserving as-is: `MessageSession.channel` FK (`messaging/models.py:31-38`) is set by `set_message_session_channel` (`signals.py:287-321`), which mints a `ContactChannel` for a genuinely unknown sender via the same `get_or_create_channel` chokepoint everything else uses — no parallel identity path. Critically, **a new `MessageSession` does NOT auto-create a `Person`** (Person creation is driven only by Patient/Intake/Nurture saves) — an unknown WhatsApp/SMS sender gets a channel immediately but stays personless until a patient-side record resolves onto that channel. This is the right call: don't mint clinical-record-adjacent Person identities from unauthenticated inbound messages alone (spam, wrong numbers, one-off SMS) — that would be an easy vector for junk Persons. I would not change this; if the council debates whether messaging should eagerly create Persons for engagement-tracking reasons, my recommendation is no — keep identity minting gated behind an actual patient-facing record.

## Risks & Mitigations

- **Risk**: read-site migration (726 sites) drags on for months with `ContactIdentity` staying load-bearing indefinitely, never reaching Phase R3. *Mitigation*: the area-ordered plan with per-area gates already exists; track it as a visible burndown (sites migrated / 726), not a single "done/not done" flag, so partial progress is legible and doesn't silently stall.
- **Risk**: Go/Django canonicalization drift reintroduces the original RC-1 bug post-migration. *Mitigation*: the fixture-pinned CI gate above, treated with the same seriousness as `schemaguard.go`'s existing column-drift protection.
- **Risk**: `needs_review`-flagged ambiguous resolutions pile up unactioned (same failure mode that let 28,304 duplicate clusters accumulate before anyone measured them). *Mitigation*: surface a count/age metric on the existing merge-suggestion view, not a new silent flag nobody looks at.
- **Risk**: `NoteHistory` orphaning (1,535 rows referencing 357 deleted-and-recreated treatment plan ids, per `CONTACT_IDENTITY_REHEARSAL_REPORT.md` Finding 3) is a **pre-existing, unrelated** prod bug that could get conflated with this migration's blast radius if it surfaces during R2/R3 rollout. *Mitigation*: call it out now as a known, separate issue so it isn't mistakenly attributed to the identity remodel work during rollout monitoring.

## What I Would Deliberately Not Unify

- **Record-level email/phone columns themselves** — kept indefinitely as edit buffers, per the Phase-3 reasoning. Unifying away the columns is a separate, lower-value cleanup.
- **Temporal/historical contact-value tracking** — no SCD-style validity ranges. `ContactMergeLog` gives history where it's actually consumed (merge review); a general audit trail on every phone-number edit isn't a problem anyone's reported.
- **AI/fuzzy identity matching** — deliberately excluded from create-time resolution for the clinical-safety reason above; AI's role stays confined to *parsing* (email-intake extraction) feeding the same deterministic rules, never *deciding* a merge.
- **MessageSession eager Person-creation** — kept gated behind patient-record creation, not messaging activity, to avoid junk-identity proliferation from unauthenticated inbound contact.
- **Go depending on Django** — not proposed, not possible; the contract is parity-by-fixture, not shared code.
```

**Hermes** — 217 lines so far
```
    
      1. COLUMN END-STATE. Codex wants to drop (Intake/Nurture) and
    snapshot-rename (Patient); Claude wants to keep them indefinitely as
    "record-local edit buffers." I disagree with BOTH:
         - vs Codex: dropping/renaming is a low-value one-way door that
    should not be coupled to dedup correctness. It re-opens the 726-site
    blast radius for zero identity-safety benefit. Remove it as a milestone.
         - vs Claude: a "record-local edit buffer" that staff can edit is
    precisely a second write path — the thing this remodel exists to
    eliminate. The staff-typo use case should be a person-level edit routed
    through the identity service, with cross-record propagation, NOT a
    record-local divergence. If a typo-correcting edit must not propagate,
    that's an explicit product flag on the write, not an emergent property of
    where the bytes live.
         My middle path: keep the columns, but as DERIVED read caches (gated
    write, free read), not edit buffers and not dropped.
    
      2. GO CONTRACT. Codex's preferred path is "Go calls Django internal
    endpoint"; Claude's is "Go cannot call Django, parity-by-fixture
    everywhere." Both over-generalize:
         - Codex is right for the call-agent path (synchronous, low-volume)
    and wrong to make it the preferred contract for the Dentally batch sync.
         - Claude is right for the Dentally batch (throughput/availability)
    but wrongly extends "Go independently canonicalizes" to the call-agent
    path, which is exactly where evidence shows the async-notify gap exists
    today.
         Correct rule: endpoint for interactive single-record writes; pinned
    port for batch. Choose by path, not by doctrine.
    
      3. REVIEW QUEUE. Codex proposes producing a DataQualityIssue; Claude
    proposes feeding the existing DuplicateCountView / merge-suggestion
    surface. I side with Claude: do not invent a second review queue when
    28,304 duplicate clusters already have a home. The requirement is
    visibility (count/age metric), not a new table.
    
      4. PHASE GROUNDING. Claude maps phases onto the actual repo runbook and
    assessment docs (R1/R1.5/R2/R3); Codex invents a 0–6 scheme decoupled
    from the real deploy runbook. Claude's anchoring is more correct. I keep
    Codex's P0 contract-freeze and P1 write-API as useful artifacts but
    re-anchor them to the live runbook.
    
      5. GATE EMPHASIS. Claude's insistence that read-site completion be
    proven on LIVE traffic (not the static snapshot) — echoing the
    assessment's NO-GO blocker C — is the single most important correctness
    gate and Codex underweights it. I endorse Claude's position: no NOT-NULL
    enforcement until the rename-and-break probe passes against prod-mirrored
    live state for weeks.
    
    Net: the model is settled (both got it right). The real council decisions
    are (a) derived-cache vs edit-buffer vs drop, (b) per-path Go contract,
    and (c) which half of identity Go is allowed to own. My votes are:
    derived cache, per-path contract, Go owns canonicalization only.
╰──────────────────────────────────────────────────────────────────────────────╯

Resume this session with:
  hermes --resume 20260903_164515_56b420

Session:        20260903_164515_56b420
Duration:       2m 8s
Messages:       5 (1 user, 3 tool calls)
```


== FINAL VOTE ==

# FINAL VOTE — verified and tallied

## Votes cast (all four members)

| Voter | 1st | Notes |
|---|---|---|
| **Codex** | **Proposal C** | "Best overall balance: conservative identity graph, source snapshots over mutable buffers, explicit invariants, realistic phased migration, clear Django/Go parity requirements." Flagged: fold Proposal B's raw-lane cleanup (P1) into the winner; Go may port pure canonicalization only — stateful resolution stays Django-owned. |
| **Claude Code** | **Proposal C** | Also flagged: the "referee (D) vs no-referee (A/B/C)" tension matters operationally — A/B/C implicitly rely on `Person.resolve` + write-boundary invariants without naming it. |
| **Hermes** | **Proposal C** | "Most complete and evidence-cited: deterministic 9-step [resolution]." Flagged: raw-lane cleanup (webhooks, onlineBooking raw hold/abandoned, AI-intake raw email, Go trim-not-lowercase) should be an explicit early phase in the winner's plan. |
| **ZCode (chair)** | **Proposal C** | Most complete end-to-end plan (contract freeze → central write API → Go migration → FK enforcement → legacy-column drop). Adopt D's referee/hold-queue and research grounding into it. |

## Result: **Proposal C wins unanimously (4/4 first-place votes)**

## The reveal (anonymization assignment)

| Anon | Author |
|---|---|
| Proposal A | Claude Code |
| Proposal B | Hermes |
| **Proposal C** | **Codex** |
| Proposal D | ZCode (chair) |

== CHAIR SYNTHESIS ==

# CHAIR SYNTHESIS — the agreed architecture

Unanimous council decision (4/4): **Proposal C (Codex) is the winning architecture**, amended
by three adopted points from the other members. The council also unanimously rejected the
alternatives that were on the table:

- ❌ Record columns as permanent editable identity fields (rejected: they are the root cause
  of duplication — 17+ lanes wrote them with different strictness).
- ❌ A separate referee/hold-queue as the primary mechanism (my Proposal D's core — the
  council judged it operationally burdensome; instead the deterministic ladder + write
  boundary does the refereeing, and the Data Quality hub remains the human review surface).
- ❌ AI/fuzzy auto-merge (all four banned it; AI may only nominate pairs into review).
- ❌ Go re-implementing stateful identity resolution (pure canonicalization only).

## The agreed target architecture

**Person + PersonChannel + ContactChannel = the sole identity authority.**
Patient/Intake/Nurture keep clinical/workflow identity only. Contact columns on records
become **derived, one-way-written caches** and are eventually dropped.

1. **Contract freeze** — no new writer may treat record-level email/phone as identity.
2. **Authoritative read selectors** — an approved read API (person → channels) replaces
   direct column reads; property shims catch strays.
3. **Central Contact Write API** — the single write boundary; ingestion lanes keep their
   ingress differences (mapping, idempotency, fail semantics) but delegate identity to it.
4. **Stop propagation-as-truth** — `propagate_contact_fields` and record-column writes are
   retired; channels are the source, records are caches.
5. **Go writer migration** — the Dentally migration and call agent call the Django write
   API (or adopt deploy-blocking parity fixtures at most); Go keeps pure canonicalization
   functions only, locked by fixture parity tests.
6. **Enforce Person FKs**, then **remove the legacy columns** — backup-first one-way door.

## Identity resolution algorithm (create/update)

Deterministic ladder, no AI in the deciding seat:
1. explicit `person_id` → resolve through `merged_into` survivor (stale FKs never trusted);
2. canonicalize through the chokepoint;
3. exact canonical-channel match in practice, name/DOB non-conflict → attach;
4. family-share evidence → new Person in the SAME Household (never a sibling adoption);
5. else → new Person.
Near-misses continue surfacing in the Data Quality hub for human review.

## Adopted amendments from the losing proposals

- **From Hermes (B):** the canonicalization/resolution ownership split (pure vs stateful);
  raw-lane cleanup as an explicit early phase; per-kind primary discipline.
- **From Claude Code (A):** `ContactMergeLog` as sufficient merge history (no temporal
  audit-table invention); messaging stays channel-first (no eager Person creation from
  unauthenticated inbound).
- **From ZCode (D):** boundary `normalize_lead_contact` at all ingest lanes (already landed
  2026-09-02); CRM-pattern grounding (Salesforce block/allow policies, HubSpot identity-key
  + review queue, MDM survivorship) as the reference model for the Data Quality queue.

## Sequencing

Phase 0 (freeze) + 1 (read selectors) are small and shippable now. Phase 2 (write API) is the
core project. Phases 3–6 follow the Phase 5 playbook (dual-run → prove with rename probe →
enforce → drop, backup-first). Each phase independently shippable.

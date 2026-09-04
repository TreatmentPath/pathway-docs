# Architecture Council Brief — Ending Contact Duplication in TreatmentPath

You are one of several AI architects in a design council. Your proposal will be synthesized
with the others into a final architecture decision. Be concrete and opinionated.

## The system

TreatmentPath: dental-practice management SaaS. Django 5.1 + DRF monolith (main backend),
a Go 1.24 service (EmailServiceGo) sharing the same Postgres, React frontends. Multi-tenant
by Practice.

## The problem

Patient/contact records keep duplicating — the same human existing as multiple Patient /
Intake / Nurture rows, or contact details drifting between records. Root causes identified
by a full ingress audit (17+ entry points):

1. **Contact data is duplicated as plain columns on every record type.**
   `Patient.email`, `Patient.phone_number`, `Patient.country_code`,
   `Patient.secondary_phone_number`, and the same on Intake and Nurture. Each of the 17+
   entry points (webhooks, marketing-form handoff, CSV import, online booking, automations,
   AI email parsing, staff UI, Go Dentally sync, Go voice-call agent) writes these columns
   with different strictness: some E.164-parse phones, some store raw; some lowercase email,
   some keep casing; online booking stores fully raw.

2. **A canonical layer already exists but is not the sole authority.** The Contact Identity
   remodel introduced: `ContactChannel` (ONE row per distinct phone/email per practice,
   unique constraint on `(practice, kind, canonical_value)`; canonical keys = E.164 phones /
   strip+lower emails, produced ONLY by canonical functions), `Person` (one human per
   practice; name, DOB, `merged_into` non-destructive merge pointer, M2M to channels through
   `PersonChannel`), `PersonChannel` (through table; `is_primary` per kind; consent flags
   `use_sms`/`use_email`/`marketing_consent`), `Household` (family grouping).
   `Patient.person` / `Intake.person` / `Nurture.person` FKs exist (nullable today), and
   pre/post-save signals auto-resolve Person + canonical channels on EVERY save of these
   models. So the channel/Person graph is consistently normalized; the inconsistent spots
   are the record-level columns.

3. **Identity logic has historically trusted the record columns.** Conversions matched
   patients by `email__iexact` etc., which caused a family-sibling bug (primary contact's
   patient adopted onto a sibling's record). We hardened the matcher (person-link first,
   email strip+lower, phone matched raw AND canonical E.164, reject cross-Person adoption),
   normalized ingestion boundaries (strip+lower email, parse phones), and made creation
   write canonical values. The Go Dentally sync re-implements person/channel resolution in
   raw SQL (E.164 + lowercase, twin adoption) because it bypasses Django.

4. **Two Go writers bypass Django's signal pipeline**: the Dentally patient migration/sync
   (raw SQL into the Django tables, identity resolution re-implemented in Go, guarded by a
   schema-guard + parity concerns) and the call-agent intake creator (GORM insert, no
   signals; Person link established by a follow-up notification to Django).

## Constraints

- Multi-service: Django ORM + Go raw SQL/GORM against the SAME tables (schema-guard
  whitelists them). Go cannot import Django code.
- ~726 backend read sites + 48 frontend files still read `Patient.contact`-era fields and
  the record-level email/phone columns; the frontend displays and edits them directly.
- Phase 5 of the running remodel drops the older `ContactIdentity` layer and enforces
  NOT NULL on `person` FKs; the record-level email/phone CharFields are NOT in any
  documented drop plan.
- Phone local formats matter (UK practice: local `07…` inputs), consent flags
  (`use_sms`/`use_email`/`marketing_consent`) live on `PersonChannel`.
- Operational constraints: one-way-door migrations need backup + staging rehearsal; the
  frontend edits emails/phones in ~48 files; Dentally is the upstream source of truth for
  some patients (its own values must not be silently re-stamped).

## The question

Design the architecture that ENDS contact duplication permanently. Address:

1. Should record-level email/phone columns be dropped and replaced by FKs to ContactChannel
   ("reference model"), or kept as normalized display caches, or something else (e.g. a
   dedicated ContactPoint value-object table with historical validity ranges)? Defend it.
2. Where does normalization live so no entry point can bypass it (Django signals vs a single
   ingestion service vs DB constraints vs app-layer only)? How do you keep the Go writer
   parity-locked with Django's rules?
3. Identity resolution at create time: what is the single algorithm (person-link first?
   canonical-channel match? name/DOB heuristics? AI?) and what happens on ambiguity
   (auto-create vs hold-for-review queue)?
4. Merge/unmerge semantics and how to prevent re-duplication after a merge.
5. Migration strategy from today's schema to the target: phases, dual-run, backfills,
   how the 726 read sites and the frontend move over, and what the rollback story is.
6. What breaks in the Go service, and the contract between the two services going forward.
7. How messaging sessions (auto-created for unknown senders) attach to the identity graph.

## Output format

Return a structured proposal: target architecture, the reasoning for the core decision
(question 1), the resolution algorithm, the migration plan in phases with gates, the Go
contract, risks with mitigations, and anything you would deliberately NOT unify (and why).
</content>

# VERIFIED EVIDENCE BASE (file:line, audited 2026-09-03)

## Canonical chokepoints (Django)
- TreatmentPlan/models.py:219 — ContactChannel.get_or_create_channel ("the ONE chokepoint for a contact channel WRITE")
- TreatmentPlan/models.py:153 — canonical_key(): phone → canonical_phone_e164, email → strip().lower()
- TreatmentPlan/models.py:167-181 — non-E.164 phones stored as bare digits + needs_review=True; _canonical_twin adoption of existing E.164 channel
- unique_channel_per_practice constraint on (practice, kind, canonical_value) — DB-enforced dedup

## Signal pipeline (fires on EVERY Patient/Intake/Nurture save)
- TreatmentPlan/contact/signals.py:32 _ensure_channels; :56 _resolve_person_for_record → Person.resolve
- TreatmentPlan/contact/signals.py:419 post_save propagate_contact_fields (email/phone edits propagate across the person's cluster; skips created=True)
- Person.resolve (models.py:539-607): name match + DOB-conflict check; reuse-or-create; deliberately creates NEW Person in same Household when names differ (docstring 553-558)
- Person.link_channels (models.py:455-491): guarantees exactly one primary per KIND (was silently missing → invisible channels bug; 6,765 unflagged combos measured on dev)
- Person.merged_into (models.py:398) — non-destructive merge pointer

## Conversion matcher (journey/mixins.py, hardened 2026-09-02)
- match_existing_patient (module fn): email strip+lower + iexact; phone matched RAW and canonical E.164; same-Person preferred; cross-Person adoption REJECTED
- _get_or_create_patient: creates with canonical values (email lower, E.164 phone); _forced_patient short-circuit (target_patient_id, "selected")
- NOTE comment block (mixins.py:174-187): record deliberately NOT repointed onto matched patient's Person — family = different humans

## Ingest lanes (phone/email handling per lane)
- intake_views.py:1215-1240 — legacy webhook: parse_phone_number primary + SECONDARY, practice default cc; NOT try/except-wrapped
- custom_webhook_views.py:426-470 — custom webhook: parse_phone_number defensive; serializer enforces email-OR-phone; idempotency cache X-Idempotency-Key
- marketingBroadcast/form_handoffs.py:262-282 — was fully raw; 2026-09-02: parse_phone_number + normalize_lead_contact
- normalize_lead_contact (contact/sync.py) — strip+lower email, trim phone; applied at all 3 webhook/form boundaries
- patient_views.py:1204 import_csv — phone E.164 :1355; email strip() ONLY (not lowercased) :1349; explicit PersonModel.resolve :1473
- onlineBooking/services.py:507 — patient created on Stripe payment with RAW hold email/phone (canonical check only for hold-vs-patient match :339)
- onlineBooking/tasks.py:118 — abandoned-session intake, RAW email/phone
- automations/actions.py:296 create_patient / :741 nurture / :855 intake / :943 dentally-patient — email strip() only; canonical_phone_e164 used for dedup checks
- services/email_intake.py:380 — AI-parsed intake: phone E.164 :349, email raw from AI extraction
- serializers/patient.py:601 PatientSerializer.create — names stripped/titled; email casing preserved on record
- messaging/models.py:251 MessageSession.get_or_create_session — auto-creates session for UNKNOWN senders; email strip+lower :272; phone split cc+local :283-298; called from SMS/Email/WhatsApp saves (:997/:1399/:2064)

## Go service (EmailServiceGo — same Postgres)
- internal/dentally/migration/service.go — the biggest identity writer: UPDATE/INSERT TreatmentPlan_patient (:1015/:1074), INSERT TreatmentPlan_person (:1438, resolve mirrored in Go), contactchannel (:1560, E.164/lowercase; canonicalTwinID :1512), personchannel (:1462), household (:1183/:1272 from Dentally family_id)
- Go channel writes: pkg/phone.CanonicalE164 + practice-region fallback; patient.phone_number column = dial/local SPLIT (not single E.164); email lowercased+trimmed in Go path
- internal/callagent/storage.go:577 createIntakeFromCall — GORM insert, BYPASSES Django signals; E.164 in normalized_phone :694; email trimmed NOT lowercased :615; notifies Django after (:757)
- cmd/worker inbound email — parses Redis mail, POSTs to Django webhook receive-email-v2 (:155/:191) — ZERO DB writes in Go
- internal/db/schemaguard.go — whitelists Django tables Go writes; fails startup on schema drift
- internal/workflows/actions/update_record.go — delegates record updates to Django REST (no direct writes)

## Known duplication incidents (evidence of the failure mode)
- Family-sibling bug: conversion matched by email__iexact across family (shared email) → primary's patient adopted onto sibling (fixed: cross-Person adoption rejected)
- 6,765 person/kind combos with channel but no primary flag (visibility bug, fixed in link_channels)
- Phase 5 assessment: ~726 backend read sites + 48 FE files still read legacy contact layer
- 202-member "household" exists on a shared generic value (oversized-group problem, flagged in Phase 5 plan)

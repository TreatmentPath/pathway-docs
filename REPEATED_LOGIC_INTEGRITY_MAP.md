# TreatmentPath Repeated Logic and Utility Map

## Purpose

This document maps logic that is repeated, reimplemented, wrapped, or partially duplicated across the TreatmentPath areas. The goal is to identify code that should become a shared global utility, an app-specific utility, a service, a common validator, or a documented canonical implementation.

This is a read-only source-mapping exercise. It does not modify application code, database data, migrations, or configuration.

## Areas in scope

1. `TreatmentPath/activityLog`
2. `TreatmentPath/automations`
3. `TreatmentPath/dataQuality`
4. `TreatmentPath/dentallyIntegration`
5. `TreatmentPath/marketingBroadcast`
6. `TreatmentPath/messaging`
7. `TreatmentPath/Notes`
8. `TreatmentPath/onlineBooking`
9. `TreatmentPath/TreatmentPlan`

Direct callers, shared services, serializers, tasks, signals, webhooks, management commands, frontend callers, and cross-area helpers are included when they implement or invoke the repeated logic.

## Sequential review protocol

Use exactly one independent subagent at a time. Never run area agents in parallel.

For each area:

1. Spawn one read-only mapping agent.
2. Have it inventory repeated or equivalent logic in that area and its direct callers.
3. Compare its results with this document and avoid duplicating mappings already recorded here.
4. Record only genuinely new mapping entries.
5. If new entries are found, update this document and restart that area’s clean-pass count.
6. Continue until the area has three consecutive passes with no genuinely new mapping entries.
7. Only then move to the next area.

A failed, unavailable, timed-out, or incomplete agent is not a clean pass. A duplicate observation does not reset the clean-pass count.

## What counts as repeated logic

Map logic when two or more implementations appear to perform the same semantic operation, including:

- Validation and normalization
- Date, time, timezone, currency, and numeric calculations
- Pagination, filtering, sorting, and search
- API request/response handling and serialization
- Authentication, authorization, feature checks, and permission gates
- Error handling, retries, logging, and status transitions
- Database query patterns and common object lookups
- Deduplication, idempotency, caching, and locking
- File, image, export, import, and upload handling
- Notification, email, SMS, and webhook delivery
- Formatting, parsing, labels, constants, and display transformations
- Background task creation and scheduling
- Frontend data fetching, mutations, state updates, and form validation
- Any business rule implemented in multiple files or apps

Do not count ordinary reuse of the same imported function as duplication unless the caller changes or bypasses its invariants. Distinguish true shared implementation from copy-pasted, wrapper-specific, or semantically divergent logic.

## Required mapping fields

Every new mapping entry must include:

1. A continuous mapping number.
2. A short name for the repeated logic family.
3. The semantic operation being implemented.
4. All discovered implementations and callers, with exact file and line evidence.
5. Whether each implementation is canonical, a wrapper, a reimplementation, or a divergent variant.
6. Inputs, outputs, side effects, and dependencies.
7. Scope classification: global utility, app-specific utility, service, component helper, or not suitable for extraction.
8. Differences in validation, matching, ordering, fallback, transactionality, idempotency, and error handling.
9. The likely maintenance, consistency, correctness, or performance consequence.
10. The proposed canonical implementation or source of truth, if one is evident.
11. Whether extraction is safe, risky, or requires design decisions.
12. Confidence and conditions required for impact.

## Continuous repeated-logic register

The register is intentionally continuous rather than divided into baseline and sequential sections. Add new entries at the end and never renumber existing entries.

**Current numbering reaches Mapping 140.** There are 139 distinct numbered mapping families because Mapping 133 is currently unassigned. All nine areas in scope have three consecutive clean passes (inventory review complete; implementation readiness is assessed separately below).

### Mapping 140 (minor) — Admin-triggered "dry-run import/commit" patient endpoint scaffold
- **Operation:** POST → resolve practice via `current_practice` (403 "No active practice.") → practice-scoped Patient fetch (404) → admin gate → `dry_run = bool(data.get("dry_run", True))` → call `run_*(patient, practice, dry_run)` → map domain error to 400 `{"detail"}` → `{"dry_run": dry_run, **report}`.
- **Implementations:** `TreatmentPlan/views/dentally_plan_import_views.py:28–62` (docstring: "Mirrors patient_accounts.views.PatientDentallyBackfillView's shape exactly"; inline admin predicate) vs `patient_accounts/views.py:1925–1961` (same shape; `_is_practice_admin` helper; `BackfillError` vs `PlanImportError`).
- **Canonical:** `run_admin_dry_run_action(request, patient_id, runner, error_cls)` helper; both views become delegates. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Mapping 139 — Outbound contact-field encryption gate (`should_encrypt_contact_info` + mask), canonical mixin + retained copies + divergent variant
- **Canonical mixin:** `TreatmentPlan/serializers/mixins.py:57–101` `EncryptContactFieldsMixin` (gate `should_encrypt_contact_info(practice, user)`; masks email/phone/secondary phone); adopted by intake/nurture serializers.
- **Documented retained copy:** `serializers/patient.py:277–305` (byte-identical block + recall overlay; mixin docstring declares it deliberate).
- **Divergent variant (undocumented, load-bearing):** `serializers/treatment_plan.py:1766–1780` — gate called **without `user`** (`should_encrypt_contact_info(instance.practice)` at 1769; missing user → returns True per encryption_utils) → plan-tab nested patient contacts **always masked even for holders of `view_unencrypted_contact_info`** while the same user sees plaintext on the patient tab; also omits `secondary_phone_number`.
- **Cross-app copy:** `messaging/serializers.py:506–533` (both MessageSession serializers; gate with `user`, over participant fields).
- **Canonical path:** extract `encrypt_contact_fields(representation, practice, user, fields=...)` used by flat and nested lanes; restore the `user` argument in the plan lane (likely a bug). **Extraction:** safe after intent confirmation. **Confidence:** high (existence), medium (impact — permission bypass + secondary-phone exposure gap).

### Mapping 138 — Contact-merge record repoint + undo pipeline (divergent table inventories, 3 implementations)
- **Canonical:** `TreatmentPlan/models.py:710–798` `Person.merge` (repoint loops at 728–738 and 748–753 incl. ActivityLog/Activity/NoteHistory; PersonChannel fold 757–778) + inverse `models.py:800–866` `Person.unmerge`.
- **View-lane reimplementation:** `TreatmentPlan/views/contact_merge_views.py:303–326` `_repoint_person_records` — third table inventory adds `messaging.MessageSession` (307, 313) but **omits activity/note tables**; runs before `Person.merge` (463 vs 499), which then re-runs as a no-op and snapshots only its own subset → **direct `Person.merge` callers (dataQuality/views.py:143, dedupe_persons.py:572, dedupe_patients.py:306) strand the loser's inbox sessions** while the UI lane moves them.
- **Raw-SQL undo variant:** `contact_merge_views.py:584–601` — regex-guarded raw cursor reversal over arbitrary table names, plus `Person.unmerge` at 605 — two undo mechanisms sharing one column across two stacked `ContactMergeLog` rows with partial `repointed` snapshots each.
- **Canonical path:** extend `Person.merge`/`unmerge` with MessageSession (or a table registry); views delegate; retire `_repoint_person_records` and the raw-SQL reversal. **Extraction:** safe; needs a behavior decision for direct-merge callers. **Confidence:** high (existence), medium (impact).

### Mapping 137 — One-time-code/OTP minting: RNG source, alphabet, length re-rolled per lane
- **Canonical:** `UserAuthentication/utils.py:503–509` `generate_otp(length=6)` — `secrets.choice` over digits; 6 conforming callers.
- **Bypasses:** `UserAuthentication/service.py:13–14` — same-name shadow using **Mersenne-Twister `random`** (predictable), live via `send_otp_to_mail` → practice_auth_views.py:50,456 (the file's own OTP-01 banner declares utils canonical, then re-rolls); `UserAuthentication/serializers.py:1012` — `random.choices(string.digits)` inline in `create_user` despite importing the canonical at line 22; `TreatmentPlan/utils/codes.py:5–21` — `random.choices` mixed-case alphanumerics minting `SixDigitVerificationCode` for the **public unauthenticated** verification lane (compounds Mapping 100's plaintext-storage flag). Deliberate variant: `magic_login_views.py:37–44` alphanumeric 8-char (documented entropy math).
- **Canonical:** `generate_code(length=6, alphabet=digits)` in shared utils, `secrets`-only. **Extraction:** safe, trivial. **Confidence:** high (existence), medium (impact — public lanes, predictable RNG). (Adjacent to but distinct from Mapping 100 hash-at-rest and Mapping 60 uniqueness-retry.)

### Mapping 136 (minor) — "priority ordering" Case annotation triplet
- `annotate(priority_order=Case(When high=3/medium=2/low=1/else 0)) .order_by("-priority_order", "-created_at")` restated: `TreatmentPlan/views/treatment_plan_views.py:607–624` (docstring itself warns "Single annotation point — avoids duplicate priority_order annotation"), a second intra-file re-roll at `:849–871` (order_by strings restated again at 883, 888), and `nurture_views.py:87–103` (byte-identical body).
- **Risk:** a new priority value (e.g. "urgent") needs 3 edits with silent default-0 divergence. **Canonical:** `annotate_priority_order(queryset)` helper. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Mapping 119 extension (pass 13) — treatment-plan lane has a third last_contacted mechanism set
- Batched: `TreatmentPlan/views/treatment_plan_views.py:627–669` (inline `sms_map` with **bare `phone_number__in`**, no `phone_match_forms` concatenated candidate; email exact); per-row fallback `serializers/treatment_plan.py:1843–1870` `get_last_contacted`. Plans tab shows stale/None for concatenated-form SMS rows while the patient list (canonical) is correct. NonRegisteredPatient subjects require Mapping 119's `(id, phone, country_code, email)` variant.

### Mapping 135 (minor) — DOB-conflict predicate re-rolled in a repair command, bypassing `canonical_dob`
- **Canonical:** `TreatmentPlan/models.py:591–608` `Person._dob_conflict` — coerces via `canonical_dob` (contact_keys.py:98–130; parity-pinned with Go `dobConflict`; docstring records the bare-`!=` string-vs-date over-split defect). **Re-roll:** `management/commands/backfill_persons_households.py:52–54` `_dob_conflict` — bare `a != b`, mirrors the pre-fix version; live at :111 in the Phase-2 Person/Household backfill. (Conforming group-level variant: `dedupe_persons.py:385–400`.)
- **Currently latent** (psycopg returns `date` objects today), but safety rests on the exact per-caller-coercion situation `canonical_dob` exists to remove. **Fix:** one-line import. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Mapping 132 — "age" quick-filter (`age` param → timedelta → `created_at__gte` cutoff), 6 copies
- **Implementations:** identical 6-key `age_mapping` dict restated in `TreatmentPlan/views/intake_views.py:164–186` (173–180), `nurture_views.py:131–151`, `patient_views.py:1062–1080`, `custom_stage_views.py:163–173`, `treatment_plan_views.py:591–604` (precomputed cutoffs variant), `archive_views.py:119–132` (filters **`archived_at`** instead of `created_at`). All silently skip unknown values; same calendar fictions (`1month=30d`, `1year=365d`) restated 6×.
- **Note:** the sibling `created_after/before` block was consolidated into `apply_date_range` (dates.py) but the adjacent `age` block was left. **Canonical:** `apply_age_filter(queryset, params, field="created_at")` beside it. **Extraction:** safe, mechanical. **Confidence:** high (existence), low (impact).

### Mapping 134 — PatientMembership benefit "used/remaining" computation, divergent
- `TreatmentPlan/views/membership_views.py:288–295` (write gate: sums **all** logs incl. negatives; rejects `quantity_used > remaining`) vs `serializers/membership.py:31–42` (display: **positive logs only**, `max(0, ...)` clamp).
- **Risk:** after a negative correction row, the panel shows more remaining than the endpoint accepts. **Canonical:** `PatientMembershipBenefit.quantity_used_total()`/`remaining()` model helpers; negatives-policy decision needed. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Mapping 130 — Phone split-form parsing `(country_code, local_number)`: repair command bypasses canonical `parse_phone_number`
- **Canonical:** `TreatmentPlan/utils/phones.py:6–120` `parse_phone_number` (all live write paths route here). **Reimplementation:** `TreatmentPlan/management/commands/split_phone_country.py:9–140` `split_number` — near byte-identical ladder (docstring examples copied verbatim), tuple return; live callers at 161, 173, 285.
- **Divergence (load-bearing):** for `0`-prefixed/unparseable numbers the canonical strips leading zeros while the command returns the number **verbatim** (`lstrip("+")` fallbacks) — backfill output is not idempotent against live `save()` normalization (models.py:73–78). (Sibling `fix_phone_country_codes.py` correctly uses the canonical; the map's "phones consolidation complete" exclusion missed this holdout.) **Fix:** wrap dict→tuple at the call site. **Extraction:** safe. **Confidence:** high (existence), low-medium (impact).

### Mapping 131 (minor) — Person name-part key normalization re-rolled in repair commands, bypassing `canonical_name_key`
- **Canonical:** `TreatmentPlan/utils/contact_keys.py:31–55` (docstring records the exact `Mary  Jane` vs `Mary Jane` whitespace-split defect). Conforming: `dedupe_persons.py`, `purge_recordless_persons.py`, `reattach_stranded_history.py`.
- **Re-rolls:** `management/commands/backfill_persons_households.py:44–46` `_name_key` and `unweld_persons.py:62–66` `_norm`/`_name_key` — strip+lower only, **no internal whitespace collapse**; the repair tools can group humans differently from `Person.resolve` on the very rows they repair. Adjacent: `backfill_canonical_contact_keys.py:226` restates `canonical_email` inline.
- **Fix:** import `canonical_name_key`/`canonical_name_part`. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Mapping 129 — Immutable PDF artifact generation task scaffold (render → sha256 `document_hash` → `rendered_pdf.save` → status dict)
- **Implementations (4 tasks + 1 inline):** `TreatmentPlan/tasks.py:35–95` (docstring: "Mirrors Notes/tasks.py's generate_letter_artifact"; no status gate, retry ×3, **no idempotency guard** — re-presenting appends artifact rows); `Notes/tasks.py:21–66` (original; finalized-only gate, no retry); `patient_accounts/tasks.py:14–62` (no gate, retry, no idempotency); `Documents/tasks.py:30–117` (divergent: idempotency guard + in-place `update_fields` update). Inline command lane `dentallyIntegration/management/commands/migrate_dentally_clinical.py:434,494,579`.
- **Supersedes the pass-1 Notes exclusion** (that two-file pair) — TreatmentPlan and patient_accounts members are new, both self-describe as mirrors; promote-if-third trigger met.
- **Canonical:** shared `create_pdf_artifact(instance, pdf_bytes, artifact_model, filename, *, gate=None, idempotent=False)`. **Extraction:** safe; Documents' in-place policy needs a flag decision. **Confidence:** high (existence), low (impact).

### Mapping 127 — Patient-guide JSON payload assembly (canonical bypassed by public/token lanes)
- **Canonical:** `TreatmentPlan/guide_payload.py:28–79` `build_guide_payload` ("The one place…"); callers `public_treatment_views.py:391`, `treatment_plan_views.py:1808`.
- **Re-rolls:** `views/public_treatment_views.py:641–704` `get_treatment_plan_public` (full inline re-roll; uses `treatment_plan.patient.practice` at 674 → **500 for non-registered-patient plans**, which the canonical's `resolve_guide_practice` handles); partial variant `:598–640` `get_treatment_plan_by_token` (no enhanced procedures/branding/financing → payload shape differs per entry link).
- **Key-vocabulary defect:** canonical pops `"practitioner"/"practitioner_profile"` (guide_payload.py:66–71) but the serializer emits `"dentist"` (serializers/treatment_plan.py:1692,1702) → **canonical suppression is a no-op, dentist data leaks when show_dentist_bio=False**; inline copies pop `"dentist"` (effective) but never `"practitioner"`.
- **Canonical path:** token/public endpoints call `build_guide_payload`; unify dentist/practitioner keys. **Extraction:** safe; key decision needed. **Confidence:** high (existence), medium (impact — public unauthenticated lanes).

### Mapping 128 — Treatment-plan final price (base − discount% − amount, floored at 0), three divergent lanes
- `models.py:3906–3916` `final_price` (float; base = `final_price_set`); `views/treatment_plan_views.py:1304–1334` `present_to_patient` (Decimal; base = **real line sum** — comment says `final_price_set` is "frequently never set"; feeds the PDF snapshot); `serializers/journey.py:124–175` (returns `final_price_set` **verbatim if set, ignores both discounts** at 146–175).
- **Risk:** journey board and payment schedule/email show different totals than the presented PDF; Decimal-vs-float fork. Dead copy excluded: `services/treatment_plan.py:171–192` (zero callers). **Canonical:** `final_price(sum_items=True)` parameterization or a pricing service. **Extraction:** safe; base-source decision needed. **Confidence:** high (existence), medium (impact — patient-facing money figures disagree).

### Mapping extensions (pass 7)
- **Mapping 42 ext:** third send-time implementation `TreatmentPlan/journey/automation.py:78–142` — docstring documents the anchor divergence as deliberate ("must never share one function"); only `_as_time` mechanically foldable. Record as documented variant.
- **Mapping 100 ext:** fourth one-time-code lane — `SixDigitVerificationCode` (`views/verification_views.py:72–98` issue; `public_treatment_views.py:368,424,489` verify) stores the code **plaintext** (models.py:4535+) and is the credential for a public unauthenticated write endpoint — the weakest member; Mapping 100's `issue_hashed_code` should absorb it.

### Mapping 10 extension (TreatmentPlan pass 6) — record-scope resolution, 4th and 5th files
- `TreatmentPlan/views/household_views.py:55–78` and `:134–148` (same param-priority loop twice intra-file; adds `person_id` as a record kind); `contact_merge_views.py:42–53` `_get_record` (string-keyed `record_type` variant, returns None). All feed Mapping 10's planned `record_for_scope_ids(practice, params)` sibling in selectors.

### Mapping 126 — Submitted-order list → `sort_order` persistence ("drag reorder endpoint"), 4 implementations
- **Implementations:** `TreatmentPlan/views/journey_stage_views.py:267–307` (strict set-equality, archive-pin, `transaction.atomic()` + `select_for_update()` at 307); `practice_treatment_views.py:195–225` and `:388–420` (intra-file pair; set-equality + dedupe; `update(sort_order=index, revision=F()+1)` at 221/413; no lock); `Stock/views/stock_category_views.py:76–100` (**lenient: silently skips unknown ids** at 92, `bulk_update`, no completeness/dedupe/lock).
- **Risk:** same malformed payload is 400 in TreatmentPlan but silently partially-applied in Stock; concurrent reorders serialized in only one lane. **Canonical:** `apply_submitted_order(queryset, ids, *, strict=True, lock=False)`. **Extraction:** safe; strictness decision needed. **Confidence:** medium-high (existence), low-medium (impact).

### Mapping 124 — Per-app immutable audit-log emission module (`diff_fields` + `_actor_name` + `emit_*_audit`), 5 apps
- **Operation:** write immutable practice-scoped audit event (`{key: {before, after}}` diff + actor snapshot + best-effort persist) into per-app `*AuditLog` models.
- **Implementations:** `HR/hr_audit.py:20–44` (original; ~80 live call sites); `TreatmentPlan/practice_treatment_audit.py:18–45` (docstring admits "Mirrors HR/hr_audit.py"; ignore-set adds `revision`); `Tasks/task_audit.py:8–36` (same `_actor_name` byte-identical); `compliance/compliance_audit.py:13–60` (+`fail_silently` fail-closed flag); `Invoices/finance_audit.py:14–47` (**raises on failure**). Fifth/sixth `diff_fields` re-rolls outside the modules: `Invoices/views/finance_config_views.py:48`; `HR/views/staff_profile_views.py:157` (no ignore-set — `updated_at` churn included).
- **Divergence:** ignore-key sets, failure policy forks 4 ways, `_actor_name` bypasses `full_name` in 4 of 5 (double-space on nullable names). **Canonical:** generic `emit_audit(log_model, ..., fail_mode="warn")` in shared utils. **Extraction:** safe; failure-policy decision needed. **Confidence:** high (existence), low-medium (impact).

### Mapping 125 — Pathway-branded PDF scaffold (`#6941c6` header/watermark/counter CSS), TreatmentPlan ↔ patient_accounts twins
- `TreatmentPlan/utils/presentation_pdf.py:13–153` (caller tasks.py:63) vs `patient_accounts/invoice_pdf.py:13–140` (docstring: "same approach … deliberately not sharing"). Identical `@page` counter, `.header` logo cell, watermark, `.totals`, `_render_html` skeleton byte-identical; only margins/h2/row-building differ. **Neither passes a `url_fetcher`** (data-URI images, but a remote `<img>` would still be fetched). Distinct from Mapping 51's `#846ce0` Stock/dentally scaffold; both share `TreatmentPath/pdf_branding.py` helpers correctly. **Canonical:** `render_branded_pdf(title, meta, sections, practice)` beside `pdf_branding.py`. **Extraction:** safe. **Confidence:** high (existence), low (impact).

### Mapping 122 extension (pass 5)
- Seventh `treatments`-normalization variant: `TreatmentPlan/serializers/journey.py:9–31` `_journey_parse_treatments` — a **superset** (also unwraps JSON-encoded and double-encoded `"["...` items); feeds the journey board serializers, so legacy CSV renders differently there than on the intake tab. Fold into Mapping 122's canonical.

### Mapping 120 — Family-household panel: `is_family` + perspective-aware `family_members` (canonical + two re-rolls + divergent variant)
- **Canonical (batched):** `TreatmentPlan/selectors.py:4–170` `build_family_maps`; consumed by intake/nurture/custom_stage views. **Per-row fallback twins:** `serializers/intake.py:331–467` and `serializers/nurture.py:312–447` (~135 lines, near byte-identical; practice-scoped email/phone Q with `_hhm__gt=1`). **Divergent variant:** `serializers/patient.py:513–560` — **no email/phone cross-identity fallback and no `practice=` filter** (`Patient.objects.filter(person__household_id=...)` at ~538) despite the lead lanes' own audit-#6 comment ("6,905 addresses exist in more than one practice").
- **Canonical path:** `build_family_maps` + `family_members_payload(viewer, family_contact, practice)`; patient lane delegates and picks up the practice filter. **Extraction:** safe. **Confidence:** high (existence), low-medium (impact).

### Mapping 121 (minor) — Dentally patient-UUID resolution ladder in lead serializers
- `TreatmentPlan/serializers/intake.py:295–330` and `nurture.py:283–310` — byte-identical ~35-line bodies (own patients' `meta_data["uuid"]`, else practice-scoped `email__iexact`/phone fallback, `.first()`). **Canonical:** `dentally_uuid_for_lead(record, context)` beside `journey_touchpoints.patient_for_dentally_id` (Mapping 22's inverse). **Extraction:** safe, mechanical. **Confidence:** high (existence), low (impact).

### Mapping 122 (minor) — `treatments` legacy CSV↔list normalization restated per file
- `serializers/intake.py:480–500` (raises ValidationError); `journey/mixins.py:260–291` (`_transform_treatments_to_array/_to_string`; returns `[]` for garbage); `views/intake_views.py:1443–1451` + `nurture_views.py:580–586` (silent drop); inline guards at `conversion_views.py:294,658`, `models.py:2492,3469,3575`. Asymmetric `", ".join` vs `split(",")` round-trip. **Canonical:** `normalize_treatments(value, *, strict=False)` / `treatments_to_csv(value)` in TreatmentPlan/utils. **Extraction:** safe. **Confidence:** high (existence), low (impact).

### Mapping 123 (minor) — `get_call_session`: first call-log id via Person, three unordered copies
- `serializers/intake.py:206–211`, `nurture.py:192–197`, `patient.py:426–431` — byte-identical `logs[0].id if logs else None`; **no ordering** (DB-order-arbitrary), per-row N+1; the sibling session fields were consolidated onto `message_sessions_for_person` (`contact/relationships.py:70`) but call-session was left behind; the `person__person_channels__channel__call_logs` prefetch (nurture_views.py:115,298) doesn't cover `person.call_logs`. **Canonical:** `first_call_log_id(person)` in `contact/relationships.py` with defined ordering. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### TreatmentPlan pass-4 exclusions (for the record)
- JourneyStage practice+slug resolution restated 4× with deliberately different failure contracts — same class as patient-search-ladders exclusion.
- `stage_label` fallback rendering pair intra-file intake_views + divergent patient_views variant — below bar.
- `get_patient_id` email→phone ladders (intake/nurture serializers) — documented "#73 never arbitrary" intentional variant of Mapping 57/77.

### Mapping 118 — Archive row creation: person_id lineage stamp written by only one of two writer families
- **Stamped writer:** `TreatmentPlan/journey/mixins.py:130–167` `_create_archive_record` (stamps `original_data["person_id"]` via `carried_person()`; 6 in-file callers).
- **Unstamped writers (no person_id in original_data):** `models.py:2408–2476` `Intake.close_intake` (create at 2463), `:2619`, `:3512`, `:3525–3593` `Nurture.close_nurture` (3579), `:2071–2100` `Intake.build_archive_snapshot` (2099), `:4047–4160` `TreatmentPlan.build_archive_snapshot` (4155). **Load-bearing consumer:** `Archive._stamped_person` (4929–4939) returns None when absent → `restore_from_archive` falls back to name-based re-derivation (the #47 second-Person-minted failure). Live callers in intake/nurture/move/treatment_plan views.
- **Canonical:** `Archive.record_for(record, *, journey_stage, ..., extra_original)` stamping person_id once. **Extraction:** safe (additive key; `_stamped_person` tolerates absence). **Confidence:** high (existence), medium (impact).

### Mapping 119 — "last_contacted" batched map builder: widened phone match never propagated to lead lanes
- **Canonical:** `TreatmentPlan/contact/last_contacted.py:29–45, 76–110` — `phone_match_forms` matches bare AND `country_code+phone_number` concatenated forms (docstring documents both occur in production; narrower match was a real data bug); `serializers/patient.py:587–604` delegates correctly.
- **Re-rolls (bare phone_number match only):** byte-identical trio `views/intake_views.py:237–278`, `nurture_views.py:192–232`, `custom_stage_views.py:201–242`; per-row fallback twins `serializers/intake.py:502–537` and `nurture.py:452–488` `get_last_contacted`.
- **Risk:** for concatenated-form SMS rows, Patient list shows real last-contacted while Intake/Nurture/Custom-stage tabs show stale/None. **Canonical:** generalize `build_last_contacted_map` with a `(id, phone, email)` variant. **Extraction:** safe, mechanical. **Confidence:** high (existence), low-medium (impact).

### TreatmentPlan pass-3 exclusions (for the record)
- `services/dentally_plan_import.py` ↔ `dentally_treatment_plan_backfill.py` `_check_item_sync_completeness` — only exception class differs; below bar, promote if a third checkpoint consumer appears.
- `views/move_views.py` three move handlers — intra-file, deliberate D-01 shape.

### Mapping 113 — Webhook secret mint + hash-at-rest (token_utils bypassed by IntakeWebhook lanes)
- **Canonical:** `TreatmentPlan/token_utils.py:38,57` `generate_secret()` (docstring: exists "so that both the views layer AND the serializers layer can import it"). Conforming: `serializers/webhook.py:345,392`; `views/custom_webhook_views.py:159–162` (rotation with grace).
- **Re-rolls:** `serializers/webhook.py:119–129, 154–165` (`secrets.token_hex(32)` + local `make_password`); `views/intake_views.py:1381–1407` `regenerate_secret` (inline mint; **overwrites hash with no grace window** — in-flight retries after rotate 401 permanently, unlike CustomJourneyWebhook's `previous_secret_hash` ladder at custom_webhook_views.py:332–339).
- **Canonical path:** `mint_webhook_secret(webhook)` helper. **Extraction:** safe. **Confidence:** high (existence), low-medium (impact).

### Mapping 114 — Outstanding-items per-patient vs batched twin (payload builders)
- `TreatmentPlan/views/patient_outstanding_views.py:73–163` vs `TreatmentPlan/outstanding_summary.py:88–306` `build_outstanding_items_map` — item dicts byte-identical (journeys/tasks/labs); documented deliberate batched mirror (Mapping 55 shape). **Batched-only fuzzy "Open Plans" pass (191–285)** means endpoint and Day-List payload can disagree. **Canonical:** `outstanding_items_for_person_ids(practice_id, person_ids)`; endpoint calls with single-id list. **Extraction:** safe. **Confidence:** high (existence), low (impact).

### Mapping 115 — NonRegisteredPatient email/phone fuzzy match (loop vs batched index)
- `TreatmentPlan/views/patient_journey_views.py:121–170` `_get_matching_non_registered_patient_ids` (O(patients×NRP) re-scan) vs `outstanding_summary.py:41–85` `_build_non_registered_patient_index` (docstring: "Mirrors … same matching semantics, one query"). **Canonical:** batched index extracted to selectors. **Extraction:** safe. **Confidence:** high (existence), low (impact).

### Mapping 116 — OpenRouter strict-schema extraction scaffold duplicated in TreatmentPlan services (Mapping 93-family ext)
- `TreatmentPlan/services/email_intake.py:53–317` `parse_intake_email_body` vs `services/email_task.py:64–242` `parse_task_email_body` — each re-rolls `OpenAI(api_key=openrouter_api_key(), base_url=openrouter_base_url())` + strict `json_schema` + `parsing_success` error-dict + JSON/except blocks; differ only in schema/prompt. **Canonical:** `complete_structured(schema, system, user)` on `TreatmentPath.llm_provider`. **Extraction:** safe. **Confidence:** high (existence), low (impact).

### Mapping 117 — Family-household relink by shared email/phone (doc-deferred, promoted)
- `TreatmentPlan/views/conversion_views.py:33–76` `_link_patient_to_family_household`/`_ensure_person_on_family_household` (moves **Patient's Person**; iexact email + phone match) vs `intake_views.py:355–379` `_sync_intake_to_family_contact` (re-points **Intake's Person**; exact phone match).
- **Canonical:** `relink_record_to_family_household(record, practice)` in `TreatmentPlan/contact/`; design decision on whether Intake records force Person moves. **Extraction:** safe after decision. **Confidence:** high (existence), low-medium (impact).

### Mapping 96 extension (pass 2) — sixth private-S3 storage class
- `TreatmentPlan/storage.py:16–21` `PrivateTreatmentPlanDocumentStorage` (docstring admits mirroring Notes/Documents); fold into Mapping 96's `private_document_storage()` factory.

### Mapping 108 — Record note-edit pipeline (NoteHistory backfill + sync_from_quick_note + mentions), 4 implementations
- **Operation:** on notes change: backfill old note if no history, write new `NoteHistory`, upsert shared Internal-Notes row via `Note.sync_from_quick_note`, resolve mentions → `omitted_mentions`.
- **Implementations:** `TreatmentPlan/serializers/intake.py:700–760`; `nurture.py:552–615` (near byte-identical); `treatment_plan.py:1230–1294` (adds normalized-notes diff); `TreatmentPlan/views/note_history_views.py:260–299` `add_contact_note` (view-lane re-roll; comment "mirrors PatientViewSet.add_note"; `-id` ladder for person resolution — adjacent to Mapping 77's convention fork).
- **Canonical:** `apply_quick_note_edit(record, content, user, mentioned_user_ids)` helper. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 109 — Client IP extraction copies in TreatmentPlan webhook lanes (Mapping 6 extension)
- `TreatmentPlan/views/custom_webhook_views.py:283–290` `_get_client_ip` (docstring: "clone of intake_views.py") vs `intake_views.py:1040–1047` `get_client_ip` — byte-identical bodies; both bypass Mapping 6's canonical `utils/request_meta.client_ip`. **Fix:** import the helper. **Extraction:** safe, trivial. **Confidence:** high.

### Mapping 110 — "Unique treatments across active journey records" endpoint, SQL vs ORM twins
- `TreatmentPlan/views/intake_views.py:1417–1464` (Python flatten of JSONField/CSV) vs `nurture_views.py:536–599` (raw-SQL `jsonb_array_elements_text` at 561 + DISTINCT + trim); byte-identical trailing choice-building block (1457–1462 ↔ 589–594).
- **Divergence:** nurture trims in SQL (whitespace dupes collapse), intake doesn't; intake handles legacy CSV shape. **Canonical:** `unique_treatment_values(model, practice)` helper. **Extraction:** safe. **Confidence:** high (existence), low (impact).

### Mapping 111 (minor) — Honorific vocabulary + leading-honorific name split, two sets in one app
- `TreatmentPlan/serializers/intake.py:21–32` `HONORIFICS` (9 entries, no `mx`; drop-token forward split at 577–587) vs `TreatmentPlan/utils/intake_fallback.py:81–92` `_FALLBACK_TITLES` (10 entries, has `mx`, no `professor`; **reversed spill order** at 279–297).
- **Canonical:** one `HONORIFICS` + `split_full_name(raw)` in TreatmentPlan/utils. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Mapping 112 (minor) — Intake practice resolution: authenticated-user vs public-slug trio
- `TreatmentPlan/views/intake_views.py:770–786` (slug-only, never tries auth lane); `serializers/intake.py:665–690` (auth-first, slug fallback, duplicated invalid-slug error string); `serializers/intake.py:640–653` `_practice_for_normalization` (auth-first but `filter(slug=...).first()` → **silent None** instead of raising).
- **Canonical:** `resolve_intake_practice(request, practice_slug, *, required=True)`. **Extraction:** safe. **Confidence:** medium-high (existence), low-medium (impact).

### TreatmentPlan borderline exclusions (pass 1)
- Global→practice library clones: `treatment_plan_views.py:398–460` `clone_global_procedure` (idempotent get_or_create) vs `review_asset_views.py:197–239` `clone_asset` (plain create + reference id) — same shape as Mapping 90, only 2 sites with deliberate policy difference; promote if a third appears.
- Patient search ladders (treatment_plan_views ×3 intra-file, patient_views, intake_views) — intentionally different tiering per endpoint; generic idiom.

### Mapping 106 — Booking slot conflict predicate (buffered time-overlap against live appointments)
- **Operation:** `filter(clinician/room=…, start_time__lt=buffered_end, end_time__gt=buffered_start, status__in=[active]).exists()`.
- **Implementations (4 sites, 3 files):** `onlineBooking/services.py:96–114` `_slot_has_conflict` (canonical-most: checks appointments **plus** online holds; buffers from service config); `Appointments/public_booking_serializers.py:216–229` (identical Q; **no hold check**; buffers from ClinicianBookingConfig); `Appointments/public_booking_views.py:258–272` (copy; **no hold check**); `Appointments/serializers.py:374–390` (room variant, no buffers). Sub-twin `_is_during_break`: `onlineBooking/services.py:81–93` vs `Appointments/public_booking_views.py:285–305`.
- **Risk:** online holds invisible to Appointments public lanes → double-booking window between systems; buffer source differs per lane. **Canonical:** `Appointment.has_conflict(practice, clinician, room, start, end, buffer_before, buffer_after)` with onlineBooking adding the holds term. **Extraction:** safe after buffer-source decision. **Confidence:** high (existence), medium (impact).

### Mapping 107 — Per-practice Stripe Checkout Session creation (account resolution + key swap + minor-units conversion)
- **Implementations:** `onlineBooking/services.py:489–542` `create_checkout_for_hold` (Decimal quantize to minor units; payment-intent metadata; ValidationError; **global `stripe.api_key` swap**); `patient_accounts/views.py:898–955` `PatientStripeCheckoutView` (**float `* 100` conversion — off-by-one pence risk**; no metadata; 502 JSON). Adjacent platform-key variants: `payments/services.py:3127` (subscription), `Admin/views/subscription_views.py:1336` (setup).
- **Canonical:** `create_practice_checkout_session(practice, ...)` beside Mapping 97/98's Stripe helpers. **Extraction:** safe (onlineBooking lane most correct). **Confidence:** high (existence), low-medium (impact).

### Mapping 104 — Practitioner display-name with email fallback (in-area bypass family; promotion trigger met)
- **Operation:** render practitioner `User` as display name, falling back to `email`.
- **Implementations:** `onlineBooking/serializers.py:421–424` `get_name` (raw f-string join or email); `onlineBooking/views.py:486–488` (same, `or ''` guards); `serializers.py:291–294` (in Mapping 102's payload); `onlineBooking/tasks.py:92` (third mechanism — `get_full_name() or email`, no strip). The app's own `views.py:683–691` was converted to canonical `TreatmentPlan.utils.names.full_name` ("Round-three G12") because nullable first/last rendered "None Smith".
- **Canonical:** `practitioner_display_name(user)` wrapper over `full_name` + email fallback; all four delegate. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Mapping 105 (minor) — Hold/payment expiry transition: sweep vs lazy, active-status vocabulary restated
- `onlineBooking/tasks.py:21–24` `expire_stale_holds` (beat-scheduled) restates active statuses as **raw string literals** instead of `OnlineBookingHold.ACTIVE_STATUSES` (models.py:251); `tasks.py:31–34` payment sweep; lazy lanes `services.py:417–420, 485–488, 749–761` use enums + update_fields; "payment expired because hold expired" rule exists in both sweeps and webhook lane with different scopes.
- **Canonical:** `OnlineBookingHold.expire_stale(qs)` classmethod + `hold.expire()` instance method. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### onlineBooking pass-5 bug aside (not duplication)
- `onlineBooking/tasks.py:53–77` `sync_stripe_connect_accounts` (beat-scheduled) writes `charges_enabled`/`payouts_enabled`/`requirements_*`/`disabled_reason` — **none of these fields exist on `PracticeStripeAccount`** (models.py:174–235); every account hits the except and is silently counted failed; the task is a silent no-op.

### Mapping 101 — Online-booking service practitioner-rules sync ("replace rules from submitted list")
- **Implementations:** `onlineBooking/serializers.py:216–263` `_sync_practitioners` (membership via `current_practice=practice` at 221 — the property the DentallyServiceMixin docstring warns is not interchangeable with relationship rows; **skips existing rules, so visibility/new-patient flags can never be edited via this lane**); `onlineBooking/views.py:234–273` `OnlineBookingServicePractitionerView.patch` (membership via active `UserPracticeRelationship` at 248; `update_or_create` at 262 updates flags).
- **Risk:** a clinician whose profile practice ≠ `current_practice` is valid in one lane, dropped in the other; flag edits are no-ops through the serializer lane. **Canonical:** `sync_service_practitioners(service, items, *, membership)` with the relationship-row predicate. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 102 — Public "bookable practitioners for this service" list (parallel rendering)
- `onlineBooking/serializers.py:284–297` `get_practitioners` (`{id, name}` only; inline name join bypassing `full_name` at 292–294) vs `onlineBooking/views.py:462–495` `OnlineBookingPublicServicePractitionerListView` (same query; adds avatar/flags/sort_order). Same page, two inconsistent practitioner cards. **Canonical:** one `public_practitioners(service, request, *, fields)` builder. **Extraction:** safe. **Confidence:** high (existence), low-medium (impact).

### Mapping 103 (minor) — `OnlineBookingProfile` default derivation stated twice
- `onlineBooking/models.py:38–51` `save()` auto-fill (authoritative) vs `serializers.py:241–249` hand-re-rolled `get_or_create` defaults (views.py:97–107 correctly rely on `save()`). **Fix:** `OnlineBookingProfile.get_or_create_for_practice(practice)` classmethod. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### onlineBooking pass-3 exclusion
- Device-trust token mint blocks `views.py:608–626` vs `756–774` — intra-file.

### Mapping 99 — Practitioner day-schedule resolution: onlineBooking re-implements `Appointments/schedule_resolver.py`
- **Canonical:** `Appointments/schedule_resolver.py:1–23` — declared "single source of truth"; all HR/Appointments consumers delegate (`HR/working_days.py:19–29` wrapper; rota/leave/accrual/forecast callers).
- **Reimplementation:** `onlineBooking/services.py` — `_adhoc_pattern_matches_date` (130–162) re-rolls `_matches_adhoc_pattern` (142–177); `generate_availability` (210–253) re-rolls the priority ladder inline.
- **Load-bearing divergences:** (1) biweekly matching `delta % (interval_weeks*7) == 0` (147–149) vs resolver's `round(days/7)` week counting (169–170) — diverge for non-multiple-of-7 offsets, so a biweekly day can be bookable in the diary but produce zero online slots; (2) monthly_ordinal "nth vs last weekday" rule diverges (158–160 vs resolver 110–131 lockstep rule); (3) rotating `PractitionerSchedulePattern` tier missing entirely → rotating-cycle practitioners get no online slots; (4) per-date queries vs batched context.
- **Fix:** onlineBooking imports `build_schedule_context`/`resolve_day_schedule`. **Extraction:** safe; online lane is the buggier one. **Confidence:** high (existence), medium (impact).

### Mapping 100 — Hashed one-time-code issue/verify triplet with divergent hash schemes
- **Implementations:** `UserAuthentication/utils.py:556–594` `store_hashed_otp`/`verify_user_otp` (**bcrypt**; docstring at models.py:736–743 bans naive storage); `UserAuthentication/models.py:828–842` `MagicLogin` pin (bcrypt, own lockout policy); `onlineBooking/models.py:449–491` `BookingEmailOTP.issue/verify` — **unsalted SHA-256** (468, 490) guarding an AllowAny public endpoint returning patient PII prefill.
- **Divergence:** hash strength (6-digit code space brute-forceable from unsalted hash), expiry (10m/24h/15m), consume-on-verify, lockout. **Canonical:** shared `issue_hashed_code(...)`; at minimum BookingEmailOTP switches to bcrypt. **Extraction:** mechanically safe; hash-scheme alignment is a policy decision. **Confidence:** medium.

### onlineBooking exclusions (pass 2)
- Mapping 58 ext: `onlineBooking/services.py:191`, `views.py:363` also use the "Europe/London" timezone fallback.
- Full-name `split(" ", 1)` storage splits (services.py:595–597, tasks.py:115–117) — generic idiom class (finding #20 scope).

### Mapping 97 — Stripe webhook receive scaffold (signature verify + idempotent event record + type dispatch)
- **Operation:** CSRF-exempt AllowAny POST: read body + `HTTP_STRIPE_SIGNATURE`, `stripe.Webhook.construct_event(...)` (ValueError→400, SignatureVerificationError→400), dedup via `get_or_create(stripe_event_id)`, type dispatch, stamp `processed`.
- **Implementations (3 live routes):** `onlineBooking/views.py:858–951` (per-practice secret via webhook_token→PracticeStripeAccount; dedup `(stripe_event_id, connected_account_id)`; tolerance=600); `payments/views.py:551–660` (global secret; `WebhookEvent` dedup; ~10 branches; the OPTIONS preflight block is a verbatim copy); `patient_accounts/views.py:1729–1764` (**weakest: no payload/sig guards, no tolerance, merged except, NO dedup table** — replay idempotency rests on `checkout.status`).
- **Canonical:** shared `verify_stripe_event(request, secret)` + `EventIngestRecord` model; onlineBooking's lane most complete. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 98 — Stripe payment → ledger provisioning block (ledger entry + payment + balance update)
- **Operation:** on confirmed payment: `compute_balance_before` → PAYMENT ledger entry → `PatientPayment` (unallocated) → `recalculate_from` → update `PatientAccount.current_balance`/`unallocated_funds`.
- **Implementations (near byte-identical ~45 lines):** `onlineBooking/services.py:660–705` (hardened by audit #20/#33; idempotency via `stripe_payment_intent_id`; broad try/except + OnlineBookingException record) vs `patient_accounts/views.py:1759–1808` (idempotency via `checkout.status`; reference = checkout id; exceptions propagate to 500 after webhook returned).
- **Canonical:** shared service in `patient_accounts` with caller-supplied idempotency policy. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### onlineBooking exclusion (pass 1)
- `stripe.api_key = ...` global mutation (services.py:40,521; tasks.py:41 + payments app 10+ sites) — SDK global idiom; noted because it makes the per-practice key swap in Mapping 97 non-thread-safe.

### Mapping 95 — Manual label attach from category ids ("clear + recreate NoteLabel rows", service bypassed)
- **Canonical (bypassed):** `Notes/services/labels.py:240–300` `apply_labels_to_note` — handles both Note and NotesLetter via isinstance, offers `preserve_existing`. **Reimplementations:** `Notes/serializers/note.py:93–111` `_process_labels` (M2M `.clear()`, silent skip); `Notes/serializers/letter.py:251–266` (queryset `.delete()`, silent skip); `Notes/views/labels.py:178–206` `perform_create` (third variant; raises NotFound/ValidationError instead of skipping).
- **Divergence:** clear mechanism, error policy (silent skip vs raise), `source` field only in viewset. **Fix:** serializers delegate to the service. **Extraction:** safe. **Confidence:** high (existence), low-medium (impact).

### Mapping 96 — Private-S3 storage subclass + `USE_SPACES` resolver, re-rolled per app
- **Implementations (5 classes, 4 files):** `Notes/storage.py:19–41` (expire 300; docstring admits mirroring HR/Documents); `HR/storage.py:20–35` (**expire 3600**); `Documents/storage.py:36,57,71–105` (two classes, 300); `compliance/storage.py:~20–60` (300). Same `default_acl="private"`, `custom_domain=None`, `querystring_auth=True` shape.
- **Risk:** the same "presigned clinical URL lifetime" policy decided per app; expiry changes need 5 edits. **Canonical:** `private_document_storage(querystring_expire=300)` factory; per-app aliases. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Out-of-area note for later passes (pass 4)
- Serializer `get_file_url`/`get_image_url` builder (`file.url`; `build_absolute_uri` if not http) — byte-identical at ≥11 sites: `Admin/serializers.py:395–407`, `compliance/serializers.py:285,1645,3166,3386`, `TreatmentPlan/serializers/mixins.py:36–47`, `treatment_plan.py:190–216 (×2)`, `settings/serializers.py:346`, `messaging/serializers.py:954–966, 1317`. Treat as one cross-area family in the TreatmentPlan/compliance passes.

### Out-of-area notes for later passes (Notes pass 6)
- Upload-validator mirror family: `HR/upload_validators.py` documented mirror of `compliance/upload_validators.py` (`_check_size`/`_check_extension`/`_check_magic_bytes`/`_MAGIC_SIGNATURES` near byte-identical); third partial variant `Labs/upload_validators.py:23` + `Labs/serializers.py:160–183`. Canonical: `utils/` factory parameterized by allowlists/exception class.
- Raw-`requests` chat-completions forward lane (Mapping 93 extension candidate): `Notes/services/ai_proxy.py:149–216`, `ChromeExtension/views.py:51–100`, `Stock/services/stock_extractor.py:1075–1110`, `Stock/services/scrape_pipeline.py:1160`, `Invoices/services/ai_extractor.py:263,344,538` — same `requests.post` OpenAI-shaped payload + Bearer guard, divergent error contracts.

### Mapping 93 — Notes "chat completion with graceful-error dict" scaffold + forked LLM provider config
- **Operation:** `client.chat.completions.create(model="gpt-4o-mini", ...)` + key guard + `{"success": bool, "error"}` dict; 7 copies differing only in prompt/temperature/max_tokens/response-key.
- **Implementations:** `Notes/services/ai.py:330–378, 381–432, 435–554, 557–689, 692–757`; `Notes/services/letters.py:1199–1290` — all resolve key via `settings.OPENAI_API_KEY` + default `OpenAI()` client. **Load-bearing fork:** the same app's newer lanes (`Notes/services/consent.py:430,551`; `Notes/services/ai_proxy.py:39–47`) use `TreatmentPath/llm_provider.py` (OpenRouter) — whose docstring declares Notes should have "exactly one configuration". With only `OPENROUTER_API_KEY` configured, all six functions permanently fail.
- **Canonical:** `complete_text(system_prompt, user_text, *, temperature, max_tokens)` on `TreatmentPath.llm_provider`. **Extraction:** mechanically safe; provider migration is a behavioral decision. **Confidence:** high (existence), medium (impact).

### Mapping 94 — "Generate template content from note via AI" + name-collision persist policy
- `Notes/views/ai.py:237–340` `create_template_from_note` (collision → 400 reject, 308) vs `Notes/views/note.py:859–990` `create_letter_from_note` (collision → **silently overwrites existing template's content_template**, returns 200 "updated", no confirmation/versioning).
- **Canonical:** `create_template_from_content(user, kind, name, content, *, on_collision)`; the 400-reject policy is likely intended. **Extraction:** safe after collision-policy decision. **Confidence:** high (existence), medium (impact).

### Notes exclusions (pass 3)
- Audio upload validation 3× intra-file in `views/transcription.py` — below bar; promote if another app copies it.
- `Notes/sentiment_analyzer.py` — zero callers (dead); heuristics re-implemented live in `services/consent.py:513–537, 399–421` — dead code + single live implementation.

### Mapping 90 — Global → personal/practice template clone lane (5 copies, 3 files, divergent policies)
- **Operation:** duplicate a global/superuser template into the requester's collection; collision policy + provenance.
- **Implementations:** `Notes/views/template.py:356–422` `clone_template` (collision→400; sets `global_template_reference_id` at 389); `template.py:557–602` `active_templates` auto-clone loop (**no reference-id** → auto-clones never count toward the clone-usage ranking subquery at template.py:195–204; silent-skip collision); `template.py:694–758` third full copy — **unreachable dead code** after return at 662; `Notes/views/letters.py:1509–1640` `clone_letter_template` (reference-id + image cloning); `Notes/views/auto_template.py:202–252` (deliberate idempotent `get_or_create` on `built_in_slug`); `Notes/views/practice_template.py:252–279, 350–373` `copy_to_mine` ×2 (rename-with-suffix collision policy, no provenance).
- **Canonical:** `clone_global_template(source, owner, *, on_collision, provenance_field)` beside the import services. **Extraction:** safe; collision policy decision needed. **Confidence:** high (existence), low-medium (impact — reference-id gap corrupts usage ranking).

### Mapping 91 — `consent_templates.json` → ConsentTemplate/ConsentPattern importer, two divergent live implementations
- `Admin/views/consent_management_views.py:26–118` (routed Admin/urls.py:762): global rows, upsert, **deletes+recreates all patterns** (85), `patterns` key only, manual cache delete. `Notes/management/commands/migrate_consent_templates.py:33–189`: per-practice, skips existing (87–90), **derives patterns from score_extraction/status_mapping/wear_types** (105–114) + `extraction_fields` (117–123), transactional, no manual cache delete (signals cover it).
- **Risk:** same JSON seeds two different row topologies and pattern sets. **Canonical:** one importer parameterized by `practice` + pattern-expansion policy. **Extraction:** safe. **Confidence:** high (existence), low-medium (impact).

### Mapping 92 — "Ranked templates" Note/Letter twins with divergent "global" semantics (privacy-relevant)
- Note lane: `Notes/views/template.py:273–353` → `Notes/models/template.py:123–182` (global = superuser-gated + deduped-to-latest). Letter lane: `Notes/views/letters.py:1464–1506` → `Notes/models/letter.py:79–99` — global = **`exclude(user=user)`: every other user's active letter template, no superuser gate, no dedupe**, despite docstring "marked as global" and sibling `global_letter_templates` (letters.py:1454) correctly filtering `user__user_type="superuser"`.
- **Risk:** cross-user template content exposure + ranking pollution via a live endpoint. **Fix:** letter lane uses superuser+latest global queryset. **Extraction:** safe after intent decision (looks like a bug). **Confidence:** high (existence), medium (impact).

### Notes pass-2 extension/dead-code notes
- **Mapping 14 ext:** `Notes/serializers/practice_template.py:49–52, 94–97` `get_shared_by_name` hand-rolled name join bypassing `get_display_name_for_user` (the app's own serializers forbid it; superuser-sharer renders a real name).
- Dead code: `Notes/views/template.py:663–802` unreachable second copy of `active_templates`.

### Mapping 88 — Global phrase "copy into practice" (canonical bypassed by REST clones, case-sensitivity fork)
- **Canonical:** `Notes/services/import_phrases.py:33–53` `_import_category_set` / `:56–84` `_import_phrase_set` — `get_or_create` keyed on **exact** `name`; callers `Notes/signals.py:50–75` + `phrases/import-defaults/`. **Clones:** `Notes/views/phrase.py:218–256` `clone_note_phrase`, `:329+` `clone_letter_phrase` (docstring admits mirroring) — resolve category with **`name__iexact`** (`phrase.py:235, 346`) and no None guard on the global category.
- **Risk:** "Hygiene" vs "hygiene" collapses in the clone lane but forks in the bulk-import lane. **Canonical path:** single-row `import_phrase(global_phrase, practice)` variant after an exact-vs-iexact decision. **Extraction:** safe after that decision. **Confidence:** high (existence), low (impact).

### Mapping 89 — Draft save/list/update/retrieve scaffolding, Note vs NotesLetter twins
- **Operation:** `save_to_draft` (force is_draft), `list_drafts`, find-in-either-state `update`/`destroy`, draft→final conversion block (`is_draft=False` + `on_record_created`).
- **Implementations:** `Notes/views/note.py:646–663, 665–708, 710–778, 744–756` vs `Notes/views/letters.py:424–557, 559–587, 589–672, 636–648`.
- **Divergences (load-bearing):** Note's `list_drafts` lists user-OR-practice drafts; Letters' lists **only user drafts** despite its comment claiming "within their practice context" (latent scoping bug); Note's `update` lacks the patient-practice gate the letters lane carries (#48 fix). **Canonical:** `DraftTransitionMixin` / `drafts_for(user, practice)` selector in Notes, letters' scoping as the security-correct shape. **Extraction:** safe for the conversion block; scope policy needs a decision. **Confidence:** high (existence), low-medium (impact).

### Notes exclusions/borderline (pass 1)
- Letter image/logo upsert-by-id (5 copies, all intra-file in `letters.py`) — below two-file bar; promote if Compliance copies it.
- `generate_letter_artifact` (Notes/tasks.py:21–58) vs `Documents/tasks.py:29–94` `generate_signed_pdf` — deliberate parallel (different models/immutability policy).
- **Bug (not duplication):** `Notes/tasks.py:30` imports `from .service import ...` but `Notes/service.py` does not exist (canonical lives in `Notes/services/letters.py`) — `generate_letter_artifact` will raise ImportError when the Celery task runs.

### Mapping 87 — Twilio phone-number search/purchase/webhook-wiring re-rolled in Admin SMS lane
- **Canonical:** `messaging/whatsapp_service.py` — `search_available_numbers` (102–160, `.local.list`, lowercase capability keys), `provision_phone_number` (162–212), `configure_webhooks` (421–457, sets `status_callback`); subaccount-scoped client (34–37).
- **Reimplementation:** `Admin/views/sms_configuration_views.py` — `search_available_numbers` (352–422; hardcoded `GB` + `.mobile`, `voice_enabled` vs `mms_enabled`, **uppercase** capability keys, DRF error Responses); `provision_phone_number` (525–617; inline webhook wiring at 605–607 **without status_callback**, compensating `.delete()` release 610–652); `change_number` (205–230, second inline webhook-wiring re-roll); parent-account credentials re-read/validated at 4 sites (207, 363, 440, 509+).
- **Divergence:** subaccount vs parent account scope (intentional); number type/country/capability filters; capability key casing; status_callback wired or not; raise vs DRF responses; compensating release only in Admin lane.
- **Canonical path:** parameterize `WhatsAppService`/`TwilioNumberService` with `client_factory`/`country`/`number_type`/`webhook_policy`. **Extraction:** safe after account-scope decision. **Confidence:** high (existence), low-medium (impact).

### Mapping 86 (minor) — Outgoing-email Message-ID minting
- **Canonical:** `messaging/utils.py:599–602` `generate_message_id(domain="pathway.dental")`; callers `message_views.py:815, 1710`. **Divergent re-roll:** `dentallyIntegration/recall_automation.py:629–633` inline `f"<recall-{uuid4().hex}@{domain}>"` (prefix convention only here). **Silent gap:** `confirmation_automation.py` and `marketingBroadcast/marketing_email_client.py` mint no Message-ID at all (unthreadable).
- **Canonical path:** `mint_message_id(domain, prefix=None)`; recall imports with `prefix="recall"`. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### messaging pass-8 extension notes (clean-count unaffected)
- **Mapping 2 ext:** `messaging/views/template_views.py:860` sends via legacy `send_email_via_sendgrid` (`UserAuthentication/utils.py:330`) + `TemplateMessageHistory` — a SendGrid lane parallel to Mapping 2's EmailServiceClient lanes; the helper itself is a single shared implementation (ordinary imports).

### Mapping 84 — WhatsApp practice-setup lifecycle re-implemented in Admin lane
- **Operation:** provision/reset `PracticeWhatsAppConfig` (Twilio subaccount → number → status transitions); disconnect = clear credential fields + reset.
- **Implementations:** `messaging/views/whatsapp_views.py:139–198` `WhatsAppSetupInitiateView.post` (get_or_create at 149–151, `subaccount_created` at 167, `number_provisioned` at 171–179, failure block 190–194) vs `Admin/whatsapp_views.py:113–173` `initiate_setup` (near-verbatim copy; docstring admits "Pattern follows messaging/views/whatsapp_views.py"). **Disconnect twins:** messaging `:86–128` (clears subaccount+token, resets "pending") vs Admin `:75–111` (preserves them, resets "subaccount_created" — documented deliberate difference, but the 12-field clear-list maintained twice). **Config-read twins:** `:51–84` vs `:48–73`.
- **Risk:** status-vocabulary/field additions need 2 edits; disconnect semantics fork the recovery path for the same practice. **Canonical:** `ensure_whatsapp_setup(config, ...)` / `reset_whatsapp_config(config, *, preserve_subaccount)` beside `WhatsAppService`. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 85 — Session spam/archive toggle with classifier feedback (4 divergent variants)
- **Operation:** set/clear `MessageSession.is_spam` with mutual-exclusion (spam clears archive, archive clears spam), best-effort `learn_spam_correction_for_sessions.delay`, `broadcast_session_flagged_spam`.
- **Implementations:** `messaging/views/spam_views.py:35–60` `apply_spam_verdict` (one-way, archived=False, broadcast); `contact_views.py:979–1022` `spam` (two-way bulk, feedback, broadcast); `utility_views.py:163–170, 182–185, 236–245` (inline rule + feedback, **no broadcast**); `session_views.py:382–396` `archive` (inverse direction + feedback, **no broadcast**).
- **Risk:** same user action removes from inbox live in two lanes, stale in the other two until refresh. **Canonical:** `set_session_spam(sessions, is_spam, *, broadcast=True)` service. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Extensions (pass 6)
- **Mappings 76/32:** `message_views.py:3250–3423` `generate_empathetic_responses` — another identifier→scope lane (email both directions 3297–3331; naive `+{normalized}` concat at 3358–3361; ignores session/channel identity).
- **Mapping 20:** conforming site added — `messaging/views/call_log_views.py:160–165` `CallLogViewSet.by_phone` (`Q(phone_number=...) | Q(channel__canonical_value=canonical_phone_e164(...))`, the correct lookup-key shape).

### messaging exclusion (pass 6)
- `template_resolution.py` consolidation is complete — all four historical copies now delegate to the canonical module.

### messaging pass-7 extension notes (conforming sites of mapped families — clean-count unaffected)
- **Mapping 73 ext:** `messaging/management/commands/debug_unread_messages.py:44–70, 99–102` re-rolls the recount predicate (5th site).
- **Mapping 76/73 ext:** `messaging/management/commands/mark_all_messages_read.py:45–46, 85–91` — bulk `.update(unread=False)` with no `PracticeUnreadCount` decrement and no `update_session_stats` (same maintenance-bypass defect as `mark_conversation_read`).
- messaging `signals.py:241–311` stop-recall-on-reply vs `recall_automation.py:893–916` lazy backstop — documented deliberate belt-and-braces pair.

### Mapping 81 — Practice resolution for message-history viewsets (divergent rule + unscoped sibling)
- **Canonical:** `utils/practice_mixins.py:19–54` `PracticeAccessMixin` (60+ messaging viewsets). **Divergent:** `messaging/views/history_views.py:22–48` `MessageHistoryViewSet.get_queryset` — hand-rolled `user_type` ladder (staff→staff_profile.practice, dentist→dentist_profile, admin→current_practice); differs from every other inbox endpoint when `current_practice` differs from profile practice. **Unscoped variant:** `history_views.py:58–75` `TemplateMessageHistoryViewSet.get_queryset` — `TemplateMessageHistory.objects.all()` with **no practice filter** (model has practice FK, models.py:761–768; routed live at urls.py:186–196, 216) — cross-practice read exposure gated only by the `inbox` feature flag.
- **Fix:** both viewsets use the mixin + `filter(practice=...)`. **Extraction:** safe, trivial. **Confidence:** high (existence), medium (impact).

### Mapping 82 — Inbox conversation-list aggregation (two live lanes, divergent mechanisms)
- `messaging/views/message_views.py:1194–1400` `conversations` — Python dict grouping of raw Email/SMS rows by identifier strings, per-message unread flags, naive `+{num}` concat at 1303–1318, `priority_levels` literal ×3 (1274, 1359, 1391), skips WhatsApp; vs `messaging/views/contact_views.py:129–205, 296+` + `serializers.py:1884–2010` `ContactViewSet.list`/`ContactListSerializer` — ContactChannel grouped by Person identity, cached session counts. **Dead third variant:** `messaging/utils.py:605–800` `group_messages_by_thread`/`get_thread_summary`/`find_related_messages`/`rebuild_thread_for_message` — zero callers project-wide (deletion candidates).
- **Risk:** same inbox renders different counts/unread/identity per endpoint (Mapping 74/79-drift-prone fields; shared-mailbox string-key mis-attribution). **Canonical:** contacts lane; `conversations` delegates or retires. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 83 (minor) — Phone last-9-digits "tail" matching, four independent re-rolls
- `messaging/signals.py:260–272` (min 7, `in` semantics); `messaging/management/commands/repair_mislabelled_sessions.py:211–222` (min 9, `endswith`); `TreatmentPlan/management/commands/split_collapsed_persons.py:86–90` `_phone_tail` (min 9, equality); `dentallyIntegration/views/recall_sequence_views.py:404–405` (DB `contains=np[-9:]`, no min).
- **Canonical:** `phone_tail(value, min_digits=9)` in `TreatmentPlan/utils/contact_keys.py`. **Extraction:** safe. **Confidence:** medium-high (existence), low (impact).

### Extensions (pass 5)
- **Mapping 76:** fourth identifier→scope site — `message_views.py:1403–1616` `conversation` GET (email `iexact` both sides vs phone `+concat`/bare variants at 1502–1518).
- **Mapping 32:** another filter-side `+{num}` concat — `message_views.py:1303–1318`.

### messaging exclusions (pass 5)
- Serializer contact twins (`serializers.py:133–202` vs `395–429`) and `clone_template` email/SMS twins — documented deliberate intra-file copies. `TranscriptionConsumer.process_audio_chunk` (consumers.py:107–170) — chunked streaming, distinct operation from Mapping 69.

### Mapping 79 — MessageSession stats recount reimplementation in `tasks.update_session_stats_async`
- **Canonical:** `messaging/models.py:583–634` `update_session_stats` — queries by session FK (docstring: old phone/email queries could miss messages when participant identifier differs from stored phone_number), maintains sms/email counts, doesn't touch `updated_at`.
- **Reimplementation:** `messaging/tasks.py:16–103` `update_session_stats_async` — rebuilds **identifier-based** querysets (`Q(from_treatment_path=participant_email)|Q(to_patient=...)` at 39–44; SMS `Q(phone_number=participant_phone_number)|Q(phone_number=full_phone)` at 55–59), bumps `updated_at`, never writes sms/email counts. **Live on a schedule:** `tasks.py:139–165` `cleanup_stale_sessions` (beat task, 100 sessions/hour) rewrites counts using the legacy matching — the exact miss the canonical fixed. Batch wrapper `tasks.py:106–133`.
- **Fix:** task delegates to canonical. **Extraction:** safe. **Confidence:** high (existence), medium (impact — scheduled count correction/erasure). (Extends Mapping 74's mechanism list.)

### Mapping 80 — Hand-built `conversation_updated` WS payload (forked key vocabulary, 5 sites)
- **Implementations:** `messaging/utils.py:1277–1295` `broadcast_new_message` (`contact_unread_count`, preview `[:100]`, `person_id=channel_id`); `session_views.py:279–293` (`total_unread_count`, no preview); `contact_views.py:875–891` (`unread_count: 0` hardcoded, `person_id=identity_id`); `whatsapp_views.py:310–331` and `:573–591` (byte-identical pair; neither unread key).
- **Risk:** three disjoint unread-key vocabularies reach the same frontend consumer (`consumers.py:266–274` forwards verbatim); `person_id` semantics differ. **Canonical:** `conversation_payload(session, ...)` builder in `messaging/utils.py`; needs key-name decision first. **Extraction:** safe after decision. **Confidence:** high (existence), medium (impact).

### Mapping 32 extension (pass 4) — WhatsApp outbound naive country-code concat
- `messaging/views/whatsapp_views.py:254–256`: bare `f"{country_code}{to_number}"` (no `+`, no E.164) at a Twilio send site; wrapped as `whatsapp:{to_number}` at `whatsapp_service.py:309,364`. Same family as Mapping 32's naive copies.

### messaging exclusion (pass 4)
- `services.py:202–434` async/sync empathetic-responses pair — intra-file with one live consumer each; promote only if a third variant appears.

### Mapping 76 — Contact identifier → message/session bulk-scope resolution
- **Operation:** resolve "all messages/sessions for this email-or-phone contact" and bulk-update.
- **Implementations:** `messaging/views/utility_views.py:115–258` (naive `+{identifier}` concat at 147–148; email matches `to_patient` only); `messaging/views/message_views.py:3543–3605` `mark_conversation_read` (same concat; email lane matches `to_patient|from_treatment_path`; **never touches MessageSession/PracticeUnreadCount — `.update()` bypasses unread maintenance**); `messaging/views/contact_views.py:219+` `_resolve_identity` → `channel_id__in` (most correct shape; used by star/archive/spam).
- **Canonical:** extend `_resolve_identity` into shared `contact_scope(practice, identifier)` selector. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 77 — Person → linked Intake/Nurture "cross-link id" getter (four conventions)
- **Canonical:** `messaging/serializers.py:1651–1700` `CallLogSerializer` uses `sole_record_for_person` (#75/#92 guarded). **Unconverted variants:** `messaging/serializers.py:573–590` (`person.intakes.all()[0]` — **no ordering, arbitrary**); `:2066–2090` `ContactListSerializer` (`order_by("id").first()`, patient getter without ambiguity guard); `TreatmentPlan/serializers/intake.py:276–287` and `nurture.py:264–273` (most-recent `created_at` convention).
- **Risk:** same human's conversation can display a relative's intake/nurture id. **Fix:** all consume `sole_record_for_person`. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 78 (minor) — Practice total unread count, two read paths
- `messaging/consumers.py:447–457` `UnreadCountConsumer` (live `Sum("unread_count")` per socket read) vs `messaging/views/session_views.py:488–505` (cached `PracticeUnreadCount` row). Disagree whenever Mapping 74's drift exists. **Canonical:** one selector. **Extraction:** safe, trivial. **Confidence:** medium (existence), low (impact).

### Mapping 32 extension (pass 3) — filter-side naive `+concat` sites
- `messaging/views/utility_views.py:40–41, 147–148`; `messaging/views/message_views.py:3568–3569` — same `if not startswith("+"): x = f"+{x}"` bug shape (local `07700900123` → `+07700900123`, matches nothing); fold into Mapping 32's canonical normalizer.

### messaging exclusions (pass 3)
- Best-effort `learn_spam_correction_for_sessions.delay` try/except blocks (4 views) — ordinary `.delay()` reuse (Mapping 29 analogue).

### Mapping 74 — MessageSession cached-count maintenance (three concurrent mechanisms + WhatsApp-blind recount)
- **Operation:** keep `message_count/unread_count/sms_count/email_count/last_message_at` in sync on message create/delete.
- **Mechanisms:** (1) async F-expression increment — `models.py:1071–1088, 1478–1490, 2140–2147` enqueue `tasks.increment_message_count` (`tasks.py:171–232`: `F()+1`, `last_message_at=now()`, auto-unarchive, unread maintenance); (2) synchronous signal recount — `signals.py:20–165` (registered apps.py:7–10): authoritative COUNT, `last_message_at=instance.created_at`, **never touches unread_count** — fires alongside mechanism 1 on every create (permanent +1 drift until cleanup); (3) full recount `models.py:583–634` `update_session_stats` (canonical); (4) `commands/update_session_counts.py:99–124` recount with **no WhatsApp term** — rewrites WhatsApp session `message_count` to 0.
- **Canonical:** all delegate to a single `session.register_message(instance)` / `update_session_stats`. Dead variant: `tasks.decrement_unread_count` (235–273, no live caller). **Extraction:** safe. **Confidence:** high (existence), medium (impact — visible count drift). (Distinct from Mapping 73's mark-read family.)

### Mapping 75 — Twilio delivery-status webhook → message status update (SMS vs WhatsApp lanes diverge)
- `messaging/views/message_views.py:446–504` `webhook`: maps Twilio vocabulary (`undelivered→failed` at 477, `accepted/sending→pending`), WS broadcast `sms_delivery_status`; `messaging/views/whatsapp_views.py:610–655` `whatsapp_status_webhook`: stores `message_status` **verbatim** (raw "queued"/"accepted" persist), adds `delivered_at/read_at`/error stamps, no broadcast, no practice scope on sid lookup.
- **Canonical:** `apply_twilio_status(message, twilio_status, error_code=None)` + per-channel broadcast hook. **Extraction:** safe. **Confidence:** high (existence), low-medium (impact). (Signature-validation divergence is Mapping 70.)

### Mapping 24 extension (pass 2) — sending-identity resolution: four more unmapped sites
- `messaging/views/message_views.py:770–830` (bulk_send_email — re-queried per recipient), `:1810–1885` (send_plain_email); `dentallyIntegration/recall_automation.py:110–133` `_resolve_sending_domain`; `dentallyIntegration/views/recall_views.py:1735–1755`; `UserAuthentication/alert_email_utils.py:39–55` (comment admits mirroring). All should call Mapping 24's `resolve_practice_sender`.

### messaging borderline exclusion (pass 2)
- Inbound Twilio ingest scaffolding `receive_sms` (message_views.py:506–600) vs `whatsapp_incoming_webhook` (whatsapp_views.py:478–607): channel-specific scaffolding around canonical helpers; below extraction bar unless `receive_email_v2` folds into the same contract.

### Mapping 69 — Whisper → GPT post-process → dental-terminology transcription pipeline
- **Operation:** transcribe via `whisper-1`, optional `gpt-4o-mini` post-process, `format_dental_terminology`, return `{success, text, error}`.
- **Implementations (4 functions, 2 apps):** `messaging/services.py:109–201` `transcribe_audio` (caller message_views.py:3130); `messaging/services.py:544–625` `transcribe_audio_stream` (near-verbatim copy of #1, uses canonical `prepare_whisper_call`; **no call site found — dormant**); `Notes/services/ai.py:42–172` + `:173+` lightening mode (~15 Notes views); `Notes/services/transcription.py:31–171` `transcribe_audio_for_letter` (+template injection).
- **Divergence:** temperature 0 vs 0.3; prompt getters; #1 not migrated to `prepare_whisper_call`. **Canonical:** extend `TreatmentPath/ai.py prepare_whisper_call` with the post-process stage; all four call it. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 70 — Twilio webhook signature validation (canonical decorator bypassed by WhatsApp lane)
- **Canonical:** `messaging/validators.py:17–107` `validate_twilio_request` (URL variation retries, DEBUG bypass); only consumer `message_views.py:504`. **Inline re-rolls:** `whatsapp_views.py:424–430` (parent token), `:502–511` (subaccount), `:622–632` (subaccount + parent fallback) — single-URL validation only, no slash/https retries.
- **Risk:** behind a rewriting proxy the WhatsApp lane rejects valid webhooks the SMS lane accepts. **Fix:** parameterize the decorator's token resolver; decorate all three. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 71 — Inbound email MIME body extraction (recursive part-walk re-rolled)
- `messaging/utils.py:165–232` `direct_parse_mime`; `:234–288` `extract_multipart_content` (regex strategy); `messaging/views/message_views.py:2362–2381` nested flanker walk in `receive_email_v2` keeping both text+html (comment: "same as old receive_email"); plus second nested walk `extract_attachments_from_part` (2779+).
- **Canonical:** `extract_mime_parts(raw_mime) -> {text, html, attachments, headers}` in `messaging/utils.py`. **Extraction:** safe. **Confidence:** high (existence), low-medium (impact).

### Mapping 72 — AI style rewrite of message text (`reformat_text_as_*` trio vs `reconstruct_text`)
- `messaging/utils.py:803–844, 886–927, 987–1028` `reformat_text_as_magic/warm/formal` (byte-identical bodies, batch twins, three routed APIViews) vs `messaging/services.py:437–533` `reconstruct_text` (divergent: inline prompts, explicit API key, string error contract vs dict).
- **Canonical:** `rewrite_message_text(text, style_prompt, *, api_key=...)`; trio becomes wrappers or is retired. **Extraction:** safe. **Confidence:** medium-high (existence), low (impact).

### Mapping 73 — Conversation mark-read with practice-unread decrement
- `messaging/views/session_views.py:222–303` `mark_read` (single session) vs `messaging/views/contact_views.py:794–906` `mark_read` (identity-bulk) — same sequence (count-unread before bulk `.update` → `PracticeUnreadCount.decrement` → `update_session_stats` → WS broadcasts); email/SMS filter-pair block copy-pasted (242–253 / 829–847). Recount sites sharing the predicate: `models.py:2299–2303`, `commands/reset_practice_unread_counts.py:78–82`, `tasks.py:65`, `Statistics/views.py:84–87`.
- **Canonical:** `mark_sessions_read(practice, session_ids)` service. **Extraction:** safe. **Confidence:** medium-high (existence), low-medium (impact).

### Mapping 68 — Email bounce/complaint suppression (status → block_reason + person block + EmailMessages status stamp)
- **Canonical:** `marketingBroadcast/views/webhook_views.py` — named maps `_EVENT_TYPE_TO_BLOCK_REASON` (23–26) / `_EVENT_TYPE_TO_MESSAGE_STATUS` (47–51), `_record_message_status` (60–85, practice-scoped `provider_message_id` update), exact-recipient person resolution with fail-closed `sole_person_for_channel` fallback (145–211), block at 362.
- **Divergent:** `messaging/views/message_views.py:3786–3840` `email_message_delivery_callback` — inline status whitelist, suppression gated on `source_type == "marketing_broadcast"` (3821–3824), inline reason ternary (3832–3834), person via `PersonChannel...first()` (3827–3830) — **the shared-mailbox `.first()` bug shape finding #61 fixed in the webhook lane**; can suppress the wrong household member. Also fires a WS broadcast and accepts `deferred`.
- **Canonical path:** shared `suppress_on_delivery_event(email_message, status)` in `consent_ledger.py`; messaging lane's `.first()` is a bug. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 65 — Schema-embedded image hoist (walk → skip-stored → decode → store loop)
- **Canonical:** `marketingBroadcast/form_images.py` — `walk_schema_images` (240–254), `walk_block_content_images` (257–263), `store_image_for_practice` (224–237), `hoist_schema_images` (266–292); caller `views/form_template_views.py:85`. **Reimplementation:** `management/commands/hoist_form_images_to_storage.py` — `_walk_schema` (200–221, byte-identical incl. docstring), `_walk_block_content` (223–228), `_store` (190–198), `_rewrite`/`_rewrite_revision_blocks` (124–188) re-rolling the hoist loop despite importing `KIND_*`/`decode_data_url`/`process_form_image` from the same module.
- **Risk:** new image slot added to the canonical walker silently missed by the backfill. **Fix:** command imports the canonical functions, keeps its tally wrapper. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Mapping 66 — Client-attempt-id UUID coercion with deterministic uuid5 fallback
- **Operation:** coerce client attempt id to UUID; on failure `uuid.uuid5(NAMESPACE_URL, str(value))` — the idempotency key of `MarketingFormSubmission`.
- **Implementations:** `views/public_form_views.py:317–321` (ValueError only, strips input); `management/commands/import_marketing_form_demo.py:243–250` `_coerce_uuid` (+AttributeError/TypeError, no strip).
- **Canonical:** `coerce_attempt_uuid(value)` beside the model. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Mapping 67 (minor) — Public submit contact-field length caps
- `form_abuse.py:25–27` named constants (100/50/254) vs `views/public_form_views.py:351–353` inline literals `[:100]/[:50]/[:254]`. Rejection gate runs before slices today; a one-sided limit change makes gate and storage disagree. **Fix:** public_form_submit imports the constants. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### marketingBroadcast exclusions (pass 7)
- "Duplicate entity as `Copy of {name}`" in 4 files — generic Django row-copy idiom with per-model policies.

### Mapping 64 — Dentally gender normalization to a 2-value vocabulary (divergent case conventions)
- **Implementations:** `marketingBroadcast/tasks.py:57–63` `_normalize_gender` (stored string; strip+lower; only male/female kept, else `""` — feeds `MarketingPatientProfile.gender`, lowercase vocabulary per models.py:149); `dentallyIntegration/tasks.py:1324–1331` `_parse_gender` (raw payload bool→female/male; **string branch is bare strip with no lower and no vocabulary restriction** — arbitrary strings enter `Patient.gender`); `management/commands/backfill_recall_dob.py:43–50` (SQL CASE, output Title-case, mirrors Go `ParseGender` writing `RecallPatient.sex` as "Male"/"Female").
- **Risk:** same human stored under two case conventions in two tables; payload lane lets non-vocabulary strings through the 2-value contract. **Canonical:** `normalize_dentally_gender(value, *, output_case)` in a shared Dentally vocab module. **Extraction:** safe. **Confidence:** high (existence), low-medium (impact — no cross-comparison site yet).

### marketingBroadcast borderline exclusions (pass 6)
- Reply-to validation split: `serializers.py:189–193` (any `@` string) vs `views/domain_views.py:136–155` (max-5 `validate_email`) — intra-app strictness divergence, below two-file bar; promote if a third site appears.
- `library_templates.py:5–8` transactional-shell copy from `UserAuthentication.alert_templates` — documented intentional (fail-closed personalisation constraint prevents import).

### Mapping 62 — WebSocket practice-access gate (canonical exists, lanes unconverted)
- **Canonical:** `utils/ws_practice.py` `reject_unless_member` / `user_may_join_practice` / `current_practice_id` (audit #78; re-reads DB because soft practice switches outlive sockets). Migrated: `TreatmentPlan/consumers.py:8`, `messaging/consumers.py:16`.
- **Unconverted copies:** `marketingBroadcast/consumers.py:96–101` (compares cached `user.practice` FK — the exact lane the canonical's docstring calls out as still wrong); `dentallyIntegration/consumers.py:144–147, 236–239` (byte-identical; previously excluded as intra-file, now cross-file with the canonical + 3 same-shape sites).
- **Risk:** stale-practice socket authorization (user keeps access to old practice's group for socket lifetime). **Fix:** delegate to `reject_unless_member`. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 63 — Public/unauthenticated practice brand-identity payload
- **Operation:** build practice identity block for anonymous public pages (name, logo URL, brand colours; blank→None normalization).
- **Implementations (5 sites, 5 files):** `marketingBroadcast/views/preferences_views.py:52–67` (raw colours, no blank→None); `marketingBroadcast/views/public_form_views.py:118–143` `public_practice_identity` (blank→None, different key names `primary`/`accent`/`cardBg`/`background`, invents shortName/phoneHref); `dentallyIntegration/views/appointment_confirm_views.py:286–301` (raw colour keys); `Documents/views/public_views.py:243–252` (`or None`, `_practice_logo_url`); `TreatmentPlan/views/public_treatment_views.py:140–166` (local `_blank_to_none`, absolute URI).
- **Risk:** same unset-colour practice renders fallback theme on some public pages and empty-string styling on others; forked key vocabulary. **Canonical:** `public_practice_brand(practice, request, *, blank_to_none=True)` builder; callers overlay page extras. **Extraction:** safe; needs key-name decision. **Confidence:** high (existence), low-medium (impact).

### Borderline exclusions (pass 4, for the record)
- Person display-name join `f"{first} {last}".strip()` ~40× project-wide (e.g. `delivery_reporting.py:340` bypassing `TreatmentPlan.utils.names.full_name` — CSV renders double space, preview collapses) — generic idiom; promote if more in-area bypasses appear.
- Address one-line joins across different models — generic.

### Mapping 60 — Retry-loop unique random short-token minting
- **Operation:** mint random short code, retry against model-uniqueness `exists()`, bounded or unbounded.
- **Implementations (4 sites):** `marketingBroadcast/models.py:99–105` (bounded 10, loud RuntimeError); `dentallyIntegration/confirm_utils.py:12–33` `generate_unique_short_token` (bounded 10, same contract; only its legacy *HMAC* pair was previously excluded — the short-token generator was unmapped); `medicalHistory/models.py:71–78` (**unbounded `while True`**, ambiguous-safe alphabet); `Documents/models.py:99–106` (**unbounded `while True`**).
- **Risk:** saturated keyspace hangs the worker in two lanes; two truncation/maintenance policies. **Canonical:** `generate_unique_token(queryset, field, length, alphabet, max_attempts=10)` in shared utils. **Extraction:** safe. **Confidence:** high (existence), low-medium (impact).

### Mapping 61 (minor) — Email masking for display
- **Operation:** render `lo***@domain`.
- **Implementations:** `marketingBroadcast/views/preferences_views.py:12–16` (keeps 1 char); `UserAuthentication/utils.py:~1428–1437` (keeps 2 chars, different no-`@` fallback).
- **Canonical:** one `mask_email(email, keep=2)`. **Extraction:** safe, trivial. **Confidence:** medium (existence), low (impact).

### Mapping 59 — Practice-unique slug suffix assignment for MarketingForm
- **Canonical:** `marketingBroadcast/views/form_views.py:42–57` `_next_slug(base, practice, ignore_pk)` (callers 99, 293). **Copy:** `form_template_views.py:87–99` (`use` action) inline re-roll of the identical suffix loop instead of importing `_next_slug`. Adjacent below-bar: `Admin/marketing_form_template_views.py:104–108` same suffix shape on a different model.
- **Fix:** template lane imports `_next_slug`. **Extraction:** safe, trivial. **Confidence:** high (existence), low (impact).

### Mapping 7 extension (pass 2) — campaign preview pagination site
- `marketingBroadcast/views/campaign_views.py:360–372, 408–430` — a third live manual-pagination site (unguarded `int()` → silent default, page_size cap 200, hand-built envelope) with the same shape as the segment lane already recorded in Mapping 7.

### Mapping 54 — Segment rule-group → queryset resolution (preview drops the ambiguity guard)
- **Canonical:** `marketingBroadcast/segment_engine.py:198–212` `resolve_segment` — filters `is_archived=False, is_ambiguous=False` (audit #37). **Copy:** `marketingBroadcast/views/segment_views.py:30–45` `_resolve_preview_queryset` — near-verbatim but **omits `is_ambiguous=False`**; callers at segment_views.py:72, 91, 140.
- **Risk:** builder preview/last_resolved_count counts fused-Person profiles that real audience resolution and sends exclude. **Fix:** view lane calls `resolve_segment`. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 55 — Marketing-consent sendability predicate (per-row vs batched)
- **Implementations:** `marketingBroadcast/consent_ledger.py:82–106` `is_marketing_eligible` (per-row; absent row → sendable) vs `segment_engine.py:297–317` batched ORM copy (comment: "deliberate batched duplicate … MUST be kept in step"). Callers both live.
- **Classification:** documented deliberate variant; **keep-separate or parameterize one predicate** (`marketing_sendability(practice, person_ids)`). **Extraction:** safe. **Confidence:** high.

### Mapping 56 — Person → primary email resolution (4 shapes, 3 files)
- `delivery_reporting.py:27–29` (non-empty guard); `views/preferences_views.py:19–21` (`_primary_email` — **no non-empty guard**; empty-string channel surfaces on the public preferences page); `views/campaign_views.py:81–98` `_emails_for` (batched); `segment_engine.py:287–295, 349–359` (same predicate ×2).
- **Canonical:** `primary_email_for_person` / `emails_for_person_ids` in `TreatmentPlan/contact` selectors. **Extraction:** safe. **Confidence:** medium-high.

### Mapping 57 — Sole-patient disambiguation: extracted canonical vs unconverted original
- **Canonical:** `TreatmentPlan/utils/sole_patient.py:27–59` `sole_patient_for_person` (docstring: "the rule already proven under #37 … lifted out"). **Unconverted original:** `marketingBroadcast/tasks.py:125–139` (also derives `is_ambiguous`; helper returns None for both "no patients" and "ambiguous").
- **Fix:** add `(patient, ambiguous)` return mode to helper; tasks.py delegates. **Extraction:** safe. **Confidence:** high.

### Mapping 58 — Practice timezone resolver (divergent fallbacks)
- **Implementations:** `marketingBroadcast/form_handoffs.py:83–87` (fallback `Europe/London`); `financialAnalytics/analytics/base.py:124–141` (practice_id input, fallback `UTC`).
- **Risk:** unset/invalid timezone → same practice's day boundaries differ by lane. **Canonical:** one `practice_timezone(practice_or_id, default=...)` in UserAuthentication/utils. **Extraction:** safe; needs fallback-policy decision. **Confidence:** medium-high.

### marketingBroadcast additional exclusions (pass 1)
- Group-send best-effort `group_send` idiom repeated across ~10 apps (Invoices/websocket_utils, staffAlerts/dispatch, messaging/utils, TreatmentPlan/websocket_utils, etc.) — channels-framework idiom, below bar.

### Mapping 51 — Styled tabular PDF export (WeasyPrint inline-HTML scaffold)
- **Operation:** render rows to styled PDF (`HTML(string=...).write_pdf()` with shared "purple header / zebra rows" CSS `#846ce0`/`#f9f7ff`).
- **Implementations:** `Stock/views/stock_export_views.py:188–245` `_build_pdf` (original; installs `_deny_all_url_fetcher` SSRF defense at 188–197, 242); `dentallyIntegration/views/dentally_views.py:2084–2179` `export_pdf` (docstring admits "reusing the exact Stock-export look"; **no url_fetcher passed** at 2171 — security posture diverges).
- **Canonical:** shared `styled_table_pdf(title, meta, columns, rows)` with `_deny_all_url_fetcher` always installed. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 52 — Same-day sibling-appointment grouping for confirmations
- **Operation:** a patient's other same-day non-cancelled appointments, to roll slots into one confirmation.
- **Implementations:** `dentallyIntegration/views/appointment_confirm_views.py:17–35` `_same_day_companions` (DB query, Python `is_cancelled_state` filter); `confirmation_automation.py:536–560` (in-memory `defaultdict` over candidates with own eligibility gate; `grouped_appointment_ids` at 604).
- **Divergence:** DB vs in-memory mechanism; manual cascade ignores targeting eligibility; `None` start_time lands only in automation group. **Canonical:** `same_day_siblings(appointment)` / `group_same_day(...)` beside `confirmation_status.py`. **Extraction:** safe. **Confidence:** medium.

### Mapping 53 — Recall segment-config defaults contract (defaults table vs inline literals)
- **Implementations:** `dentallyIntegration/views/recall_config_views.py:28–62` `DEFAULT_SEGMENT_PARAMS`/`SEGMENT_PARAM_FIELDS` (authoritative, shown in admin at 88, zero external consumers) vs `recall_views.py` inline literals for the same keys (`not_seen_months` 24 at 211, 374, 608, 1208–1211, 3766–3768; `first_visit_months` 12 / `max_records` 5 at 375, 609, 2813–2819; `exam_hygiene_months` 15; `predictable_visits` 3; `min_spend` 500; etc.).
- **Risk:** changing a default in the table silently leaves the engine on the old value — admin preview and actual engine disagree. **Canonical:** move `DEFAULT_SEGMENT_PARAMS` to models.py beside `RecallSegmentConfig`. **Extraction:** safe. **Confidence:** high (existence), medium (impact).

### Mapping 22 extension (pass 7) — additional Dentally-id resolution sites
- `dentallyIntegration/serializers.py:562–597` `_get_patient` (MultipleObjectsReturned → newest-`.first()` — ambiguity-binds-arbitrary-match shape) and `recall_views.py:703–710` (`meta_data__id=pid` newest-first).

### Mapping 49 — Recall channel-sendability predicate ("contact present AND Dentally consent")
- **Operation:** can-contact = non-empty contact detail AND synced `RecallPatient.use_email`/`use_sms`; missing-row default (True, True).
- **Implementations (4 sites, 3 files):** `dentallyIntegration/serializers.py:1250–1268` (strip, fail-open default); `dentallyIntegration/views/recall_views.py:3299–3321` (ORM Q — **no strip**; inner join → **fail-closed** on missing row); `recall_sequence_views.py:166–176` (strip, **fail-open**, comment: "An absent RecallPatient row is a sync gap, not an archive"); `recall_views.py:1719–1821` `bulk_send` (two-phase presence/consent checks with per-reason skip).
- **Risk:** whitespace-only email passes the list filter but fails serializer/enroll/bulk-send; missing sync row flips open/closed per lane. **Canonical:** `recall_sendable_q(channel)` / `is_sendable(record, consent_row, channel)` with fail-mode parameter. **Extraction:** safe. **Confidence:** high.

### Mapping 50 — Dentally REST fetch loop re-rolled outside `DentallyAPIService`
- **Canonical:** `dentallyIntegration/services.py:53–75,176–251,295–317` (Bearer, environment→BASE_URLS, header-aware 429 backoff, 403-as-rate-limit). **Reimplementation:** `management/commands/backfill_recall_sendability.py:41–90` — inline auth + own paging, blind `sleep(5)` 429 policy, no 403 handling. **Fix:** generator/all-pages variant on `DentallyAPIService.get_patients`. **Extraction:** safe (low impact, one-off command). **Confidence:** high.

### dentallyIntegration additional exclusion (pass 6)
- Attendance-segment classifier: `recall_views.py:343–484` batched `_attendance_segment_map` mirrors per-row classifier at `recall_views.py:489–745` — intra-file only; promote to a mapping if a third site appears.

### Mapping 48 — Fernet field-encryption helper (env→settings→DEBUG key resolution + encrypt/decrypt wrappers)
- **Operation:** resolve symmetric Fernet key, wrap `encrypt/decrypt` with empty-passthrough, for third-party secrets at rest.
- **Implementations:** `dentallyIntegration/encryption.py:15–91` (DEBUG → fresh random key per call; **decrypt failure returns ciphertext as plaintext**); `onlineBooking/encryption.py:9–56` (DEBUG → **hardcoded committed key** `b"kRxvTNGQNQ_ownP4Emm-..."`; decrypt failure raises in prod); `messaging/models.py:1923–1953` (`PracticeWhatsAppConfiguration.set/get_auth_token` — **no env/settings key; derives from `SECRET_KEY[:32]`**, incompatible scheme, SECRET_KEY rotation breaks stored tokens).
- **Canonical:** single `field_crypto.py` with `get_field_cipher(env_var_name)`. **Extraction:** requires re-encrypt migration per consumer + fallback/passthrough policy decision. **Confidence:** high (existence), medium (impact — latent security/availability issues).

### Mapping 33 extension (pass 5) — Dentally date-string coercion copies still live
- `TreatmentPlan/utils/dates.py:24–27` documents consolidation of `_as_date` in `dentallyIntegration/opportunity_classifier.py:64–76` (5 live call sites) and `management/commands/backfill_daylist_patients.py:27–34` — **both still live and unconverted**. Two further unmapped copies: `tasks.py:1313–1322` `_parse_date` (`parse_date(str(value)[:10])`), `management/commands/migrate_dentally_clinical.py:387–393` `_parse_iso_date`. Aware-datetime variant: `tasks.py:570–586` `parse_date` (`fromisoformat` + `make_aware`) — fourth variant of the `parse_request_datetime` family missed by dates.py's own family list. All drop-in replaceable by `parse_date_value`/`parse_request_datetime`. **Extraction:** safe. **Confidence:** high.

### Mapping 46 — Messaging outreach-event reporting query (per-channel filter + events/count modes)
- **Operation:** query outgoing EmailMessages/SMSMessage (+WhatsApp/CallLog) by practice + purpose/source_type + direction + `is_sandbox=False` + window; emit merged event list (limit `min(100,500)`) or `{total_contact_attempts, by_channel, by_status, ...}` aggregate.
- **Copy-paste pair:** `dentallyIntegration/views/recall_sequence_views.py:297–546` (4 channels, `message_purpose="recall"`) ↔ `dentallyIntegration/views/daylist_reporting_views.py:58–209` (2 channels, `message_purpose="automation"`, docstring: "Mirrors RecallReportingViewSet's shape exactly"). **Batched variant:** `recall_views.py:1441–1620` `_recall_contact_data`.
- **Divergence:** channel sets, scope key (single channel_id vs `channel_id__in`), WhatsApp scoped `to_number__contains=np[-9:]`, email always-"sent"; endpoints disagree on what a "contact attempt" is. **Canonical:** parameterized `messaging_outreach_report(practice, purpose|source_types, channels, window, scope, mode)` in messaging. **Extraction:** safe. **Confidence:** high.

### Mapping 47 — Recall record → contact-identity (Person/ContactChannel) resolution
- **Operation:** resolve RecallRecord (Dentally patient id + raw email/phone) to messaging contact identity.
- **Implementations (3):** `dentallyIntegration/serializers.py:1125–1220` `_get_contact` (per-row; own-Patient-first, name disambiguation via `canonical_full_name_key`); `dentallyIntegration/views/recall_views.py:2099–2248` `_build_contact_map` (batched mirror, docstring: "matching the per-row _get_contact exactly", uses `lookup_key`); `dentallyIntegration/views/recall_sequence_views.py:248–296` `_resolve_contact` (**weaker: no own-Patient-first, no name disambiguation — can attribute recall report to a family member sharing the channel**).
- **Canonical:** one resolver in `TreatmentPlan.contact` (`person_for_recall_record` + bulk wrapper); `_resolve_contact` delegates. **Extraction:** safe; needs email-vs-phone probe-order decision. **Confidence:** high.

### Mapping 36 extension (pass 4) — Go mirror of Dentally family grouping
- `EmailServiceGo/internal/dentally/migration/service.go:1142–1200` `applyDentallyFamilyGrouping` mirrors `dentallyIntegration/tasks.py:108–170` (comment admits it). **Divergent:** Go creates a fresh Household per run with `Dentally Family <id>` label and blind-UPDATEs all siblings (overwriting existing households); Django prefers an existing multi-member household and preserves assignments. Same family → different household topology per import lane. Not extractable cross-service; needs contract documentation + alignment. **Confidence:** medium-high.

### Mapping 42 — Sequence step send-time parsing + due-at computation
- **Operation:** resolve step send_time (step → sequence default → fallback) and compute UTC due instant in practice timezone.
- **Implementations:** `dentallyIntegration/recall_automation.py:40–110` (anchored on enrollment date, ADDS offset_days, `DEFAULT_SEND_TIME=9:00` fallback, Day-0 immediate-send); `dentallyIntegration/confirmation_automation.py:37–77` (anchored on appointment date, SUBTRACTS offset_days; docstrings admit "copied, not imported"/"mirror image"; **no fallback — unparseable send_time can raise**).
- **Canonical:** shared `as_time` + parameterized `compute_step_due_at(..., direction=...)` in a sequences timing module. **Extraction:** safe; fallback difference is a latent bug in confirmation lane. **Confidence:** high.

### Mapping 43 — Terminology-mapping category word-set extraction
- **Canonical:** `dentallyIntegration/views/recall_helpers.py:22–35` `_exam_hygiene_words` (4 live call sites in recall_views). **Reimplementation:** `dentallyIntegration/recall_automation.py:190–199` private `_words` closure, semantically identical.
- **Canonical path:** relocate helper to `treatment_vocab.py`; recall_automation imports. **Extraction:** safe. **Confidence:** high.

### Mapping 44 — Patient-facing confirmation-link minting contract
- **Operation:** stamp `confirmation_channel`, mint `confirmation_token` via `generate_unique_short_token` only if absent, save(update_fields), build `{CONFIRM_BASE_URL}/{token}`.
- **Implementations:** `dentallyIntegration/confirmation_automation.py:110–131` `get_confirmation_link` (docstring: "mirrors AppointmentConfirmLinkViewSet.retrieve's contract exactly"); `dentallyIntegration/views/appointment_confirm_views.py:69–85` (inline original). Both duplicate hardcoded fallback `"https://dev.confirm.dental"` (127 / 83).
- **Canonical:** `get_confirmation_link` moved to `confirm_utils.py`; fallback defined once. **Extraction:** safe. **Confidence:** high.

### Mapping 45 — Email template context builder (`patient`/`clinic`/`dentist` + flat placeholders)
- **Operation:** build structured `{patient, clinic, dentist, current_user}` + flat placeholder context for message rendering.
- **Implementations:** `messaging/views/template_views.py:421–560` `_build_email_template_context`; `dentallyIntegration/recall_automation.py:245–289` `_recall_template_context` (docstring: "SAME shape … keeps recall-automation rendering identical" — manual mirror contract).
- **Divergence:** recall splits `rec.patient_name` on first space vs real name fields; `practice` alias; always-empty `current_user`; messaging adds plan_info. **Canonical:** `build_practice_clinic_info(practice)` + assembler in `messaging`; recall overlays its placeholders. **Extraction:** safe for clinic block. **Confidence:** medium-high.

### Mapping 41 — Dentally migration progress cache-key contract + raw-Redis fallback read
- **Operation:** producer/consumers share a Redis-cached migration progress blob keyed `dentally_migration_{practice_id}` (+ `dentally_migration_cancel_{practice_id}`); views bypass the Django cache API and read raw Redis (`:1:` django-redis prefix, `db=1`, `pickle.loads`).
- **Producer:** `dentallyIntegration/tasks.py:258–259, 307, 359` (`cache.set`, timeout 86400). **Consumers:** `dentallyIntegration/views/dentally_views.py:364–382, 538–562, 616–646, 741+` — 4× intra-file copy of the direct-Redis block (`redis_key = f":1:{cache_key}"` at 375/552/630/647; comment: "Return Redis data directly since Django cache.get() is unreliable").
- **Risk:** key rename or cache backend/serializer/db change in one file silently breaks the other; hardcoded db/timeout on both sides. **Canonical:** `migration_progress_key(practice_id)` / `read_migration_progress(practice_id)` helpers. **Extraction:** safe, low priority. **Confidence:** high (existence), medium (impact).

### Mapping 39 — "Patient has a live/booked future appointment" predicate
- **Operation:** decide whether a Dentally patient has an upcoming booked appointment via `RecallAppointment` state + start time.
- **Canonical (fixed):** `dentallyIntegration/recall_automation.py:296,307–322` `patient_has_future_appt` — `Lower("state")__in BOOKED_APPOINTMENT_STATES` + `start_time__gte=now`; docstring documents the old unanchored-regex bug ("reschedule_pending"/"unconfirmed" counted as booked, silently suppressing recall sequences).
- **Unfixed regex copies (×5 live sites):** `dentallyIntegration/views/recall_views.py:585, 940, 3543, 3588, 3856` — `state__iregex=r"(pending|confirmed|arrived|in.?surgery)"`; unanchored (false positives) and time semantics differ (`start_time__date__gte`/midnight cutoff vs `now`).
- **Risk:** same patient "has future appt" in UI but not in the automation (recall sends while UI shows resolved). **Canonical:** bulk selector `patient_ids_with_future_appt(practice, ids)` using Lower+now. **Extraction:** safe. **Confidence:** high.

### Mapping 40 — Dentally cancelled/completed appointment-state vocabulary (`CANCELLED_STATES`)
- **Canonical:** `dentallyIntegration/confirmation_status.py:31,34,39–44` (`CANCELLED_STATES`/`COMPLETED_STATES` + strip+lower predicates; line 30 comment: "Keep this in step with opportunity_classifier.CANCELLED_STATES"). Correct consumers: `confirmation_automation.py:493`, `next_appointment.py:29,42`.
- **Copy:** `dentallyIntegration/opportunity_classifier.py:60` (identical literal set; used at 335 with `.lower()` but **no `.strip()`** — messy "cancelled " data counts live in opportunity suppression but dead in confirmation logic).
- **Fix:** opportunity_classifier imports canonical. **Extraction:** safe. **Confidence:** high.

### dentallyIntegration borderline exclusions (do not re-derive)
- `treatment_vocab.py` ↔ `opportunity_vocab.py` structurally parallel vocab loaders — documented analogue, semi-deliberate divergence, below bar.
- recall_views.py intra-file: "spend over period" recomputed 4× (715–730, 3635–3657, 2968–3010, 1173–1189); dormant tab omits it (serializer fallback at serializers.py:1462–1467).
- `consumers.py:144,236` duplicated `check_practice_access` — intra-file.
- `(patient.meta_data or {}).get("id")` one-line idiom ~10 sites — generic; `migrate_dentally_clinical.py:748` lacks None guard (management command).

### Mapping 37 — Dentally import error-message → `failure_type` classification chain
- **Operation:** classify failed import from error text into `failure_type` (→ DataQualityIssue.classifier via `_record_import_issue`).
- **Implementations:** `dentallyIntegration/tasks.py:1033–1213` (superset: exact-case "Duplicate"/"Validation error"/"Invalid phone"/"required"/"missing data"; DB-constraint branch at 1093); `automations/actions.py:732–745` (lowercase matching; **no invalid_phone branch** → such issues become `other_import_error` and are bulk-retried forever by `dataQuality/views.py:527–584` `retry_errors`); intra-file variant `automations/actions.py:639–657` (serializer lane).
- **Risk:** ordering divergence ("Validation error: ... required field" → validation in tasks.py but missing_data in actions.py); case sensitivity changes classifier per lane. **Canonical:** `classify_dentally_import_failure(...)` beside `_record_import_issue`. **Extraction:** safe. **Confidence:** high.

### Mapping 37 extension (dataQuality pass 3) — Go mirror of the classification + issue-writer contract
- **Go lane:** `EmailServiceGo/internal/dentally/migration/service.go:2402–2503` — `_failureTypeToClassifier` (comment admits "mirrors Django's… keep both in sync") + `persistImportIssue` (same 9 detail keys; update-then-insert keyed `practice_id AND dentally_patient_id`). **Third classifier ordering:** duplicate → invalid phone → missing data/required → validation; "Validation error: … required" → `validation_error` in Django tasks.py but `missing_data_import` in Go — same retry-economics divergence as Mapping 37's Python lanes.
- **Unschematized cluster contract:** `dataQuality/views.py:69–77` writes `dismissed_member_ids` (sorted) in `detail`; `EmailServiceGo/internal/dentally/scheduler/dataquality_sweep.go:279–315` re-derives the re-open rule from that JSON key in raw SQL; `record_id = "channel-<id>"` format duplicated (views.py:56 docstring, sweep.go:257, legacy_backfill.py:83). Each side implements half the protocol with no constant/schema source.
- **Classification:** cross-service mirror contract — cannot be extracted into shared code; remedy is a documented schema note + single Python anchor for the classifier map and record_id format. **Confidence:** medium-high.

### Mapping 38 — Post-failure conflicting-patient identification for `conflict_details`
- **Operation:** after a duplicate/import failure, find which existing Patient caused the conflict; stamp `conflict_details`.
- **Implementations:** `dentallyIntegration/tasks.py:1095–1178` (multi-match report: email + canonical-phone scans, `duplicate_matches` list with match_type/multiple_matches/all_matches); `automations/actions.py:655–688` (single patient via `email__iexact` or `find_conflicting_patient_by_phone`, first-match-wins — a both-key conflict reported as email_only only).
- **Canonical:** derive from Mapping 25's planned `find_dentally_duplicate(practice, ...) -> (patient|None, matched_by)`. **Extraction:** requires Mapping 25's design decision to land first. **Confidence:** medium.

### Mapping 36 — "Ensure Person has a Household" create-or-assign snippet
- **Operation:** if Person has no household, create `Household(practice, label)`, assign, persist; optionally assign to other Persons.
- **Implementations (6 sites, 5 files):** `dataQuality/views.py:273–282` (label `f"{first} {last}"[:255] or "Family"`); `dentallyIntegration/tasks.py:419–426` (identical shape, label `f"Family {family_id}"` with different truncation order); `TreatmentPlan/views/contact_merge_views.py:422–428` and `481–486` (label `""`); `TreatmentPlan/management/commands/unweld_persons.py:347–356` (label `f"Household {id}"`); `TreatmentPlan/management/commands/split_collapsed_persons.py:496–500` (label `""`, writes via `.filter(...).update(...)` — bypasses signals); `TreatmentPlan/views/patient_views.py:890` (label `""`, id-assign).
- **Risk:** label policy forks 4 ways; write mechanism forks (signals vs no signals); edits need 5 files. **Canonical:** `Household.get_or_create_for_person(person, label)` on TreatmentPlan models. **Extraction:** safe. **Confidence:** high.

### dataQuality borderline exclusions (recorded so later passes don't re-derive)
- `force_import` vs `create_patient` (dataQuality/views.py:244–349 vs 458–525): ~55-line intra-file copy — intra-file only.
- `_FAILURE_TYPE_TO_CLASSIFIER` duplicated `dataQuality/legacy_backfill.py:14–20` ↔ `dentallyIntegration/tasks.py:24–30`; legacy_backfill's only callers are migrations (excluded).

### Mapping 35 — Workflow entity-type vocabulary split (`VALID_ENTITY_TYPES` ↔ `MODEL_TO_ENTITY_TYPE` ↔ handler keys)
- **Operation:** canonical workflow entity-type string vocabulary, validated outbound (events.py gate), re-derived from model names (record_events), re-keyed inbound (actions.py handlers).
- **Implementations:** `automations/events.py:48–61` `VALID_ENTITY_TYPES` (11 values; invalid → event silently dropped, gate at 118–122); `TreatmentPath/record_events.py:38–52` `MODEL_TO_ENTITY_TYPE` (model_name mapping + fallback `model_name.lower()`, plural hack at 47); `automations/actions.py:263–268, 2304, 2626` (inbound handler dicts, disjoint subsets, intra-file).
- **Risk:** outbound whitelist includes `appointment`/`invoice` only via model_name-fallback coincidence; any newly whitelisted type requires a hand-edit in record_events or events are silently dropped. **Canonical:** export the set from `automations/events.py`; record_events references/validates it at import time. **Extraction:** safe. **Confidence:** medium.

### Mapping 34 — Twilio `TwilioRestException` → user-facing error taxonomy
- **Operation:** classify failed `messages.create` by `str(e).lower()` substring + error codes (21212, 21211, 21608, funds, blocked/spam) into `user_message`; persist failed `SMSMessage` with error fields.
- **Implementations:** `automations/actions.py:1932–1955` (500 JSON response; no 21602 branch); `messaging/views/message_views.py:388–430` (superset — adds bare "from number"/"to number" substrings and `21602`/"invalid message" at 424; DRF ValidationError 400; different fallback message/truncation). (Corrects Mapping 1's earlier "only actions.py maps Twilio error codes" note.)
- **Risk:** 21602 sends report generic message from automations but specific from Inbox; taxonomy edits need two files. **Canonical:** `twilio_error_to_user_message(e, ...)` pure helper; response shaping stays per-site. **Extraction:** safe. **Confidence:** high.

### Mapping 33 — Optional ISO-date-string coercion with silent fallback (`str` → `date`)
- **Operation:** parse optional payload field as `%Y-%m-%d`; on failure silently keep unset.
- **Canonical (unused by actions.py):** `TreatmentPlan/utils/dates.py:97–112` `parse_date_value` (truncates `[:10]`, returns None). **Reimplementation:** `automations/actions.py:1184–1218` — three inline copies inside `_create_treatment_plan_record` (discount_valid_until, start_date, end_date), exact `%Y-%m-%d`, silent `except: pass`. **Adjacent naive copies:** ~25+ view sites (`Statistics/views.py` ×9, `Appointments/views.py` ×5, `Assets`, `compliance`, `Invoices`, …) with per-site failure modes (silent vs 400 vs 500).
- **Canonical:** `parse_date_value`; actions.py lane is a drop-in. Wider view-lane consolidation needs a failure-policy decision per endpoint. **Extraction:** safe (actions lane); medium confidence overall.

### Mapping 32 — Outbound SMS "to"-number E.164 resolution at Twilio send sites
- **Operation:** decide the `to` for `client.messages.create`: keep `+`-prefixed, else canonicalise via country source.
- **Canonical variants:** `Documents/utils/notifications.py:258–280` `_normalise_phone` (docstring calls the naive shape the "+GB7911123456" bug, audit #10); `messaging/views/message_views.py:247–272`; `automations/actions.py:1811–1860` (superset with patient-scan country inference — scan portion under Mapping 9).
- **Naive copies (still `f"+{phone}"` concat — unfixed "+GB" bug shape, messages silently never send):** `TreatmentPlan/journey/dispatch.py:218`; `dentallyIntegration/recall_automation.py:669` (internally inconsistent — line 367 uses canonical for lookup); `dentallyIntegration/confirmation_automation.py:317`; `dentallyIntegration/views/recall_views.py:1899`; `messaging/views/template_views.py:1650–1652`.
- **Canonical path:** generalize `_normalise_phone` to `resolve_sms_to_number(phone, practice, patient=None, country_code=None)` in phone utils. **Extraction:** safe. **Confidence:** high.

### Mapping 31 — EmailServiceGo base-URL + shared-secret outbound resolution
- **Operation:** resolve the Go service base URL and/or outbound `X-Workflow-Secret` header, each site re-rolling env/settings lookup with its own hardcoded fallback instead of `settings.py` (EMAIL_SERVICE_URL at settings.py:919, WORKFLOW_SERVICE_SECRET at 947).
- **Implementations (7 files):** `automations/events.py:38,43–45,158–161` (fallback **:9000**); `marketingBroadcast/marketing_email_client.py:26–34` (**:8080** + secret fallback); `UserAuthentication/email_address_provisioning.py:13–17` (**:8080** + secret); `UserAuthentication/email_service_client.py:33–36` (os.environ-first, **:9000**, Bearer API key); `Tasks/views.py:1412` (os.environ-first, **:9000**, Bearer); `dentallyIntegration/views/noshow_views.py:74` (os.environ-first, **:8082**, forwards user JWT); `messaging/email_service.py:72` (:8080).
- **Divergence (load-bearing):** three distinct fallback ports for the same service; settings-first vs os.environ-first precedence; X-Workflow-Secret vs Bearer vs forwarded-JWT auth; fire-and-forget notify sub-pattern differs in observability (events.py logs vs noshow_views.py bare `except: pass`).
- **Canonical:** settings.py as sole fallback source + one `email_service_base_url()` resolver; auth schemes stay per-caller (intentional). **Extraction:** safe for URL/secret resolution. **Confidence:** high.

### Mapping 3 extension (pass 9, for the record)
- Fifth/sixth semantic variants of automated task creation: `automations/actions.py:3080–3324` `_create_task_action` (N-task fan-out + `task_group_id` at 3255, `workflow_execution_id` idempotency at 3081–3099) and `Tasks/tasks.py:135–194` `generate_instances_for_rule` (rule-driven fan-out). Distinct shapes, not copies — record under Mapping 3 as variants.

### Mapping 30 — Lead `source` whitelist + workflow default source value
- **Operation:** validate/normalize lead `source` and supply workflow default.
- **Implementations:** `automations/actions.py:864–878` `_create_nurture_record` local `valid_sources` (11 values, missing `call-agent`; invalid → silent coerce `"other"`); `TreatmentPlan/models.py:3266–3276` `Nurture` field choices (12 values incl. `call-agent`); `TreatmentPlan/models.py:2110–2124` `Intake.SOURCE_CHOICES` (**divergent**: snake_case `workflow_automation`, adds `call-agent`, `online_booking_abandoned`); `automations/actions.py:952–953` `_create_intake_record` writes default `"Workflow Automation"` with **no validation** — an out-of-choices value persisted into Intake.
- **Risk:** downstream source targeting/grouping sees unrecognized `"Workflow Automation"`; new source needs 3+ edits. **Canonical:** model `SOURCE_CHOICES`/field choices; actions.py validates against model choices and uses `workflow_automation` for Intake. **Extraction:** safe (constant alignment). **Confidence:** high.

### Mapping 5 extension (pass 8, for the record)
- Additional inbound `X-Workflow-Secret` checks: `messaging/views/message_views.py:2280, 3660, 3734, 3790, 3871`; `UserAuthentication/email_address_provisioning.py:17`.

### Mapping 29 — Post-send Stripe SMS usage metering (best-effort `record_sms_usage` enqueue)
- **Operation:** after persisting an outgoing SMS, best-effort report metered usage via `usage.tasks.record_sms_usage` in a local-import + try/except + logger.warning block.
- **Implementations (5 sites, 4 files):** `automations/actions.py:1906–1914` (async `.delay`); `Tasks/tasks.py:315–319` (async); `messaging/views/message_views.py:294–302` and `1113–1118` (**synchronous in-request**, warning strings still say "queue"); `messaging/models.py:1087–1092` (`SMSMessage.save` — synchronous, meters `self.direction` → **only lane metering inbound SMS**; swallows with bare `pass`).
- **Risk:** billing consistency — sync lanes block requests on Stripe/Celery; inbound metering only in the model lane; metering-rule changes need 5 edits. **Canonical:** `record_sms_usage_best_effort(practice_id, direction)` wrapper in `usage/tasks.py`, async default. **Extraction:** safe. **Confidence:** medium-high.

### Mapping 28 — Dentally payload → Patient contact-fields pipeline (parse loop + rejection policy)
- **Operation:** raw Dentally payload → `first/last/email/phone_number/secondary_phone_number/country_code` via `pick_dentally_phone` per field, first parsed country wins, email "None"/""→None, then rejection rules ("all phones invalid", "no email and no phone").
- **Implementations:** `automations/actions.py:485–581` (full variant: `build_international_phone` branch 517–523, 15-digit guard 526–534, rejection → issue + HTTP 400 547–581); `dentallyIntegration/tasks.py:697–825` (near byte-identical, rejection raises Exception 815–825, live via beat schedule tasks.py:1648); `dentallyIntegration/views/dentally_views.py:837–909` `bulk_import_patients` (**divergent: no `build_international_phone` branch, no 15-digit rejection** — >15-digit garbage accepted, national-format numbers silently dropped). Fourth lane `dataQuality/patient_import_utils.py:24–31` properly uses canonical `extract_dentally_phone_fields`.
- **Note:** `TreatmentPlan/utils/phones.py:553–570` docstring asserts all three lanes "join via build_international_phone and reject over 15 digits" — **false for the views lane**; downstream has drifted from its own documented contract.
- **Canonical:** `dentally_payload_contact_fields(practice, payload) -> (patient_data, rejected_phones)` in `TreatmentPlan/utils/phones.py`, parameterized rejection policy. **Extraction:** confirm views-lane missing branches are bugs, then safe. **Confidence:** high.

### Mapping 26 — Lead contact-field ingest normalization (email strip+lower, phone split-form)
- **Operation:** normalize inbound lead contact fields — `normalize_email` + `parse_phone_number(phone, practice.default_country_code)` → `(email, phone_number, secondary, country_code)` canonical split form.
- **Implementations:** `automations/actions.py:145–170` `_normalized_lead_contact_fields` (superset; primary country wins over secondary); `TreatmentPlan/models.py:4898–4922` `_normalized_original_contact` (near-copy, inputs not stripped); `onlineBooking/tasks.py:117–132` (inline copy; **divergent: computes then discards `country_code`, so intake phone stored without country code**).
- **Canonical:** `normalize_lead_contact_fields(practice, data)` in `TreatmentPlan/contact/sync.py`. **Extraction:** safe. **Confidence:** high.

### Mapping 27 — Identity-carry Person resolution for new records (#47 rule)
- **Operation:** when recreating/carrying a record for a known human, resolve the Person; return None rather than guess or cross practices.
- **Implementations:** `automations/actions.py:99–142` `_carried_person` (**no `merged_into__isnull` guard — can carry a merged/absorbed Person**); `TreatmentPlan/models.py:4924–4940` `_stamped_person` (guard present; project convention per `TreatmentPlan/contact/signals.py:102`, `channel_owner.py:53`). Both docstrings cite finding #47.
- **Canonical:** `carry_person_for_record(...)` / `carry_person_for_id(...)` in `TreatmentPlan/contact`, models.py semantics generalized. **Extraction:** safe (add guard to actions lane). **Confidence:** medium-high.

### Mapping 25 — Dentally lead duplicate-patient match (name + email/phone composite) with divergent merge policy
- **Operation:** find existing Patient by `first_name__iexact + last_name__iexact` scoped to practice, refine via `email__iexact`, fall back to phone Q-match.
- **Implementations (3 copy-paste lanes + 1 variant + 1 bypassed canonical):** `automations/actions.py:602–631` (auto-MERGES — overwrites existing patient with Dentally payload via partial serializer); `dentallyIntegration/tasks.py:841–885` (never merges — appends to `possible_duplicates` for review; comment at 824–836 documents deliberate no-auto-merge policy); `dentallyIntegration/views/dentally_views.py:949–997` (REJECTS import into `results["failed"]`); variant `dentallyIntegration/tasks.py:1048–1061` (all-at-once exact match, conflict_details only); bypassed canonical `TreatmentPlan/journey/mixins.py:31` `match_existing_patient` (email-first, own-Person guard, lookup_key phone key — none of the three lanes call it).
- **Risk:** identical match → silent overwrite vs flag vs reject per lane; all three Dentally blocks use bare `canonical_phone_e164` (see Mapping 20) and lack the family/sibling guard, so a household-shared email merges a spouse in the workflow lane. **Canonical:** `find_dentally_duplicate(practice, ...) -> (patient|None, matched_by)` with per-lane merge-policy parameter, lookup_key phone key. **Extraction:** requires a merge-policy design decision. **Confidence:** high.

### automations additional intra-file/borderline exclusions (do not re-report)
- TreatmentPlan status whitelist duplicated `automations/actions.py:1226–1235` ↔ `TreatmentPlan/views/conversion_views.py:1023–1034` (silent coerce to "pending" vs 400) — candidate constant `TreatmentPlan.VALID_STATUSES`.
- actions.py:47–48 double-import shadowing of `parse_phone_number` — hygiene, not logic.

### Mapping 23 — Practice SMS readiness gate
- **Operation:** resolve `PracticeSMSConfiguration`, gate on `is_active` + `phone_number_status == "active"` + configured number before SMS send.
- **Implementations:** `automations/actions.py:1782–1800` (403 per reason); `Documents/utils/notifications.py:120–137` (ValueError; docstring admits mirroring message_views); `messaging/views/message_views.py:190–215` and `942–957` (ValidationError; **auto-creates pending config on DoesNotExist** at 215); `Tasks/tasks.py:275–291` (silent skip).
- **Risk:** error channel and missing-config side effect diverge (read-only vs auto-create vs silent); readiness rule edits need 4+ changes. **Canonical:** `PracticeSMSConfiguration.is_sms_ready(practice) -> (bool, reason)`. **Extraction:** safe. **Confidence:** high.

### Mapping 24 — Practice sending-identity resolution for EmailService (domain_id + from_email with inbox fallback)
- **Operation:** pick domain_id/from_email via `get_domains()` / `get_or_create_practice_inbox()`.
- **Implementations:** `automations/actions.py:2092–2108` (arbitrary `domains[0]`; final fallback hardcoded `noreply@app.pathway.dental` — can send from unowned domain); `messaging/views/message_views.py:1725–1785` (practice-selected `is_selected=True` domain first; 500 on inbox-domain mismatch); `messaging/views/domain_views.py:80–100` (same inbox→domain matching snippet re-rolled).
- **Canonical:** `resolve_practice_sender(client, practice)` in `messaging/email_service.py` with strict/non-strict mode. **Extraction:** requires a fallback-policy decision, then safe. **Confidence:** medium-high.

### Out-of-area observations for later passes (recorded to avoid re-flagging)
- Family-household relink lookup duplicated across `TreatmentPlan/views/conversion_views.py:44–70, 300–330, 620–650` and `TreatmentPlan/views/intake_views.py:355–375`; bare `email__iexact` lead-match sites in those files → TreatmentPlan pass.

### Mapping 21 — Clinician/practitioner eligibility predicate ("who is a dentist at this practice")
- **Operation:** select practice members eligible as clinicians via `role == "dentist" OR clinician-flag`.
- **ORM copies:** `UserAuthentication/views/practice_team_views.py:298,326,389` (de facto source of truth); `Invoices/services/practitioner_matching.py:57` (docstring admits mirroring); `HR/views/rota_views.py:1844`; `onlineBooking/views.py:805`; `Invoices/views/invoice_views.py:2218–2223`.
- **Divergent Python variants:** `UserAuthentication/alert_email_utils.py:271–272` (`role=="admin"` gate — role-string check); `automations/actions.py:3224–3238` (`is_practice_admin and is_clinician` — staff clinicians fall out of BOTH all_clinicians and all_non_clinicians task sets). `compliance/views.py:239–256` documents the convention in a third shape.
- **Risk:** load-bearing disagreement on staff-clinicians and admin detection. **Canonical:** shared selector `clinician_relationships(practice)` in UserAuthentication. **Extraction:** requires design decisions (which semantics is correct). **Confidence:** high.

### Mapping 22 — Resolve Patient from Dentally identity (meta_data id/uuid/account_id)
- **Canonical:** `TreatmentPlan/services/journey_touchpoints.py:69–99` `patient_for_dentally_id` (string+number id forms; ambiguity → None + report).
- **Divergent:** `TreatmentPlan/views/patient_views.py:232–258` `_resolve_by_dentally` (uuid-first ordering, ambiguity → `.first()` — can silently bind arbitrary match); `automations/actions.py:464–475` (delegates id to canonical, adds unique uuid + account_id fallbacks).
- **Risk:** ambiguity handling and key coverage differ per lane. **Canonical path:** extend `patient_for_dentally_id` with uuid/account_id keys + explicit ambiguity policy. **Extraction:** requires design decisions. **Confidence:** medium-high. (Distinct from Mapping 9 — Dentally-id identity, not email/phone/name.)

### Mapping 19 — Consent-reminder SMS "gate + send" sequence
- **Operation:** fetch SigningRequest, gate on resendiability, build body, call `send_consent_sms`.
- **Implementations:** `automations/actions.py:1482–1598` (denylist gate `TERMINAL_CONSENT_STATUSES` at 1381/1528, practice-scoped fetch 1517–1525, suppress on ValueError); `Documents/tasks.py:323–365` (allowlist gate `not in ("sent","viewed","pending")` at 340, unscoped fetch 333–335, retrying task max_retries=3).
- **Risk:** gates diverge — a new non-terminal status is sendable from the workflow endpoint but skipped by the task; double-messaging vs suppression inconsistency. **Canonical:** the Celery task; the endpoint should delegate via `.delay(...)` as its email sibling already does (actions.py:1449–1454). **Extraction:** safe. **Confidence:** high.

### Mapping 20 — Phone contact-match Q predicate (raw column OR ContactChannel canonical_value)
- **Operation:** `Q(phone_number=<raw>) | Q(person__person_channels__channel__kind=PHONE, ...canonical_value=<derived>)` duplicate-contact check.
- **Implementations:** `automations/actions.py:80–96, 380–388, 613–625, 976–990, 1096–1107` (5 copies in one file); `TreatmentPlan/views/patient_views.py:389–396, 620–636`; `TreatmentPlan/journey/mixins.py:93–108`; `TreatmentPlan/views/intake_views.py:1537–1550`; adjacent: `dentallyIntegration/tasks.py:865`, `dentallyIntegration/views/dentally_views.py:985`, `onlineBooking/services.py:389` (inside Mapping 9's scope).
- **Divergence (load-bearing):** TreatmentPlan sites derive the channel key via `ContactChannel.lookup_key(practice, PHONE, phone, country_code)` (audit #7 fix); automations dup-check branches still use bare `canonical_phone_e164(phone, country_code)` (actions.py:80, and the shape called out in actions.py:1092–1095 comment) — same number can be flagged duplicate in one lane and silently created in another.
- **Canonical:** `ContactChannel.phone_match_q(practice, phone, country_code)` helper (lookup_key-based). **Extraction:** safe. **Confidence:** medium-high (adjacent to but distinct from Mapping 9's resolver family).

### automations intra-file repetitions (below two-file bar, do not re-report)
- Practice-fetch/"Practice with id X not found" scaffold (~10 sites in actions.py); consent endpoints' shared fetch/gate scaffolding; `_update_*` allowed-fields loops (2324–2547); `_serialize_*` dicts (2731–2860); `_query_*` handlers (2863–3017); `valid_priorities` fallbacks (860, 958, 1221 — semantics differ from Tasks/views.py:931 and intake_views.py:1202 whitelists).

### Mapping 17 — Audit session folding with `merged_field_diffs` merge rule
- **Operation:** group flat audit events into sessions keyed by (actor, entity_type, entity_id, action_family) within a 10-minute window, merging `field_diffs` and collecting children.
- **Copy-paste pair:** `Invoices/views/finance_audit_views.py:66–138` (merge rule 102–109); `HR/views/audit_views.py:63–126` (merge rule 95–99, character-identical logic, identical docstrings).
- **Divergent variant:** `activityLog/views.py:812–828` (`_build_sessions`) — normalizes `old/new`→`before/after`; on collision overwrites both before (earliest) and after (latest), whereas finance/HR keep the FIRST event's `after`.
- **Canonical:** shared `fold_into_sessions(events, key_fn, window)` + `normalise_diff()` in `activity_change_utils.py`. **Extraction:** requires design decisions (which merge rule is correct — they disagree on collision semantics). **Confidence:** high.

### Mapping 18 — Person-scoped NoteHistory base query in activityLog (+ redundant identity re-resolution)
- `activityLog/views.py:420–424` (`patient_activities`) and `activityLog/views.py:892–908` (`patient_audit`) — same three-clause scope (`filter(practice=practice).filter(person_id__in=...).select_related("created_by")`); audit adds date/search filters and ordering.
- **Defect:** `patient_audit` calls `resolve_target_person_ids(patient_id, practice)` twice (views.py:644 and 894), redundantly re-running contact-identity ambiguity resolution per request.
- **Canonical:** `notes_for_person_ids(practice, person_ids)` selector; reuse the person-id set from `_get_base_queryset`. **Extraction:** safe. **Confidence:** high.

### Pass-2 extension of Mapping 14
- `activityLog/views.py:426–447` `_note_item` inline actor formatting — a fourth instance of the already-mapped actor-name bypass family (mapped set previously listed views.py:742, 985, serializers.py:431).

### Mapping 13 — ActivityLog write bypass (raw `ActivityLog.objects.create` mirror)
- **Operation:** write audit/mirror ActivityLog row best-effort (try/except + logger), actor = user if authenticated else system.
- **Canonical:** `activityLog/models.py:480` `ActivityLogHelper.log`. **Copies:** `patient_accounts/views.py:137`; `Appointments/views.py:105` (`_log_appointment`, byte-similar pattern incl. `getattr(patient, "person", None)`); `marketingBroadcast/form_audit.py:52,65` (documented divergent variant — no-person events must not raise; keep separate or absorb as `best_effort=True` mode).
- **Divergence:** helper raises on missing person; copies silently drop; marketingBroadcast truncates description to 500.
- **Extraction:** safe (add best-effort tolerant mode to helper). **Confidence:** high.

### Mapping 14 — Inline actor-name formatting (bypassing `get_display_name_for_user`)
- **Canonical:** `TreatmentPath/utils.py` `get_display_name_for_user` (activityLog/models.py:311–315, 427–431 docstrings assert it is the one helper).
- **Reimplementations (4+ copies):** `activityLog/views.py:742–747` (`_actor_name`); `activityLog/views.py:985–991` (`_get_actor_name`); `activityLog/serializers.py:431–437` (`get_actor_name`); `activityLog/views.py:426–447` (`_note_item`); `medicalHistory/audit.py:24` (inline first/last join with email fallback).
- **Risk:** none apply the superuser→"System Admin" anonymity rule — superuser actor renders as real name in patient audit tab/CSV but "System Admin" elsewhere. **Extraction:** safe. **Confidence:** high.

### Mapping 15 — entity_type → human-label maps (three divergent copies)
- `activityLog/serializers.py:12–24` `ENTITY_TYPE_MAP` (10 entries); `activityLog/views.py:297–307` local `entity_type_map` (9 entries); `activityLog/views.py:706–720` `ENTITY_TYPE_LABELS` (12 entries; adds invoice/ledger/task keys, drops archive).
- **Risk:** new entity types need up to 3 edits; label fallback differs between feed and grouped audit. **Canonical:** promote serializers' `ENTITY_TYPE_MAP` merged to superset into models/utils. **Extraction:** safe. **Confidence:** high.

### Mapping 16 — ActivityLog summary derivation (`description` else `metadata["summary"]`)
- `activityLog/views.py:786–792` (with entity-label fallback); `activityLog/views.py:980–983` (`_get_summary`); `activityLog/serializers.py:439–442` (byte-identical to _get_summary).
- **Divergence:** only _build_sessions has final entity-label fallback → blank vs labeled summary in flat vs grouped audit. **Canonical:** `ActivityLog.summary_label` helper. **Extraction:** safe. **Confidence:** high.

### Mapping 1 — Outgoing SMS delivery (Twilio create + SMSMessage row)
- **Operation:** send one SMS via Twilio and persist an `SMSMessage` audit row.
- **Implementations:** `automations/actions.py:1716` (raw Client at 1885, own error-code mapping ~1935–1955, no status_callback); `dentallyIntegration/recall_automation.py:555,670,676` (via `_Delivery`, no status_callback, `source_type="recall_automation"`); `dentallyIntegration/confirmation_automation.py:286,325,338` (via `_Delivery`, WITH status_callback); `messaging/views/message_views.py:276,1085` (inline, module client at 144); `messaging/views/template_views.py:1655` (client at 89); `TreatmentPlan/journey/dispatch.py:219`; `dentallyIntegration/views/recall_views.py:1900` (ad-hoc clients from auth header at 1762–1771).
- **Classification:** canonical = `recall_automation._Delivery` extended with status_callback + `source_type` param; others are wrappers/reimplementations.
- **Divergence:** only confirmation wires status_callback (others freeze at "sent"); only actions.py maps Twilio error codes; client construction differs (global SID vs practice config vs per-request JWT).
- **Scope:** shared service. **Risk:** delivery-status tracking and error taxonomy drift per caller. **Extraction:** requires design decisions (auth context). **Confidence:** high.

### Mapping 2 — Outgoing email delivery (EmailServiceClient + EmailMessages row)
- **Operation:** send one email via Go EmailService and persist an `EmailMessages` row.
- **Implementations:** `recall_automation.py:555–655` (create at 647); `confirmation_automation.py:355,391`; `automations/actions.py:1988,2143,2176,2201`; `messaging/views/message_views.py:1617,800,1731` + `messaging/email_service.py:189,329,382,407`; `recall_views.py:1762` + manual token extraction also in `messaging/views/domain_views.py:52`; `marketingBroadcast/delivery_reporting.py:119,201` (parallel `MarketingEmailServiceClient` of same Go contract).
- **Canonical:** `recall_automation._Delivery` email path, parameterized by source_type. **Scope:** shared service. **Extraction:** risky (multiple auth contexts). **Confidence:** high.

### Mapping 3 — Automated team-task creation ("assign-to-all" Task handoff)
- **Operation:** create `Tasks.Task` for an automation (`title[:255]`, priority whitelist fallback, `is_assigned_to_all = no assignee`, `(ok, reason)` contract).
- **Implementations:** `recall_automation.py:724,734`; `confirmation_automation.py:880,893` (near byte-identical copy); `TreatmentPlan/journey/dispatch.py:337` (priority default "low" vs "medium"); `marketingBroadcast/form_handoffs.py:379` (intentionally divergent: individual assignee, working-day due date, action ledger — keep separate).
- **Scope:** `create_automation_task(...)` helper in `Tasks` (recall's version most complete). **Extraction:** safe for the three automations. **Confidence:** high.

### Mapping 4 — Due-enrollment claim/advance loop (select_for_update scanner)
- **Operation:** scan active enrollments `next_due_at__lte=now`, claim under `select_for_update(skip_locked=True)` per-row transaction, fire one step.
- **Implementations:** `recall_automation.py:816–870` (lock 856, ordered scan); `confirmation_automation.py:1105–1150` (lock 1123–1128, safer `of=("self",)`, throttle stats, NO order_by → starvation risk); `TreatmentPlan/journey/automation.py:439–478`.
- **Scope:** shared generic `claim_due_enrollments(qs, now, ...)` context manager (recall loop + confirmation's `of=("self",)`). **Extraction:** requires design decisions (kill-switch field names differ). **Confidence:** high.

### Mapping 5 — Internal Go→Django webhook shared-secret auth
- **Operation:** check `X-Workflow-Secret` against `WORKFLOW_SERVICE_SECRET`.
- **Implementations:** `automations/actions.py:57,173–186` (decorator); `messaging/views/spam_views.py:84`; `marketingBroadcast/views/webhook_views.py:231–235` (docstring admits mirroring spam_views).
- **Risk:** all three replicate hardcoded fallback secret `"workflow-service-secret-key"` and use non-constant-time `!=` (security issue). **Canonical:** the decorator + `hmac.compare_digest`, drop default secret. **Extraction:** safe. **Confidence:** high.

### Mapping 6 — Client IP extraction (X-Forwarded-For first hop)
- **Canonical:** `utils/request_meta.py:31` `client_ip` (docstring records eleven prior copies; `marketingBroadcast/views/preferences_views.py:24–36` already a wrapper).
- **Live bypass:** `marketingBroadcast/form_abuse.py:98–116` — two verbatim private copies (split at 103 and 113).
- **Extraction:** safe (import the helper). **Confidence:** high.

### Mapping 7 — Manual inline pagination (bypassing DRF pagination classes)
- **Canonical:** `TreatmentPath/pagination.py:6` `StandardResultsSetPagination` (marketingBroadcast subclasses are proper — not flagged).
- **Reimplementations:** `dentallyIntegration/views/recall_views.py:3553–3559` (unguarded `int()` → 500) and second variant at 3843–3847; `marketingBroadcast/views/segment_views.py:171–176` + `_paginated_preview` 85–95; `TreatmentPlan/views/patient_views.py:2303–2310`; `TreatmentPlan/views/patient_journey_views.py:75–83`.
- **Divergence:** max size 100/200/500; malformed input 500 vs silent default; hand-built envelopes. **Extraction:** safe. **Confidence:** high.

### Mapping 8 — Message session-id derivation (md5 key)
- **Canonical owner:** `messaging/models.py:319–368` (`get_or_create_session`, md5 at 364, legacy fallback).
- **Reimplementation:** `TreatmentPlan/contact/sync.py:77–115` (`_build_session_identifier`, md5 at 84 and 108; also re-does phone normalization).
- **Risk:** documented past session_id format bug; a change in either derivation forks sessions. **Fix:** `MessageSession.build_session_id(...)` classmethod. **Extraction:** safe. **Confidence:** high.

### Mapping 9 — Patient identity matching (email/phone/name → Patient)
- **Canonical:** `TreatmentPlan/contact/identity_resolution.py:81,168`.
- **Variants:** `onlineBooking/services.py:341–405` `_match_patient` (deliberately conservative ambiguity→None; docstring catalogs audit findings #18/#19/#28/#34); `automations/actions.py:1826–1840` (N+1 scan with un-indexed `phones_match` — weak third variant).
- **Scope:** canonical business rule — `resolve_patient_identity` parameterized by strictness. **Extraction:** requires design decisions (differences are load-bearing). **Confidence:** high.

### Mapping 10 — Resolve Person from contact-scope query params
- **Operation:** map `person_id|patient_id|intake_id|nurture_id|treatment_plan_id` (or `contact_id`) → practice-scoped Person.
- **Implementations:** `TreatmentPlan/views/notes_views.py:33–69` `_resolve_person` (five id kinds; **correction: earlier cited as `Notes/views/notes_views.py`, which does not exist — the file is in TreatmentPlan**); `TreatmentPlan/views/patient_outstanding_views.py:56–71` (`contact_id`/`patient_id` only, different param names); **(pass-3 extensions)** `activityLog/views.py:377–394` (inline Intake/Nurture → `person_id` union blocks — the second same-shape implementation that makes the family cross-file); the Intake/Nurture branch of the same `TreatmentPlan/views/notes_views.py:49–59`.
- **Note:** shared helper `person_ids_for_patient_or_person_id` (TreatmentPlan/contact/selectors.py) covers only patient_id→person, which is why lead-stage scopes are re-rolled per view. **Canonical home:** extend selectors with `person_for_scope_ids(practice, person_id=, patient_id=, intake_id=, nurture_id=, treatment_plan_id=)`.
- **Risk:** API surface forked (`person_id` vs `contact_id`). **Extraction:** safe. **Confidence:** medium-high.

### Mapping 11 — Best-effort Activity append to earliest ActivityLog session
- **Canonical:** `TreatmentPath/record_events.py:744–748,859–863`, wrapper `emit_activity_best_effort` at 919 (confirmation_automation uses it correctly).
- **Reimplementations:** `dentallyIntegration/views/appointment_confirm_views.py:471–486`; `dentallyIntegration/views/dentally_views.py:2045–2060` — hand-rolled query + create, different `is_system_generated`, skip metadata/entity-linking.
- **Extraction:** safe (call the wrapper). **Confidence:** high.

### Mapping 12 (minor) — Duplicated `canon_phone`/`norm_name` wrappers in dedupe commands
- `TreatmentPlan/management/commands/dedupe_report.py:29–51` and `dedupe_persons.py:60–82` — byte-identical 23-line `canon_phone` wrapper (both delegate to `canonical_phone_e164`) plus duplicated `norm_name`.
- **Risk:** merge-key policy edited twice or commands disagree. **Fix:** single wrapper in `TreatmentPlan.utils.contact_keys`. **Extraction:** safe. **Confidence:** high.

### Exclusions (do not re-report)
- `TreatmentPlan/utils/phones.py` normalization: consolidation complete; `messaging/models.py:208` is a documented compatible wrapper; old dedupe-command copies fixed (see Mapping 12).
- `marketingBroadcast/pagination.py`: proper subclasses of shared class.
- `messaging/views/permissions.py FeatureAccessPermission` + `utils/practice_mixins.py PracticeAccessMixin`: correctly reused (60+ sites); no divergent practice-scope reimplementation beyond Mapping 10.
- `dentallyIntegration/confirm_utils.py` HMAC tokens and `onlineBooking/services.py:408–409` hold locking: single implementations each.
- `TreatmentPath/assistant_proxy.py:74` sets X-Forwarded-For (forwarding, not extraction).
- Workflow event data-payload conventions: `TreatmentPath/record_events.py:363+` `_build_workflow_data` (bare keys) vs `Documents/views/signing_views.py:105–123` (patient-prefixed keys + template snapshot) — judged intentionally distinct payload schemas for different entity kinds, not duplication.
- automations `_apply_conditions` (actions.py:2646–2728) vs `marketingBroadcast/segment_engine.py`: both translate rule dicts to Q objects but with disjoint operator DSLs and no shared contract — generic idiom, below the bar.
- `workflow_execution_id` idempotency check (actions.py:3082–3099): single implementation project-wide.
- dataQuality: `EmailServiceGo/internal/dentally/scheduler/name_similarity.go` is a documented Go port of Django's `_name_similar`, but the cited Python original exists only in a stale worktree — single live implementation, dead counterpart (dead code, not a mapping). The dataQuality frontend `CLASSIFIER_CONFIG`/TS union is a typed API-contract mirror of `models.py CLASSIFIER_CHOICES`, below the logic bar. The `perfect-pixel-playground-project` frontend is a whole-repo structural fork — repo-level observation, not an area logic family.
- `activityLog/models.py:544–566` `ActivityLogHelper.get_entity_activities` has zero callers project-wide (the live query is inline at `activityLog/views.py:179–194`). Dead code — cleanup candidate, not a repeated-logic mapping.
- activityLog single-implementation logic confirmed singular: `_field_diffs_to_str` (views.py:966), relative-time formatting (serializers.py:365–386), `ACTIVITY_TYPE_TO_FAMILY` (views.py:725), HTTP-300 ambiguity response (views.py:559–575), CSV export scaffolding (views.py:1003–1023).

## Area status

| Area | Clean pass 1 | Clean pass 2 | Clean pass 3 | Status |
|---|---:|---:|---:|---|
| activityLog | ✓ | ✓ | ✓ | complete |
| automations | ✓ | ✓ | ✓ | complete |
| dataQuality | ✓ | ✓ | ✓ | complete |
| dentallyIntegration | ✓ | ✓ | ✓ | complete |
| marketingBroadcast | ✓ | ✓ | ✓ | complete |
| messaging | ✓ | ✓ | ✓ | complete |
| Notes | ✓ | ✓ | ✓ | complete |
| onlineBooking | ✓ | ✓ | ✓ | complete |
| TreatmentPlan | ✓ | ✓ | ✓ | complete |

## Implementation-readiness validation (2026-09-13)

### Verdict

This document is supported as a repeated-logic inventory, but it is **not approved as a batch implementation plan**. The mappings identify real duplication and divergence; they do not, by themselves, establish that every proposed extraction can be applied unchanged without regressions.

Treat each mapping as an independent change requiring its own behavior decision, regression tests, and verification. Do not interpret `Extraction: safe` as permission to combine mappings or skip those gates. In particular, a mapping described as safe while also naming a policy, key-vocabulary, fallback, identity, transactionality, idempotency, or authorization decision is only safe **after** that decision is resolved and encoded in tests.

### Validation performed

- Rechecked the current Django implementation on branch `dedup-normalization-unification` at commit `1e591f23` (`2026-09-12`).
- Directly reconfirmed representative high-risk and recent findings, including the permission-sensitive contact masking (M139), Person merge/repoint/undo split (M138), non-cryptographic code generation (M137), public guide payload divergence (M127), treatment-plan pricing divergence (M128), and online-booking schedule-resolution divergence (M99).
- Confirmed the M140 endpoint scaffold remains duplicated. This is a low-risk extraction candidate, but parity tests must cover authentication, practice scope, admin authorization, `dry_run` default/coercion, domain-error mapping, and response shape before consolidation.
- Confirmed a numbering integrity defect: Mapping 133 is absent. Existing mappings must not be renumbered; reserve 133 as intentionally unassigned unless the original missing entry can be recovered from source history.
- Ran `python manage.py test TreatmentPlan.tests.test_guide_payload TreatmentPlan.tests.test_dentally_plan_import --keepdb`: **14 tests passed**.
- Attempted the broader targeted run including `TreatmentPlan.tests.test_journey_business_workflows`: collection failed because that test module imports the missing `TreatmentPlan.journey_automation`. This is a pre-existing test-baseline blocker, not a failure caused by this audit document.

### Corrections and clarifications

- **M127:** the non-registered-patient failure condition applies to both public temporary-link functions currently reading `treatment_plan.patient.practice`: `get_treatment_plan_by_token` and `get_treatment_plan_public`. Consolidation onto `build_guide_payload` must cover both routes and preserve their authorization, expiry, and view-count behavior.
- **Count wording:** `140` is the highest assigned mapping number, not the number of distinct numbered families currently present. The distinct count is 139 until Mapping 133 is recovered or deliberately added.
- **Completion wording:** the area table proves completion of the discovery protocol only. It does not mean the proposed canonical implementations have been approved, implemented, or regression-tested.

### Required gate before applying any mapping

1. Re-resolve every cited implementation and caller against the commit being changed; line numbers are evidence locators, not stable identifiers.
2. State the intended invariant and choose the behavior for every documented divergence before extracting shared code.
3. Add characterization tests for each existing lane, including authorization and practice scoping where applicable.
4. Implement one mapping (or one tightly coupled family) at a time; do not apply the register as a single refactor.
5. Run the focused tests plus the owning app's suite. Repair the `test_journey_business_workflows` import baseline before using that module as merge/journey regression evidence.
6. For security-, identity-, payment-, scheduling-, or public-endpoint mappings, require an explicit manual review in addition to passing tests.

### Readiness interpretation

- **Mechanical candidate:** behavior is already identical and the change only redirects callers to one implementation. Still requires parity tests.
- **Decision required:** implementations differ in observable behavior; no consolidation should begin until the intended contract is selected.
- **High-risk correction:** the mapping concerns authorization, practice boundaries, identity, payments, public endpoints, scheduling, encryption, or irreversible state transitions. Treat it as a bugfix/migration with rollback planning, not as routine deduplication.

Overall status: **inventory validated with the corrections above; batch application rejected; phased implementation conditionally approved only through the required gate.**

## Canonical logic candidates

Decisions on which implementation should become the source of truth per family. Status: `agreed` / `proposed` / `needs decision`.

| Logic family | Candidate source of truth | Competing implementations | Decision/status |
|---|---|---|---|
| SMS delivery (M1) | `recall_automation._Delivery` + status_callback + `source_type` param | actions.py raw client; messaging views inline; recall_views ad-hoc; dispatch.py | proposed — needs auth-context design |
| Email delivery (M2) | `recall_automation._Delivery` email path, parameterized | actions.py; message_views; email_service.py; MarketingEmailServiceClient | proposed |
| Webhook secret auth (M5) | `automations.actions.workflow_auth_required` decorator + `hmac.compare_digest`, no default secret | spam_views inline; webhook_views inline; message_views ×5; provisioning | proposed — remove hardcoded fallback secret first |
| Client IP extraction (M6) | `utils/request_meta.client_ip` | form_abuse ×2; custom_webhook_views; intake_views; preferences (converted) | agreed — mechanical import |
| Pagination (M7) | `StandardResultsSetPagination` / one manual helper | recall_views ×2; segment/campaign/archive views; patient_views; patient_journey_views | proposed |
| Session-id derivation (M8) | `MessageSession.build_session_id(...)` classmethod (new) | contact/sync.py inline | proposed |
| Patient identity matching (M9) | `identity_resolution.resolve_patient_identity` parameterized by strictness | onlineBooking `_match_patient`; actions.py N+1 scan | needs decision (ambiguity rule per lane) |
| Person/scope-id resolution (M10) | `TreatmentPlan/contact/selectors.person_for_scope_ids` / `record_for_scope_ids` | notes_views; patient_outstanding_views; activityLog ×2; household_views ×2; contact_merge_views | proposed |
| Automation task creation (M3) | `Tasks.create_automation_task(...)` helper | recall/confirmation/dispatch copies; form_handoffs stays separate | agreed for 3 lanes |
| Enrollment claim scanner (M4) | `claim_due_enrollments(qs, now, ...)` context manager | recall/confirmation/journey scanners | needs decision (kill-switch fields differ) |
| Twilio error taxonomy (M34) | `twilio_error_to_user_message(e, ...)` pure helper | actions.py; message_views (superset) | proposed |
| SMS to-number E.164 (M32) | `resolve_sms_to_number(phone, practice, ...)` | 5 naive `f"+{phone}"` lanes | proposed — naive lanes are bugs |
| OTP mint + hash (M100, M137) | `generate_otp` (secrets) + `store_hashed_otp` (bcrypt) via `issue_hashed_code` | service.py random; serializers inline; TreatmentPlan codes.py; BookingEmailOTP SHA-256; SixDigitVerificationCode plaintext | needs decision (hash-scheme + RNG migration) |
| Encryption gate (M139) | `EncryptContactFieldsMixin` / extracted `encrypt_contact_fields(...)` with `user` | treatment_plan serializer (missing `user`); patient.py retained copy; messaging twins | proposed — confirm missing-user is a bug |
| Provider-configured LLM calls (M93, M116) | `TreatmentPath.llm_provider` (`complete_text` / `complete_structured`) | 6 Notes functions; email_intake/email_task services; raw-requests lanes (Notes ai_proxy, ChromeExtension, Stock, Invoices) | needs decision (OpenAI→OpenRouter migration) |
| Archive creation (M118) | `Archive.record_for(...)` stamping person_id | 6 model-lane create blocks; journey/mixins (canonical today) | proposed |
| Practitioner schedule (M99) | `Appointments.schedule_resolver` | onlineBooking re-roll (buggier) | agreed — onlineBooking imports resolver |
| Family-household (M36, M117) | `Household.get_or_create_for_person` + `relink_record_to_family_household` | 6 ensure-household sites; conversion/intake relink lanes; Go mirror (align only) | needs decision (Intake Person-move policy) |
| Practice timezone (M58) | `practice_timezone(practice_or_id, default=...)` in UserAuthentication | form_handoffs (Europe/London); financialAnalytics (UTC); onlineBooking ×4 | needs decision (fallback default) |
| Dedupe/identity keys (M12, M131, M135) | `TreatmentPlan/utils/contact_keys` + `phones.py` | split_phone_country; backfill_persons_households; unweld_persons | agreed — mechanical imports |
| PDF scaffolds (M51, M125) | `styled_table_pdf(...)` + `render_branded_pdf(...)` with `_deny_all_url_fetcher` | Stock↔dentally pair; presentation_pdf↔invoice_pdf pair | proposed |
| Audit-log emission (M124) | generic `emit_audit(log_model, ..., fail_mode=...)` | HR / practice_treatment / task / compliance / finance modules + 2 inline diff re-rolls | needs decision (failure policy) |
| Storage subclass (M96) | `private_document_storage(querystring_expire=300)` factory | Notes/HR/Documents×2/compliance/TreatmentPlan classes | agreed — mechanical |
| Merge repoint/undo (M138) | `Person.merge`/`Person.unmerge` with MessageSession term + table registry | contact_merge_views `_repoint_person_records`; raw-SQL undo | needs decision (direct-merge caller behavior) |

## Cross-area dependency map

Which areas produce shared logic and which consume it. Change producers with consumers in mind.

| Logic family | Producing areas | Consuming areas | Shared inputs/outputs | Drift risk |
|---|---|---|---|---|
| SMS/email delivery (M1, M2) | messaging (service contract), dentallyIntegration (`_Delivery`) | automations, messaging views, TreatmentPlan journey, dentallyIntegration, marketingBroadcast (parallel client) | practice/user auth context, SMSMessage/EmailMessages rows, Go EmailService API | High — status_callback, source_type, auth context per lane |
| X-Workflow-Secret auth (M5, M31) | automations (inbound decorator), EmailServiceGo (issuer) | messaging, marketingBroadcast, UserAuthentication, dentallyIntegration, Tasks (inbound checks; outbound URL/secret resolution) | `WORKFLOW_SERVICE_SECRET` header; `EMAIL_SERVICE_URL` | High — hardcoded fallback secret in 7+ files; three fallback ports |
| Client IP (M6) | TreatmentPath/utils | marketingBroadcast, dentallyIntegration, TreatmentPlan webhooks, Notes/public surfaces | request → IP string | Low — mechanical |
| Patient identity (M9, M22, M25, M57) | TreatmentPlan/contact + sole_patient + journey_touchpoints | onlineBooking, automations, dentallyIntegration, marketingBroadcast, messaging | Person/Patient/ContactChannel; Dentally meta_data ids | High — ambiguity rules differ per lane; wrong-person merges/assigns |
| NoteHistory/Activity emission (M11, M13, M108) | activityLog + TreatmentPath/record_events + TreatmentPlan models | dentallyIntegration, patient_accounts, Appointments, automations, marketingBroadcast | person, entity, content_type, best-effort contract | Medium — mirror writers skip metadata/entity-linking |
| Enrollment scanners (M4) | dentallyIntegration (recall/confirmation), TreatmentPlan/journey | Celery beat → deliveries → M1/M2 send lanes | Enrollment querysets, `next_due_at`, kill-switch field | High — safety fixes landed one scanner at a time |
| Contact-identity selectors (M10, M20, M120, M138) | TreatmentPlan/contact + selectors | activityLog, Notes, messaging, onlineBooking, dentallyIntegration, dataQuality, automations | practice-scoped query params, lookup_key/phone_match_forms | High — table inventories and param vocabularies fork |
| Practice SMS/WhatsApp readiness + Twilio ops (M23, M70, M75, M84, M87) | messaging (config, validators, service) | automations, Documents, Tasks, Admin | PracticeSMSConfiguration, PracticeWhatsAppConfig, Twilio clients | Medium — gate semantics and provisioning lanes fork |
| Last-contacted maps (M119) | TreatmentPlan/contact/last_contacted | messaging serializers, TreatmentPlan views/serializers | SMSMessage/EmailMessages by phone/email forms | Medium — widened phone match not propagated |
| OTP/verification codes (M100, M137) | UserAuthentication (canonical), onlineBooking, TreatmentPlan | auth flows, public booking, public treatment verification | code, hash, expiry, consumer | High — 3 RNG schemes, 4 storage schemes |
| Docling-free doc pipelines: PDF artifacts (M51, M125, M129) | TreatmentPath/pdf_branding + per-app renderers | Stock, dentallyIntegration, TreatmentPlan, patient_accounts, Notes, Documents | html → WeasyPrint → *Artifact rows | Medium — SSRF fetcher and idempotency policy differ |
| Household/family (M36, M117, M120) | TreatmentPlan models + contact + selectors | dentallyIntegration (Django + Go mirror), dataQuality, Notes (consent audience) | Person.household, family_id, dismissed_member_ids JSON | High — Python/Go topologies already diverge |
| Import-issue pipeline (M37, M38) | dataQuality (models) + dentallyIntegration (writer) + Go mirror | automations, dataQuality retry_errors, staff UI | DataQualityIssue rows, failure_type vocabulary, Go classifier mirror | High — classifier ordering differs per service |
| WS practice gate + broadcasts (M62, M80) | utils/ws_practice + per-app consumers | TreatmentPlan, messaging, marketingBroadcast, dentallyIntegration consumers | practice group name, event payload schema | Medium — payload key vocabulary forked |
| Compliance/consent exports & masking (M96, M139, M48) | TreatmentPlan/serializers (mixins), per-app storage | messaging, Notes, marketingBroadcast, patient_accounts | encrypt toggle, `Pathway_{hash8}` mask, presigned URLs | High — user-blind gate in one lane; expiry per app |

## Review notes and exclusions

Record safe intentional differences, generated code, dead/quarantined code, and ordinary wrappers here so future agents do not repeatedly report them as new mappings.

## Completion rule

The mapping review is complete only when all nine areas have three consecutive clean passes under the protocol above, the continuous mapping count is current, and every cross-area mapping has implementation evidence or is explicitly marked unresolved.

# Agent prompt: Sweep Django for patient bugs

You are auditing the Django app at /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath for PATIENT IDENTITY bugs.

READ-ONLY INVESTIGATION. Do not modify any file. Report findings only.

Environment: `source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate`, manage.py is in TreatmentPath/. A live copy of production data is in database `prod_control` (use `DB_NAME=prod_control python manage.py shell` or psql) — USE IT to prove or disprove each hypothesis with real data. Never write to it.

CONTEXT — bugs already found in this codebase, so you know the SHAPE of what to look for:

1. NAME SPLIT MISMATCH (just bit a user). A patient stored as first_name="Christopher (Chris)" last_name="Mooney" was looked up with first_name="Christopher" last_name="(Chris) Mooney". The JOINED name is identical but the split differs, so a lookup matching `first_name__iexact` AND `last_name__iexact` separately fails. Any code that splits a full name at the first/last space, or matches on the two fields separately, is suspect.

2. READ-ONLY FIELD SILENTLY DISCARDED. TreatmentPlan serializer declares `patient = serializers.SerializerMethodField()` (read-only) and `patient_id = IntegerField()` (writable). A client sending `patient: 31408` has it silently dropped by DRF, then falls into a fallback path that fails. Look for other writable-vs-read-only mismatches where a client would plausibly send the wrong key.

3. SERIALIZER/MODEL CONTRADICTION. TreatmentPlan serializer deliberately allows creating a plan with no patient ("If neither email nor phone_number is provided, allow creating treatment plan without patient link") but TreatmentPlan.clean() raises "must have either a registered or non-registered patient". Look for other places where a serializer permits what the model forbids.

4. IDENTITY KEY DIVERGENCE. The canonical definitions are TreatmentPlan/utils/contact_keys.py (canonical_email, canonical_name_key, canonical_name_part, canonical_dob) and TreatmentPlan/utils/phones.py (canonical_phone_e164). Any private re-implementation is a bug. Two dedupe commands had their own `canon_phone` that invented a +44 country code — already fixed.

5. Person.resolve MUST receive a `dob` when the caller has one, or it cannot refuse to merge two same-named people with different birth dates.

YOUR TASK — find MORE instances of these classes, and any other patient-identity defect:

a) Every place a full name is SPLIT (partition/split on space) or JOINED then compared. Check whether a nickname/middle-name/double-barrelled name breaks it. PROVE it against prod_control data (e.g. count patients whose first_name contains a space — those are the vulnerable ones).
b) Every patient LOOKUP path: by name, email, phone, id. Which ones can fail to find a patient that exists? Test against real rows.
c) DRF serializers where a plausible client payload key is read-only or differently named from the writable one.
d) Serializer-permits / model-forbids contradictions (grep for clean() raising ValidationError and check the serializers that create those models).
e) Any remaining private normalisation of email/phone/name/dob.
f) Patient or Person lookups missing practice scoping (a cross-practice leak).

For EACH finding report: file:line, the exact defect, a concrete failing example using REAL data from prod_control (names/ids), and severity. Distinguish PROVEN (you reproduced it against data) from SUSPECTED. Do not speculate without evidence — if you cannot prove it, say so explicitly. Prioritise ruthlessly: I want real defects, not style opinions.

# Agent prompt: Sweep Go service for patient bugs

You are auditing the Go service at /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo for PATIENT IDENTITY bugs.

READ-ONLY INVESTIGATION. Do not modify any file. Report findings only. You may run `go build ./...` and `go test` to understand behaviour, and you may write a THROWAWAY test file to probe a function's behaviour — but delete it before finishing.

CRITICAL CONTEXT: this Go service writes DIRECTLY to the same PostgreSQL database as a Django app — it does not call Django. So the same identity rules are implemented TWICE, and drift between them has already caused real damage: on 2026-07-27 an import merged whole families onto one Person because Go matched on contact channel only, with no name or DOB check. 812 Person rows each swallowed a family.

A live copy of production data is in database `prod_control` on localhost:5432 (user mannie). Use psql to prove or disprove hypotheses with REAL data. Never write to it.

The Django counterparts you must compare against:
- TreatmentPathBackend/TreatmentPath/TreatmentPlan/models.py -> Person.resolve, ContactChannel.canonical_key, Person._dob_conflict
- TreatmentPathBackend/TreatmentPath/TreatmentPlan/utils/phones.py -> canonical_phone_e164
- TreatmentPathBackend/TreatmentPath/TreatmentPlan/utils/contact_keys.py -> canonical_email, canonical_name_key, canonical_dob
- A shared fixture pins some of this: TreatmentPlan/tests/person_resolution_fixtures.json and internal/dentally/migration/testdata/person_resolution_fixtures.json MUST be identical.

BUG CLASSES ALREADY FOUND HERE (find MORE of the same shape):
1. Go preferred Dentally's `*_normalized` phone field over the raw one. Dentally's normalisation is WRONG for non-UK numbers (it prefixes +44 onto a number that already has a country code), so foreign patients silently became uncontactable. Fixed via pickUsablePhone.
2. Channel linking was INSERT-only (ON CONFLICT DO NOTHING), so a corrected contact detail left the wrong one attached forever. Fixed via releaseStaleChannels.
3. A name split at the wrong space produces a first/last pair that no longer matches the stored record.

YOUR TASK:
a) Compare Go's identity resolution (internal/dentally/migration/service.go -> linkPatientToPersonAndChannels, dobConflict, ParsePhoneToDialCode, extractPhones) against Django's line by line. Any case where they would resolve DIFFERENTLY is a defect — construct the concrete input that diverges and prove it.
b) Verify the two person_resolution_fixtures.json copies are byte-identical.
c) Find every place Go writes to TreatmentPlan_person, TreatmentPlan_patient, TreatmentPlan_contactchannel or TreatmentPlan_personchannel. For each: can it create a duplicate Person, attach a channel to the wrong Person, or write a non-canonical key?
d) Check pkg/phone, pkg/email, pkg/personname against their Django equivalents for divergent edge cases (empty, nil, whitespace, unicode, 00-prefix, ISO vs dial country codes, invalid numbers).
e) The Go-written mirror tables (recall_patient, daylist_patient, callagent) — do they store names/contacts in a form that could later be matched incorrectly?
f) Any name splitting or joining in Go that could mis-split a nickname or double-barrelled name.

For EACH finding report: file:line, the exact defect, a concrete diverging input proving Go and Django disagree, and severity. Distinguish PROVEN from SUSPECTED — if you cannot prove it, say so explicitly. Prioritise real defects over style.

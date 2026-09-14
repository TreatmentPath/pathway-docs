# Agent prompt: Thorough sweep of unswept apps

Exhaustive hunt for PATIENT IDENTITY defects in areas a previous sweep did NOT cover.

READ-ONLY. Do not modify any file. Report findings only.

Repo: /home/mannie/Desktop/Projects/treatmentpath
Django: TreatmentPathBackend/TreatmentPath (venv at TreatmentPathBackend/venv, manage.py in TreatmentPath/)
Go: EmailServiceGo
Live production copy: database `prod_control` on localhost:5432 user `mannie`. USE IT to prove every claim. NEVER write to it.

A prior audit found 16 defects, written up in docs/PATIENT_IDENTITY_AUDIT_2026-09-06.md — READ THAT FIRST. Do not re-report anything already in it. Your job is to find what it MISSED.

It distilled all 16 into FOUR ROOT PATTERNS. Hunt for more instances of each:

**PATTERN A — a name split at a space position.**
Names are stored as first_name + last_name. Any code doing `.split(" ", 1)`, `partition(" ")`, or building `f"{first} {last}"` then re-splitting, produces a pair that no longer matches the stored record. Also: comparing first_name AND last_name as separate fields, when the two systems disagree about where the split falls. 1,686 patients have a space in first_name.

**PATTERN B — practice scoping present on one path, missing on its sibling.**
Every one found so far holds on create/read and fails on update/lookup. Look for: writable FK fields (`PrimaryKeyRelatedField`) whose queryset is unscoped; `.filter()` on Patient/Person/Intake/Nurture with no `practice=`; `validate_*` methods that check some FKs and not others; update() paths that don't re-validate what create() validated.

**PATTERN C — the reader computes a key differently from the writer.**
Canonical definitions: TreatmentPlan/utils/contact_keys.py (canonical_email, canonical_name_key, canonical_dob), TreatmentPlan/utils/phones.py (canonical_phone_e164), ContactChannel.canonical_key. Look for: case-sensitive email comparison; raw `==` on phones; `canonical_phone_e164` called without default_region when the writer passes one; string-concatenating a country code; hand-rolled normalisation.

**PATTERN D — one Person standing in for two humans.**
Look for: code assigning `record.person = <someone else's person>`; resolution that returns "the first"/"the most recent" record on a Person instead of the specific one asked for; `.first()` on a queryset that can hold several people.

**ALSO hunt this fifth class the prior audit under-covered:**
**PATTERN E — silently discarded input.** A DRF field that is read-only (SerializerMethodField, or a model @property) while a client would plausibly send that key; a writable field with a DIFFERENT name; `except SomeError: pass` swallowing an identity assignment; validated_data keys never read.

APPS TO SWEEP (the prior audit explicitly did NOT cover these):
patient_accounts, medicalHistory, Labs, Notes, Documents, Stock, Invoices, Statistics, Tasks, activityLog, referralProgram, teamChat, productAnalytics, practiceConnection, settings — plus any patient-touching code in compliance and HR. Also re-examine onlineBooking and marketingBroadcast for PATTERN E specifically.

For EACH finding report: severity, file:line, plain-English explanation of what breaks, a CONCRETE example using REAL data from prod_control (ids and names), and the count of affected rows. Mark PROVEN (you demonstrated it with data or by executing code) vs SUSPECTED (control-flow reasoning only) — be rigorous about that distinction; a previous agent's "highest-value lead" turned out to be a false positive because it never checked which serializer a view actually used.

Prioritise ruthlessly. I want real defects that affect real patients, not style opinions. If an app has nothing, say so in one line and move on.

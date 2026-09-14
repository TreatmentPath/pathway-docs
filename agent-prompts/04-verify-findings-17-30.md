# Agent prompt: Independent verification of findings #17–#30

Independently verify a set of claimed patient-identity defects. Your job is to
DISPROVE them. Assume the write-up is wrong until the data says otherwise.

READ-ONLY. Do not modify any file. Never write to any database.

Repo: /home/mannie/Desktop/Projects/treatmentpath
Django: TreatmentPathBackend/TreatmentPath (venv at TreatmentPathBackend/venv, manage.py in TreatmentPath/)
Live production copy: database `prod_control` on localhost:5432 user `mannie`.
NOTE: table names are mixed-case and MUST be double-quoted in psql, e.g.
`SELECT * FROM "TreatmentPlan_patient"` — an unquoted name errors as "does not exist".

The claims are in docs/PATIENT_IDENTITY_AUDIT_2026-09-06.md, section
"Second sweep — findings #17-#30". Read ONLY that section (the earlier #1-#16 are
being verified separately — ignore them).

## Your method — this is the whole point

Do NOT read the write-up and confirm it reads plausibly. For each of #17 through #30:

1. **Re-derive every number from scratch.** Write your own SQL. Do not copy the
   query implied by the finding — if you write the same query you will get the same
   answer whether or not it measures the right thing. State your query and your
   number. If it differs from the claim, say so loudly.

2. **Read the cited file:line yourself** and quote what is actually there. Line
   numbers drift. If the quoted code is not at that line, or the surrounding context
   changes its meaning, that is a finding about the finding.

3. **Check reachability.** A serializer with an unscoped queryset is only a defect if
   a real route reaches it AND nothing upstream already rejected the input. Trace:
   URL -> view -> which serializer class -> is there a validate_* / get_queryset /
   permission that already blocks this? One earlier agent reported an unscoped
   `Patient.objects.filter(id=...)` in Stock that was already guarded by
   `validate_patient_id` on the serializer the view actually used. That is the exact
   failure mode to avoid, in both directions.

4. **Fetch the named example rows back individually.** Every finding names specific
   ids. Look each one up. Confirm the person/patient/practice/text is as described.
   A finding whose example row does not exist, or does not say what is claimed, is
   not verified regardless of how good the reasoning looks.

5. **Ask whether the count measures the harm.** Several findings distinguish a large
   "exposure" number from a small "leaking today" number. Check both, separately, and
   check that the "today" number is derived from actual rows and not from the
   exposure number filtered by an assumption.

## Specific things to attack

- **#17** claims `Q(patients__id=patient_id) | Q(id=patient_id)` in
  activityLog/views.py merges a stranger's timeline into a patient's. Verify the OR
  is really there, that BOTH the activities query and the notes query filter on the
  resulting id set, and that practice scoping does not already eliminate the
  collisions in practice. Re-derive the collision count. Then find how many
  collisions are in the SAME practice — the claim says scoping keeps it in-practice,
  so cross-practice collisions are harmless and should not be counted.

- **#30** claims `PublicMedicalHistorySubmitView` writes without ever checking that
  the DOB/name verification happened. Try hard to find the gate: a middleware, a
  permission class, a signed token in the URL, a session flag, a status transition
  that submit implicitly requires. If a link's `code` is itself high-entropy and
  unguessable, say so and reassess the severity — a capability URL is a legitimate
  design, and the finding is then about the *absence of a second factor*, not about
  an open endpoint. Check how `code` is generated and how long it is.

- **#26** claims 894 patients have a NULL date_of_birth column but a real DOB in
  meta_data. Verify that number AND verify the meta_data values are actually valid
  dates rather than empty strings, nulls-as-text, or placeholder values.

- **#18/#19/#20** are all "0 rows today" online-booking findings. Confirm the feature
  really is unused (no holds, no `source='online'` appointments) — and confirm the
  code paths are the live ones, not dead code behind a feature flag that is off
  permanently or an app not in INSTALLED_APPS.

- **#22/#23/#24** claim writable patient FKs with unscoped querysets. For each, prove
  reachability per step 3 above, and separately confirm the "0 mismatched today"
  number with your own query.

## Output

One section per finding, #17 through #30, each with exactly one verdict:

- **CONFIRMED** — you independently reproduced it. Show your query and your number.
- **OVERSTATED** — the defect is real but a number, severity, or consequence is
  wrong. Say precisely which, and give the correct value.
- **WRONG** — it does not hold. Show what disproves it.
- **COULD NOT DISPROVE** — you could not reproduce it either way. Say what you tried
  and what evidence would settle it.

That last verdict is not a failure and you should use it whenever it is honest.
"Could not disprove" and "confirmed" are different claims and must not be merged.

Finally: anything the write-up MISSED in the same files. You are reading this code
closely; if you see a defect of the same class that is not listed, report it.

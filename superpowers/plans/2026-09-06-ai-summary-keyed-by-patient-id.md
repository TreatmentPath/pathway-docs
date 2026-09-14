# Finding #5 — key clinical AI summaries by Dentally patient id, not name

**Patient safety.** `dentally_patient_ai_summary` is UNIQUE on `(practice, patient_name)`.
Practice 21 has four different William Smiths and one summary between them.

## What the investigation added to the audit

The audit describes it as a storage-key problem. It is worse: **the whole pipeline is
name-keyed at the entry point**, so the summary CONTENT is blended across same-named
patients, not merely shared.

- `sync_handler.go:141` drives the loop from `SELECT DISTINCT patient_name` — four
  William Smiths produce ONE iteration.
- `db.go:91` fetches the appointments for that iteration with
  `LOWER(TRIM(a.patient_name)) = LOWER(TRIM(?))` — so `RecentEncounters`, `raw_notes`
  and the AI prompt are built from all four patients' clinical data at once.
- `opportunity/evidence_adapter.go:73` `PatientIDForName` resolves a name to the FIRST
  matching id, so the deterministic opportunity flags come from an arbitrary one.

So the fix is not "add a column and change the conflict target"; it is "carry the
Dentally patient id through the pipeline and use the name only for display".

## Steps

1. **Django model + migration.** Add `dentally_patient_id = IntegerField(null=True,
   db_index=True)`. Replace `uniq_patient_per_practice` with
   `UniqueConstraint(fields=["practice", "dentally_patient_id"],
   name="uniq_ai_summary_per_dentally_patient")`. Keep `patient_name` for display.
   Nullable + no Python-only `default=` — the Go writer inserts these rows, and a
   Django `default=` is invisible to it (schema-drift rule).
2. **Data migration (RunPython, reversible=noop).** For each row resolve
   `(practice, patient_name)` → `dentally_patient_id` via `dentally_appointment`.
   Exactly one id → stamp it. Several → **delete the row**: it is unattributable, and
   `data_hash` regenerates it per patient on the next sync. Zero → delete (orphan).
3. **Go writer.** `StoreAISummary` / `UpdateRawNotes` / `GetExistingDataHash` /
   `GetAISummary` take `dentallyPatientID int` alongside the name; `ON CONFLICT
   (practice_id, dentally_patient_id)`.
4. **Go reader of clinical data.** `GetPatientData` filters
   `a.dentally_patient_id = ?` instead of matching on the name. This is the
   content-contamination fix.
5. **Go loop.** `sync_handler.go` selects `DISTINCT dentally_patient_id, patient_name`
   and iterates ids. `AnalyzeOpportunitiesFromEvidence` takes the id directly rather
   than calling `PatientIDForName`.
6. **On-demand endpoint** (`handler.go`) — resolve the id for the requested name+date
   and refuse when the name is ambiguous, rather than silently picking one.
7. **Django reader.** `build_ai_summary_map` keys by `dentally_patient_id`;
   `attach_daylist_ai_fields` looks up by id. Update the module docstring, which
   currently documents the name key as a justification.
8. **Tests.** Go: same-named patients get separate rows and separate content. Django:
   the map no longer collides; the migration's split/delete rule.
9. **Verify on `prod_control`**: `GROUP BY practice_id, dentally_patient_id HAVING
   count(*) > 1` → 0, and the four William Smiths in practice 21 each get their own
   row or none.

## Traps

- Go↔Django parity: steps 1 and 3 must land together or the Go insert breaks.
- `patient_name` stays NOT NULL and keeps being written — it is the display value and
  the AI prompt input.
- The unique index must be partial or the column non-null-safe: many existing rows will
  be deleted by step 2, but new Go inserts always carry an id.

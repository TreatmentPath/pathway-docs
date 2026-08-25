# Go Import-Issue Writer Cutover & Schema Guard — Implementation Plan (Phase 3 of 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Repoint Go's migration-importer failure writer (`persistImportIssue`) from the
two legacy tables to `dataquality_dataqualityissue`, matching Phase 2's Django cutover —
then register the new table in the schema guard and confirm `/parity-check` picks it up
automatically, closing the dual-writer drift risk this whole project started from.

**Architecture:** `persistImportIssue` currently branches on `failureType` to pick one of
two raw table names. Replace that branch with a `failureType → classifier` map (mirroring
Django's `_FAILURE_TYPE_TO_CLASSIFIER` from Phase 2) and one target table. No change to
when/why this function is called — only what it writes and where.

**Tech Stack:** Go 1.x, GORM, PostgreSQL (shared DB with Django).

**Spec:** `docs/superpowers/specs/2026-08-13-unified-data-quality-design.md` (Section 2,
"Import-time" writer; Section 3, schema-guard/parity-check integration)

**Prerequisites:**
- Phase 1 (`docs/superpowers/plans/2026-08-13-dataquality-app-and-model.md`) applied —
  `dataquality_dataqualityissue` table must exist locally before any test in this plan
  can run.
- Phase 2 is independent of this plan (Django and Go writers can ship in either order —
  both target the same table, neither depends on the other's code), but both SHOULD land
  before Phase 5 per the Phase 2 deployment-sequencing note.

---

### Task 1: Repoint `persistImportIssue` to `dataquality_dataqualityissue`

**Files:**
- Test: `internal/dentally/migration/persist_import_issue_test.go` (new)
- Modify: `internal/dentally/migration/service.go:1959-2031` (`persistImportIssue`)

- [ ] **Step 1: Write the failing test**

Follows the existing transaction-rollback test pattern from `canonical_twin_test.go`
(`openMigrationTestDB` + `db.Begin()` + `t.Cleanup(tx.Rollback)`), reusing the same
`practiceID = 16` test practice already used elsewhere in this package.

```go
// internal/dentally/migration/persist_import_issue_test.go
package migration

import (
	"context"
	"encoding/json"
	"testing"
)

func persistTestService(t *testing.T) (*MigrationService, context.Context) {
	t.Helper()
	db := openMigrationTestDB(t)
	tx := db.Begin()
	if tx.Error != nil {
		t.Fatalf("begin tx: %v", tx.Error)
	}
	t.Cleanup(func() { tx.Rollback() })
	return &MigrationService{DB: tx}, context.Background()
}

type dqIssueRow struct {
	Classifier        string
	Source            string
	Status            string
	RecordType        string
	RecordID          string
	DentallyPatientID int64
	Detail            string // raw jsonb text, decoded by the test as needed
}

func readDataQualityIssue(t *testing.T, svc *MigrationService, practiceID int, dentallyPatientID int64) *dqIssueRow {
	t.Helper()
	var row dqIssueRow
	err := svc.DB.Raw(
		`SELECT classifier, source, status, record_type, record_id,
		        dentally_patient_id, detail::text
		 FROM dataquality_dataqualityissue
		 WHERE practice_id = ? AND dentally_patient_id = ?`,
		practiceID, dentallyPatientID,
	).Row().Scan(&row.Classifier, &row.Source, &row.Status, &row.RecordType,
		&row.RecordID, &row.DentallyPatientID, &row.Detail)
	if err != nil {
		return nil
	}
	return &row
}

func TestPersistImportIssue(t *testing.T) {
	const practiceID = 16

	t.Run("duplicate failure writes duplicate_import classifier", func(t *testing.T) {
		svc, ctx := persistTestService(t)
		dentally := map[string]any{
			"id": float64(77001), "first_name": "Sam", "last_name": "Reed",
			"email_address": "sam.reed@example.com",
		}
		phone := "07700900555"
		err := &importPatientError{
			FailureType: "duplicate",
			Message:     "duplicate key value violates unique constraint",
			PhoneNumber: &phone,
			ConflictDetails: map[string]any{
				"conflict_type": "phone_only",
			},
		}

		svc.persistImportIssue(ctx, practiceID, "go-task-1", dentally, err)

		row := readDataQualityIssue(t, svc, practiceID, 77001)
		if row == nil {
			t.Fatal("expected a DataQualityIssue row, found none")
		}
		if row.Classifier != "duplicate_import" {
			t.Errorf("classifier = %q, want duplicate_import", row.Classifier)
		}
		if row.Source != "go_migration" {
			t.Errorf("source = %q, want go_migration", row.Source)
		}
		if row.Status != "open" {
			t.Errorf("status = %q, want open (from db_default)", row.Status)
		}
		if row.RecordType != "dentally_patient" {
			t.Errorf("record_type = %q, want dentally_patient", row.RecordType)
		}
		if row.RecordID != "77001" {
			t.Errorf("record_id = %q, want 77001", row.RecordID)
		}

		var detail map[string]any
		if err := json.Unmarshal([]byte(row.Detail), &detail); err != nil {
			t.Fatalf("detail is not valid JSON: %v", err)
		}
		if detail["first_name"] != "Sam" {
			t.Errorf("detail.first_name = %v, want Sam", detail["first_name"])
		}
	})

	t.Run("non-importPatientError falls back to message sniffing, other_import_error classifier", func(t *testing.T) {
		svc, ctx := persistTestService(t)
		dentally := map[string]any{"id": float64(77002), "first_name": "Pat", "last_name": "Lee"}
		err := errorf("some unrelated DB error")

		svc.persistImportIssue(ctx, practiceID, "go-task-2", dentally, err)

		row := readDataQualityIssue(t, svc, practiceID, 77002)
		if row == nil {
			t.Fatal("expected a DataQualityIssue row, found none")
		}
		if row.Classifier != "other_import_error" {
			t.Errorf("classifier = %q, want other_import_error", row.Classifier)
		}
	})

	t.Run("second call with same dentally id updates in place, not a duplicate row", func(t *testing.T) {
		svc, ctx := persistTestService(t)
		dentally := map[string]any{"id": float64(77003), "first_name": "Kim", "last_name": "Park"}
		errFirst := errorf("validation error: bad date")
		errSecond := errorf("validation error: still bad date")

		svc.persistImportIssue(ctx, practiceID, "go-task-3a", dentally, errFirst)
		svc.persistImportIssue(ctx, practiceID, "go-task-3b", dentally, errSecond)

		var count int
		if err := svc.DB.Raw(
			`SELECT count(*) FROM dataquality_dataqualityissue WHERE practice_id = ? AND dentally_patient_id = ?`,
			practiceID, 77003,
		).Row().Scan(&count); err != nil {
			t.Fatalf("count query: %v", err)
		}
		if count != 1 {
			t.Errorf("row count = %d, want 1 (expected update-in-place)", count)
		}
	})
}

// errorf builds a plain error (not *importPatientError) for the message-sniffing
// fallback branch of persistImportIssue.
func errorf(msg string) error { return &plainErr{msg} }

type plainErr struct{ msg string }

func (e *plainErr) Error() string { return e.msg }
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd EmailServiceGo && go test ./internal/dentally/migration/... -run TestPersistImportIssue -v`
Expected: FAIL — `pq: relation "dataquality_dataqualityissue" does not exist` if Phase 1
hasn't been migrated locally yet (fix by running Phase 1's `python manage.py migrate
dataQuality` first), otherwise the test fails because `persistImportIssue` still writes
to the two legacy tables, so `readDataQualityIssue` finds nothing (`row == nil`).

- [ ] **Step 3: Replace `persistImportIssue`**

Replace the entire function body (`service.go` lines 1959-2031) with:

```go
// _failureTypeToClassifier mirrors Django's _FAILURE_TYPE_TO_CLASSIFIER
// (TreatmentPathBackend/TreatmentPath/dentallyIntegration/tasks.py) exactly — keep
// both in sync if a new failure type is ever added on either side.
var _failureTypeToClassifier = map[string]string{
	"duplicate":     "duplicate_import",
	"validation":    "validation_error",
	"missing_data":  "missing_data_import",
	"invalid_phone": "invalid_phone_import",
	"other":         "other_import_error",
}

func (m *MigrationService) persistImportIssue(ctx context.Context, practiceID int, taskID string, dentally map[string]any, importErr error) {
	failureType := "other"
	errorMessage := importErr.Error()
	var phone *string
	var existingID *int
	conflict := map[string]any{}

	if pe, ok := importErr.(*importPatientError); ok {
		if pe.FailureType != "" {
			failureType = pe.FailureType
		}
		if pe.PhoneNumber != nil {
			phone = pe.PhoneNumber
		}
		if pe.ExistingPatientID != nil {
			existingID = pe.ExistingPatientID
		}
		if pe.ConflictDetails != nil {
			conflict = pe.ConflictDetails
		}
	} else {
		lower := strings.ToLower(errorMessage)
		switch {
		case strings.Contains(lower, "duplicate"):
			failureType = "duplicate"
		case strings.Contains(lower, "invalid phone"):
			failureType = "invalid_phone"
		case strings.Contains(lower, "missing data"), strings.Contains(lower, "required"):
			failureType = "missing_data"
		case strings.Contains(lower, "validation"):
			failureType = "validation"
		}
	}

	classifier, ok := _failureTypeToClassifier[failureType]
	if !ok {
		classifier = "other_import_error"
	}

	dentallyID := anyToInt64(dentally["id"])

	detail := map[string]any{
		"migration_task_id":   taskID,
		"first_name":          toCleanString(dentally["first_name"]),
		"last_name":           toCleanString(dentally["last_name"]),
		"email":               toCleanString(dentally["email_address"]),
		"phone_number":        phone,
		"existing_patient_id": existingID,
		"error_message":       errorMessage,
		"conflict_details":    conflict,
		"dentally_data":       dentally,
	}
	detailJSON, _ := json.Marshal(detail)

	record := map[string]any{
		"practice_id":         practiceID,
		"classifier":          classifier,
		"source":              "go_migration",
		"record_type":         "dentally_patient",
		"record_id":           fmt.Sprintf("%d", dentallyID),
		"dentally_patient_id": dentallyID,
		"detail":              gorm.Expr("CAST(? AS jsonb)", string(detailJSON)),
	}

	// Table name is a literal at both call sites (not a variable/const) — the
	// schema-guard's coverage test (reGormTable regex in
	// schema_guard_coverage_test.go) only matches a literal string directly inside
	// .Table("...") to detect a write; a `const table = "..."` indirection (tried
	// first, execution-time discovery) is invisible to it and left the table
	// undetected as either registered-and-written or stale.
	tx := m.DB.WithContext(ctx).Table("dataquality_dataqualityissue").
		Where("practice_id = ? AND dentally_patient_id = ?", practiceID, dentallyID).
		Updates(record)
	if tx.Error != nil {
		fmt.Printf("[Migration] failed to update DataQualityIssue practice=%d patient=%d: %v\n", practiceID, dentallyID, tx.Error)
		return
	}
	if tx.RowsAffected == 0 {
		record["created_at"] = time.Now().UTC()
		record["updated_at"] = time.Now().UTC()
		// status is intentionally omitted — db_default 'open' applies (see
		// dataQuality/models.py DataQualityIssue.status, Phase 1).
		if err := m.DB.WithContext(ctx).Table("dataquality_dataqualityissue").Create(record).Error; err != nil {
			fmt.Printf("[Migration] failed to record DataQualityIssue practice=%d patient=%d: %v\n", practiceID, dentallyID, err)
		}
	}
}
```

Do NOT remove the two `dentallyintegration_dentallyimport*` raw-SQL references
elsewhere in this file if any exist outside `persistImportIssue` — check with
`grep -n "dentallyintegration_dentallyimport" internal/dentally/migration/service.go`
before finishing this step; if the grep returns only lines inside the function you just
replaced, you're done. (Per the earlier design investigation, this was the only Go
write site — confirmed via `grep -rln "dentallyimportfailure\|dentallyimporterror" ...`
returning only `service.go` and `pile.go`, and `pile.go` only references the tables in
a comment, not code.)

- [ ] **Step 4: Run the test to verify it passes**

Run: `go test ./internal/dentally/migration/... -run TestPersistImportIssue -v`
Expected: `--- PASS` for all three subtests.

- [ ] **Step 5: Run the full migration package test suite to confirm no regression**

Run: `go test ./internal/dentally/migration/... -v`
Expected: all existing tests still pass (`TestCanonicalTwin`, page-dedupe tests, etc. —
none of them call `persistImportIssue`, so this is a smoke check, not expected to
surface anything new).

- [ ] **Step 6: Commit**

```bash
cd EmailServiceGo
git add internal/dentally/migration/service.go internal/dentally/migration/persist_import_issue_test.go
git commit -m "feat(dataQuality): persistImportIssue writes to dataquality_dataqualityissue"
```

(Leave the actual commit to the user per project convention.)

---

### Task 2: Register the table in the schema guard

**Files:**
- Modify: `internal/db/schemaguard.go` (`requiredColumns` map)

- [ ] **Step 1: Generate the column snapshot**

Run (against the LOCAL database, which must already have Phase 1's migration applied):

```bash
cd EmailServiceGo
DATABASE_URL="postgres://mannie:maniZolas1008@localhost:5432/treatmentpath_db?sslmode=disable" \
  go run ./cmd/schemaguard_gen dataquality_dataqualityissue
```

Actual generated output:

```go
"dataquality_dataqualityissue": {
	"classifier", "created_at", "detail", "practice_id",
	"record_id", "record_type", "source", "updated_at",
},
```

`status` correctly does NOT appear (Phase 1's `db_default="open"` makes it drift-safe)
and `dentally_patient_id` correctly does NOT appear (nullable). `detail` DOES appear,
which the plan's original prediction got wrong — `models.JSONField(default=dict,
blank=True)` only fills the value at the Django ORM layer; `default=` without
`db_default=` creates no actual DB-level default, so the live column is NOT NULL with no
default, same as any other required column. This is harmless here because the Go writer
already always supplies `detail` in both the `Updates` and `Create` calls — but it does
mean `detail` needed registering, which the generator caught correctly. `id` does not
appear either (Django `BigAutoField` — an identity column with its own DB-level
default). Do not hand-edit the tool's output beyond adding the explanatory comment in
Step 2.

- [ ] **Step 2: Paste the generated entry into `requiredColumns`**

Add the generated block to `internal/db/schemaguard.go`'s `requiredColumns` map
(alphabetical-within-table-group per the file's existing convention — add near the
`dentally_*` entries since it's alphabetically adjacent), with a one-line dated comment:

```go
	// dataquality_dataqualityissue: registered 2026-08-13 — new unified data-quality
	// model (Phase 3 of the data-quality unification), written by
	// persistImportIssue (migration importer) and, starting Phase 4, the nightly
	// duplicate-contact/missing-info sweep.
	"dataquality_dataqualityissue": {
		"classifier", "created_at", "id", "practice_id", "record_id",
		"record_type", "source", "updated_at",
	},
```

(Replace the column list with whatever Step 1 actually generated if it differs from the
placeholder shown here — the generator's live output is authoritative, not this plan.)

- [ ] **Step 3: Run the schema-guard coverage tests**

Run: `go test ./internal/db/... -run TestSchemaGuard -v`
Expected: `TestSchemaGuard_EveryGoWrittenTableIsRegistered` PASS (confirms
`dataquality_dataqualityissue` is now correctly registered — this test derives the
Go-written table set from source and would have FAILED before Step 2, since Task 1
started writing to a table absent from the snapshot).

- [ ] **Step 4: Run the full schema-guard gate against the local DB**

Run: `DATABASE_URL="postgres://mannie:maniZolas1008@localhost:5432/treatmentpath_db?sslmode=disable" SCHEMA_GUARD_REQUIRE_DB=1 go test ./internal/db/... -run TestSchemaGuard_RequiredColumnsPinned -v`
(`SCHEMA_GUARD_REQUIRE_DB=1` alone errors with "DATABASE_URL is empty" — both must be set together.)
Expected: PASS. This is the actual pre-deploy gate — confirms the snapshot matches the
live local schema exactly (not just "table is registered" from Step 3, but "the
registered columns are correct").

- [ ] **Step 5: Commit**

```bash
git add internal/db/schemaguard.go
git commit -m "chore(schemaguard): register dataquality_dataqualityissue"
```

(Leave the actual commit to the user per project convention.)

---

### Task 3: Confirm via `/parity-check`

**Files:** none (verification-only task, no code changes)

- [ ] **Step 1: Run a local parity audit**

Run the `/parity-check` slash command (`local` target, read-only, no `fix`) — this
invokes the `go-django-parity-guard` agent, which re-derives the Go-written table set
from source on every run, so it should now include `dataquality_dataqualityissue`
without any change to the agent definition itself.

- [ ] **Step 2: Confirm the report lists the new table cleanly**

Expected: the Parity Report's "Findings table" either omits
`dataquality_dataqualityissue` entirely (meaning it found zero issues — the goal) or
lists it with a PASS/non-blocking status. If it reports the table as unregistered or
mismatched, Task 2 has a mistake — go back and re-run `schemaguard_gen`, don't hand-patch
the snapshot to make the report quiet.

- [ ] **Step 3: No commit needed** — this task only verifies Tasks 1-2, it doesn't
  change any files.

---

## Self-review notes

- **Spec coverage**: implements the Go half of Section 2 ("Import-time" writer) and all
  of Section 3 (schema-guard/parity-check integration) from the spec.
- **No placeholders**: every step has complete, runnable code, including the
  schema-guard entry — flagged explicitly as "replace with the tool's real output" only
  where the actual value is generated at run-time and can't be hand-predicted in a
  written plan (that's not a placeholder, it's an instruction to use a generator tool
  that exists specifically so no one hand-writes this list).
- **Type consistency**: `_failureTypeToClassifier`'s keys (`duplicate`, `validation`,
  `missing_data`, `invalid_phone`, `other`) match the `failureType` values already
  produced earlier in `persistImportIssue`'s existing branching logic (unchanged from
  the original function, re-read from the live source during planning) and match
  Django's `_FAILURE_TYPE_TO_CLASSIFIER` keys from Phase 2 exactly — cross-referenced in
  a code comment so future edits to one side get flagged to check the other.
- **Verified dependency**: confirmed via source grep during design that `service.go` is
  the ONLY Go writer of the legacy tables (`pile.go` only mentions them in a comment) —
  Task 1 Step 3 repeats this check as a safety net rather than assuming it silently.

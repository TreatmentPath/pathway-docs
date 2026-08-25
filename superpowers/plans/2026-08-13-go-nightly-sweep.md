# Go Nightly Duplicate-Contact / Missing-Info Sweep — Implementation Plan (Phase 4 of 5)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Materialize the two scan-based issue types (`duplicate_contact`, `missing_info`)
as `DataQualityIssue` rows via a new nightly Go cron job, replacing the live-computed
Django scans (`DataQualityDuplicatesView`, `DataQualityMissingInfoView`) as the source of
truth for these classifiers.

**Architecture:** One new file, `internal/dentally/scheduler/dataquality_sweep.go`, with
two detector functions (`sweepDuplicateContacts`, `sweepMissingInfo`) run per-practice
inside the existing worker-pool pattern (`scheduler.go`'s semaphore+WaitGroup shape), plus
a ported `nameSimilar`/`sequenceMatcherRatio` pair mirroring Django's `_name_similar`
exactly (Ratcliff/Obershelp ratio, no junk heuristics — irrelevant at name-string length).
Wired into a new nightly cron entry alongside the existing `0 2 * * *` daily sync.

**Tech Stack:** Go 1.x, GORM, PostgreSQL, `robfig/cron/v3`.

**Spec:** `docs/superpowers/specs/2026-08-13-unified-data-quality-design.md` (Section 2,
"Scan-time" writer)

**Prerequisites:** Phase 1 applied (table exists). Independent of Phases 2/3 (this sweep
never touches the import-time classifiers or the legacy tables at all).

**Known limitation, accepted rather than solved here:** Django (`TreatmentPathBackend`)
and Go (`EmailServiceGo`) are separate git repositories with independent CI. There is no
single-source-of-truth location both can reference at test time without assuming a
specific local sibling-directory layout. This plan duplicates the name-similarity truth
table fixture into both repos with a cross-reference comment in each, rather than a true
shared file — a real (smaller) residual drift risk versus one canonical file, but the
same risk profile the project already accepts elsewhere (e.g. the undocumented duplicated
constants this project's schema-parity tooling was built to catch after the fact). Flagged
explicitly rather than silently assumed solved.

---

### Task 1: Port `_name_similar` — `nameSimilar` + `sequenceMatcherRatio`, with shared fixture

**Files:**
- Create: `EmailServiceGo/internal/dentally/scheduler/name_similarity.go`
- Create: `EmailServiceGo/internal/dentally/scheduler/name_similarity_test.go`
- Create: `EmailServiceGo/internal/dentally/scheduler/testdata/name_similarity_cases.json`
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/tests/fixtures/name_similarity_cases.json`
- Create: `TreatmentPathBackend/TreatmentPath/dataQuality/tests/test_name_similarity_fixture.py`

- [ ] **Step 1: Write the shared fixture (identical content in both locations)**

```json
[
  {"a": "John", "b": "John", "expected": true, "note": "exact match"},
  {"a": "Jon", "b": "John", "expected": true, "note": "typo/spelling variant, ratio >= 0.82"},
  {"a": "Jon", "b": "Jonathan", "expected": true, "note": "prefix match"},
  {"a": "Paul", "b": "Paul Thomas", "expected": true, "note": "prefix, dropped middle name"},
  {"a": "J", "b": "John", "expected": true, "note": "single initial matches first letter"},
  {"a": "j", "b": "John", "expected": true, "note": "case-insensitive single initial"},
  {"a": "McGrath", "b": "Mc Grath", "expected": true, "note": "hyphenation/spacing variant"},
  {"a": "John", "b": "Jane", "expected": false, "note": "clearly different names"},
  {"a": "Smith", "b": "Jones", "expected": false, "note": "clearly different surnames"},
  {"a": "", "b": "John", "expected": false, "note": "empty string never matches"},
  {"a": "John", "b": "", "expected": false, "note": "empty string never matches, reversed"},
  {"a": "", "b": "", "expected": false, "note": "both empty — _name_similar returns False for empty input, not vacuous true"},
  {"a": "Alexandra", "b": "Alex", "expected": true, "note": "prefix match, reversed order"},
  {"a": "Katherine", "b": "Kate", "expected": false, "note": "nickname NOT caught by prefix or ratio — a real gap in the heuristic, preserved intentionally rather than 'fixed' during porting"}
]
```

Write this exact JSON content to BOTH locations listed above — byte-identical. The
`Katherine`/`Kate` case matters: it documents a known false-negative in the existing
Python heuristic (not something this port should silently "improve", since that would
make Go and Python diverge in real detection behavior, not just in test coverage).

- [ ] **Step 2: Write the failing Go test**

```go
// internal/dentally/scheduler/name_similarity_test.go
package scheduler

import (
	"encoding/json"
	"os"
	"testing"
)

type nameSimilarityCase struct {
	A        string `json:"a"`
	B        string `json:"b"`
	Expected bool   `json:"expected"`
	Note     string `json:"note"`
}

func loadNameSimilarityFixture(t *testing.T) []nameSimilarityCase {
	t.Helper()
	data, err := os.ReadFile("testdata/name_similarity_cases.json")
	if err != nil {
		t.Fatalf("read fixture: %v", err)
	}
	var cases []nameSimilarityCase
	if err := json.Unmarshal(data, &cases); err != nil {
		t.Fatalf("parse fixture: %v", err)
	}
	return cases
}

func TestNameSimilar_MatchesFixture(t *testing.T) {
	for _, c := range loadNameSimilarityFixture(t) {
		got := nameSimilar(c.A, c.B)
		if got != c.Expected {
			t.Errorf("nameSimilar(%q, %q) = %v, want %v (%s)", c.A, c.B, got, c.Expected, c.Note)
		}
	}
}

func TestSequenceMatcherRatio_KnownValues(t *testing.T) {
	// Cross-checked against Python's difflib.SequenceMatcher(None, a, b).ratio()
	// directly (not just via the >= 0.82 threshold) for a few concrete cases, so a
	// bug in the ratio computation itself (not just the threshold check) is caught.
	cases := []struct {
		a, b string
		want float64
	}{
		{"jon", "john", 6.0 / 7.0},   // SequenceMatcher(None, "jon", "john").ratio()
		{"same", "same", 1.0},
		{"", "", 1.0},
		{"abc", "xyz", 0.0},
	}
	for _, c := range cases {
		got := sequenceMatcherRatio(c.a, c.b)
		if diff := got - c.want; diff > 1e-9 || diff < -1e-9 {
			t.Errorf("sequenceMatcherRatio(%q, %q) = %v, want %v", c.a, c.b, got, c.want)
		}
	}
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `cd EmailServiceGo && go test ./internal/dentally/scheduler/... -run 'TestNameSimilar|TestSequenceMatcherRatio' -v`
Expected: FAIL — `undefined: nameSimilar` / `undefined: sequenceMatcherRatio` (functions
don't exist yet).

- [ ] **Step 4: Implement the port**

```go
// internal/dentally/scheduler/name_similarity.go
package scheduler

import "strings"

// nameSimilar ports Django's _name_similar exactly
// (TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/data_quality_views.py) —
// fuzzy name match for duplicate-contact detection: catches nicknames, dropped middle
// names, hyphenation, initials and typos, while treating clearly-different names as
// distinct so real family members sharing a channel aren't flagged as duplicates.
//
// Keep this in exact sync with the Python original — see the shared truth-table
// fixture at testdata/name_similarity_cases.json (byte-identical copy at
// TreatmentPathBackend/TreatmentPath/dataQuality/tests/fixtures/name_similarity_cases.json)
// asserted by both this package's test suite and dataQuality's Python test suite.
func nameSimilar(a, b string) bool {
	a = strings.ToLower(strings.TrimSpace(a))
	b = strings.ToLower(strings.TrimSpace(b))
	if a == "" || b == "" {
		return false
	}
	if a == b {
		return true
	}
	if strings.HasPrefix(a, b) || strings.HasPrefix(b, a) {
		return true
	}
	ra, rb := []rune(a), []rune(b)
	if (len(ra) == 1 && ra[0] == rb[0]) || (len(rb) == 1 && rb[0] == ra[0]) {
		return true
	}
	aNoSpace := strings.ReplaceAll(a, " ", "")
	bNoSpace := strings.ReplaceAll(b, " ", "")
	return sequenceMatcherRatio(aNoSpace, bNoSpace) >= 0.82
}

// sequenceMatcherRatio ports Python's difflib.SequenceMatcher(None, a, b).ratio() —
// the Ratcliff/Obershelp algorithm: recursively find the longest matching block, then
// recurse on the unmatched left and right remainders; ratio = 2*matches / (len(a)+len(b)).
// Deliberately omits difflib's "junk"/"autojunk" heuristics — those only ever activate
// for sequences of 200+ elements, irrelevant at name-string length, and Django's
// SequenceMatcher(None, a, b) call passes no junk function either.
func sequenceMatcherRatio(a, b string) float64 {
	ra, rb := []rune(a), []rune(b)
	if len(ra) == 0 && len(rb) == 0 {
		return 1.0
	}
	matches := 0
	for _, block := range matchingBlocks(ra, rb) {
		matches += block.size
	}
	return 2.0 * float64(matches) / float64(len(ra)+len(rb))
}

type matchBlock struct {
	aStart, bStart, size int
}

func matchingBlocks(a, b []rune) []matchBlock {
	type span struct{ aLo, aHi, bLo, bHi int }
	queue := []span{{0, len(a), 0, len(b)}}
	var blocks []matchBlock

	for len(queue) > 0 {
		s := queue[len(queue)-1]
		queue = queue[:len(queue)-1]

		i, j, k := longestMatch(a, b, s.aLo, s.aHi, s.bLo, s.bHi)
		if k > 0 {
			blocks = append(blocks, matchBlock{i, j, k})
			if s.aLo < i && s.bLo < j {
				queue = append(queue, span{s.aLo, i, s.bLo, j})
			}
			if i+k < s.aHi && j+k < s.bHi {
				queue = append(queue, span{i + k, s.aHi, j + k, s.bHi})
			}
		}
	}
	return blocks
}

// longestMatch finds the longest run of consecutive matching runes between
// a[aLo:aHi] and b[bLo:bHi], returning its start indices and length. O(n*m), fine for
// name-length strings (this is not called on anything longer than ~100 characters).
func longestMatch(a, b []rune, aLo, aHi, bLo, bHi int) (besti, bestj, bestsize int) {
	besti, bestj, bestsize = aLo, bLo, 0
	j2len := map[int]int{}
	for i := aLo; i < aHi; i++ {
		newJ2Len := map[int]int{}
		for j := bLo; j < bHi; j++ {
			if a[i] != b[j] {
				continue
			}
			k := j2len[j-1] + 1
			newJ2Len[j] = k
			if k > bestsize {
				besti, bestj, bestsize = i-k+1, j-k+1, k
			}
		}
		j2len = newJ2Len
	}
	return besti, bestj, bestsize
}
```

- [ ] **Step 5: Run the Go test to verify it passes**

Run: `go test ./internal/dentally/scheduler/... -run 'TestNameSimilar|TestSequenceMatcherRatio' -v`
Expected: `--- PASS` for both tests. If `TestNameSimilar_MatchesFixture` fails on the
`Katherine`/`Kate` case, re-check the fixture file was written with `"expected": false`
for that case (it's an intentionally-preserved false-negative, not a bug to fix).

- [ ] **Step 6: Write and run the matching Python fixture test**

```python
# dataQuality/tests/test_name_similarity_fixture.py
import json
from pathlib import Path

from django.test import SimpleTestCase

from TreatmentPlan.views.data_quality_views import _name_similar

FIXTURE_PATH = Path(__file__).parent / "fixtures" / "name_similarity_cases.json"


class NameSimilarityFixtureTests(SimpleTestCase):
    def test_all_fixture_cases_match_python_implementation(self):
        cases = json.loads(FIXTURE_PATH.read_text())
        for case in cases:
            with self.subTest(a=case["a"], b=case["b"]):
                self.assertEqual(
                    _name_similar(case["a"], case["b"]),
                    case["expected"],
                    msg=case.get("note", ""),
                )
```

Run: `cd TreatmentPathBackend/TreatmentPath && source ../venv/bin/activate && python manage.py test dataQuality.tests.test_name_similarity_fixture --keepdb -v 2`
Expected: `OK`. This confirms the fixture itself is correct against the ORIGINAL Python
implementation, not just internally consistent with the new Go port — both sides are
now pinned to the same truth table.

- [ ] **Step 7: Commit**

```bash
cd EmailServiceGo
git add internal/dentally/scheduler/name_similarity.go internal/dentally/scheduler/name_similarity_test.go internal/dentally/scheduler/testdata/name_similarity_cases.json
git commit -m "feat(dataQuality): port _name_similar to Go with shared fixture"

cd ../TreatmentPathBackend
git add TreatmentPath/dataQuality/tests/fixtures/name_similarity_cases.json TreatmentPath/dataQuality/tests/test_name_similarity_fixture.py
git commit -m "test(dataQuality): pin _name_similar against shared Go/Python fixture"
```

(Leave the actual commits to the user per project convention — two separate repos, two
separate commits.)

---

### Task 2: `missing_info` detector

**Files:**
- Create: `EmailServiceGo/internal/dentally/scheduler/dataquality_sweep.go`
- Create: `EmailServiceGo/internal/dentally/scheduler/dataquality_sweep_test.go`

- [ ] **Step 1: Write the failing test**

Reuses the transaction-rollback pattern from the migration package's tests
(`openMigrationTestDB`-equivalent) — this package needs its own DB-test helper since
`scheduler` doesn't currently have one; add it alongside the test.

```go
// internal/dentally/scheduler/dataquality_sweep_test.go
package scheduler

import (
	"context"
	"os"
	"testing"
	"time"

	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	gormLogger "gorm.io/gorm/logger"
)

func sweepTestDB(t *testing.T) *gorm.DB {
	t.Helper()
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		dsn = "postgres://mannie:maniZolas1008@localhost:5432/treatmentpath_db?sslmode=disable"
	}
	db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
		Logger: gormLogger.Default.LogMode(gormLogger.Silent),
	})
	if err != nil {
		t.Skipf("skipping: cannot connect to DB: %v", err)
	}
	tx := db.Begin()
	if tx.Error != nil {
		t.Fatalf("begin tx: %v", tx.Error)
	}
	t.Cleanup(func() { tx.Rollback() })
	return tx
}

// seedPractice inserts a practice row and returns its id. Supplies every
// NOT-NULL/no-DB-default column on UserAuthentication_practice — 23 in total.
// Execution-time correction: the plan as originally written supplied only 6 columns
// (name, slug, is_active, is_archived, created_at, archive_reason). Django's
// blank=True (address, archive_reason, colour fields, etc.) is Python-only and
// creates no DB-level default — a minimal INSERT fails at the DB with "null value
// in column address violates not-null constraint", the same trap already
// documented on DataQualityIssue.detail in the Phase 3 plan. Verified the full
// required-column list empirically via information_schema before rewriting this.
func seedPractice(t *testing.T, db *gorm.DB, name string) int {
	t.Helper()
	now := time.Now().UTC()
	var id int
	if err := db.Raw(
		`INSERT INTO "UserAuthentication_practice"
		 (name, slug, address, archive_reason, phone_number, practice_type, timezone,
		  brand_primary_colour, brand_secondary_colour, brand_accent_colour, brand_background_colour,
		  financing_terms, clock_location_radius,
		  is_active, is_archived, encrypt_patient_contact_info, use_default_email,
		  journeys_board_hide_empty_stages, require_clock_location, require_clock_photo,
		  show_dentist_bio, smtp_use_ssl, created_at)
		 VALUES (?, ?, '', '', '', '', '',
		         '', '', '', '',
		         '{}', 0,
		         true, false, false, false,
		         false, false, false,
		         false, false, ?)
		 RETURNING id`,
		name, name+"-slug-"+time.Now().Format("150405.000000"), now,
	).Scan(&id).Error; err != nil {
		t.Fatalf("seed practice: %v", err)
	}
	return id
}

// seedPatientMissingInfo inserts a Patient with no email and no phone. Supplies
// every NOT-NULL/no-DB-default column — 19 in total, same blank=True trap as
// seedPractice above (execution-time correction, same rationale).
func seedPatientMissingInfo(t *testing.T, db *gorm.DB, practiceID int, firstName string) int {
	t.Helper()
	now := time.Now().UTC()
	var id int
	if err := db.Raw(
		`INSERT INTO "TreatmentPlan_patient"
		 (practice_id, first_name, last_name, title, middle_name, preferred_name, gender,
		  county, town, postcode, nhs_number, occupation, recall_method,
		  emergency_contact_name, emergency_contact_phone, emergency_contact_relationship,
		  medical_alert, medical_alert_text, use_email, use_sms, field_sources,
		  created_at, updated_at)
		 VALUES (?, ?, '', '', '', '', '',
		         '', '', '', '', '', '',
		         '', '', '',
		         false, '', false, false, '{}',
		         ?, ?)
		 RETURNING id`,
		practiceID, firstName, now, now,
	).Scan(&id).Error; err != nil {
		t.Fatalf("seed patient: %v", err)
	}
	return id
}

func TestSweepMissingInfo(t *testing.T) {
	db := sweepTestDB(t)
	ctx := context.Background()
	practiceID := seedPractice(t, db, "Missing Info Sweep Test")
	patientID := seedPatientMissingInfo(t, db, practiceID, "NoContact")

	sweepMissingInfo(ctx, db, practiceID)

	var count int
	if err := db.Raw(
		`SELECT count(*) FROM dataquality_dataqualityissue
		 WHERE practice_id = ? AND classifier = 'missing_info' AND record_type = 'patient' AND record_id = ? AND status = 'open'`,
		practiceID, patientIDStr(patientID),
	).Row().Scan(&count); err != nil {
		t.Fatalf("count query: %v", err)
	}
	if count != 1 {
		t.Errorf("expected 1 open missing_info issue for patient %d, got %d", patientID, count)
	}
}

func TestSweepMissingInfo_ReconcilesResolvedWhenNoLongerMissing(t *testing.T) {
	db := sweepTestDB(t)
	ctx := context.Background()
	practiceID := seedPractice(t, db, "Missing Info Reconcile Test")
	patientID := seedPatientMissingInfo(t, db, practiceID, "TempMissing")

	sweepMissingInfo(ctx, db, practiceID)

	// Patient now has an email — condition no longer holds.
	if err := db.Exec(
		`UPDATE "TreatmentPlan_patient" SET email = 'now-has-email@example.com' WHERE id = ?`,
		patientID,
	).Error; err != nil {
		t.Fatalf("update patient: %v", err)
	}

	sweepMissingInfo(ctx, db, practiceID)

	var status string
	if err := db.Raw(
		`SELECT status FROM dataquality_dataqualityissue
		 WHERE practice_id = ? AND classifier = 'missing_info' AND record_id = ?`,
		practiceID, patientIDStr(patientID),
	).Row().Scan(&status); err != nil {
		t.Fatalf("read status: %v", err)
	}
	if status != "resolved" {
		t.Errorf("status = %q, want resolved (condition cleared)", status)
	}
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `go test ./internal/dentally/scheduler/... -run TestSweepMissingInfo -v`
Expected: FAIL — `undefined: sweepMissingInfo` / `undefined: patientIDStr`.

- [ ] **Step 3: Implement `sweepMissingInfo`**

```go
// internal/dentally/scheduler/dataquality_sweep.go
package scheduler

import (
	"context"
	"fmt"
	"time"

	"gorm.io/gorm"
)

func patientIDStr(id int) string { return fmt.Sprintf("%d", id) }

// sweepMissingInfo finds every Patient/Intake row in the practice with neither an
// email nor a phone number, upserts an open DataQualityIssue for each, and resolves
// any previously-open missing_info issue whose record no longer qualifies (the
// contact was edited to add an email or phone since the last sweep).
//
// Mirrors DataQualityMissingInfoView's predicate exactly
// (TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/data_quality_views.py):
// missing = (email IS NULL OR email = '') AND (phone_number IS NULL OR phone_number = '').
func sweepMissingInfo(ctx context.Context, db *gorm.DB, practiceID int) {
	type row struct {
		Kind string
		ID   int
	}
	var rows []row

	if err := db.WithContext(ctx).Raw(
		`SELECT 'patient' AS kind, id FROM "TreatmentPlan_patient"
		 WHERE practice_id = ? AND (email IS NULL OR email = '') AND (phone_number IS NULL OR phone_number = '')
		 UNION ALL
		 SELECT 'intake' AS kind, id FROM "TreatmentPlan_intake"
		 WHERE practice_id = ? AND (email IS NULL OR email = '') AND (phone_number IS NULL OR phone_number = '')`,
		practiceID, practiceID,
	).Scan(&rows).Error; err != nil {
		fmt.Printf("[DataQuality sweep] missing_info query failed practice=%d: %v\n", practiceID, err)
		return
	}

	seen := make(map[string]bool, len(rows))
	now := time.Now().UTC()
	for _, r := range rows {
		recordID := fmt.Sprintf("%d", r.ID)
		key := r.Kind + ":" + recordID
		seen[key] = true

		record := map[string]any{
			"practice_id": practiceID,
			"classifier":  "missing_info",
			"source":      "go_sweep",
			"record_type": r.Kind,
			"record_id":   recordID,
			"status":      "open",
			"updated_at":  now,
		}
		tx := db.WithContext(ctx).Table("dataquality_dataqualityissue").
			Where("practice_id = ? AND classifier = 'missing_info' AND record_type = ? AND record_id = ?", practiceID, r.Kind, recordID).
			Updates(record)
		if tx.Error != nil {
			fmt.Printf("[DataQuality sweep] missing_info update failed practice=%d %s=%d: %v\n", practiceID, r.Kind, r.ID, tx.Error)
			continue
		}
		if tx.RowsAffected == 0 {
			record["created_at"] = now
			record["detail"] = gorm.Expr("'{}'::jsonb")
			if err := db.WithContext(ctx).Table("dataquality_dataqualityissue").Create(record).Error; err != nil {
				fmt.Printf("[DataQuality sweep] missing_info insert failed practice=%d %s=%d: %v\n", practiceID, r.Kind, r.ID, err)
			}
		}
	}

	// Reconcile: any OPEN missing_info issue not in this sweep's result set no
	// longer qualifies — resolve it.
	resolveStaleIssues(ctx, db, practiceID, "missing_info", seen)
}

// resolveStaleIssues flips every open issue of the given classifier whose
// "kind:record_id" key is absent from `stillOpen` to status=resolved. Shared by both
// sweep detectors — the reconciliation rule is identical for missing_info and
// duplicate_contact ("if the sweep doesn't see it anymore, the condition cleared").
func resolveStaleIssues(ctx context.Context, db *gorm.DB, practiceID int, classifier string, stillOpen map[string]bool) {
	type openRow struct {
		ID         int
		RecordType string
		RecordID   string
	}
	var open []openRow
	if err := db.WithContext(ctx).Raw(
		`SELECT id, record_type, record_id FROM dataquality_dataqualityissue
		 WHERE practice_id = ? AND classifier = ? AND status = 'open'`,
		practiceID, classifier,
	).Scan(&open).Error; err != nil {
		fmt.Printf("[DataQuality sweep] reconcile query failed practice=%d classifier=%s: %v\n", practiceID, classifier, err)
		return
	}

	now := time.Now().UTC()
	for _, o := range open {
		key := o.RecordType + ":" + o.RecordID
		if stillOpen[key] {
			continue
		}
		if err := db.WithContext(ctx).Exec(
			`UPDATE dataquality_dataqualityissue SET status = 'resolved', resolved_at = ?, updated_at = ? WHERE id = ?`,
			now, now, o.ID,
		).Error; err != nil {
			fmt.Printf("[DataQuality sweep] reconcile update failed practice=%d issue=%d: %v\n", practiceID, o.ID, err)
		}
	}
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `go test ./internal/dentally/scheduler/... -run TestSweepMissingInfo -v`
Expected: `--- PASS` for both subtests.

- [ ] **Step 5: Commit**

```bash
git add internal/dentally/scheduler/dataquality_sweep.go internal/dentally/scheduler/dataquality_sweep_test.go
git commit -m "feat(dataQuality): add sweepMissingInfo nightly detector"
```

(Leave the actual commit to the user.)

---

### Task 3: `duplicate_contact` detector

**Files:**
- Modify: `EmailServiceGo/internal/dentally/scheduler/dataquality_sweep.go`
- Modify: `EmailServiceGo/internal/dentally/scheduler/dataquality_sweep_test.go`

- [ ] **Step 1: Write the failing test**

```go
// append to dataquality_sweep_test.go

// seedPersonWithPhoneChannel creates a Person and a shared phone ContactChannel,
// linking them via PersonChannel — the minimal setup DataQualityDuplicatesView's
// (and this sweep's) shared-channel query groups on.
func seedPersonWithPhoneChannel(t *testing.T, db *gorm.DB, practiceID int, firstName, lastName, phoneCanonical string) int {
	t.Helper()
	now := time.Now().UTC()
	var personID int
	if err := db.Raw(
		`INSERT INTO "TreatmentPlan_person" (practice_id, first_name, last_name, created_at, updated_at)
		 VALUES (?, ?, ?, ?, ?) RETURNING id`,
		practiceID, firstName, lastName, now, now,
	).Scan(&personID).Error; err != nil {
		t.Fatalf("seed person: %v", err)
	}

	var channelID int
	if err := db.Raw(
		`SELECT id FROM "TreatmentPlan_contactchannel" WHERE practice_id = ? AND kind = 'phone' AND canonical_value = ?`,
		practiceID, phoneCanonical,
	).Scan(&channelID).Error; err != nil || channelID == 0 {
		if err := db.Raw(
			`INSERT INTO "TreatmentPlan_contactchannel"
			 (practice_id, kind, canonical_value, raw_sample, needs_review, created_at, updated_at)
			 VALUES (?, 'phone', ?, '', false, ?, ?) RETURNING id`,
			practiceID, phoneCanonical, now, now,
		).Scan(&channelID).Error; err != nil {
			t.Fatalf("seed channel: %v", err)
		}
	}

	if err := db.Exec(
		`INSERT INTO "TreatmentPlan_personchannel" (person_id, channel_id, role, is_primary, created_at, updated_at)
		 VALUES (?, ?, '', true, ?, ?)`,
		personID, channelID, now, now,
	).Error; err != nil {
		t.Fatalf("link person to channel: %v", err)
	}

	// A patient row so the cluster has 2+ contacts (sweepDuplicateContacts, like the
	// Python view, requires len(contacts) >= 2 across the whole cluster, not per person).
	if err := db.Exec(
		`INSERT INTO "TreatmentPlan_patient" (practice_id, person_id, first_name, last_name, created_at, updated_at)
		 VALUES (?, ?, ?, ?, ?, ?)`,
		practiceID, personID, firstName, lastName, now, now,
	).Error; err != nil {
		t.Fatalf("seed patient for person: %v", err)
	}

	return personID
}

func TestSweepDuplicateContacts_DetectsSharedChannelSimilarName(t *testing.T) {
	db := sweepTestDB(t)
	ctx := context.Background()
	practiceID := seedPractice(t, db, "Dup Contact Sweep Test")

	seedPersonWithPhoneChannel(t, db, practiceID, "Jon", "Smith", "+447700900901")
	seedPersonWithPhoneChannel(t, db, practiceID, "John", "Smith", "+447700900901")

	sweepDuplicateContacts(ctx, db, practiceID)

	var count int
	if err := db.Raw(
		`SELECT count(*) FROM dataquality_dataqualityissue
		 WHERE practice_id = ? AND classifier = 'duplicate_contact' AND status = 'open'`,
		practiceID,
	).Row().Scan(&count); err != nil {
		t.Fatalf("count query: %v", err)
	}
	if count != 1 {
		t.Errorf("expected 1 open duplicate_contact cluster, got %d", count)
	}
}

func TestSweepDuplicateContacts_SkipsDifferentDOBs(t *testing.T) {
	db := sweepTestDB(t)
	ctx := context.Background()
	practiceID := seedPractice(t, db, "Dup Contact DOB Test")

	fatherID := seedPersonWithPhoneChannel(t, db, practiceID, "John", "Smith", "+447700900902")
	sonID := seedPersonWithPhoneChannel(t, db, practiceID, "John", "Smith", "+447700900902")
	if err := db.Exec(`UPDATE "TreatmentPlan_person" SET dob = '1970-01-01' WHERE id = ?`, fatherID).Error; err != nil {
		t.Fatalf("set father dob: %v", err)
	}
	if err := db.Exec(`UPDATE "TreatmentPlan_person" SET dob = '2005-06-15' WHERE id = ?`, sonID).Error; err != nil {
		t.Fatalf("set son dob: %v", err)
	}

	sweepDuplicateContacts(ctx, db, practiceID)

	var count int
	if err := db.Raw(
		`SELECT count(*) FROM dataquality_dataqualityissue WHERE practice_id = ? AND classifier = 'duplicate_contact'`,
		practiceID,
	).Row().Scan(&count); err != nil {
		t.Fatalf("count query: %v", err)
	}
	if count != 0 {
		t.Errorf("expected 0 duplicate_contact issues for a father/son household (different DOBs), got %d", count)
	}
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `go test ./internal/dentally/scheduler/... -run TestSweepDuplicateContacts -v`
Expected: FAIL — `undefined: sweepDuplicateContacts`.

- [ ] **Step 3: Implement `sweepDuplicateContacts`**

Append to `dataquality_sweep.go`:

```go
type channelGroupRow struct {
	ChannelID int
	Kind      string
	PersonID  int
	FirstName string
	LastName  string
	DOB       *time.Time
}

type dqContact struct {
	ID    string `json:"id"`
	Name  string `json:"name"`
	Email string `json:"email"`
	Phone string `json:"phone"`
	Type  string `json:"type"`
}

// sweepDuplicateContacts ports DataQualityDuplicatesView.get exactly
// (TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/data_quality_views.py) —
// group Persons sharing a contact channel, flag pairs whose names are similar
// (nameSimilar) and whose DOBs don't actively conflict, skip pathological
// mega-channels (40+ distinct persons — a shared generic value, not a real cluster).
func sweepDuplicateContacts(ctx context.Context, db *gorm.DB, practiceID int) {
	var rows []channelGroupRow
	if err := db.WithContext(ctx).Raw(
		`SELECT pc.channel_id AS channel_id, cc.kind AS kind, p.id AS person_id,
		        p.first_name AS first_name, p.last_name AS last_name, p.dob AS dob
		 FROM "TreatmentPlan_personchannel" pc
		 JOIN "TreatmentPlan_person" p ON p.id = pc.person_id
		 JOIN "TreatmentPlan_contactchannel" cc ON cc.id = pc.channel_id
		 WHERE p.merged_into_id IS NULL AND p.practice_id = ?
		   AND (p.first_name <> '' OR p.last_name <> '')
		   AND pc.channel_id NOT IN (
		     SELECT channel_id FROM "TreatmentPlan_duplicateclusterdismissal" WHERE practice_id = ?
		   )`,
		practiceID, practiceID,
	).Scan(&rows).Error; err != nil {
		fmt.Printf("[DataQuality sweep] duplicate_contact query failed practice=%d: %v\n", practiceID, err)
		return
	}

	byChannel := make(map[int][]channelGroupRow)
	kindByChannel := make(map[int]string)
	for _, r := range rows {
		byChannel[r.ChannelID] = append(byChannel[r.ChannelID], r)
		kindByChannel[r.ChannelID] = r.Kind
	}

	seen := make(map[string]bool)
	now := time.Now().UTC()

	for channelID, members := range byChannel {
		distinct := make(map[int]channelGroupRow)
		for _, m := range members {
			distinct[m.PersonID] = m
		}
		if len(distinct) > 40 {
			continue // pathological mega-channel — a shared generic value, not a cluster
		}

		ids := make([]int, 0, len(distinct))
		for id := range distinct {
			ids = append(ids, id)
		}

		dupIDs := make(map[int]bool)
		for i := 0; i < len(ids); i++ {
			for j := i + 1; j < len(ids); j++ {
				pi, pj := distinct[ids[i]], distinct[ids[j]]
				if pi.DOB != nil && pj.DOB != nil && !pi.DOB.Equal(*pj.DOB) {
					continue
				}
				if nameSimilar(pi.FirstName, pj.FirstName) && nameSimilar(pi.LastName, pj.LastName) {
					dupIDs[pi.PersonID] = true
					dupIDs[pj.PersonID] = true
				}
			}
		}
		if len(dupIDs) < 2 {
			continue
		}

		personIDs := make([]int, 0, len(dupIDs))
		for id := range dupIDs {
			personIDs = append(personIDs, id)
		}

		contacts := loadContactsForPersons(ctx, db, practiceID, personIDs)
		if len(contacts) < 2 {
			continue
		}

		recordID := fmt.Sprintf("channel-%d", channelID)
		seen["person_cluster:"+recordID] = true

		matchType := "phone"
		if kindByChannel[channelID] == "email" {
			matchType = "email"
		}
		detail := map[string]any{
			"channel_id": channelID,
			"match_type": matchType,
			"contacts":   contacts,
		}
		detailJSON, _ := json.Marshal(detail)

		record := map[string]any{
			"practice_id": practiceID,
			"classifier":  "duplicate_contact",
			"source":      "go_sweep",
			"record_type": "person_cluster",
			"record_id":   recordID,
			"status":      "open",
			"detail":      gorm.Expr("CAST(? AS jsonb)", string(detailJSON)),
			"updated_at":  now,
		}
		tx := db.WithContext(ctx).Table("dataquality_dataqualityissue").
			Where("practice_id = ? AND classifier = 'duplicate_contact' AND record_id = ?", practiceID, recordID).
			Updates(record)
		if tx.Error != nil {
			fmt.Printf("[DataQuality sweep] duplicate_contact update failed practice=%d channel=%d: %v\n", practiceID, channelID, tx.Error)
			continue
		}
		if tx.RowsAffected == 0 {
			record["created_at"] = now
			if err := db.WithContext(ctx).Table("dataquality_dataqualityissue").Create(record).Error; err != nil {
				fmt.Printf("[DataQuality sweep] duplicate_contact insert failed practice=%d channel=%d: %v\n", practiceID, channelID, err)
			}
		}
	}

	resolveStaleIssues(ctx, db, practiceID, "duplicate_contact", seen)
}

// loadContactsForPersons fetches a display-ready contact summary (Patient + Intake
// rows) for a set of Person ids — same shape as Django's _dq_contact.
func loadContactsForPersons(ctx context.Context, db *gorm.DB, practiceID int, personIDs []int) []dqContact {
	var contacts []dqContact

	var patients []struct {
		ID        int
		FirstName string
		LastName  string
		Email     string
		Phone     string
	}
	db.WithContext(ctx).Raw(
		`SELECT id, first_name, last_name, COALESCE(email, '') AS email, COALESCE(phone_number, '') AS phone
		 FROM "TreatmentPlan_patient" WHERE practice_id = ? AND person_id IN ?`,
		practiceID, personIDs,
	).Scan(&patients)
	for _, p := range patients {
		contacts = append(contacts, dqContact{
			ID: fmt.Sprintf("%d", p.ID), Name: strings.TrimSpace(p.FirstName + " " + p.LastName),
			Email: p.Email, Phone: p.Phone, Type: "patient",
		})
	}

	var intakes []struct {
		ID        int
		FirstName string
		LastName  string
		Email     string
		Phone     string
	}
	db.WithContext(ctx).Raw(
		`SELECT id, first_name, last_name, COALESCE(email, '') AS email, COALESCE(phone_number, '') AS phone
		 FROM "TreatmentPlan_intake" WHERE practice_id = ? AND person_id IN ?`,
		practiceID, personIDs,
	).Scan(&intakes)
	for _, i := range intakes {
		contacts = append(contacts, dqContact{
			ID: fmt.Sprintf("%d", i.ID), Name: strings.TrimSpace(i.FirstName + " " + i.LastName),
			Email: i.Email, Phone: i.Phone, Type: "intake",
		})
	}

	return contacts
}
```

Add `"encoding/json"` and `"strings"` to this file's imports alongside the existing
`"context"`, `"fmt"`, `"time"`, `"gorm.io/gorm"`.

- [ ] **Step 4: Run the test to verify it passes**

Run: `go test ./internal/dentally/scheduler/... -run TestSweepDuplicateContacts -v`
Expected: `--- PASS` for both subtests.

- [ ] **Step 5: Run the full sweep test file together**

Run: `go test ./internal/dentally/scheduler/... -run 'TestSweep|TestNameSimilar|TestSequenceMatcherRatio' -v`
Expected: all `PASS`.

- [ ] **Step 6: Commit**

```bash
git add internal/dentally/scheduler/dataquality_sweep.go internal/dentally/scheduler/dataquality_sweep_test.go
git commit -m "feat(dataQuality): add sweepDuplicateContacts nightly detector"
```

(Leave the actual commit to the user.)

---

### Task 4: Wire both detectors into a nightly cron entry

**Files:**
- Modify: `EmailServiceGo/internal/dentally/scheduler/scheduler.go`

- [ ] **Step 1: Add the cron registration**

In `scheduler.go`'s `Start()` method, add a new cron entry alongside the existing
`0 2 * * *` daily-sync registration (same slot, per the spec's approved "nightly" cadence
decision — this can safely run in the same 2:00 UTC window since it queries independent
tables from the daily Dentally sync and doesn't block on it):

```go
	// Nightly data-quality sweep: duplicate-contact clusters + missing-info gaps,
	// materialized as DataQualityIssue rows (see
	// docs/superpowers/specs/2026-08-13-unified-data-quality-design.md, Section 2).
	// Runs for EVERY active practice, not just Dentally-integrated ones — these
	// issue types aren't Dentally-import-specific.
	_, err = s.cron.AddFunc("0 2 * * *", func() {
		logger.Info("Starting nightly data-quality sweep at 2:00 UTC")
		s.runDataQualitySweep()
	})
	if err != nil {
		return fmt.Errorf("failed to schedule data-quality sweep: %w", err)
	}
```

Add this block immediately after the existing daily-sync `AddFunc` call (before the
recall auto-sync registration), so `Start()`'s error-handling `if err != nil { return
... }` chain stays intact.

- [ ] **Step 2: Implement `runDataQualitySweep`**

Add this method to `scheduler.go`, following the exact worker-pool pattern already used
by `runDailySync` (semaphore + WaitGroup, `MaxConcurrentWorkers` limit):

```go
// runDataQualitySweep runs sweepDuplicateContacts + sweepMissingInfo for every
// active practice (not gated on Dentally integration — these issue types apply to
// any Patient/Intake/Person regardless of origin).
func (s *DailySyncScheduler) runDataQualitySweep() {
	ctx := context.Background()

	var practices []PracticeInfo
	if err := s.db.WithContext(ctx).Raw(
		`SELECT id, name FROM "UserAuthentication_practice" WHERE is_active = true AND is_archived = false`,
	).Scan(&practices).Error; err != nil {
		logger.Error("data-quality sweep: failed to list active practices", zap.Error(err))
		return
	}

	semaphore := make(chan struct{}, MaxConcurrentWorkers)
	var wg sync.WaitGroup

	for _, p := range practices {
		wg.Add(1)
		semaphore <- struct{}{}
		go func(practiceID int) {
			defer func() {
				<-semaphore
				wg.Done()
			}()
			sweepCtx, cancel := context.WithTimeout(ctx, 10*time.Minute)
			defer cancel()
			sweepDuplicateContacts(sweepCtx, s.db, practiceID)
			sweepMissingInfo(sweepCtx, s.db, practiceID)
		}(p.ID)
	}
	wg.Wait()

	logger.Info("Data-quality sweep completed", zap.Int("practices_swept", len(practices)))
}
```

- [ ] **Step 3: Build to confirm no compile errors**

Run: `cd EmailServiceGo && go build ./...`
Expected: no output, exit code 0.

- [ ] **Step 4: Run the scheduler package's existing test suite to confirm no regression**

Run: `go test ./internal/dentally/scheduler/... -v`
Expected: all tests pass, including the new ones from Tasks 1-3.

- [ ] **Step 5: Commit**

```bash
git add internal/dentally/scheduler/scheduler.go
git commit -m "feat(dataQuality): wire nightly duplicate-contact/missing-info sweep into cron"
```

(Leave the actual commit to the user.)

---

## Self-review notes

- **Spec coverage**: implements Section 2's "Scan-time" writer in full, including the
  "nightly" cadence decision and the "entirely in Go" architecture decision, both
  confirmed with the user during brainstorming.
- **No placeholders**: every step has complete, runnable code.
- **Type consistency**: `sweepDuplicateContacts`/`sweepMissingInfo`/`resolveStaleIssues`
  signatures (`ctx context.Context, db *gorm.DB, practiceID int`, plus classifier/map
  args on `resolveStaleIssues`) match between their Task 2/3 definitions and the Task 4
  call sites in `runDataQualitySweep` — re-checked after writing both.
- **Known limitation flagged explicitly** at the top of this plan: the shared fixture is
  duplicated (not a single canonical file) because Django and Go are separate repos with
  independent CI — a real, smaller residual drift risk versus a true single source of
  truth, called out rather than silently assumed away.
- **Reconciliation is shared** between both detectors via `resolveStaleIssues` (DRY —
  the "if the sweep doesn't see it anymore, resolve it" rule is identical for both
  classifiers, implemented once).
- **Cross-phase bug found and fixed during Phase 5**: `sweepDuplicateContacts`'s
  `detail` payload originally carried only `channel_id`/`match_type`/`contacts` — it
  never wrote `member_person_ids`, but Phase 5's `DataQualityIssueViewSet.merge`
  action requires exactly that key to authorize which persons can be merged. Every
  merge request against a real sweep-detected cluster would have been silently
  rejected (empty member-id set). Fixed here: `personIDs` (already computed, now
  sorted for determinism) is included in `detail` as `member_person_ids`, and a new
  assertion in `TestSweepDuplicateContacts_DetectsSharedChannelSimilarName` pins it.
  Same discovery also added `PersonID`/`person_id` to `dqContact`/`loadContactsForPersons`
  — without it, the frontend merge UI (Phase 5 Task 7) has no way to map a named
  contact the user picks back to the Person id `merge` actually operates on.

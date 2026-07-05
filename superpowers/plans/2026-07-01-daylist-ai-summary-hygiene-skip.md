# AI Summary Hygiene-Skip Wiring Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire the already-built, already-parity-proven `daylist/classification` engine into the AI summary's visit-selection logic, so hygiene-only visits are skipped when picking the "last two visits" to summarize — walking back through history until real clinical content is found, using each practice's real configured terminology instead of guessing.

**Architecture:** Extract the AI summary's inline "group treatment-plan-items into visit-days, take the first two" logic out of `GetPatientData` into a small, pure, unit-testable function (`selectRecentEncounters`), teach it to skip visit-days that classify as hygiene-and-nothing-else via the classification engine, then wire `GetPatientData` to fetch the practice's real terminology (falling back to defaults on any error) and pass it through.

**Tech Stack:** Go 1.24, the `internal/dentally/daylist/classification` package (built and parity-proven in a prior plan), `gorm.io/gorm`, `gorm.io/driver/sqlite` (for the one DB-touching unit test).

**Scope note:** This plan covers ONLY the AI summary's hygiene-skip wiring (spec doc §4 point 1). Migrating `daylist/opportunity/classifiers.go` and `daylist/ai/db.go`'s `classifyCategory()` to the same engine (spec §4 points 2-3), and the two-round objective accuracy benchmark (spec §6.4-6.5), remain separate, not-yet-written follow-up plans — this plan produces one complete, working, testable change on its own.

**Reference spec:** `docs/superpowers/specs/2026-07-01-daylist-classification-engine-design.md` (§4 point 1, §5)
**Reference plan (prior, already executed):** `docs/superpowers/plans/2026-07-01-daylist-classification-engine.md`

---

## Task 1: Extract encounter selection into a pure, testable function — with hygiene-skip

**Files:**
- Modify: `EmailServiceGo/internal/dentally/daylist/ai/db.go:41-193` (the `GetPatientData` function — see exact before/after below)
- Test: `EmailServiceGo/internal/dentally/daylist/ai/encounters_test.go` (new)

### Background: exact current code being replaced

Lines 41-48 of `db.go` (function signature, unchanged):
```go
func GetPatientData(ctx context.Context, db *gorm.DB, practiceID int, patientName string, syncDate string) (*PatientData, error) {
	patientName = NormalizeName(patientName)
	data := &PatientData{
		PatientName:       patientName,
		TodayAppointments: make([]TodayAppointment, 0),
		RecentEncounters:  make([]Encounter, 0),
	}
```

Lines 147-193 (the block being replaced — a locally-scoped `encRow` type plus an inline loop that groups treatment-plan-items by visit day and takes the first two distinct days, unconditionally):
```go
	// Last two completed clinical encounters: treatment_plan_items.notes grouped by
	// visit date (completed_at, else date). PRD §6 last-two-encounter source selection.
	type encRow struct {
		ID           string    `gorm:"column:id"`
		Notes        string    `gorm:"column:notes"`
		Nomenclature string    `gorm:"column:nomenclature"`
		EncDate      time.Time `gorm:"column:enc_date"`
	}
	var encRows []encRow
	db.WithContext(ctx).Raw(`
		SELECT id::text AS id, notes, nomenclature,
		       COALESCE(completed_at, date) AS enc_date
		FROM "dentallyIntegration_treatmentplanitem"
		WHERE practice_id = ?
		  AND completed = true
		  AND COALESCE(notes, '') <> ''
		  AND dentally_patient_id = ?
		ORDER BY COALESCE(completed_at, date) DESC
	`, practiceID, dentallyPatientID).Scan(&encRows)

	seenDate := make(map[string]int) // visit date -> index in data.RecentEncounters
	for _, r := range encRows {
		day := r.EncDate.Format("2006-01-02")
		idx, ok := seenDate[day]
		if !ok {
			if len(data.RecentEncounters) >= 2 {
				continue // PRD: do not silently expand beyond two encounters
			}
			data.RecentEncounters = append(data.RecentEncounters, Encounter{
				SourceDate: day,
				Category:   classifyCategory(r.Nomenclature),
			})
			idx = len(data.RecentEncounters) - 1
			seenDate[day] = idx
		}
		enc := &data.RecentEncounters[idx]
		if strings.TrimSpace(r.Notes) != "" {
			if enc.Notes != "" {
				enc.Notes += "\n"
			}
			enc.Notes += r.Notes
		}
		enc.TPIIDs = append(enc.TPIIDs, r.ID)
		if enc.Category == "other" {
			enc.Category = classifyCategory(r.Nomenclature)
		}
	}
```

### What changes

`encRow` moves to package level (so it can be a parameter/return type shared between `GetPatientData` and the new pure function — and so the test file can construct rows directly, without a database). The grouping+selection logic moves into a new package-level function `selectRecentEncounters(rows []encRow, terminology map[string][]string) []Encounter`, which does the same grouping as before but skips any visit-day where every treatment-plan-item on it classifies as hygiene-and-nothing-else (via a new `isHygieneOnlyDay` helper), continuing to walk back through older visits until two non-hygiene-only days are found (or history runs out).

- [ ] **Step 1: Write the failing tests**

Create `EmailServiceGo/internal/dentally/daylist/ai/encounters_test.go`:

```go
package ai

import (
	"reflect"
	"testing"
	"time"

	"github.com/treatmentpath/email-service/internal/dentally/daylist/classification"
)

func day(s string) time.Time {
	t, _ := time.Parse("2006-01-02", s)
	return t
}

func TestSelectRecentEncountersSkipsHygieneOnlyDayAndWalksBack(t *testing.T) {
	terminology := classification.DefaultTerminology()
	rows := []encRow{
		{ID: "1", Notes: "Scale and polish, gums healthy", Nomenclature: "Hygiene visit - scale and polish", EncDate: day("2026-06-01")},
		{ID: "2", Notes: "Crown prep completed, patient tolerated well", Nomenclature: "Crown prep UL6", EncDate: day("2026-05-01")},
		{ID: "3", Notes: "Root canal completed, obturation done", Nomenclature: "Root canal UL6", EncDate: day("2026-04-01")},
	}
	got := selectRecentEncounters(rows, terminology)
	if len(got) != 2 {
		t.Fatalf("expected 2 encounters, got %d: %+v", len(got), got)
	}
	if got[0].SourceDate != "2026-05-01" || got[1].SourceDate != "2026-04-01" {
		t.Fatalf("expected the hygiene-only 2026-06-01 visit to be skipped, got dates %s, %s", got[0].SourceDate, got[1].SourceDate)
	}
}

func TestSelectRecentEncountersKeepsMixedDayNotPurelyHygiene(t *testing.T) {
	terminology := classification.DefaultTerminology()
	rows := []encRow{
		{ID: "1", Notes: "Scale and polish", Nomenclature: "Hygiene visit - scale and polish", EncDate: day("2026-06-01")},
		{ID: "2", Notes: "Small filling placed same visit", Nomenclature: "Crown prep UL6", EncDate: day("2026-06-01")},
	}
	got := selectRecentEncounters(rows, terminology)
	if len(got) != 1 {
		t.Fatalf("expected 1 encounter (mixed day kept, not skipped), got %d: %+v", len(got), got)
	}
	if got[0].SourceDate != "2026-06-01" {
		t.Fatalf("expected the mixed-content day to be kept, got %s", got[0].SourceDate)
	}
}

func TestSelectRecentEncountersConcatenatesNotesForSameDay(t *testing.T) {
	terminology := classification.DefaultTerminology()
	rows := []encRow{
		{ID: "1", Notes: "First note", Nomenclature: "Crown prep UL6", EncDate: day("2026-06-01")},
		{ID: "2", Notes: "Second note", Nomenclature: "Crown fit UL6", EncDate: day("2026-06-01")},
	}
	got := selectRecentEncounters(rows, terminology)
	if len(got) != 1 {
		t.Fatalf("expected 1 encounter, got %d", len(got))
	}
	if got[0].Notes != "First note\nSecond note" {
		t.Fatalf("expected concatenated notes, got %q", got[0].Notes)
	}
	if !reflect.DeepEqual(got[0].TPIIDs, []string{"1", "2"}) {
		t.Fatalf("expected both TPI ids collected in order, got %v", got[0].TPIIDs)
	}
}

func TestSelectRecentEncountersCapsAtTwoEvenWithMoreNonHygieneHistory(t *testing.T) {
	terminology := classification.DefaultTerminology()
	rows := []encRow{
		{ID: "1", Notes: "n1", Nomenclature: "Crown prep UL6", EncDate: day("2026-06-01")},
		{ID: "2", Notes: "n2", Nomenclature: "Root canal UL6", EncDate: day("2026-05-01")},
		{ID: "3", Notes: "n3", Nomenclature: "Extraction UL8", EncDate: day("2026-04-01")},
	}
	got := selectRecentEncounters(rows, terminology)
	if len(got) != 2 {
		t.Fatalf("expected cap of 2 encounters even though 3 non-hygiene days exist, got %d", len(got))
	}
}

func TestSelectRecentEncountersReturnsFewerThanTwoWhenHistoryIsAllHygiene(t *testing.T) {
	terminology := classification.DefaultTerminology()
	rows := []encRow{
		{ID: "1", Notes: "n1", Nomenclature: "Hygiene visit - scale and polish", EncDate: day("2026-06-01")},
		{ID: "2", Notes: "n2", Nomenclature: "Scale and polish", EncDate: day("2026-05-01")},
	}
	got := selectRecentEncounters(rows, terminology)
	if len(got) != 0 {
		t.Fatalf("expected 0 encounters when entire available history is hygiene-only, got %d: %+v", len(got), got)
	}
}

func TestIsHygieneOnlyDayTrueWhenAllRowsAreHygiene(t *testing.T) {
	terminology := classification.DefaultTerminology()
	rows := []encRow{
		{Nomenclature: "Hygiene visit - scale and polish"},
		{Nomenclature: "Scale and polish"},
	}
	if !isHygieneOnlyDay(rows, terminology) {
		t.Fatal("expected true when every row classifies as hygiene-only")
	}
}

func TestIsHygieneOnlyDayFalseWhenOneRowIsNotHygiene(t *testing.T) {
	terminology := classification.DefaultTerminology()
	rows := []encRow{
		{Nomenclature: "Hygiene visit - scale and polish"},
		{Nomenclature: "Crown prep UL6"},
	}
	if isHygieneOnlyDay(rows, terminology) {
		t.Fatal("expected false when at least one row is not hygiene-only")
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/ai/... -run 'TestSelectRecentEncounters|TestIsHygieneOnlyDay' -v`
Expected: FAIL — `undefined: encRow` / `undefined: selectRecentEncounters` / `undefined: isHygieneOnlyDay` (these currently only exist as a local type and inline logic inside `GetPatientData`, not as package-level symbols).

- [ ] **Step 3: Make the changes to `db.go`**

Add this import to the existing import block at the top of `db.go` (alongside the existing imports):
```go
	"github.com/treatmentpath/email-service/internal/dentally/daylist/classification"
```

Replace the entire block shown in "Background" above (lines 147-193) with:
```go
	// Last two completed clinical encounters: treatment_plan_items.notes grouped by
	// visit date (completed_at, else date), skipping routine hygiene-only visits so
	// the AI summary walks back to real clinical content. PRD §6 last-two-encounter
	// source selection.
	var encRows []encRow
	db.WithContext(ctx).Raw(`
		SELECT id::text AS id, notes, nomenclature,
		       COALESCE(completed_at, date) AS enc_date
		FROM "dentallyIntegration_treatmentplanitem"
		WHERE practice_id = ?
		  AND completed = true
		  AND COALESCE(notes, '') <> ''
		  AND dentally_patient_id = ?
		ORDER BY COALESCE(completed_at, date) DESC
	`, practiceID, dentallyPatientID).Scan(&encRows)

	terminology := resolveTerminologyWithFallback(ctx, db, practiceID)
	data.RecentEncounters = selectRecentEncounters(encRows, terminology)
```

Add these package-level declarations to `db.go` (e.g. right after the `GetPatientData` function, before `appendUnique`):
```go
// encRow is one completed treatment-plan-item row contributing to a patient's
// visit history. Package-level (not a local type inside GetPatientData) so
// selectRecentEncounters and isHygieneOnlyDay are unit-testable without a
// database.
type encRow struct {
	ID           string    `gorm:"column:id"`
	Notes        string    `gorm:"column:notes"`
	Nomenclature string    `gorm:"column:nomenclature"`
	EncDate      time.Time `gorm:"column:enc_date"`
}

// selectRecentEncounters groups completed treatment-plan-item rows (already
// ordered newest-first by enc_date) into per-visit-day Encounters, skipping
// any day where isHygieneOnlyDay is true, until two non-hygiene-only days are
// found (or history runs out). PRD §6: do not silently expand beyond two
// encounters.
func selectRecentEncounters(rows []encRow, terminology map[string][]string) []Encounter {
	type dayGroup struct {
		day  string
		rows []encRow
	}
	var groups []dayGroup
	groupIndex := make(map[string]int)
	for _, r := range rows {
		day := r.EncDate.Format("2006-01-02")
		idx, ok := groupIndex[day]
		if !ok {
			groups = append(groups, dayGroup{day: day})
			idx = len(groups) - 1
			groupIndex[day] = idx
		}
		groups[idx].rows = append(groups[idx].rows, r)
	}

	var encounters []Encounter
	for _, g := range groups {
		if len(encounters) >= 2 {
			break // PRD: do not silently expand beyond two encounters
		}
		if isHygieneOnlyDay(g.rows, terminology) {
			continue // routine hygiene visit — walk back further for real content
		}
		enc := Encounter{SourceDate: g.day}
		for i, r := range g.rows {
			if i == 0 {
				enc.Category = classifyCategory(r.Nomenclature)
			}
			if strings.TrimSpace(r.Notes) != "" {
				if enc.Notes != "" {
					enc.Notes += "\n"
				}
				enc.Notes += r.Notes
			}
			enc.TPIIDs = append(enc.TPIIDs, r.ID)
			if enc.Category == "other" {
				enc.Category = classifyCategory(r.Nomenclature)
			}
		}
		encounters = append(encounters, enc)
	}
	return encounters
}

// isHygieneOnlyDay reports whether every treatment-plan-item on this day
// classifies as hygiene and nothing else — i.e. a routine scale-and-polish
// visit with no other notable content. Used by selectRecentEncounters to skip
// hygiene-only visits when picking the AI summary's "last two visits": walk
// back through history until real clinical content is found, rather than
// summarizing a routine hygiene note.
func isHygieneOnlyDay(rows []encRow, terminology map[string][]string) bool {
	for _, r := range rows {
		concepts := classification.ConceptsForText(r.Nomenclature, terminology)
		if len(concepts) != 1 || concepts[0] != "hygiene" {
			return false
		}
	}
	return true
}
```

Note: `resolveTerminologyWithFallback` (used above) is defined in Task 2 of this plan — it doesn't exist yet after this step, so the package will not compile until Task 2 is also done. This is expected and fine within a single subagent's work if you do Tasks 1 and 2 back to back, but if executing task-by-task with review gates between them, add a temporary local definition in this task:
```go
func resolveTerminologyWithFallback(ctx context.Context, db *gorm.DB, practiceID int) map[string][]string {
	return classification.DefaultTerminology() // Task 2 replaces this with the real per-practice fetch+fallback
}
```
placed in `db.go` near the other new functions, so Task 1 compiles and its tests (which don't call `GetPatientData` directly, only `selectRecentEncounters`/`isHygieneOnlyDay`) can run in isolation. Task 2 will replace this stub with the real implementation.

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/ai/... -v`
Expected: PASS — all 7 new tests, plus every pre-existing test in the `ai` package (this package has existing tests like `high_risk_summary_test.go`, `summary_test.go`, `types_test.go` — none of those should regress).

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go build ./...`
Expected: no output, exit code 0.

- [ ] **Step 5: Commit**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
git add internal/dentally/daylist/ai/db.go internal/dentally/daylist/ai/encounters_test.go
git commit -m "feat: skip hygiene-only visits when selecting AI summary's last two encounters"
```

---

## Task 2: Wire real per-practice terminology into `GetPatientData`

**Files:**
- Modify: `EmailServiceGo/internal/dentally/daylist/ai/db.go` (replace the Task-1 stub `resolveTerminologyWithFallback`, or add it fresh if Task 1 and 2 are being done in the same sitting)
- Test: `EmailServiceGo/internal/dentally/daylist/ai/terminology_fallback_test.go` (new)

### What changes

Task 1 either left a temporary stub `resolveTerminologyWithFallback` (if executed as a standalone reviewed task) that just returns `classification.DefaultTerminology()` unconditionally, ignoring the practice entirely — or (if Tasks 1+2 were done together) doesn't have the function yet at all. This task implements the REAL version: fetch the practice's saved terminology overrides from the shared database, merge them onto defaults, and gracefully fall back to pure defaults (logging, not erroring) if the fetch fails — a terminology-lookup outage must never block AI summary generation.

- [ ] **Step 1: Write the failing test**

Create `EmailServiceGo/internal/dentally/daylist/ai/terminology_fallback_test.go`:

```go
package ai

import (
	"context"
	"reflect"
	"testing"

	"github.com/treatmentpath/email-service/internal/dentally/daylist/classification"
	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

// TestResolveTerminologyWithFallbackUsesDefaultsWhenTableMissing simulates a
// real query failure (no daylist_opportunity_terminology table exists at all)
// and proves the function falls back to exact defaults rather than erroring
// out or panicking — a terminology-lookup outage must never block AI summary
// generation.
func TestResolveTerminologyWithFallbackUsesDefaultsWhenTableMissing(t *testing.T) {
	db, err := gorm.Open(sqlite.Open("file::memory:?cache=shared"), &gorm.Config{})
	if err != nil {
		t.Fatalf("open sqlite: %v", err)
	}
	// Deliberately do NOT create daylist_opportunity_terminology — the query
	// inside FetchPracticeOverrides will fail with "no such table".

	got := resolveTerminologyWithFallback(context.Background(), db, 999)
	want := classification.DefaultTerminology()
	if !reflect.DeepEqual(got, want) {
		t.Fatalf("expected fallback to exact defaults on fetch error, got %v, want %v", got, want)
	}
}

// TestResolveTerminologyWithFallbackAppliesRealPracticeOverride proves the
// happy path: when the practice HAS a saved, enabled override, it's actually
// applied (not silently ignored).
func TestResolveTerminologyWithFallbackAppliesRealPracticeOverride(t *testing.T) {
	type row struct {
		PracticeID int    `gorm:"column:practice_id"`
		Mappings   string `gorm:"column:mappings"`
		Enabled    bool   `gorm:"column:enabled"`
	}
	db, err := gorm.Open(sqlite.Open("file::memory:?cache=shared"), &gorm.Config{})
	if err != nil {
		t.Fatalf("open sqlite: %v", err)
	}
	// Minimal ad-hoc table matching the real daylist_opportunity_terminology
	// shape, scoped to this test only.
	if err := db.Exec(`CREATE TABLE daylist_opportunity_terminology (practice_id INTEGER, mappings TEXT, enabled BOOLEAN)`).Error; err != nil {
		t.Fatalf("create table: %v", err)
	}
	if err := db.Exec(
		`INSERT INTO daylist_opportunity_terminology (practice_id, mappings, enabled) VALUES (?, ?, ?)`,
		4, `{"hygiene": {"words": ["myword123"]}}`, true,
	).Error; err != nil {
		t.Fatalf("insert fixture row: %v", err)
	}

	got := resolveTerminologyWithFallback(context.Background(), db, 4)
	if !reflect.DeepEqual(got["hygiene"], []string{"myword123"}) {
		t.Fatalf("expected practice override applied to hygiene concept, got %v", got["hygiene"])
	}
	// Untouched concepts still fall back to defaults.
	defaults := classification.DefaultTerminology()
	if !reflect.DeepEqual(got["exam"], defaults["exam"]) {
		t.Fatalf("expected untouched 'exam' concept to remain default, got %v", got["exam"])
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/ai/... -run TestResolveTerminologyWithFallback -v`
Expected: if Task 1 left the stub version (`return classification.DefaultTerminology()` unconditionally), `TestResolveTerminologyWithFallbackAppliesRealPracticeOverride` FAILS (the stub ignores the practice's override entirely) while `TestResolveTerminologyWithFallbackUsesDefaultsWhenTableMissing` incidentally passes (since the stub always returns defaults). If Task 1+2 are being done together and neither function nor stub exists yet, both FAIL with `undefined: resolveTerminologyWithFallback`.

- [ ] **Step 3: Implement (or replace the stub with) the real function**

In `db.go`, add or replace `resolveTerminologyWithFallback` with:
```go
// resolveTerminologyWithFallback fetches a practice's saved "counts as
// hygiene" (and other concept) terminology overrides and merges them onto the
// built-in defaults. On any fetch/parse error it logs and falls back to pure
// defaults — a terminology-lookup outage must never block AI summary
// generation.
func resolveTerminologyWithFallback(ctx context.Context, db *gorm.DB, practiceID int) map[string][]string {
	overrides, err := classification.FetchPracticeOverrides(ctx, db, practiceID)
	if err != nil {
		logger.Error("fetching opportunity terminology overrides failed; using defaults",
			zap.Int("practice_id", practiceID), zap.Error(err))
		overrides = &classification.PracticeOverrides{}
	}
	return classification.ResolveTerminology(overrides, classification.DefaultTerminology())
}
```

(This makes `GetPatientData`'s existing call `terminology := resolveTerminologyWithFallback(ctx, db, practiceID)` from Task 1 actually do real per-practice lookups — no further change needed in `GetPatientData` itself.)

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/ai/... -v`
Expected: PASS — both new tests, plus everything from Task 1 and every pre-existing `ai` package test.

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go build ./...`
Expected: no output, exit code 0.

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go vet ./internal/dentally/daylist/ai/...`
Expected: no output, exit code 0.

- [ ] **Step 5: Commit**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
git add internal/dentally/daylist/ai/db.go internal/dentally/daylist/ai/terminology_fallback_test.go
git commit -m "feat: wire real per-practice terminology into AI summary hygiene-skip"
```

---

## What's Next (not in this plan)

- Migrate `daylist/opportunity/classifiers.go`'s hardcoded term lists to call the `classification` engine (spec §4 point 2) — separate plan.
- Migrate `daylist/ai/db.go`'s `classifyCategory()`'s overlapping buckets (hygiene/exam/treatment-ish) to the engine (spec §4 point 3) — separate plan. Note: after this plan, `classifyCategory()` and the classification engine's `ConceptsForText` co-exist and are BOTH consulted (the former still labels the visible `Category` field sent to the AI prompt; the latter now ALSO independently decides hygiene-skip). This is intentional — this plan only changes the skip decision, not the display category — but a future migration could unify them.
- Two-round objective accuracy benchmark against 100 + 100 real appointments (spec §6.4-6.5), and before/after regression diffs on the opportunity badge engine specifically (spec §6.6-6.7), before either of the above migrations ship.

---

## Plan Self-Review

**Spec coverage:** Implements design doc §4 point 1 (hygiene-skip in `daylist/ai`'s visit selection) and the data flow in §5 for that specific path. §4 points 2-3, §6.4-6.8 explicitly deferred (see "What's Next").

**Placeholder scan:** No TBD/TODO markers. Task 1's temporary stub for `resolveTerminologyWithFallback` is explicitly a real, compilable piece of code with a clear comment explaining it's superseded by Task 2 — not a vague placeholder; it's there so Task 1 is independently compilable and testable if run through separate review gates.

**Type consistency:** `encRow` (fields ID/Notes/Nomenclature/EncDate) is defined once in Task 1 and used identically in both tasks' test files. `selectRecentEncounters(rows []encRow, terminology map[string][]string) []Encounter` and `isHygieneOnlyDay(rows []encRow, terminology map[string][]string) bool` signatures match between their Task 1 definition and test usage. `resolveTerminologyWithFallback(ctx context.Context, db *gorm.DB, practiceID int) map[string][]string` signature is consistent between its Task 1 stub, Task 2's real implementation, and both tasks' test/call-site usage.

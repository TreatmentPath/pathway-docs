# Daylist Classification Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a standalone, fully-tested Go package (`internal/dentally/daylist/classification`) that ports Django's `opportunity_vocab.py` "counts as" terminology-matching logic — canonical/word-list vocabulary, per-practice override resolution, and text-to-concept matching — proven byte-for-byte faithful to Django's actual runtime output via a golden fixture.

**Architecture:** A new Go package embeds a verbatim copy of Django's `opportunity_group.csv`, loads it into a canonical→words vocabulary once, expands the same hardcoded default concept→canonical mapping Django uses, merges in a practice's real per-practice overrides (read from the shared `daylist_opportunity_terminology` table Django already writes to), and matches cleaned free text against the resolved word lists. No existing code is modified in this plan — this package is not yet wired into any consumer (that's a follow-up plan, once this one is reviewed and proven).

**Tech Stack:** Go 1.24, `gorm.io/gorm` (raw SQL + sqlite in-memory for tests), Go's `encoding/csv` and `encoding/json` stdlib, Python/Django management command (fixture generation only, not shipped code).

**Reference spec:** `docs/superpowers/specs/2026-07-01-daylist-classification-engine-design.md`

---

## Task 1: Package scaffold, embedded CSV, and `CleanText`

**Files:**
- Create: `EmailServiceGo/internal/dentally/daylist/classification/data/opportunity_group.csv`
- Create: `EmailServiceGo/internal/dentally/daylist/classification/clean.go`
- Test: `EmailServiceGo/internal/dentally/daylist/classification/clean_test.go`

- [ ] **Step 1: Create the data directory and copy the CSV verbatim**

This is a byte-for-byte copy of Django's `dentallyIntegration/data/opportunity_group.csv` — the shared source of truth for canonical→word-list vocabulary. Copying (not regenerating) guarantees identical expansion on both sides.

Run:
```bash
mkdir -p /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo/internal/dentally/daylist/classification/data
cp /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/data/opportunity_group.csv \
   /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo/internal/dentally/daylist/classification/data/opportunity_group.csv
```
Expected: the new file exists and `diff` against the Django source shows no differences.
```bash
diff /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/dentallyIntegration/data/opportunity_group.csv \
     /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo/internal/dentally/daylist/classification/data/opportunity_group.csv
```
Expected output: no output (files identical).

- [ ] **Step 2: Write the failing test for `CleanText`**

```go
package classification

import "testing"

func TestCleanText(t *testing.T) {
	cases := []struct {
		name string
		in   string
		want string
	}{
		{"lowercases", "Scale AND Polish", "scale and polish"},
		{"strips html tags", "PA <b>radiograph</b> taken", "pa radiograph taken"},
		{"strips punctuation", "UL6 - crown, fitted!", "ul6 crown fitted"},
		{"collapses whitespace", "hygiene   visit\tcompleted", "hygiene visit completed"},
		{"trims ends", "  crown prep  ", "crown prep"},
		{"empty input", "", ""},
		{"numbers survive", "ul8 wisdom review", "ul8 wisdom review"},
	}
	for _, c := range cases {
		t.Run(c.name, func(t *testing.T) {
			got := CleanText(c.in)
			if got != c.want {
				t.Fatalf("CleanText(%q) = %q, want %q", c.in, got, c.want)
			}
		})
	}
}
```

- [ ] **Step 3: Run the test to verify it fails**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/classification/... -run TestCleanText -v`
Expected: FAIL — `undefined: CleanText` (package doesn't exist yet).

- [ ] **Step 4: Implement `CleanText`**

```go
// Package classification ports Django's opportunity_vocab.py "counts as"
// terminology-matching logic into Go: a canonical/word-list vocabulary
// (sourced from the same opportunity_group.csv Django uses), per-practice
// override resolution, and text-to-concept matching.
package classification

import (
	"regexp"
	"strings"
)

var (
	htmlTagRe = regexp.MustCompile(`<[^>]+>`)
	nonWordRe = regexp.MustCompile(`[^a-z0-9 ]+`)
	wsRe      = regexp.MustCompile(`\s+`)
)

// CleanText normalises a nomenclature/description string for matching:
// strip HTML tags, lowercase, strip anything that isn't a letter/number/space,
// collapse whitespace. Mirrors Django's opportunity_vocab.clean_text exactly —
// HTML is stripped BEFORE lowercasing so tags never leak into tokens.
func CleanText(s string) string {
	if s == "" {
		return ""
	}
	out := htmlTagRe.ReplaceAllString(s, " ")
	out = strings.ToLower(out)
	out = nonWordRe.ReplaceAllString(out, " ")
	out = wsRe.ReplaceAllString(out, " ")
	return strings.TrimSpace(out)
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/classification/... -run TestCleanText -v`
Expected: PASS (all 7 subtests).

- [ ] **Step 6: Commit**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
git add internal/dentally/daylist/classification/
git commit -m "feat: scaffold daylist classification package with CleanText"
```

---

## Task 2: Vocabulary loader and default terminology

**Files:**
- Create: `EmailServiceGo/internal/dentally/daylist/classification/vocab.go`
- Test: `EmailServiceGo/internal/dentally/daylist/classification/vocab_test.go`

- [ ] **Step 1: Write the failing tests**

```go
package classification

import "testing"

func TestLoadVocabIncludesEveryCanonicalRegardlessOfKind(t *testing.T) {
	v := loadVocab()
	// "wisdom" and "monitor" are both kind=exclude in the CSV, but Django's
	// _vocab() loads every row unconditionally — kind is metadata, not a filter.
	for _, canon := range []string{"hygiene", "exam", "radiograph", "crown", "root_canal",
		"missing", "implant", "bridge", "denture", "wisdom", "monitor", "scan", "extraction", "filling"} {
		if len(v[canon]) == 0 {
			t.Fatalf("expected non-empty word list for canonical %q", canon)
		}
	}
}

func TestLoadVocabWordsAreCleaned(t *testing.T) {
	v := loadVocab()
	for _, w := range v["hygiene"] {
		if w != CleanText(w) {
			t.Fatalf("word %q in hygiene vocab is not pre-cleaned", w)
		}
	}
}

func TestDefaultTerminologyHasAllElevenConcepts(t *testing.T) {
	dt := DefaultTerminology()
	want := []string{"hygiene", "exam", "radiograph", "scan", "crown", "root_canal",
		"missing", "replacement", "restoration", "wisdom", "watch"}
	if len(dt) != len(want) {
		t.Fatalf("DefaultTerminology() has %d concepts, want %d: %v", len(dt), len(want), dt)
	}
	for _, c := range want {
		if len(dt[c]) == 0 {
			t.Fatalf("expected non-empty default words for concept %q", c)
		}
	}
}

func TestDefaultTerminologyMissingConceptCombinesTwoCanonicals(t *testing.T) {
	dt := DefaultTerminology()
	// "missing" concept = canonicals ["missing", "extraction"] combined (see
	// opportunity_vocab.CONCEPT_DEFAULT_CANONICALS).
	hasMissingWord, hasExtractionWord := false, false
	for _, w := range dt["missing"] {
		if w == "edentulous" {
			hasMissingWord = true
		}
		if w == "extraction" {
			hasExtractionWord = true
		}
	}
	if !hasMissingWord || !hasExtractionWord {
		t.Fatalf("expected 'missing' concept to combine words from both missing and extraction canonicals, got %v", dt["missing"])
	}
}

func TestDefaultTerminologyWatchConceptMapsToMonitorCanonical(t *testing.T) {
	dt := DefaultTerminology()
	found := false
	for _, w := range dt["watch"] {
		if w == "declined" {
			found = true
		}
	}
	if !found {
		t.Fatalf("expected 'watch' concept to pull words from the 'monitor' canonical, got %v", dt["watch"])
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/classification/... -run 'TestLoadVocab|TestDefaultTerminology' -v`
Expected: FAIL — `undefined: loadVocab` / `undefined: DefaultTerminology`.

- [ ] **Step 3: Implement the vocab loader and default terminology**

```go
package classification

import (
	_ "embed"
	"encoding/csv"
	"strings"
	"sync"
)

// The curated opportunity vocabulary, shipped as a reference CSV embedded at
// build time. This is a verbatim copy of Django's
// dentallyIntegration/data/opportunity_group.csv — see Task 1, Step 1.
// Columns: canonical, kind, member_words. `kind` is documentation only (it is
// NOT used to filter rows — e.g. "wisdom" and "monitor" are kind=exclude but
// still contribute real words to the "wisdom" and "watch" concepts below).
//
//go:embed data/opportunity_group.csv
var opportunityGroupCSV string

var (
	vocabOnce sync.Once
	vocab     map[string][]string // canonical (lowercase) -> cleaned member words
)

// loadVocab parses the embedded CSV once and caches it. Every row is loaded
// regardless of its `kind` column, mirroring Django's opportunity_vocab._vocab().
func loadVocab() map[string][]string {
	vocabOnce.Do(func() {
		v := make(map[string][]string)
		r := csv.NewReader(strings.NewReader(opportunityGroupCSV))
		r.FieldsPerRecord = -1
		rows, err := r.ReadAll()
		if err != nil {
			vocab = v
			return
		}
		for i, row := range rows {
			if i == 0 || len(row) == 0 { // header
				continue
			}
			canon := strings.ToLower(strings.TrimSpace(row[0]))
			if canon == "" {
				continue
			}
			var words []string
			if len(row) >= 3 {
				for _, w := range strings.Fields(row[2]) {
					if cw := CleanText(w); cw != "" {
						words = append(words, cw)
					}
				}
			}
			v[canon] = words
		}
		vocab = v
	})
	return vocab
}

// conceptDefaultCanonicals: classifier concepts -> the canonicals that, by
// default, stand for each. Ported verbatim from Django's
// opportunity_vocab.CONCEPT_DEFAULT_CANONICALS — keep these two in sync.
var conceptDefaultCanonicals = map[string][]string{
	"hygiene":     {"hygiene"},
	"exam":        {"exam"},
	"radiograph":  {"radiograph"},
	"scan":        {"scan"},
	"crown":       {"crown"},
	"root_canal":  {"root_canal"},
	"missing":     {"missing", "extraction"},
	"replacement": {"implant", "bridge", "denture"},
	"restoration": {"filling"},
	"wisdom":      {"wisdom"},
	"watch":       {"monitor"},
}

var (
	defaultTerminologyOnce sync.Once
	defaultTerminologyMap  map[string][]string
)

// DefaultTerminology returns the built-in concept -> words mapping (the
// fallback used when a practice has no saved override, or an override is
// disabled). Cached after first call. Mirrors Django's default_terminology().
func DefaultTerminology() map[string][]string {
	defaultTerminologyOnce.Do(func() {
		v := loadVocab()
		out := make(map[string][]string, len(conceptDefaultCanonicals))
		for concept, canonicals := range conceptDefaultCanonicals {
			var words []string
			seen := make(map[string]struct{})
			for _, canon := range canonicals {
				for _, w := range v[canon] {
					if _, dup := seen[w]; !dup {
						seen[w] = struct{}{}
						words = append(words, w)
					}
				}
			}
			out[concept] = words
		}
		defaultTerminologyMap = out
	})
	return defaultTerminologyMap
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/classification/... -v`
Expected: PASS (all tests in the package so far).

- [ ] **Step 5: Commit**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
git add internal/dentally/daylist/classification/
git commit -m "feat: add vocab loader and default terminology to classification package"
```

---

## Task 3: Concept matching (`ConceptsForText`)

**Files:**
- Create: `EmailServiceGo/internal/dentally/daylist/classification/match.go`
- Test: `EmailServiceGo/internal/dentally/daylist/classification/match_test.go`

- [ ] **Step 1: Write the failing tests**

```go
package classification

import (
	"reflect"
	"testing"
)

func TestConceptsForTextSingleWordTokenMatch(t *testing.T) {
	terminology := map[string][]string{
		"hygiene": {"scale", "polish"},
		"crown":   {"crown", "onlay"},
	}
	got := ConceptsForText("Scale and polish performed", terminology)
	want := []string{"hygiene"}
	if !reflect.DeepEqual(got, want) {
		t.Fatalf("ConceptsForText() = %v, want %v", got, want)
	}
}

func TestConceptsForTextMultiWordPhraseMatchesAsSubstring(t *testing.T) {
	terminology := map[string][]string{
		"crown": {"porcelain crown"},
	}
	got := ConceptsForText("Fitted a porcelain crown today", terminology)
	want := []string{"crown"}
	if !reflect.DeepEqual(got, want) {
		t.Fatalf("ConceptsForText() = %v, want %v", got, want)
	}
}

func TestConceptsForTextNoMatchReturnsNil(t *testing.T) {
	terminology := map[string][]string{"hygiene": {"scale", "polish"}}
	got := ConceptsForText("General chat about the weekend", terminology)
	if got != nil {
		t.Fatalf("ConceptsForText() = %v, want nil", got)
	}
}

func TestConceptsForTextMultipleConceptsSortedAlphabetically(t *testing.T) {
	terminology := map[string][]string{
		"hygiene": {"scale"},
		"crown":   {"crown"},
	}
	got := ConceptsForText("Scale then crown prep same visit", terminology)
	want := []string{"crown", "hygiene"}
	if !reflect.DeepEqual(got, want) {
		t.Fatalf("ConceptsForText() = %v, want %v (alphabetical)", got, want)
	}
}

func TestConceptsForTextIgnoresSingleCharacterTokens(t *testing.T) {
	// Django's text_matches only considers tokens with len >= 2.
	terminology := map[string][]string{"weird": {"a"}}
	got := ConceptsForText("a b c", terminology)
	if got != nil {
		t.Fatalf("ConceptsForText() = %v, want nil (single-char tokens excluded)", got)
	}
}

func TestConceptsForTextEmptyTextReturnsNil(t *testing.T) {
	terminology := map[string][]string{"hygiene": {"scale"}}
	got := ConceptsForText("", terminology)
	if got != nil {
		t.Fatalf("ConceptsForText() = %v, want nil", got)
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/classification/... -run TestConceptsForText -v`
Expected: FAIL — `undefined: ConceptsForText`.

- [ ] **Step 3: Implement `ConceptsForText`**

```go
package classification

import (
	"sort"
	"strings"
)

// ConceptsForText returns every concept whose word list matches the given
// raw text, sorted alphabetically for deterministic output. Mirrors Django's
// concepts_for_text()/text_matches(): single-word concepts match as whole
// tokens (len >= 2), multi-word phrases match as substrings of the cleaned
// text. Returns nil if nothing matches.
func ConceptsForText(text string, terminology map[string][]string) []string {
	cleaned := CleanText(text)
	if cleaned == "" {
		return nil
	}
	tokens := make(map[string]struct{})
	for _, tok := range strings.Fields(cleaned) {
		if len(tok) >= 2 {
			tokens[tok] = struct{}{}
		}
	}

	var matched []string
	for concept, words := range terminology {
		if wordsMatch(cleaned, tokens, words) {
			matched = append(matched, concept)
		}
	}
	sort.Strings(matched)
	return matched
}

func wordsMatch(cleanedText string, tokens map[string]struct{}, words []string) bool {
	for _, w := range words {
		if strings.Contains(w, " ") {
			if strings.Contains(cleanedText, w) {
				return true
			}
		} else if _, ok := tokens[w]; ok {
			return true
		}
	}
	return false
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/classification/... -v`
Expected: PASS (all tests in the package so far).

- [ ] **Step 5: Commit**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
git add internal/dentally/daylist/classification/
git commit -m "feat: add ConceptsForText matcher to classification package"
```

---

## Task 4: Per-practice override resolution

**Files:**
- Create: `EmailServiceGo/internal/dentally/daylist/classification/terminology.go`
- Test: `EmailServiceGo/internal/dentally/daylist/classification/terminology_test.go`

- [ ] **Step 1: Write the failing tests**

```go
package classification

import (
	"context"
	"reflect"
	"testing"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

// ── ResolveTerminology (pure logic, no DB) ──────────────────────────────────

func TestResolveTerminologyNilOverridesReturnsDefaults(t *testing.T) {
	defaults := map[string][]string{"hygiene": {"scale", "polish"}}
	got := ResolveTerminology(nil, defaults)
	if !reflect.DeepEqual(got, defaults) {
		t.Fatalf("ResolveTerminology(nil, ...) = %v, want %v", got, defaults)
	}
}

func TestResolveTerminologyDisabledOverridesReturnsDefaults(t *testing.T) {
	defaults := map[string][]string{"hygiene": {"scale", "polish"}}
	overrides := &PracticeOverrides{
		Enabled:  false,
		Mappings: map[string][]string{"hygiene": {"myword"}},
	}
	got := ResolveTerminology(overrides, defaults)
	if !reflect.DeepEqual(got, defaults) {
		t.Fatalf("disabled overrides should fall back fully to defaults, got %v", got)
	}
}

func TestResolveTerminologyOverridesOnlyConfiguredConcepts(t *testing.T) {
	defaults := map[string][]string{
		"hygiene": {"scale", "polish"},
		"exam":    {"exam", "checkup"},
	}
	overrides := &PracticeOverrides{
		Enabled:  true,
		Mappings: map[string][]string{"hygiene": {"customword"}},
	}
	got := ResolveTerminology(overrides, defaults)
	if !reflect.DeepEqual(got["hygiene"], []string{"customword"}) {
		t.Fatalf("hygiene should be overridden, got %v", got["hygiene"])
	}
	if !reflect.DeepEqual(got["exam"], []string{"exam", "checkup"}) {
		t.Fatalf("exam should remain default (untouched concept), got %v", got["exam"])
	}
}

func TestResolveTerminologyUnknownConceptIsIgnored(t *testing.T) {
	defaults := map[string][]string{"hygiene": {"scale"}}
	overrides := &PracticeOverrides{
		Enabled:  true,
		Mappings: map[string][]string{"not_a_real_concept": {"whatever"}},
	}
	got := ResolveTerminology(overrides, defaults)
	if !reflect.DeepEqual(got, defaults) {
		t.Fatalf("unknown concept override should be ignored, got %v", got)
	}
	if _, exists := got["not_a_real_concept"]; exists {
		t.Fatalf("unknown concept should not appear in the result")
	}
}

func TestResolveTerminologyEmptyWordListForConceptFallsBackToDefault(t *testing.T) {
	defaults := map[string][]string{"hygiene": {"scale", "polish"}}
	overrides := &PracticeOverrides{
		Enabled:  true,
		Mappings: map[string][]string{"hygiene": {}}, // saved but empty
	}
	got := ResolveTerminology(overrides, defaults)
	if !reflect.DeepEqual(got["hygiene"], []string{"scale", "polish"}) {
		t.Fatalf("empty override word list should fall back to default, got %v", got["hygiene"])
	}
}

func TestResolveTerminologyDoesNotMutateDefaultsArgument(t *testing.T) {
	defaults := map[string][]string{"hygiene": {"scale", "polish"}}
	overrides := &PracticeOverrides{
		Enabled:  true,
		Mappings: map[string][]string{"hygiene": {"customword"}},
	}
	_ = ResolveTerminology(overrides, defaults)
	if !reflect.DeepEqual(defaults["hygiene"], []string{"scale", "polish"}) {
		t.Fatalf("ResolveTerminology must not mutate the defaults map it was given, got %v", defaults["hygiene"])
	}
}

// ── FetchPracticeOverrides (real sqlite in-memory DB) ───────────────────────

type opportunityTerminologyRow struct {
	PracticeID int    `gorm:"column:practice_id"`
	Mappings   string `gorm:"column:mappings"`
	Enabled    bool   `gorm:"column:enabled"`
}

func (opportunityTerminologyRow) TableName() string { return "daylist_opportunity_terminology" }

func openTestDB(t *testing.T) *gorm.DB {
	t.Helper()
	db, err := gorm.Open(sqlite.Open("file::memory:?cache=shared"), &gorm.Config{})
	if err != nil {
		t.Fatalf("open sqlite: %v", err)
	}
	if err := db.AutoMigrate(&opportunityTerminologyRow{}); err != nil {
		t.Fatalf("migrate: %v", err)
	}
	return db
}

func TestFetchPracticeOverridesNoRowReturnsDisabledEmpty(t *testing.T) {
	db := openTestDB(t)
	got, err := FetchPracticeOverrides(context.Background(), db, 999)
	if err != nil {
		t.Fatalf("FetchPracticeOverrides: %v", err)
	}
	if got.Enabled {
		t.Fatalf("expected Enabled=false when no row exists, got true")
	}
	if len(got.Mappings) != 0 {
		t.Fatalf("expected empty Mappings when no row exists, got %v", got.Mappings)
	}
}

func TestFetchPracticeOverridesParsesStoredMappings(t *testing.T) {
	db := openTestDB(t)
	row := opportunityTerminologyRow{
		PracticeID: 4,
		Enabled:    true,
		Mappings:   `{"hygiene": {"canonicals": ["hygiene"], "words": ["scale", "polish"]}}`,
	}
	if err := db.Create(&row).Error; err != nil {
		t.Fatalf("insert fixture row: %v", err)
	}

	got, err := FetchPracticeOverrides(context.Background(), db, 4)
	if err != nil {
		t.Fatalf("FetchPracticeOverrides: %v", err)
	}
	if !got.Enabled {
		t.Fatalf("expected Enabled=true")
	}
	want := []string{"scale", "polish"}
	if !reflect.DeepEqual(got.Mappings["hygiene"], want) {
		t.Fatalf("Mappings[hygiene] = %v, want %v", got.Mappings["hygiene"], want)
	}
}
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/classification/... -run 'TestResolveTerminology|TestFetchPracticeOverrides' -v`
Expected: FAIL — `undefined: ResolveTerminology` / `undefined: PracticeOverrides` / `undefined: FetchPracticeOverrides`.

- [ ] **Step 3: Implement `PracticeOverrides`, `FetchPracticeOverrides`, and `ResolveTerminology`**

```go
package classification

import (
	"context"
	"encoding/json"
	"fmt"

	"gorm.io/gorm"
)

// PracticeOverrides is a practice's saved terminology customization, read
// from the shared `daylist_opportunity_terminology` table (Django's
// OpportunityTerminologyMapping model). A practice with no saved row, or a
// disabled row, resolves to Enabled=false / Mappings=nil, which
// ResolveTerminology treats identically: fall back entirely to defaults.
type PracticeOverrides struct {
	Enabled  bool
	Mappings map[string][]string // concept -> configured (uncleaned) words
}

type opportunityTerminologyMappingSpec struct {
	Words []string `json:"words"`
}

// FetchPracticeOverrides reads the practice's saved terminology row from the
// shared database. Safe to call with a practice that has no row (Django
// hasn't created one until the admin saves a customization) — returns
// Enabled=false in that case, same as an explicitly disabled row, so
// ResolveTerminology's fallback-to-defaults behavior is identical either way.
func FetchPracticeOverrides(ctx context.Context, db *gorm.DB, practiceID int) (*PracticeOverrides, error) {
	type row struct {
		Mappings []byte `gorm:"column:mappings"`
		Enabled  bool   `gorm:"column:enabled"`
	}
	var r row
	err := db.WithContext(ctx).Raw(
		`SELECT mappings, enabled FROM daylist_opportunity_terminology WHERE practice_id = ? LIMIT 1`,
		practiceID,
	).Scan(&r).Error
	if err != nil {
		return nil, err
	}
	if len(r.Mappings) == 0 {
		return &PracticeOverrides{Enabled: r.Enabled}, nil
	}

	var parsed map[string]opportunityTerminologyMappingSpec
	if err := json.Unmarshal(r.Mappings, &parsed); err != nil {
		return nil, fmt.Errorf("parsing daylist_opportunity_terminology.mappings for practice %d: %w", practiceID, err)
	}
	mappings := make(map[string][]string, len(parsed))
	for concept, spec := range parsed {
		mappings[concept] = spec.Words
	}
	return &PracticeOverrides{Enabled: r.Enabled, Mappings: mappings}, nil
}

// ResolveTerminology returns the effective concept -> words mapping for a
// practice: defaults, with any enabled, non-empty concept override applied on
// top. Mirrors Django's opportunity_vocab.resolve_terminology() exactly:
//   - nil/disabled overrides -> defaults, unchanged
//   - an override for an unknown concept is ignored
//   - an override with an empty (post-cleaning) word list falls back to the default
//   - never mutates the `defaults` argument
func ResolveTerminology(overrides *PracticeOverrides, defaults map[string][]string) map[string][]string {
	effective := make(map[string][]string, len(defaults))
	for concept, words := range defaults {
		cp := make([]string, len(words))
		copy(cp, words)
		effective[concept] = cp
	}
	if overrides == nil || !overrides.Enabled {
		return effective
	}
	for concept, rawWords := range overrides.Mappings {
		if _, known := effective[concept]; !known {
			continue
		}
		cleaned := make([]string, 0, len(rawWords))
		for _, w := range rawWords {
			if cw := CleanText(w); cw != "" {
				cleaned = append(cleaned, cw)
			}
		}
		if len(cleaned) > 0 {
			effective[concept] = cleaned
		}
	}
	return effective
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/classification/... -v`
Expected: PASS (all tests in the package so far).

- [ ] **Step 5: Commit**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
git add internal/dentally/daylist/classification/
git commit -m "feat: add per-practice override resolution to classification package"
```

---

## Task 5: Django golden fixture generation

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/management/commands/dump_opportunity_parity_fixture.py`

This command calls Django's REAL `opportunity_vocab.resolve_terminology()` + `concepts_for_text()` — the reference implementation being ported — against synthetic (non-PHI) scenarios and text, and writes the results to a JSON fixture. No live DB practice data is used, so the fixture is fully reproducible in any environment.

- [ ] **Step 1: Write the command**

```python
"""
Dump a golden fixture of Django's opportunity_vocab classifier's real output,
for verifying the ported Go implementation in EmailServiceGo's
internal/dentally/daylist/classification package matches it exactly.

Uses synthetic (non-PHI) override scenarios and text — this fixture proves
algorithmic parity, not real-patient accuracy (that's a separate benchmark).

    python manage.py dump_opportunity_parity_fixture > fixture.json
"""

import json

from django.core.management.base import BaseCommand

from dentallyIntegration import opportunity_vocab as ov


class _FakeMappingRow:
    """Minimal stand-in for an OpportunityTerminologyMapping instance —
    resolve_terminology() only reads .enabled and .mappings."""

    def __init__(self, enabled, mappings):
        self.enabled = enabled
        self.mappings = mappings


SCENARIOS = [
    {"name": "no_row", "row": None},
    {"name": "enabled_no_overrides", "row": _FakeMappingRow(True, {})},
    {
        "name": "enabled_hygiene_custom_word",
        "row": _FakeMappingRow(True, {"hygiene": {"words": ["myword123"]}}),
    },
    {
        "name": "disabled_with_overrides",
        "row": _FakeMappingRow(False, {"hygiene": {"words": ["myword123"]}}),
    },
    {
        "name": "unknown_concept_ignored",
        "row": _FakeMappingRow(True, {"not_a_real_concept": {"words": ["whatever"]}}),
    },
]

TEXTS = [
    "Scale and polish performed, gingivitis noted",
    "Routine examination completed, no issues",
    "PA <b>radiograph</b> taken of UL6",
    "Crown prep UL6, temporary crown fitted",
    "RCT completed on UL6, obturation done",
    "Tooth UL8 extracted, missing since childhood",
    "Wisdom tooth ul8 review, asymptomatic",
    "Intraoral scan taken with iTero",
    "Patient advised to monitor gum recession, declined treatment",
    "General chat about half term holidays",
    "myword123 mentioned during hygiene visit",
    "",
]


class Command(BaseCommand):
    help = "Dump a golden fixture of opportunity_vocab's real classifier output for Go parity testing."

    def handle(self, *args, **opts):
        entries = []
        for scenario in SCENARIOS:
            terminology = ov.resolve_terminology(scenario["row"])
            row = scenario["row"]
            for text in TEXTS:
                concepts = sorted(ov.concepts_for_text(text, terminology))
                entries.append(
                    {
                        "scenario": scenario["name"],
                        "enabled": bool(row.enabled) if row else False,
                        "mappings": row.mappings if row else {},
                        "text": text,
                        "expected_concepts": concepts,
                    }
                )
        self.stdout.write(json.dumps(entries, indent=2))
```

- [ ] **Step 2: Run it and verify output**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
source ../venv/bin/activate
python manage.py dump_opportunity_parity_fixture > /tmp/opportunity_parity_fixture.json
python -c "import json; data = json.load(open('/tmp/opportunity_parity_fixture.json')); print(f'{len(data)} entries')"
```
Expected: `60 entries` (5 scenarios × 12 texts), and no errors/tracebacks from the management command.

Spot-check one entry makes sense:
```bash
python -c "
import json
data = json.load(open('/tmp/opportunity_parity_fixture.json'))
for e in data:
    if e['scenario'] == 'enabled_hygiene_custom_word' and e['text'] == 'myword123 mentioned during hygiene visit':
        print(e)
"
```
Expected: an entry with `"expected_concepts": ["hygiene"]` (matches via the custom override word `myword123`, since the default hygiene words wouldn't match this exact synthetic string).

- [ ] **Step 3: Copy the fixture into the Go module's testdata**

```bash
mkdir -p /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo/internal/dentally/daylist/classification/testdata
cp /tmp/opportunity_parity_fixture.json \
   /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo/internal/dentally/daylist/classification/testdata/opportunity_parity_fixture.json
```

- [ ] **Step 4: Commit both sides**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend
git add TreatmentPath/dentallyIntegration/management/commands/dump_opportunity_parity_fixture.py
git commit -m "feat: add management command to dump opportunity classifier golden fixture"

cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
git add internal/dentally/daylist/classification/testdata/
git commit -m "test: add Django golden fixture for opportunity classifier parity testing"
```

---

## Task 6: Go parity test against the golden fixture

**Files:**
- Create: `EmailServiceGo/internal/dentally/daylist/classification/parity_test.go`

- [ ] **Step 1: Write the failing test**

```go
package classification

import (
	"encoding/json"
	"os"
	"reflect"
	"testing"
)

type parityFixtureEntry struct {
	Scenario          string                                       `json:"scenario"`
	Enabled           bool                                         `json:"enabled"`
	Mappings          map[string]struct{ Words []string `json:"words"` } `json:"mappings"`
	Text              string                                       `json:"text"`
	ExpectedConcepts  []string                                     `json:"expected_concepts"`
}

// TestGoMatchesDjangoOpportunityClassifier proves the ported Go engine
// produces byte-for-byte identical concept sets to Django's real
// opportunity_vocab.resolve_terminology()/concepts_for_text(), across every
// scenario in the golden fixture (see Task 5). This is the core parity proof
// for the whole package.
func TestGoMatchesDjangoOpportunityClassifier(t *testing.T) {
	raw, err := os.ReadFile("testdata/opportunity_parity_fixture.json")
	if err != nil {
		t.Fatalf("reading fixture: %v", err)
	}
	var entries []parityFixtureEntry
	if err := json.Unmarshal(raw, &entries); err != nil {
		t.Fatalf("parsing fixture: %v", err)
	}
	if len(entries) == 0 {
		t.Fatal("fixture is empty — did Task 5 run correctly?")
	}

	defaults := DefaultTerminology()
	for _, e := range entries {
		e := e
		t.Run(e.scenarioAndTextLabel(), func(t *testing.T) {
			mappings := make(map[string][]string, len(e.Mappings))
			for concept, spec := range e.Mappings {
				mappings[concept] = spec.Words
			}
			overrides := &PracticeOverrides{Enabled: e.Enabled, Mappings: mappings}

			terminology := ResolveTerminology(overrides, defaults)
			got := ConceptsForText(e.Text, terminology)

			want := e.ExpectedConcepts
			if len(want) == 0 {
				want = nil // Django's [] and Go's nil both mean "no matches"
			}
			if !reflect.DeepEqual(got, want) {
				t.Fatalf("scenario=%q text=%q: got %v, want %v (Django)", e.Scenario, e.Text, got, want)
			}
		})
	}
}

func (e parityFixtureEntry) scenarioAndTextLabel() string {
	label := e.Scenario + "/" + e.Text
	if label == e.Scenario+"/" {
		return e.Scenario + "/empty_text"
	}
	return label
}
```

- [ ] **Step 2: Run the test to verify it fails (until Task 5's fixture is in place) or passes**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/classification/... -run TestGoMatchesDjangoOpportunityClassifier -v`
Expected: if Task 5 was completed first, this should PASS immediately (all 60 sub-tests green) — this task is verification, not new production code. If it fails, the failure output tells you exactly which scenario/text diverges between Go and Django; do not edit the test to make it pass — fix the mismatch in `terminology.go`/`match.go`/`vocab.go` until Go's output matches Django's.

- [ ] **Step 3: Run the full package test suite one more time**

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go test ./internal/dentally/daylist/classification/... -v`
Expected: PASS — every test written in Tasks 1–6.

Run: `cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo && go build ./...`
Expected: no output, exit code 0 (confirms the new package doesn't break the existing build — it isn't imported anywhere yet, so this should be a no-op check, but run it to be sure).

- [ ] **Step 4: Commit**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
git add internal/dentally/daylist/classification/
git commit -m "test: add Go/Django parity test for opportunity classifier"
```

---

## What's Next (not in this plan)

This plan ends with a complete, unit-tested, Django-parity-proven classification engine that **nothing calls yet**. Follow-up work (a separate plan, per the design doc's §4 and §6.4–6.8):

1. Wire hygiene-skip logic into `daylist/ai`'s visit-selection query (walk back through history until 2 non-hygiene visits are found).
2. Migrate `daylist/opportunity/classifiers.go`'s hardcoded term lists to call this engine.
3. Migrate `daylist/ai/db.go`'s `classifyCategory()`'s overlapping buckets (hygiene/exam/treatment-ish) to call this engine.
4. Two-round objective accuracy benchmark against 100 + 100 real appointments (design doc §6.4–6.5).
5. Before/after regression diffs on both real consumers (design doc §6.6–6.7) before cutover.

---

## Plan Self-Review

**Spec coverage:** This plan implements spec §3 (Architecture: CSV embed, `CleanText`, vocab loader, `DefaultTerminology`, `FetchPracticeOverrides`, `ResolveTerminology`, `ConceptsForText`, concept name mapping) and spec §6.1–6.3 (golden fixture, Go parity test, unit tests). Spec §4 (wiring), §6.4–6.8 (accuracy benchmark, regression, ship gate), and §7 (already resolved in the spec doc) are explicitly deferred to the follow-up plan noted above — this plan is scoped to producing a complete, working, independently-testable engine, not yet wiring any consumer.

**Placeholder scan:** No TBD/TODO markers; every step has complete, runnable code and exact commands with expected output.

**Type consistency:** `PracticeOverrides{Enabled bool; Mappings map[string][]string}` is defined once in Task 4 and used identically in Task 6's parity test. `DefaultTerminology() map[string][]string`, `ResolveTerminology(overrides *PracticeOverrides, defaults map[string][]string) map[string][]string`, and `ConceptsForText(text string, terminology map[string][]string) []string` are defined in Tasks 2–4 and consumed with matching signatures in Task 6.

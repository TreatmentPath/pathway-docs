# Daylist "Counts As" Classification Engine — Design

**Date:** 2026-07-01
**Status:** Approved design — ready for implementation planning
**Scope:** Port Django's "counts as" terminology-matching classifier (`opportunity_vocab.py`) into a reusable Go package, wire it into the daylist opportunity engine and the AI-summary pipeline, and use it to make the AI summary skip hygiene visits when picking the "last two visits" it summarizes.

---

## 1. Background & Problem

Today there are **three independent, uncoordinated word-matching systems** for deciding "does this appointment/treatment-plan-item count as concept X" in this codebase:

1. **Django's `opportunity_vocab.py`** (`dentallyIntegration/opportunity_vocab.py`) — the real, admin-configurable classifier. Loads canonical→word-list vocab from `data/opportunity_group.csv`, lets each practice override which canonicals count toward each concept via `OpportunityTerminologyMapping` (table `daylist_opportunity_terminology`), and matches cleaned text against the resolved word list.
2. **Go's `daylist/opportunity/classifiers.go`** — a separate, hardcoded set of term lists (`hygieneTerms`, `missingToothTerms`, etc.) used by `opportunity/engine.go`'s `Analyze()`.
3. **Go's `daylist/ai/db.go`'s `classifyCategory()`** — a third, separate hardcoded 5-bucket keyword switch, used to label the AI summary's input visits and (currently) to decide which visits count as "hygiene."

None of these three currently agree with each other, and none of the two Go ones respect a practice's actual admin-configured terminology — only Django's does.

**The trigger for this work:** the AI summary currently picks a patient's "last two completed visits with notes" indiscriminately, regardless of type. If one or both are hygiene visits, the summary is less useful (hygiene visits rarely have clinically notable narrative). The ask: skip hygiene visits and walk back through history until 2 non-hygiene visits are found, using a real "is this hygiene" classifier — not a fresh hardcoded guess.

While scoping that, we found the deeper issue above and decided to fix it properly: build **one** reusable Go classification engine, sourced from the same real per-practice config Django already uses, and migrate both existing Go classifiers onto it.

---

## 2. Goals & Non-Goals

### Goals
1. A new, reusable Go package that ports Django's `opportunity_vocab.py` matching logic faithfully (`CleanText`, `ResolveTerminology`, `ConceptsForText`).
2. Read real per-practice terminology overrides from the shared DB (`daylist_opportunity_terminology`), falling back to CSV-derived defaults — mirroring Django's `resolve_terminology()` exactly.
3. Use the engine to add hygiene-skip logic to the AI summary's visit selection (walk back through history until 2 non-hygiene visits are found).
4. Migrate `daylist/opportunity/classifiers.go`'s hardcoded term lists to the new engine, for every concept where Django and Go overlap.
5. Migrate `daylist/ai/db.go`'s `classifyCategory()`'s overlapping buckets (`hygiene`, `exam`, and the treatment-ish concepts) to the new engine.
6. Prove parity + correctness before cutover via the test plan in §6, so neither existing consumer (opportunity badges, AI summary) regresses.

### Non-Goals (explicitly deferred)
- Porting Django's `OpportunityConfig` thresholds, suppression rules, or `confidence_by_source` logic — **only** the terminology/word-matching ("counts as") layer is in scope.
- Replacing `classifyCategory()`'s `emergency`, `consultation`, and `other` buckets — these have no equivalent concept in Django's opportunity vocabulary (hygiene, exam, radiograph, scan, crown, root_canal, missing, replacement, restoration, wisdom, watch), so they remain Go-only heuristics, unchanged.
- Any change to the Django side (`opportunity_vocab.py` stays exactly as-is; it's the reference implementation being ported, not touched).
- Any change to `internal/dentally/recall`'s separate `treatment_group.csv`/drop-word pipeline (different file, different feature, out of scope).

---

## 3. Architecture

New Go package: `internal/dentally/daylist/classification`.

| Django (`opportunity_vocab.py`) | Go (`daylist/classification`) | Notes |
|---|---|---|
| `data/opportunity_group.csv` | `data/opportunity_group.csv` (embedded copy) | Same file, copied verbatim — guarantees identical canonical→word expansion. |
| `clean_text(s)` | `CleanText(s string) string` | HTML-strip → lowercase → strip non-alphanumeric → collapse whitespace. |
| `_vocab()` / `all_canonicals()` | `loadVocab()` (`sync.Once`-cached) | canonical → member words, parsed from the embedded CSV. |
| `CONCEPT_DEFAULT_CANONICALS` / `default_terminology()` | `DefaultTerminology() map[string][]string` | Ported 1:1, `sync.Once`-cached. |
| `OpportunityTerminologyMapping` DB row | `FetchPracticeOverrides(ctx, db, practiceID)` | Raw-SQL read of `daylist_opportunity_terminology` — first Go read of a Django **config** table (Go already reads Django data tables elsewhere). |
| `resolve_terminology(mapping_row)` | `ResolveTerminology(overrides, enabled, defaults) map[string][]string` | Same merge semantics: untouched concepts fall back to defaults; a disabled row falls back entirely to defaults. |
| `text_matches` / `concepts_for_text` | `ConceptsForText(text string, terminology map[string][]string) []string` | Single-word concepts match as whole tokens; multi-word phrases match as substrings. |

Concept name mapping (Go's existing names → Django's concept keys, so nothing is silently renamed):

| Go (today) | Django concept |
|---|---|
| `hygieneTerms` | `hygiene` |
| `missingToothTerms` | `missing` |
| `monitorDeclineTerms` | `watch` |
| (crown classifier) | `crown` |
| (implant classifier) | `replacement` |
| (radiograph classifier) | `radiograph` |

**Fetch cadence:** `ResolveTerminology` is called **once per practice per sync run**, not per patient — the resolved `map[string][]string` is passed down and reused across every patient/visit in that run.

---

## 4. Wiring — call sites

1. **New: AI-summary hygiene-skip.** `daylist/ai`'s visit-selection query currently grabs the last 2 completed treatment-plan-items with notes, unconditionally. It changes to: fetch a larger candidate window (e.g. last 10), resolve the practice's terminology once, then walk the candidates newest-first, skipping any where `"hygiene" ∈ ConceptsForText(nomenclature, terminology)`, until 2 non-hygiene visits are collected or candidates run out.
2. **Migrate: `daylist/opportunity/classifiers.go`.** Its hardcoded term-list classifiers (`classifyHygiene`, `classifyCrown`, `classifyImplant`, `classifyRadiographs`, etc.) call the new engine instead of their own term lists. This means the opportunity badges (Crown/Implant/Hygiene/Xrays-due) start respecting each practice's actual configured terminology instead of a Go-only hardcoded list.
3. **Migrate (partial): `daylist/ai/db.go`'s `classifyCategory()`.** Its `hygiene`/`exam`/treatment-ish cases route through the new engine; `emergency`/`consultation`/`other` are untouched (no Django equivalent — see Non-Goals).

*(Open question resolved during discovery, see §7: confirming whether Go's `opportunity/engine.go` or Django computes the badges the user actually sees today — this determines the blast radius/visibility of change #2 above.)*

---

## 5. Data flow

```
Practice sync run starts
  → FetchPracticeOverrides(ctx, db, practiceID)   [1 query]
  → ResolveTerminology(overrides, defaults)        [1 merge, cached for the run]
      │
      ├─→ opportunity/engine.go Analyze() per patient
      │      → ConceptsForText(nomenclature, terminology) per TPI/appointment
      │      → OpportunityFlag{Hygiene, Crown, Implant, ...}
      │
      └─→ daylist/ai per patient
             → candidate visits (last N completed TPIs w/ notes, newest first)
             → for each candidate: ConceptsForText(nomenclature, terminology)
             → skip if "hygiene" present; collect until 2 non-hygiene found
             → BuildUserMessage(selected 2 visits) → AI summary
```

---

## 6. Testing & Parity Plan

This touches production behavior two existing features already depend on (opportunity badges, AI summary), so proving faithfulness comes before any cutover.

- [ ] **6.1 Golden fixture from Django.** Script/management command that, for a representative sample of practices (≥1 using pure defaults, ≥1 with customized overrides, ≥1 with the mapping disabled) and a batch of real nomenclature/reason strings, calls Django's actual `concepts_for_text()` and records input→output pairs to a fixture file.
- [ ] **6.2 Go parity test.** A Go test loads that fixture and asserts the new engine returns the identical concept set for every input. Must pass before any call site is rewired.
- [ ] **6.3 Unit tests for the engine itself.**
  - [ ] `CleanText`: HTML stripping, punctuation, whitespace collapse.
  - [ ] `ConceptsForText`: single-word token matching vs. multi-word phrase substring matching.
  - [ ] `ResolveTerminology`: untouched concepts fall back to defaults; configured concepts override cleanly; a disabled mapping row falls back entirely to defaults.
- [ ] **6.4 Objective accuracy benchmark (round 1).** Pull 100 real completed appointments/treatment-plan-items. Independently determine the objectively-correct classification for each (ground truth — not just "what Django currently outputs," since Django's classifier could itself be subtly wrong). Run the new Go engine against them and measure accuracy.
- [ ] **6.5 Objective accuracy benchmark (round 2).** Repeat 6.4 with a fresh, different 100 appointments, to confirm accuracy holds and round 1 wasn't a fluke of that sample.
- [ ] **6.6 Regression: `opportunity/engine.go`.** Before/after diff of `Analyze()`'s flag output (Crown/Implant/Hygiene/Cosmetic/etc.) against a sample of real patients/practices. Note (per §7): this output feeds the AI-generated narrative's opportunity wording (`dentallyai.BuildOpportunities`), **not** the live badge pills — Django recomputes those fresh from its own config every page load, so they're unaffected by this migration. Existing test suite for this package must still pass unchanged.
- [ ] **6.7 Regression: `daylist/ai/db.go`.** Before/after diff of `classifyCategory()`'s output and the new hygiene-skip visit-selection behavior against a sample of real patients; existing AI-summary tests must still pass unchanged.
- [ ] **6.8 Ship gate.** All of 6.1–6.7 green before the new engine replaces either existing classifier in the live path.

---

## 7. Open Question — resolved during discovery

**Q: For the live daylist screen today, are opportunity badges (Crown/Implant/Hygiene/Xrays-due) computed by Django or by Go's `opportunity/engine.go`?**

**Resolved: Django, at request time — Go's computation never reaches the badge.**

- Go's `opportunity.Engine.Analyze()` (via `AnalyzeEvidence()`, `opportunity/engine.go:24`, called from `service.go:169-186`) runs from two places: the daily background scheduler (`scheduler.go:741`) and an on-demand regenerate endpoint (`ai/handler.go:132`). Its output is written to the `DentallyPatientAISummary` table (`hygiene_due`, `crown_opportunity`, etc.) — it does **not** go directly to the frontend.
- Django's `DentallyDayListViewSet` (`dentally_views.py:1906-1922`) reads those Go-written columns via a subquery **but then explicitly overwrites them**, unconditionally, with a fresh call to `opportunity_classifier.compute_opportunities()` (which uses `opportunity_vocab.py` + `OpportunityConfig`) before serializing the response. The code comment at `dentally_views.py:1873-1879` states this override exists specifically so **"editing the config takes effect on the very next page load, with no re-sync."**
- **So:** the badge pill a user sees is 100% Django-computed, always fresh. Go's `opportunity` package output is a *different, adjacent* consumer — it feeds `dentallyai.BuildOpportunities(opps)`, which becomes part of the free-text AI-generated summary narrative (`ai_report`/`ai_recommendations`), not the badge itself.

**Effect on this design:** migrating `daylist/opportunity/classifiers.go` (§4 point 2) has a **smaller blast radius than assumed** — it only changes the opportunity-related wording fed into the AI narrative summary, not the live badge pills (those are safe regardless, since Django recomputes them fresh every page load from its own config). This lowers the risk of that migration; the regression test in §6.6 should target the AI narrative's opportunity content, not the badge pills.

---

## 8. Checklist Summary (for quick reference while implementing)

- [ ] Copy `dentallyIntegration/data/opportunity_group.csv` into `internal/dentally/daylist/classification/data/opportunity_group.csv`
- [ ] Implement `CleanText`
- [ ] Implement vocab loader (`loadVocab`, `sync.Once`)
- [ ] Implement `DefaultTerminology` (port `CONCEPT_DEFAULT_CANONICALS`)
- [ ] Implement `FetchPracticeOverrides` (raw SQL against `daylist_opportunity_terminology`)
- [ ] Implement `ResolveTerminology`
- [ ] Implement `ConceptsForText`
- [ ] Unit tests (§6.3)
- [ ] Django golden fixture + Go parity test (§6.1–6.2)
- [ ] Objective accuracy benchmark rounds 1 & 2 (§6.4–6.5)
- [ ] Resolve §7 open question
- [ ] Wire hygiene-skip into `daylist/ai` visit selection
- [ ] Migrate `daylist/opportunity/classifiers.go` overlapping concepts
- [ ] Migrate `daylist/ai/db.go`'s `classifyCategory()` overlapping buckets
- [ ] Regression tests §6.6–6.7
- [ ] Ship gate §6.8

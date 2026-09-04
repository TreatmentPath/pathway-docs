# Cross-language parity — logic that exists in BOTH Django and Go

Some identity logic is implemented twice: once in Django and once in Go, because
the Dentally sync and the call agent write to the shared database directly and
never go through Django. When those copies disagree, the same human resolves to
two different Persons depending on which service saw them first — which is the
duplicate-contact problem itself.

**Find every one of them:**

```bash
grep -rn "CROSS-LANGUAGE PARITY" --include=*.py --include=*.go .
```

## The pairs

| What | Django | Go | Pinned by |
|---|---|---|---|
| Canonical **phone** key | `TreatmentPlan/utils/phones.py` `canonical_phone_e164` | `pkg/phone` `CanonicalE164` | `phone_fixtures.json` (two identical copies) |
| Canonical **email** key | `TreatmentPlan/utils/contact_keys.py` `canonical_email` | `pkg/email` `Canonical` | `contact_key_fixtures.json` (two identical copies) |
| Canonical **name** key | `TreatmentPlan/utils/contact_keys.py` `canonical_name_part` | `pkg/personname` `CanonicalPart` | `contact_key_fixtures.json` |
| **Person resolution** algorithm | `TreatmentPlan/models.py` `Person.resolve` | `internal/dentally/migration/service.go` `linkPatientToPersonAndChannels` | **nothing — comment only** |
| **DOB conflict** rule | `TreatmentPlan/models.py` `Person._dob_conflict` | `internal/dentally/migration/service.go` `dobConflict` | **nothing — comment only** |

## The two tiers, and why the difference matters

**Fixture-pinned (the three keys).** Both languages assert against one shared
set of expected values. If either side changes, its own test suite goes red —
you cannot ship the drift. When adding a case, add it to BOTH copies of the
fixture file.

**Comment-only (resolution + dob).** Nothing mechanical stops these drifting.
This happened on 2026-09-04: Django's name key was fixed to collapse internal
whitespace and the Go copy was left on the old rule, so for a few hours the two
services disagreed about whether "Mary  Jane" and "Mary Jane" were one person.
It was caught by a manual sweep, not by a test.

If you extend either of these, consider whether the logic can be moved behind a
fixture-pinned key instead — that is the only guard that actually holds.

## Not a parity pair

`internal/dentally/scheduler/name_similarity.go` (`nameSimilar`,
`sequenceMatcherRatio`) ports Python's `difflib.SequenceMatcher`, but Django has
no fuzzy name matcher of its own — this is Go-only, used by the duplicate
*detector*, not the resolver. Nothing to keep in step.

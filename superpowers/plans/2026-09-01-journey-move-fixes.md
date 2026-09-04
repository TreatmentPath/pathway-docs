# Journey Move Fixes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix the four defects captured in the ScreenKite recording of the Journeys "Move patient" flow: (1) raw DRF "Priority: This field may not be null" 400, (2) a failed side-PATCH reported as a failed move (causing retries that then hit "Cannot move to current stage"), (3) no success feedback and an unhelpful already-moved error, (4) full names with honorifics ("MRS NICOLA I BUTCHER") stored wholesale in `first_name` by workflow-driven intake creation.

**Architecture:** The Move dialog (`MovePatientDialog`) calls `useJourneyConvert.onConfirm`, which POSTs the move and then PATCHes the record's details (`buildEditPatch`). The bug chain: `buildEditPatch` always sends `priority` (null when the card has none); `IntakeSerializer` (ModelSerializer over a non-nullable CharField) rejects explicit `null`, the PATCH returns `{"error":"Validation failed","field_errors":{"priority":["This field may not be null."]}}`, `applyRecordPatch` surfaces that message and `onConfirm` throws — so the dialog shows an error even though the move POST succeeded. The user retries, the second POST hits `source_slug == destination_slug` and shows "Cannot move to current stage."

**Tech Stack:** Django 5.1 + DRF (TreatmentPathBackend), React 18 + Vitest (perfect-pixel-playground-project), Go 1.24 (EmailServiceGo workflows).

---

### Task 1: Stop sending `priority: null` on record patches

**Files:**
- Modify: `perfect-pixel-playground-project/src/utils/journeyEditPatch.ts:23-27`
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/serializers/intake.py` (`validate`, ~line 481)
- Test: `perfect-pixel-playground-project/src/utils/journeyEditPatch.test.ts` (create if missing)
- Test: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/` (extend existing intake serializer/view tests)

- [ ] **Step 1: Write failing frontend test** — `priority` must be absent (not null) when the card has no priority; present when set.

```ts
it('omits priority when the record has none instead of sending null', () => {
  const { mainPatch } = buildEditPatch('intake', {
    stageSlug: 'deposits',
    priority: undefined,
    clinician: null,
    assignee: null,
    estimatedValue: null,
    quickNote: null,
  } as JourneyBoardManageUpdates);
  expect('priority' in mainPatch.payload).toBe(false);
});

it('keeps a real priority value', () => {
  const { mainPatch } = buildEditPatch('intake', {
    stageSlug: 'deposits',
    priority: 'HIGH',
    clinician: null,
    assignee: null,
    estimatedValue: null,
    quickNote: null,
  } as JourneyBoardManageUpdates);
  expect(mainPatch.payload.priority).toBe('high');
});
```

- [ ] **Step 2: Run** `npx vitest run src/utils/journeyEditPatch.test.ts` — expect FAIL (currently `priority: null` present).
- [ ] **Step 3: Implement** in `buildEditPatch`:

```ts
const payload: Record<string, unknown> = {
  estimated_value: updates.estimatedValue ?? null,
  notes: updates.quickNote?.text ?? "",
};
// Priority is a non-nullable CharField on Intake: sending an explicit null is
// rejected ("Priority: This field may not be null."), so the key is only sent
// when there is a value to set.
if (updates.priority) payload.priority = updates.priority.toLowerCase();
```

- [ ] **Step 4: Backend tolerance** — in `IntakeSerializer.validate` (partial updates only), drop a null priority so older clients can't 400:

```python
def validate(self, data):
    # A partial update that sends priority=None (older client builds did)
    # must be treated as "no change", not a validation error: priority is a
    # non-nullable CharField with a model default.
    if self.partial and data.get("priority") is None:
        data.pop("priority", None)
    ...  # existing body continues
```

- [ ] **Step 5: Backend test** — extend the existing intake update test module with: PATCH `{"priority": null, "notes": "x"}` returns 200 and leaves priority unchanged.
- [ ] **Step 6: Run backend test** (`python manage.py test TreatmentPlan.tests.<module> -v` via venv), expect PASS.
- [ ] **Step 7: Commit** `fix: stop sending null priority on journey record patches (move-flow 400)`.

### Task 2: A failed details-PATCH must not read as a failed move

**Files:**
- Modify: `perfect-pixel-playground-project/src/hooks/useJourneyConvert.ts` (`applyRecordPatch` usage in all three loops + return type)
- Modify: `perfect-pixel-playground-project/src/components/journeys/MovePatientDialog.tsx` (`handleSubmit`/`handleRelationshipSubmit` + onConfirm type)
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/move_views.py:136,244` (clearer detail message)
- Test: `perfect-pixel-playground-project/src/hooks/useJourneyConvert.test.tsx` (extend)
- Test: `perfect-pixel-playground-project/src/components/journeys/MovePatientDialog.test.tsx` (extend)
- Test: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_move_views.py` (extend)

- [ ] **Step 1: Failing hook test** — move POST 200 + patch 400 resolves with warnings (does NOT throw); move POST 400 still throws. Existing tests pin the throw-on-any-error behaviour, so they must be updated with the new contract.
- [ ] **Step 2: Implement** — `onConfirm` returns `Promise<{ warnings: string[] }>`; in each loop, patch errors collect into `warnings` (only move errors into `errors`/throw). Hook returns warnings to the caller.
- [ ] **Step 3: Dialog** — on success, if `warnings.length`, show a non-blocking amber notice via toast (`use-toast`): "Moved, but the details didn't save: <warning>". Success otherwise: toast "Moved to <destination name>." then close (dialog already closes).
- [ ] **Step 4: Backend message** — in both views replace the detail with the stage name:

```python
"detail": f"This record is already in the {source_slug} stage.",
```

- [ ] **Step 5: Update/extend tests** (hook contract change, dialog warning toast, backend detail text).
- [ ] **Step 6: Run** `npx vitest run src/hooks/useJourneyConvert.test.tsx src/components/journeys/MovePatientDialog.test.tsx` and the backend move tests; fix fallout in other callers of `onConfirm` (tables pass `journeyConvert.onConfirm` straight through — type-compatible since return value is additive).
- [ ] **Step 7: Commit** `fix: journey move succeeds visibly when the follow-up details PATCH fails`.

### Task 3: Split honorific + full names at workflow-driven intake creation

**Files:**
- Modify: `EmailServiceGo/internal/workflows/actions/create_record.go` (`buildRecordData`, after the common-fields loop)
- Test: `EmailServiceGo/internal/workflows/actions/create_record_test.go` (create/extend)

- [ ] **Step 1: Failing Go test**:

```go
func TestBuildRecordDataSplitsFullNameWithHonorific(t *testing.T) {
    a := NewCreateRecordAction()
    data := a.buildRecordData(
        JSONMap{"entity_type": "intake"},
        JSONMap{"name": "MRS NICOLA I BUTCHER"},
    )
    if data["first_name"] != "Nicola" { t.Errorf("first_name = %v", data["first_name"]) }
    if data["last_name"] != "I Butcher" { t.Errorf("last_name = %v", data["last_name"]) }
}
```

- [ ] **Step 2: Run** `go test ./internal/workflows/actions/ -run TestBuildRecordData -v` — FAIL (first_name = "MRS NICOLA I BUTCHER").
- [ ] **Step 3: Implement** in `buildRecordData` before `return data`:

```go
normalizePersonName(data)
```

with:

```go
var honorifics = map[string]bool{
    "mr": true, "mrs": true, "miss": true, "ms": true, "dr": true,
    "sir": true, "madam": true, "prof": true, "professor": true, "rev": true,
}

// normalizePersonName splits a full name that arrived through the "name" alias
// into first/last. Website forms send "MRS NICOLA I BUTCHER" as one string;
// without this the whole string lands in first_name. The honorific is a form
// artifact (Intake has no title column) and is dropped. Only fires when
// last_name is absent, so explicit field mappings always win.
func normalizePersonName(data map[string]interface{}) {
    if _, hasLast := data["last_name"]; hasLast && data["last_name"] != "" {
        return
    }
    raw, ok := data["first_name"].(string)
    if !ok {
        return
    }
    tokens := strings.Fields(raw)
    if len(tokens) < 2 {
        return
    }
    if honorifics[strings.ToLower(strings.TrimSuffix(tokens[0], "."))] {
        tokens = tokens[1:]
    }
    if len(tokens) < 2 {
        data["first_name"] = strings.Join(tokens, " ")
        return
    }
    data["first_name"] = tokens[0]
    data["last_name"] = strings.Join(tokens[1:], " ")
}
```

- [ ] **Step 4: Run** the Go test again — PASS. Also run `go build ./...`.
- [ ] **Step 5: Commit** `fix: split honorific full names into first/last on workflow intake creation`.

### Task 4: Verification sweep

- [ ] Frontend: `npm run test:run` (full suite) in `perfect-pixel-playground-project`.
- [ ] Backend: targeted `python manage.py test TreatmentPlan.tests.test_move_views` + intake tests, `--keepdb`.
- [ ] Go: `go test ./internal/workflows/...`.
- [ ] Lint/format: black + isort on touched Python files; eslint/prettier per project on touched TS; gofmt on Go. Note: `.pre-commit-config.yaml` exists in TreatmentPathBackend/TreatmentPath.
- [ ] Delete scratch files; no secrets added.

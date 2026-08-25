# Recall Sync Livelock Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop the recall auto-sync from cancelling and restarting a group child's in-flight sync every 30 minutes, so the delta watermark can advance, spend data lands, and the Dentally quota stops being exhausted.

**Architecture:** Six independent defects in one causal chain, fixed at their own layers. The scheduler learns to check whether the *target* practice (not the group head) is already syncing, and to skip rather than pre-empt. The recall engine refuses to be pre-empted by a scheduled run and resumes a consistent interrupted delta instead of restarting from patient #1. The rate gate is re-keyed from practice to Dentally account, since a group shares one API key and one hourly quota. Finally the completion stamp that drives Django reconciliation is written to each synced child, not only the head.

**Tech Stack:** Go 1.x (`EmailServiceGo`, GORM + raw SQL, zap logging, robfig/cron), Django 5 (`TreatmentPathBackend`, Celery), PostgreSQL (shared between both services).

## Global Constraints

- **Read-only on Django-owned schema.** Go writes to Django-owned tables via raw SQL but must not add/alter columns. No migrations in this plan. If a change seems to need one, stop and escalate.
- **No behaviour change for standalone practices.** A practice with zero rows in `dentally_site_assignment` must behave byte-for-byte as it does today. Every task's tests must include a standalone-practice case proving this.
- **Manual/HTTP sync keeps pre-emption.** `POST` handlers (`handler.go:688`, `handler.go:643-645`) exist so a human can force a re-run; they must still cancel an in-flight run. Only the *scheduled* path changes.
- **Go integration tests use a real Postgres inside a rolled-back transaction.** Follow the existing pattern in `internal/dentally/scheduler/scheduler_integration_test.go`: `openTestDB(t)`, `tx := db.Begin()`, `t.Cleanup(func() { tx.Rollback() })`, then `&DailySyncScheduler{db: tx}`. Set `DATABASE_URL=postgres://mannie@localhost:5432/treatmentpath_db?sslmode=disable`. The local DB has `recall_sync_checkpoint`, `recall_auto_sync_config` and `dentally_site_assignment` (verified 2026-08-11).
- **Django tests run with `--keepdb` and never `--noinput`** (see the persistent-test-DB constraint in project memory).
- **Commit after every task.** Do not batch tasks into one commit.

---

## Background: the causal chain

Read this before starting — every task below is one link in it.

1. `getRecallAutoSyncPractices` gates the 30-minute tick on the **head** practice, including the "not already running" guard `ck.status NOT IN ('running','syncing')` joined on `ck.practice_id = p.id`.
2. But `runRecallAutoSync` calls `RunRecallDeltaSync(target)` for each **child** returned by `resolveSyncTargets`. The `running` checkpoint therefore belongs to the child, and the head's own row reads `completed`. **The guard never fires for a group head.**
3. `runRecallSync` (`engine.go:224`) begins by calling `prev()` — cancelling any in-flight run for that practice. So each tick kills the previous one.
4. A cancelled run never reaches the `completed` branch of `markRecallCheckpointStatus`, so the `last_synced_at = started_at` write never happens. **The watermark freezes.**
5. The next delta re-selects everything changed since that frozen watermark — the same ~1,282 patients — and `resume=false` resets `last_patient_id` to 0, so it re-grinds the same first ~150 patients.
6. That re-fetching (×4 API calls per patient) exhausts the shared Dentally quota (measured: 409 remaining of 3600/hr), and Dentally starts returning **403** — which the engine treats as a per-patient failure, burning ~21s in retries per patient and marking them owed, making the run slower and the cancellation more certain.

Production evidence, practice 27 (Church View Dental Clinic, child of head 24 SmileHQ), captured 2026-08-11:

```
practice_id | status  | mode  | total_recalls | processed | started_at          | completed_at | last_synced_at
27          | running | delta | 1282          | 91        | 2026-08-11 13:02:25 | NULL         | 2026-07-03 09:30:02
24          | completed| delta| 8             | 8         | 2026-08-11 13:00:00 | 13:01:14     | 2026-08-11 13:00:00
```

Measured throughput ~7 patients/min → 1,282 patients needs ~3 hours against a 30-minute tick. It can never finish.

---

## File Structure

| File | Responsibility | Change |
|---|---|---|
| `EmailServiceGo/internal/dentally/scheduler/recall_targets.go` | **New.** Target-scoped busy check + stale-run threshold, kept out of the 900-line `scheduler.go`. | Create |
| `EmailServiceGo/internal/dentally/scheduler/recall_targets_test.go` | **New.** Integration tests for the busy check. | Create |
| `EmailServiceGo/internal/dentally/scheduler/scheduler.go` | Tick orchestration: skip busy targets, call the scheduled (non-pre-empting) entry point, stamp per-child completion. | Modify (`runRecallAutoSync` ~L458-523, `SyncService` iface ~L37-38) |
| `EmailServiceGo/internal/dentally/recall/engine.go` | Add non-pre-empting scheduled entry point; resume a consistent interrupted delta. | Modify (~L199-251) |
| `EmailServiceGo/internal/dentally/recall/engine_scheduled_test.go` | **New.** Unit tests for pre-emption refusal. | Create |
| `EmailServiceGo/internal/dentally/recall/throttle.go` | Re-key the rate gate from practice to Dentally account; stop clearing it per run. | Modify |
| `EmailServiceGo/internal/dentally/recall/throttle_test.go` | **New.** Unit tests for account-scoped keying. | Create |
| `EmailServiceGo/internal/workflows/actions/dentally_client.go` | Expose the already-resolved key-holding practice id so the gate can key on it. | Modify (~L140-215) |
| `EmailServiceGo/internal/dentally/service.go` | Pass through the new scheduled entry point. | Modify (~L465-467) |
| `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests/test_reconciliation_group_children.py` | **New.** Regression test that group children become due for reconciliation. | Create |

Tasks 1, 2, 4 and 6 are independent and may be done in any order. Task 3 depends on Task 2. Task 5 depends on Task 4.

---

### Task 1: Target-scoped busy guard with a staleness escape hatch

The root cause. Today the scheduler asks "is the head busy?" when it should ask "is the practice I am about to sync busy?".

A naive `status = 'running'` check would deadlock forever if a run died without cleanup, so the check also treats a run whose `updated_at` has not moved for `recallStaleRunMinutes` as abandoned. `updateRecallCheckpoint` touches `updated_at` after every patient, so `updated_at` is a reliable liveness signal.

**Files:**
- Create: `EmailServiceGo/internal/dentally/scheduler/recall_targets.go`
- Create: `EmailServiceGo/internal/dentally/scheduler/recall_targets_test.go`
- Modify: `EmailServiceGo/internal/dentally/scheduler/scheduler.go:502-519` (the target loop inside `runRecallAutoSync`)

**Interfaces:**
- Consumes: `DailySyncScheduler.db` (`*gorm.DB`), `resolveSyncTargets(ctx, practiceID) []int` — both already exist.
- Produces:
  - `func (s *DailySyncScheduler) recallTargetBusy(ctx context.Context, targetID int) bool`
  - `func recallStaleRunMinutes() int` (env `RECALL_STALE_RUN_MINUTES`, default 45)

- [ ] **Step 1: Write the failing test**

Create `EmailServiceGo/internal/dentally/scheduler/recall_targets_test.go`:

```go
package scheduler

import (
	"context"
	"testing"
)

// A checkpoint that is 'running' and recently updated means a live sync — the
// scheduler must skip that target rather than pre-empt it (the Church View
// livelock: every 30-min tick killed the in-flight run, so the delta watermark
// never advanced).
func TestRecallTargetBusy(t *testing.T) {
	db := openTestDB(t)
	tx := db.Begin()
	if tx.Error != nil {
		t.Fatalf("begin transaction: %v", tx.Error)
	}
	t.Cleanup(func() { tx.Rollback() })

	const targetID = 999001
	s := &DailySyncScheduler{db: tx}
	ctx := context.Background()

	cases := []struct {
		name       string
		status     string
		updatedAgo string // interval expression
		want       bool
	}{
		{"live running run is busy", "running", "1 minute", true},
		{"live syncing run is busy", "syncing", "30 seconds", true},
		{"stale running run is not busy", "running", "3 hours", false},
		{"completed run is not busy", "completed", "1 minute", false},
		{"interrupted run is not busy", "interrupted", "1 minute", false},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			tx.Exec(`DELETE FROM recall_sync_checkpoint WHERE practice_id = ?`, targetID)
			if err := tx.Exec(`
				INSERT INTO recall_sync_checkpoint
					(practice_id, status, mode, generate_records, last_patient_id,
					 processed_count, synced_count, failed_count, total_recalls,
					 created_count, patients, owed_patient_ids, started_at, updated_at)
				VALUES (?, ?, 'delta', true, 0, 0, 0, 0, 0, 0,
				        '[]'::jsonb, '[]'::jsonb, NOW(), NOW() - `+tc.updatedAgo+`::interval)
			`, targetID, tc.status).Error; err != nil {
				t.Fatalf("seed checkpoint: %v", err)
			}
			if got := s.recallTargetBusy(ctx, targetID); got != tc.want {
				t.Errorf("recallTargetBusy(%s, updated %s ago) = %v, want %v",
					tc.status, tc.updatedAgo, got, tc.want)
			}
		})
	}
}

// No checkpoint row at all (first ever sync, and every standalone practice
// before its first run) must never read as busy.
func TestRecallTargetBusy_NoCheckpointRow(t *testing.T) {
	db := openTestDB(t)
	tx := db.Begin()
	if tx.Error != nil {
		t.Fatalf("begin transaction: %v", tx.Error)
	}
	t.Cleanup(func() { tx.Rollback() })

	s := &DailySyncScheduler{db: tx}
	if s.recallTargetBusy(context.Background(), 999002) {
		t.Error("recallTargetBusy = true for a practice with no checkpoint row, want false")
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd EmailServiceGo
DATABASE_URL='postgres://mannie@localhost:5432/treatmentpath_db?sslmode=disable' \
  go test ./internal/dentally/scheduler/ -run TestRecallTargetBusy -v
```

Expected: FAIL — `s.recallTargetBusy undefined (type *DailySyncScheduler has no field or method recallTargetBusy)`.

- [ ] **Step 3: Write minimal implementation**

Create `EmailServiceGo/internal/dentally/scheduler/recall_targets.go`:

```go
package scheduler

import (
	"context"
	"os"
	"strconv"

	"github.com/treatmentpath/email-service/internal/logger"
	"go.uber.org/zap"
)

// recallStaleRunMinutes is how long a checkpoint may sit in 'running'/'syncing'
// without its updated_at moving before we treat the run as abandoned and allow a
// fresh one. updateRecallCheckpoint touches updated_at after every patient, so a
// live run refreshes this constantly. Without the escape hatch a process killed
// mid-sync would block that practice forever (ReconcileInterruptedSyncs only
// clears such rows at service startup).
func recallStaleRunMinutes() int {
	if v := os.Getenv("RECALL_STALE_RUN_MINUTES"); v != "" {
		if n, err := strconv.Atoi(v); err == nil && n > 0 {
			return n
		}
	}
	return 45
}

// recallTargetBusy reports whether the practice we are about to sync already has
// a LIVE sync in flight.
//
// This must be checked per TARGET, not per gating practice. For a group head the
// gating query in getRecallAutoSyncPractices joins the checkpoint on the head's
// own practice_id, but runRecallAutoSync fans the actual work out to the head's
// CHILDREN — so the head's row reads 'completed' while a child's reads 'running',
// and the head-level guard never fires. That mismatch is what let every 30-minute
// tick pre-empt Church View's in-flight sync for five weeks.
//
// Fails OPEN (returns false) on a query error: a monitoring failure must not
// silently stop syncing altogether.
func (s *DailySyncScheduler) recallTargetBusy(ctx context.Context, targetID int) bool {
	var busy bool
	err := s.db.WithContext(ctx).Raw(`
		SELECT EXISTS (
			SELECT 1 FROM recall_sync_checkpoint
			WHERE practice_id = ?
			  AND status IN ('running', 'syncing')
			  AND updated_at > NOW() - make_interval(mins => ?)
		)
	`, targetID, recallStaleRunMinutes()).Scan(&busy).Error
	if err != nil {
		logger.Warn("recall auto-sync: busy check failed; proceeding",
			zap.Int("practice_id", targetID), zap.Error(err))
		return false
	}
	return busy
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd EmailServiceGo
DATABASE_URL='postgres://mannie@localhost:5432/treatmentpath_db?sslmode=disable' \
  go test ./internal/dentally/scheduler/ -run TestRecallTargetBusy -v
```

Expected: PASS, all 7 subtests.

- [ ] **Step 5: Wire the guard into the tick**

In `EmailServiceGo/internal/dentally/scheduler/scheduler.go`, replace the target loop inside `runRecallAutoSync` (currently lines 502-519) with:

```go
				for _, target := range s.resolveSyncTargets(ctx, p.ID) {
					// Skip a target that is already mid-sync. Checked per TARGET
					// because the head-level gate in getRecallAutoSyncPractices
					// only ever sees the HEAD's checkpoint — see recallTargetBusy.
					if s.recallTargetBusy(ctx, target) {
						logger.Info("Recall auto-sync: target already syncing, skipping",
							zap.Int("practice_id", target),
							zap.Int("head_practice_id", p.ID),
							zap.String("practice_name", p.Name))
						continue
					}
					result, err := s.svc.RunRecallDeltaSync(ctx, target)
					if err != nil {
						logger.Error("Recall auto-sync: delta sync failed",
							zap.Int("practice_id", target),
							zap.Int("head_practice_id", p.ID),
							zap.String("practice_name", p.Name),
							zap.Error(err))
						continue
					}
					logger.Info("Recall auto-sync: delta sync completed",
						zap.Int("practice_id", target),
						zap.Int("head_practice_id", p.ID),
						zap.String("practice_name", p.Name),
						zap.Int("total", result.Total),
						zap.Int("synced", result.Synced),
						zap.Int("failed", result.Failed))
				}
```

- [ ] **Step 6: Verify the whole scheduler package still builds and passes**

```bash
cd EmailServiceGo
go build ./... && \
DATABASE_URL='postgres://mannie@localhost:5432/treatmentpath_db?sslmode=disable' \
  go test ./internal/dentally/scheduler/ -v
```

Expected: build clean, all tests PASS.

- [ ] **Step 7: Commit**

```bash
git add EmailServiceGo/internal/dentally/scheduler/recall_targets.go \
        EmailServiceGo/internal/dentally/scheduler/recall_targets_test.go \
        EmailServiceGo/internal/dentally/scheduler/scheduler.go
git commit -m "fix(recall): skip targets already mid-sync instead of pre-empting them

The recall auto-sync gate joins recall_sync_checkpoint on the gating practice,
but a group head fans work out to its children — so the head's row read
'completed' while a child's read 'running' and the guard never fired. Every
30-minute tick cancelled the child's in-flight sync, so the delta watermark
never advanced and the same ~1282 patients were re-fetched forever.

Check the checkpoint of the practice actually being synced, with a staleness
escape hatch so an abandoned run cannot block a practice permanently."
```

---

### Task 2: A scheduled sync must not pre-empt an in-flight run

Defence in depth behind Task 1. Even if the DB guard is bypassed (racing ticks, replica lag, a future caller), the engine itself should refuse to let a *scheduled* run kill a live one. Manual runs keep today's pre-empting behaviour so "Sync now" still works.

**Files:**
- Modify: `EmailServiceGo/internal/dentally/recall/engine.go:199-251`
- Modify: `EmailServiceGo/internal/dentally/service.go:465-467`
- Modify: `EmailServiceGo/internal/dentally/scheduler/scheduler.go:37-38` (the `SyncService` interface) and the call site from Task 1 Step 5
- Create: `EmailServiceGo/internal/dentally/recall/engine_scheduled_test.go`

**Interfaces:**
- Consumes: `RecallService.mu`, `RecallService.cancel` (`map[int]context.CancelFunc`), `RecallService.cancelGen` — all existing private fields.
- Produces:
  - `func (r *RecallService) RunRecallDeltaSyncScheduled(ctx context.Context, practiceID int) (*RecallSyncResult, error)`
  - `func (s *Service) RunRecallDeltaSyncScheduled(ctx context.Context, practiceID int) (*recall.RecallSyncResult, error)`
  - `func (r *RecallService) syncInFlight(practiceID int) bool`
  - `var ErrSyncAlreadyRunning = errors.New("recall sync already running for this practice")`

- [ ] **Step 1: Write the failing test**

Create `EmailServiceGo/internal/dentally/recall/engine_scheduled_test.go`:

```go
package recall

import (
	"context"
	"errors"
	"testing"
)

// syncInFlight is the in-process half of the anti-pre-emption guard: the
// scheduler consults the DB checkpoint, this consults the live cancel map.
func TestSyncInFlight(t *testing.T) {
	r := &RecallService{
		cancel:    map[int]context.CancelFunc{},
		cancelGen: map[int]int{},
	}

	if r.syncInFlight(42) {
		t.Error("syncInFlight = true with an empty cancel map, want false")
	}

	_, cancelFn := context.WithCancel(context.Background())
	t.Cleanup(cancelFn)
	r.mu.Lock()
	r.cancel[42] = cancelFn
	r.mu.Unlock()

	if !r.syncInFlight(42) {
		t.Error("syncInFlight = false while a run is registered, want true")
	}
	if r.syncInFlight(43) {
		t.Error("syncInFlight = true for an unrelated practice, want false")
	}
}

// A SCHEDULED delta must decline rather than cancel a live run. This is the
// exact behaviour whose absence caused the Church View livelock.
func TestRunRecallDeltaSyncScheduled_DeclinesWhenInFlight(t *testing.T) {
	r := &RecallService{
		cancel:    map[int]context.CancelFunc{},
		cancelGen: map[int]int{},
	}

	runCtx, cancelFn := context.WithCancel(context.Background())
	t.Cleanup(cancelFn)
	r.mu.Lock()
	r.cancel[42] = cancelFn
	r.mu.Unlock()

	result, err := r.RunRecallDeltaSyncScheduled(context.Background(), 42)

	if !errors.Is(err, ErrSyncAlreadyRunning) {
		t.Fatalf("err = %v, want ErrSyncAlreadyRunning", err)
	}
	if result != nil {
		t.Errorf("result = %+v, want nil", result)
	}
	// The critical assertion: the in-flight run was NOT cancelled.
	select {
	case <-runCtx.Done():
		t.Error("scheduled sync cancelled the in-flight run; it must decline instead")
	default:
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd EmailServiceGo
go test ./internal/dentally/recall/ -run 'TestSyncInFlight|TestRunRecallDeltaSyncScheduled' -v
```

Expected: FAIL — `r.syncInFlight undefined`, `r.RunRecallDeltaSyncScheduled undefined`, `undefined: ErrSyncAlreadyRunning`.

- [ ] **Step 3: Write minimal implementation**

In `EmailServiceGo/internal/dentally/recall/engine.go`, add `"errors"` to the import block, then insert immediately after `RunRecallDeltaSync` (after line 212):

```go
// ErrSyncAlreadyRunning is returned by the SCHEDULED entry points when a run for
// this practice is already in flight. It is an expected outcome, not a fault.
var ErrSyncAlreadyRunning = errors.New("recall sync already running for this practice")

// syncInFlight reports whether a run is currently registered for this practice.
func (r *RecallService) syncInFlight(practiceID int) bool {
	r.mu.Lock()
	defer r.mu.Unlock()
	_, ok := r.cancel[practiceID]
	return ok
}

// RunRecallDeltaSyncScheduled is the entry point for the recurring auto-sync
// tick. It differs from RunRecallDeltaSync in exactly one way: it DECLINES when a
// run is already in flight instead of cancelling it.
//
// runRecallSync pre-empts by design, so a human clicking "Sync now" replaces a
// stuck run. That is wrong for a timer: a sync longer than the tick interval was
// killed and restarted forever, and because a cancelled run never reaches the
// 'completed' branch of markRecallCheckpointStatus, its last_synced_at watermark
// never advanced — so the next run re-selected the same work. Church View sat in
// that loop from 2026-07-03 to 2026-08-11.
func (r *RecallService) RunRecallDeltaSyncScheduled(ctx context.Context, practiceID int) (*RecallSyncResult, error) {
	if r.syncInFlight(practiceID) {
		log.Printf("[RecallEngine] scheduled delta for practice %d declined — a run is already in flight", practiceID)
		return nil, ErrSyncAlreadyRunning
	}
	return r.RunRecallDeltaSync(ctx, practiceID)
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd EmailServiceGo
go test ./internal/dentally/recall/ -run 'TestSyncInFlight|TestRunRecallDeltaSyncScheduled' -v
```

Expected: PASS.

- [ ] **Step 5: Expose it through the service and the scheduler interface**

In `EmailServiceGo/internal/dentally/service.go`, after `RunRecallDeltaSync` (line 467):

```go
// RunRecallDeltaSyncScheduled runs an incremental sync for the recurring tick,
// declining rather than pre-empting a run that is already in flight.
func (s *Service) RunRecallDeltaSyncScheduled(ctx context.Context, practiceID int) (*recall.RecallSyncResult, error) {
	return s.Recall.RunRecallDeltaSyncScheduled(ctx, practiceID)
}
```

In `EmailServiceGo/internal/dentally/scheduler/scheduler.go`, add to the `SyncService` interface after line 38:

```go
	RunRecallDeltaSyncScheduled(ctx context.Context, practiceID int) (*recall.RecallSyncResult, error)
```

Then in `runRecallAutoSync`, change the call added in Task 1 Step 5 from `s.svc.RunRecallDeltaSync(ctx, target)` to `s.svc.RunRecallDeltaSyncScheduled(ctx, target)`, and make a declined run quiet rather than an error — replace the `if err != nil` block with:

```go
					if errors.Is(err, recall.ErrSyncAlreadyRunning) {
						logger.Info("Recall auto-sync: target already syncing, skipping",
							zap.Int("practice_id", target),
							zap.Int("head_practice_id", p.ID))
						continue
					}
					if err != nil {
						logger.Error("Recall auto-sync: delta sync failed",
							zap.Int("practice_id", target),
							zap.Int("head_practice_id", p.ID),
							zap.String("practice_name", p.Name),
							zap.Error(err))
						continue
					}
```

Add `"errors"` to `scheduler.go`'s import block if not already present.

- [ ] **Step 6: Verify build and full package tests**

```bash
cd EmailServiceGo
go build ./... && go vet ./internal/dentally/... && \
DATABASE_URL='postgres://mannie@localhost:5432/treatmentpath_db?sslmode=disable' \
  go test ./internal/dentally/recall/ ./internal/dentally/scheduler/ -v
```

Expected: build clean, vet clean, tests PASS. If any mock implementing `SyncService` fails to compile, add the new method to it.

- [ ] **Step 7: Commit**

```bash
git add EmailServiceGo/internal/dentally/recall/engine.go \
        EmailServiceGo/internal/dentally/recall/engine_scheduled_test.go \
        EmailServiceGo/internal/dentally/service.go \
        EmailServiceGo/internal/dentally/scheduler/scheduler.go
git commit -m "fix(recall): scheduled delta declines instead of pre-empting an in-flight run

runRecallSync cancels any in-flight run for the practice on entry. That is right
for a human clicking Sync now and wrong for a 30-minute timer: a sync longer than
the interval was killed and restarted forever. Add a scheduled entry point that
declines with ErrSyncAlreadyRunning; manual/HTTP runs keep pre-empting."
```

---

### Task 3: Resume a consistent interrupted delta instead of restarting from patient #1

`RunRecallDeltaSync` passes `resume=false`, so `saveRecallCheckpointStart` zeroes `last_patient_id` and `processed_count`. That is why Church View re-ground the same first ~150 patients every cycle and never reached the tail. Tasks 1-2 stop the interruptions; this stops a single interruption from throwing away hours of work.

Safety rests on three existing properties: `checkpointConsistent` rejects a checkpoint whose saved patient list does not match the run total (i.e. clobbered by an overlapping run); `resumeStartIndex` returns 0 when `last_patient_id` is not found in the rebuilt list; and every per-patient write is an idempotent upsert. So the worst case of a wrong resume is repeated work, never lost or corrupted work.

**Files:**
- Modify: `EmailServiceGo/internal/dentally/recall/engine.go:202-212` (`RunRecallDeltaSync`)
- Modify: `EmailServiceGo/internal/dentally/recall/checkpoint_resume_test.go` (extend the existing suite)

**Interfaces:**
- Consumes: `loadRecallCheckpoint(ctx, practiceID) (*RecallSyncProgress, []recallPatientData, error)`, `checkpointConsistent(progress, patients) bool` — both existing.
- Produces: `func (r *RecallService) deltaShouldResume(ctx context.Context, practiceID int) bool`

- [ ] **Step 1: Write the failing test**

Append to `EmailServiceGo/internal/dentally/recall/checkpoint_resume_test.go`:

```go
// A delta may resume only from an interrupted delta whose saved patient list is
// self-consistent. Anything else must restart, because resuming from a clobbered
// or foreign checkpoint would silently skip patients.
func TestDeltaShouldResume(t *testing.T) {
	cases := []struct {
		name     string
		progress *RecallSyncProgress
		patients []recallPatientData
		want     bool
	}{
		{
			name:     "interrupted delta with a consistent list resumes",
			progress: &RecallSyncProgress{Status: "interrupted", Mode: "delta", TotalRecalls: 2, LastPatientID: 11},
			patients: []recallPatientData{{DentallyPatientID: 11}, {DentallyPatientID: 12}},
			want:     true,
		},
		{
			name:     "completed run does not resume",
			progress: &RecallSyncProgress{Status: "completed", Mode: "delta", TotalRecalls: 2, LastPatientID: 11},
			patients: []recallPatientData{{DentallyPatientID: 11}, {DentallyPatientID: 12}},
			want:     false,
		},
		{
			name:     "full-sync checkpoint does not resume a delta",
			progress: &RecallSyncProgress{Status: "interrupted", Mode: "sync", TotalRecalls: 2, LastPatientID: 11},
			patients: []recallPatientData{{DentallyPatientID: 11}, {DentallyPatientID: 12}},
			want:     false,
		},
		{
			name:     "list length mismatch (clobbered by an overlapping run) does not resume",
			progress: &RecallSyncProgress{Status: "interrupted", Mode: "delta", TotalRecalls: 5702, LastPatientID: 11},
			patients: []recallPatientData{{DentallyPatientID: 11}},
			want:     false,
		},
		{
			name:     "nothing processed yet does not resume",
			progress: &RecallSyncProgress{Status: "interrupted", Mode: "delta", TotalRecalls: 2, LastPatientID: 0},
			patients: []recallPatientData{{DentallyPatientID: 11}, {DentallyPatientID: 12}},
			want:     false,
		},
		{
			name:     "no checkpoint at all does not resume",
			progress: nil,
			patients: nil,
			want:     false,
		},
	}

	for _, tc := range cases {
		t.Run(tc.name, func(t *testing.T) {
			if got := deltaResumable(tc.progress, tc.patients); got != tc.want {
				t.Errorf("deltaResumable = %v, want %v", got, tc.want)
			}
		})
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd EmailServiceGo
go test ./internal/dentally/recall/ -run TestDeltaShouldResume -v
```

Expected: FAIL — `undefined: deltaResumable`.

- [ ] **Step 3: Write minimal implementation**

In `EmailServiceGo/internal/dentally/recall/checkpoint.go`, append:

```go
// deltaResumable reports whether an interrupted DELTA run can be safely continued
// rather than restarted from its first patient.
//
// All four conditions matter:
//   - status "interrupted"/"running": a completed run has nothing to resume, and
//     "cancelled" means a human stopped it deliberately.
//   - mode "delta": a full-sync checkpoint has a different patient universe.
//   - checkpointConsistent: rejects a list clobbered by an overlapping run.
//   - LastPatientID > 0: nothing was processed, so a restart costs nothing.
//
// Resuming wrongly is bounded: resumeStartIndex falls back to 0 when the id is
// absent from the rebuilt list, and every per-patient write is an idempotent
// upsert — so the worst case is repeated work, never skipped work.
func deltaResumable(progress *RecallSyncProgress, patients []recallPatientData) bool {
	if progress == nil {
		return false
	}
	if progress.Status != "interrupted" && progress.Status != "running" {
		return false
	}
	if progress.Mode != "delta" {
		return false
	}
	if progress.LastPatientID <= 0 {
		return false
	}
	return checkpointConsistent(progress, patients)
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd EmailServiceGo
go test ./internal/dentally/recall/ -run TestDeltaShouldResume -v
```

Expected: PASS, all 6 subtests.

- [ ] **Step 5: Use it in RunRecallDeltaSync**

Replace the body of `RunRecallDeltaSync` in `engine.go` (lines 202-212) with:

```go
func (r *RecallService) RunRecallDeltaSync(ctx context.Context, practiceID int) (*RecallSyncResult, error) {
	since, err := r.getRecallLastSyncedAt(ctx, practiceID)
	if err != nil {
		log.Printf("[RecallEngine] delta: could not read last_synced_at for practice %d: %v — running full sync", practiceID, err)
		since = ""
	}
	if since == "" {
		log.Printf("[RecallEngine] delta: no prior sync for practice %d — running full sync", practiceID)
	}

	// Continue an interrupted delta rather than re-grinding it from patient #1.
	// Before this, every interrupted cycle restarted at the top, so a delta longer
	// than the tick interval processed the same leading patients forever and never
	// reached its tail (Church View: ~150 of 1282, every 30 minutes, for 5 weeks).
	resume := false
	if progress, patients, cerr := r.loadRecallCheckpoint(ctx, practiceID); cerr != nil {
		log.Printf("[RecallEngine] delta: checkpoint read failed practice=%d: %v — starting fresh", practiceID, cerr)
	} else if deltaResumable(progress, patients) {
		resume = true
		log.Printf("[RecallEngine] delta: resuming practice %d after patient %d (%d/%d already processed)",
			practiceID, progress.LastPatientID, progress.ProcessedCount, progress.TotalRecalls)
	}

	return r.runRecallSync(ctx, practiceID, true, resume, since)
}
```

- [ ] **Step 6: Run the full recall suite**

```bash
cd EmailServiceGo
go build ./... && \
DATABASE_URL='postgres://mannie@localhost:5432/treatmentpath_db?sslmode=disable' \
  go test ./internal/dentally/recall/ -v
```

Expected: build clean, all tests PASS — in particular the pre-existing `checkpoint_resume_test.go` cases must be unaffected.

- [ ] **Step 7: Commit**

```bash
git add EmailServiceGo/internal/dentally/recall/checkpoint.go \
        EmailServiceGo/internal/dentally/recall/checkpoint_resume_test.go \
        EmailServiceGo/internal/dentally/recall/engine.go
git commit -m "fix(recall): resume an interrupted delta instead of restarting at patient #1

RunRecallDeltaSync passed resume=false, so saveRecallCheckpointStart zeroed
last_patient_id every cycle and an interrupted delta re-ground its leading
patients forever without reaching the tail. Resume when the checkpoint is an
interrupted delta with a self-consistent patient list."
```

---

### Task 4: Key the rate gate on the Dentally account, not the practice

`throttle.go` keys `rateGate` by `practiceID`, but practices 24, 26 and 27 share **one** Dentally account, one API key and one 3600/hr quota (the client already resolves this: a child with no integration of its own borrows the head's). Three practices each pacing as if they owned the whole quota over-consume it by up to 3×. Measured on production 2026-08-11: `x-ratelimit-remaining: 409` of `3600`, and Dentally answering per-patient fetches with **403**.

**Files:**
- Modify: `EmailServiceGo/internal/workflows/actions/dentally_client.go` (~L140-215, `GetIntegrationConfig`)
- Modify: `EmailServiceGo/internal/dentally/recall/throttle.go`
- Create: `EmailServiceGo/internal/dentally/recall/throttle_test.go`

**Interfaces:**
- Consumes: `DentallyClient.GetIntegrationConfig(ctx, practiceID) (*DentallyIntegrationConfig, error)`, and the local `keyPracticeID` it already computes.
- Produces:
  - New field `DentallyIntegrationConfig.KeyPracticeID int` (json `key_practice_id`)
  - `func (r *RecallService) rateGateKey(ctx context.Context, practiceID int) int`

- [ ] **Step 1: Write the failing test**

Create `EmailServiceGo/internal/dentally/recall/throttle_test.go`:

```go
package recall

import (
	"testing"
	"time"

	"github.com/treatmentpath/email-service/internal/workflows/actions"
)

// Practices sharing one Dentally account share one hourly quota, so they must
// share one pacing gate. Keying per practice let a 3-child group burn the
// account quota ~3x faster than the gate believed, which is what pushed the
// SmileHQ account to 409/3600 remaining and triggered Dentally 403s.
func TestRateGateSharedAcrossAccountPractices(t *testing.T) {
	r := &RecallService{rateGate: map[int]time.Time{}}

	// Head 24 holds the key; children 26 and 27 borrow it.
	r.rateGateKeyOverride = map[int]int{24: 24, 26: 24, 27: 24}

	next := time.Now().Add(2 * time.Second)
	r.setRateGateFor(24, next)

	for _, pid := range []int{24, 26, 27} {
		if got := r.rateGateUntil(pid); !got.Equal(next) {
			t.Errorf("practice %d gate = %v, want the shared account gate %v", pid, got, next)
		}
	}

	// A standalone practice keeps its own independent gate.
	r.rateGateKeyOverride[99] = 99
	if got := r.rateGateUntil(99); !got.IsZero() {
		t.Errorf("standalone practice 99 gate = %v, want zero (unaffected)", got)
	}
}

// updateRateGate must store under the account key so siblings observe it.
func TestUpdateRateGateStoresUnderAccountKey(t *testing.T) {
	r := &RecallService{rateGate: map[int]time.Time{}}
	r.rateGateKeyOverride = map[int]int{27: 24}

	r.updateRateGate(27, &actions.DentallyResponseMeta{
		StatusCode:         200,
		RateLimitLimit:     3600,
		RateLimitRemaining: 100,
		RateLimitResetUnix: time.Now().Add(time.Hour).Unix(),
	})

	if _, ok := r.rateGate[24]; !ok {
		t.Error("updateRateGate(27) did not store under account key 24")
	}
	if _, ok := r.rateGate[27]; ok {
		t.Error("updateRateGate(27) stored under the practice key 27; must use the account key")
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd EmailServiceGo
go test ./internal/dentally/recall/ -run 'TestRateGate|TestUpdateRateGate' -v
```

Expected: FAIL — `r.rateGateKeyOverride undefined`, `r.setRateGateFor undefined`, `r.rateGateUntil undefined`.

- [ ] **Step 3: Expose the key-holding practice on the client config**

In `EmailServiceGo/internal/workflows/actions/dentally_client.go`, add to the `DentallyIntegrationConfig` struct (after the `SiteID` field, ~line 63):

```go
	// KeyPracticeID is the practice whose Dentally integration supplies the API
	// key for this request — itself for a standalone practice, the group HEAD for
	// a mapped child. Callers that must respect the ACCOUNT-wide rate limit (one
	// quota per Dentally account, shared by every practice in the group) key their
	// throttling on this, not on the requesting practice id.
	KeyPracticeID int `json:"key_practice_id"`
```

Then in `GetIntegrationConfig`, where the config is built (the literal containing `SiteID: siteID,` at ~line 210), add:

```go
		KeyPracticeID: keyPracticeID,
```

- [ ] **Step 4: Re-key the gate**

Replace `EmailServiceGo/internal/dentally/recall/throttle.go`'s gate accessors. Keep `waitRateGate`/`updateRateGate`/`gatedRequest` signatures unchanged so no caller changes; only the key changes.

```go
// rateGateKey returns the id this practice's pacing should be stored under: the
// practice that HOLDS the Dentally API key. A Dentally group is ONE account with
// ONE hourly quota shared by the head and every child, so pacing each practice
// separately lets an N-practice group consume the quota ~N times faster than the
// gate believes. Falls back to the practice's own id when the config cannot be
// read — the previous behaviour, and safe for standalone practices.
func (r *RecallService) rateGateKey(ctx context.Context, practiceID int) int {
	if r.rateGateKeyOverride != nil { // tests
		if k, ok := r.rateGateKeyOverride[practiceID]; ok {
			return k
		}
	}
	cfg, err := r.Client.GetIntegrationConfig(ctx, practiceID)
	if err != nil || cfg == nil || cfg.KeyPracticeID == 0 {
		return practiceID
	}
	return cfg.KeyPracticeID
}

// rateGateUntil reads the next-allowed time for a practice's ACCOUNT.
func (r *RecallService) rateGateUntil(practiceID int) time.Time {
	key := r.rateGateKey(context.Background(), practiceID)
	r.rateGateMu.Lock()
	defer r.rateGateMu.Unlock()
	return r.rateGate[key]
}

// setRateGateFor stores a next-allowed time under an already-resolved account key.
func (r *RecallService) setRateGateFor(accountKey int, until time.Time) {
	r.rateGateMu.Lock()
	defer r.rateGateMu.Unlock()
	r.rateGate[accountKey] = until
}

// waitRateGate blocks until this practice's account is allowed its next call.
func (r *RecallService) waitRateGate(ctx context.Context, practiceID int) {
	if d := time.Until(r.rateGateUntil(practiceID)); d > 0 {
		select {
		case <-ctx.Done():
		case <-time.After(d):
		}
	}
}

// updateRateGate recomputes the next-allowed time from a response's rate-limit
// headers and stores it against the ACCOUNT, so sibling practices in the same
// Dentally group observe the same pacing.
func (r *RecallService) updateRateGate(practiceID int, meta *actions.DentallyResponseMeta) {
	if meta == nil {
		return
	}
	r.setRateGateFor(
		r.rateGateKey(context.Background(), practiceID),
		migration.ComputeNextAllowedAt(meta, rateGateBaseDelay),
	)
}
```

Add the `rateGateKeyOverride` test seam to the `RecallService` struct in `engine.go` (next to `rateGate`):

```go
	// rateGateKeyOverride short-circuits rateGateKey in tests so they need no DB.
	// nil in production.
	rateGateKeyOverride map[int]int
```

Add `"context"` to `throttle.go`'s imports if not already present.

- [ ] **Step 5: Run test to verify it passes**

```bash
cd EmailServiceGo
go build ./... && go test ./internal/dentally/recall/ -run 'TestRateGate|TestUpdateRateGate' -v
```

Expected: build clean, PASS.

- [ ] **Step 6: Commit**

```bash
git add EmailServiceGo/internal/workflows/actions/dentally_client.go \
        EmailServiceGo/internal/dentally/recall/throttle.go \
        EmailServiceGo/internal/dentally/recall/throttle_test.go \
        EmailServiceGo/internal/dentally/recall/engine.go
git commit -m "fix(recall): pace the rate gate per Dentally account, not per practice

A Dentally group is one account with one 3600/hr quota shared by the head and
every child, but rateGate was keyed by practice id, so an N-practice group
consumed the quota ~Nx faster than the gate believed. Measured on production:
409 of 3600 remaining, with Dentally answering per-patient fetches 403."
```

---

### Task 5: Stop discarding learned pacing at the start of every run

`runRecallSync` calls `r.clearRateGate(practiceID)` on entry ("start fresh"). With the livelock that happened every 30 minutes, so the pacing learned from Dentally's own `x-ratelimit-*` headers was thrown away and each restart burst until it re-learned. Pacing state describes the *account's* quota, which does not reset because a run restarted — it resets when Dentally's window resets, which `ComputeNextAllowedAt` already accounts for.

**Files:**
- Modify: `EmailServiceGo/internal/dentally/recall/throttle.go` (`clearRateGate`)
- Modify: `EmailServiceGo/internal/dentally/recall/engine.go:251` (the call site)
- Modify: `EmailServiceGo/internal/dentally/recall/throttle_test.go` (extend Task 4's suite)

**Interfaces:**
- Consumes: `rateGateKey`, `setRateGateFor`, `rateGateUntil` from Task 4.
- Produces: no new symbols; `clearRateGate` is deleted.

- [ ] **Step 1: Write the failing test**

Append to `EmailServiceGo/internal/dentally/recall/throttle_test.go`:

```go
// Pacing describes the ACCOUNT's remaining quota, which does not reset just
// because a sync run restarted. Clearing it per run made every restart burst
// until it re-learned the limit — and during the livelock that was every 30 min.
func TestRateGateSurvivesRunRestart(t *testing.T) {
	r := &RecallService{rateGate: map[int]time.Time{}}
	r.rateGateKeyOverride = map[int]int{27: 24}

	next := time.Now().Add(5 * time.Second)
	r.setRateGateFor(24, next)

	// Simulate what a new run does on entry. After the fix there is no gate reset,
	// so the learned pacing must still be in force.
	if got := r.rateGateUntil(27); !got.Equal(next) {
		t.Errorf("gate after run restart = %v, want preserved %v", got, next)
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Before removing the call, temporarily prove the old behaviour was wrong by adding `r.clearRateGate(27)` just before the assertion, then:

```bash
cd EmailServiceGo
go test ./internal/dentally/recall/ -run TestRateGateSurvivesRunRestart -v
```

Expected: FAIL — gate reads the zero time instead of the preserved value. Remove the temporary line before continuing.

- [ ] **Step 3: Delete clearRateGate and its call site**

In `EmailServiceGo/internal/dentally/recall/throttle.go`, delete the whole `clearRateGate` function.

In `EmailServiceGo/internal/dentally/recall/engine.go`, delete line 251 and replace the comment above it:

```go
	// NB: the rate gate is deliberately NOT reset here. It tracks the Dentally
	// ACCOUNT's remaining hourly quota, which does not reset because a run
	// restarted — ComputeNextAllowedAt already handles the real window reset.
	// Clearing it per run made every restart burst until it re-learned the limit.
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd EmailServiceGo
go build ./... && go test ./internal/dentally/recall/ -v
```

Expected: build clean (no remaining references to `clearRateGate`), all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add EmailServiceGo/internal/dentally/recall/throttle.go \
        EmailServiceGo/internal/dentally/recall/throttle_test.go \
        EmailServiceGo/internal/dentally/recall/engine.go
git commit -m "fix(recall): keep learned rate pacing across run restarts

runRecallSync cleared the rate gate on entry, so each restart burst until it
re-learned Dentally's limit from response headers. Pacing tracks the account's
remaining quota, which does not reset because a run restarted."
```

---

### Task 6: Stamp sync completion on each synced child, so group children reconcile

`_due_reconciliation_practices` reads `RecallAutoSyncConfig.last_completed_at` **per practice**, but `runRecallAutoSync` only ever stamps the **head's** row. Church View's own row still reads `2026-07-03 09:41` — it has not been reconciled since the day its watermark froze. Group children silently never reconcile.

Stamping happens per target on genuine success only. The head's existing deferred stamp is left exactly as it is (it fires on every exit path by design, so a permanently failing practice still gets reconciled).

**Files:**
- Modify: `EmailServiceGo/internal/dentally/scheduler/scheduler.go` (`stampRecallAutoSyncComplete` and the target loop)
- Create: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests/test_reconciliation_group_children.py`

**Interfaces:**
- Consumes: `DailySyncScheduler.db`, `stampRecallAutoSyncComplete(ctx, practiceID)` — existing.
- Produces: no new Go symbols; `stampRecallAutoSyncComplete` gains a second call site.

- [ ] **Step 1: Write the failing Django test**

Create `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests/test_reconciliation_group_children.py`:

```python
"""
Regression: a group CHILD practice must become due for reconciliation once its
own recall sync completes.

_due_reconciliation_practices reads RecallAutoSyncConfig.last_completed_at per
practice, but the Go scheduler only stamped the group HEAD's row. Church View
(practice 27, child of head 24) therefore last reconciled on 2026-07-03 — the
same day its delta watermark froze — because nothing ever moved its own
last_completed_at.
"""

from datetime import timedelta
from unittest.mock import patch

from django.test import TestCase, override_settings
from django.utils import timezone

from UserAuthentication.models import Practice

from ..models import RecallAutoSyncConfig
from ..tasks import _due_reconciliation_practices


@override_settings(SECURE_SSL_REDIRECT=False)
class GroupChildReconciliationDueTests(TestCase):
    def setUp(self):
        self.head = Practice.objects.create(name="Group Head", slug="group-head-recon")
        self.child = Practice.objects.create(name="Group Child", slug="group-child-recon")
        # Completed well outside the settle grace so the only variable under test
        # is whether the child's own timestamp moved.
        self.done = timezone.now() - timedelta(hours=2)

    def _configs(self, head_done, child_done):
        RecallAutoSyncConfig.objects.create(
            practice=self.head, enabled=True, frequency_minutes=30,
            last_completed_at=head_done,
        )
        RecallAutoSyncConfig.objects.create(
            practice=self.child, enabled=True, frequency_minutes=30,
            last_completed_at=child_done,
        )

    def test_child_is_due_when_its_own_sync_completed(self):
        self._configs(head_done=self.done, child_done=self.done)
        with patch(
            "dentallyIntegration.reconciliation.active_practice_ids",
            return_value=[self.head.id, self.child.id],
        ):
            due = _due_reconciliation_practices()
        self.assertIn(
            self.child.id, due,
            "a group child whose own recall sync completed must be due for reconciliation",
        )

    def test_child_is_not_due_when_only_the_head_was_stamped(self):
        # The pre-fix production state: the head is stamped every 30 minutes while
        # the child's timestamp is frozen weeks in the past.
        self._configs(
            head_done=self.done,
            child_done=timezone.now() - timedelta(days=39),
        )
        from ..models import ReconciliationStatus

        ReconciliationStatus.objects.create(
            practice=self.child, last_delta_run_at=timezone.now() - timedelta(days=38)
        )
        with patch(
            "dentallyIntegration.reconciliation.active_practice_ids",
            return_value=[self.head.id, self.child.id],
        ):
            due = _due_reconciliation_practices()
        self.assertNotIn(
            self.child.id, due,
            "documents the broken state: a child never stamped never becomes due",
        )
```

- [ ] **Step 2: Run the tests**

```bash
source TreatmentPathBackend/venv/bin/activate
cd TreatmentPathBackend/TreatmentPath
python manage.py test dentallyIntegration.tests.test_reconciliation_group_children --keepdb -v 2
```

Expected: `test_child_is_not_due_when_only_the_head_was_stamped` PASSES (it pins the broken behaviour the Go fix removes). `test_child_is_due_when_its_own_sync_completed` PASSES too — this pair proves the Django trigger is correct and the defect is purely that Go never moves the child's timestamp. If either fails, the reconciliation trigger has a second bug; stop and report it rather than adjusting the test.

- [ ] **Step 3: Stamp each successfully synced target in Go**

In `EmailServiceGo/internal/dentally/scheduler/scheduler.go`, inside `runRecallAutoSync`, immediately after the successful "delta sync completed" log added in Task 2 Step 5:

```go
					// Stamp completion on the TARGET's own config row as well as the
					// head's. Django's _due_reconciliation_practices reads
					// RecallAutoSyncConfig.last_completed_at PER PRACTICE, so a group
					// child whose row never moves is never reconciled — Church View
					// last reconciled 2026-07-03 for exactly this reason. No-op when
					// the target has no config row, and a no-op re-stamp when the
					// target IS the head.
					if target != p.ID {
						s.stampRecallAutoSyncComplete(ctx, target)
					}
```

- [ ] **Step 4: Verify Go builds and the scheduler suite passes**

```bash
cd EmailServiceGo
go build ./... && go vet ./internal/dentally/... && \
DATABASE_URL='postgres://mannie@localhost:5432/treatmentpath_db?sslmode=disable' \
  go test ./internal/dentally/scheduler/ -v
```

Expected: build clean, vet clean, PASS.

- [ ] **Step 5: Run the Django formatter/linter on the new test**

```bash
cd TreatmentPathBackend
source venv/bin/activate
pre-commit run --files TreatmentPath/dentallyIntegration/tests/test_reconciliation_group_children.py
```

Expected: black / isort / pyflakes PASS (black may reformat — re-run until clean).

- [ ] **Step 6: Commit**

```bash
git add EmailServiceGo/internal/dentally/scheduler/scheduler.go \
        TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests/test_reconciliation_group_children.py
git commit -m "fix(recall): stamp sync completion on each synced child practice

Django's _due_reconciliation_practices reads RecallAutoSyncConfig.last_completed_at
per practice, but the scheduler only stamped the group head's row, so group
children never became due. Church View last reconciled 2026-07-03."
```

---

### Task 7: Full-stack verification and the production unblock note

**Files:**
- Create: `TreatmentPathBackend/to-run-inprod/2026-08-11-recall-livelock-fix.txt`

- [ ] **Step 1: Run every touched suite together**

```bash
cd EmailServiceGo
go build ./... && go vet ./internal/... && \
DATABASE_URL='postgres://mannie@localhost:5432/treatmentpath_db?sslmode=disable' \
  go test ./internal/dentally/... -count=1

cd ../TreatmentPathBackend && source venv/bin/activate && cd TreatmentPath
python manage.py test dentallyIntegration --keepdb -v 1
```

Expected: all PASS. Record the actual output — do not claim success without it.

- [ ] **Step 2: Write the production runbook note**

Create `TreatmentPathBackend/to-run-inprod/2026-08-11-recall-livelock-fix.txt`:

```
RECALL SYNC LIVELOCK FIX — production notes (2026-08-11)

NO MIGRATIONS. NO MANUAL DB CHANGES REQUIRED.

Deploy target: the Go `email-service` container on SMTP-SERVER-PROD
(NOT Treatmentpath-PROD — the Go service does not run there).

Self-healing on deploy:
  Practice 27's checkpoint is currently status='running' with a recent
  updated_at, which the new recallTargetBusy guard would treat as live.
  Restarting the container runs ReconcileInterruptedSyncs, which flips every
  'running'/'syncing' row to 'interrupted' — so the guard clears itself on the
  deploy restart. No manual UPDATE needed.

Optional env tuning (defaults are fine):
  RECALL_STALE_RUN_MINUTES  default 45

What to expect after deploy, practice 27 (Church View):
  1. First tick starts a delta and is NOT cancelled 30 minutes later.
     Confirm: recall_sync_checkpoint.processed_count for practice_id=27 keeps
     climbing past ~150 and started_at STOPS changing every 30 minutes.
  2. The run takes ~3 hours at the measured ~7 patients/min (faster once the
     403s stop). On completion, last_synced_at jumps from 2026-07-03 to today.
     THIS IS THE SUCCESS SIGNAL.
  3. The next delta should drop from ~1282 patients to single digits.
  4. total_spend backfills as patients pass the financial stage. Expect the
     zero-spend share for practice 27 to fall from ~36% toward the group norm
     (~16%). This needs no backfill command — the sync fills it in.
  5. Dentally quota should recover. Watch x-ratelimit-remaining trend up from
     ~409/3600.

Verification queries (read-only, run on Treatmentpath-PROD):
  docker exec treatmentpathbackend_db_1 psql -U treatmentpath_user -d treatmentpath_db -c "
    SELECT practice_id, status, total_recalls, processed_count,
           started_at, completed_at, last_synced_at
    FROM recall_sync_checkpoint WHERE practice_id IN (24,26,27) ORDER BY practice_id;"

  docker exec treatmentpathbackend_db_1 psql -U treatmentpath_user -d treatmentpath_db -c "
    SELECT practice_id, count(*) AS patients,
           count(*) FILTER (WHERE COALESCE(total_spend,0)=0) AS zero_spend
    FROM recall_patient WHERE practice_id IN (24,26,27) GROUP BY 1 ORDER BY 1;"

ROLLBACK: redeploy the previous image. No schema or data changes to undo.
```

- [ ] **Step 3: Commit**

```bash
git add TreatmentPathBackend/to-run-inprod/2026-08-11-recall-livelock-fix.txt
git commit -m "docs: production runbook for the recall sync livelock fix"
```

---

## Self-Review

**Spec coverage** — the six defects from the investigation, each mapped to a task:

| # | Defect | Task |
|---|---|---|
| 1 | Concurrency guard joins the checkpoint on the head, not the synced target | 1 |
| 2 | `runRecallSync` pre-empts an in-flight run on scheduler re-entry | 2 |
| 3 | Delta restarts at patient #1 (`resume=false`) instead of continuing | 3 |
| 4 | Rate gate keyed per practice while the quota is per Dentally account | 4 |
| 5 | `clearRateGate` discards learned pacing on every run start | 5 |
| 6 | Completion stamped only on the head, so group children never reconcile | 6 |

The 403s are not given their own task: they were proved to be a *consequence* of quota exhaustion, not a permissions fault (direct API calls for the failing patients returned 200), so tasks 1-5 are their fix. If 403s persist after deploy with healthy quota headers, that is a new investigation.

**Placeholder scan** — no TBDs, no "add error handling", every code step carries real code, every test step names the exact command and expected result.

**Type consistency** — `recallTargetBusy` / `recallStaleRunMinutes` (Task 1) are used only in Task 1. `ErrSyncAlreadyRunning`, `syncInFlight`, `RunRecallDeltaSyncScheduled` (Task 2) are consumed by Task 6's target loop, which sits after Task 2's edit to the same block. `deltaResumable` (Task 3) is defined in `checkpoint.go` and called from `engine.go`. `KeyPracticeID`, `rateGateKey`, `rateGateUntil`, `setRateGateFor`, `rateGateKeyOverride` (Task 4) are all consumed by Task 5. `stampRecallAutoSyncComplete` (Task 6) is pre-existing and unchanged in signature.

**Ordering note** — Tasks 1, 2 and 6 all edit the same target loop in `runRecallAutoSync`. Do them in numeric order; if executing out of order, re-read the current state of that loop before editing.

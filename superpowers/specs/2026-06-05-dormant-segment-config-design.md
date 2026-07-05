# Admin-Configurable Dormant Segment — Design (Slice 1)

**Date:** 2026-06-05
**Status:** Approved design — ready for implementation planning
**Scope:** First vertical slice of an admin-configurable segment system, delivering the **Dormant** segment end-to-end (config persistence → live evaluation → Dormant tab rendering).

---

## 1. Background & Problem

Today the recall backend **auto-classifies** patients into 7 fixed segments (`vip, loyal, at_risk, lapsed, new, high_revenue, chronic_dna`) via a hard-coded scoring engine in Go (`recall/engine.go` `assignSegment` and `classification.go` `classAssignSegment`). See `RECALL_KNOWLEDGE_BASE.md` §11.

The desired model is the inverse: **the recall data already exists, and the practice admin defines what each segment means** via the Recalls Administration UI. That admin surface (`src/pages/recalls/components/administration/`) is currently **UI-only** — every section holds local React state and "Save" only fires a toast; the file comments state it is built *"before the configuration API exists."*

This slice builds that configuration API for the **Dormant** segment and wires the Dormant tab to render from it.

### Key facts established during discovery
- **`last_seen` is not stored.** The authoritative source of a patient's last completed visit is the `recall_appointment` table (`MAX(start_time) WHERE state='completed'`). `recall_record.last_completed_date` is *treatment*-based, not visit-based, and `recall_record.last_completed_appointment_id` is defined but never populated — do **not** use either for `last_seen`.
- **Two dormant config surfaces exist.** The dedicated `DormantSection` (admin → Dormant tab) and the "Dormant Patient" card inside `SegmentsSection`. **Decision:** Dormant becomes part of the Segments model; `SegmentsSection`'s card is the canonical editor. `DormantSection` is transitional and will be removed by the user later — it is **not** wired by this slice.
- **The Dormant *tab* is not wired to dormant data.** `RecallsPage` only branches `administration` vs. the default recalls table, so the "Dormant Patients" tab currently shows the same "All Recalls" list.

---

## 2. Goals & Non-Goals

### Goals
1. Persist an admin-defined Dormant rule (`not_seen_months`, default 24) in a **generalizable** per-practice segment-config model.
2. Wire the `SegmentsSection` Dormant card to load/save that config.
3. Render the Dormant tab as a live list of patients whose last completed visit is older than the configured threshold (evaluated on each request).

### Non-Goals (explicitly deferred)
- Long-term no-visit cutoff (the 6-year exclusion).
- Eligibility exclusions: archived patients, contactability (no email/phone), upcoming-booked appointments.
- The other 6 attendance segments (Regular, Lapsed, Hygiene, Exam, Treatment-Led, New).
- Any recall-engine / precompute / `recall_record` schema change.
- Removing `DormantSection` (the user will do this separately).
- A broader "settings model" for all admin sections (may come later).

---

## 3. Evaluation Model

**Live query** (chosen over precompute). On each request to the Dormant list endpoint, the backend reads the saved Dormant config and evaluates membership against existing recall data. Config edits (e.g. 24 → 18 months) and the preview count reflect **instantly**, with no engine run or schema change. Per-practice data volumes (thousands of appointments) make a live subquery acceptable; a precompute/materialized `last_seen` path remains open for the future if needed.

---

## 4. Data Model

New Django model in the `dentallyIntegration` app:

```python
class RecallSegmentConfig(models.Model):
    id          = UUIDField(primary_key=True, default=uuid4, editable=False)
    practice    = ForeignKey("UserAuthentication.Practice", on_delete=CASCADE,
                             related_name="recall_segment_configs")
    segment_key = CharField(max_length=50)        # "dormant" (only one wired now)
    enabled     = BooleanField(default=True)
    params      = JSONField(default=dict)         # dormant → {"not_seen_months": 24}
    created_at  = DateTimeField(auto_now_add=True)
    updated_at  = DateTimeField(auto_now=True)

    class Meta:
        db_table = "recall_segment_config"
        unique_together = ("practice", "segment_key")
        indexes = [Index(fields=["practice", "segment_key"])]
```

- One migration. No changes to existing recall tables or the Go engine.
- `params` is JSON so each future segment carries its own shape without a schema change.
- **Defaults:** if no row exists for `(practice, "dormant")`, the system behaves as `{enabled: true, params: {"not_seen_months": 24}}`. A row is created on first save.

---

## 5. Config API (read/save the Dormant rule)

`RecallSegmentConfigViewSet` (DRF), `permission_classes = [IsAuthenticated]`, scoped to `request.user.practice`.

### `GET /api/backend/dentally/recall-segments/dormant/`
Returns the saved row, or the default if none exists (so the card always renders):
```json
{ "segment_key": "dormant", "enabled": true, "params": { "not_seen_months": 24 } }
```

### `PUT /api/backend/dentally/recall-segments/dormant/`
Body: `{ "enabled": true, "params": { "not_seen_months": 18 } }`. Upserts the `(practice, "dormant")` row.

**Validation:**
- `not_seen_months` must be a positive integer (`>= 1`). Reject otherwise with 400.
- `segment_key` is taken from the URL; only `"dormant"` is accepted this slice (others → 404/400).
- `enabled` optional, defaults to existing/true.

---

## 6. Dormant Evaluation + List Endpoint

New action on the existing `RecallRecordViewSet` (reuses its serializer, pagination, and patient-panel link resolution):

### `GET /api/backend/dentally/recalls/dormant/`
1. Read the practice's `dormant` config → `not_seen_months` (fallback default 24). The `enabled` flag is **stored but not acted on in this slice** (see §9) — the endpoint always computes the list.
2. Annotate each `RecallRecord` with `last_seen`:
   `Subquery(MAX(recall_appointment.start_time) WHERE practice = P AND dentally_patient_id = OuterRef AND state = 'completed')` — the same Subquery/OuterRef shape as the existing `spend_in_period` annotation in `RecallRecordViewSet.list`.
3. **Dormant filter:** `last_seen IS NULL OR last_seen < (now − not_seen_months months)`.
   - `last_seen IS NULL` (never completed a visit) **counts as Dormant** (per product decision — the most "gone quiet").
   - Threshold cutoff date = `today - relativedelta(months=not_seen_months)` (calendar months).
4. Reuse existing pagination (`page`, `page_size` ≤ 100), default sort, search, and the `{count, total_pages, page, page_size, recalls:[…]}` envelope. `count` is the "matches N patients" preview number.

**"Last completed visit" definition:** `recall_appointment.state = 'completed'` only. A booked/pending future appointment does not count as "seen."

---

## 7. Frontend Wiring

### Config: `SegmentsSection` Dormant card
- New hook `useSegmentConfig('dormant')` (or inline fetch): on mount `GET`s the config and seeds the card's `not_seen_months` (`config.dormant.notSeenMonths`); on the section's **Save**, `PUT`s `{ enabled, params: { not_seen_months } }`.
- **Only the Dormant field is persisted this slice.** The other segment cards in `SegmentsSection` keep their current local-state/toast behavior until wired in later slices.
- Uses `fetchWithAuth` and the `API_ENDPOINTS` config pattern in `src/config/api.ts` (Django base URL via `getApiUrl`).

### Dormant tab: `RecallsPage`
- Branch `activeTab === 'dormant'` to render a Dormant list instead of the default "All Recalls" list.
- New hook `useDormantRecalls` (mirrors `useRecallsData`'s list-fetch shape, minus sync/websocket) calling `GET /dentally/recalls/dormant/` with pagination.
- Renders the existing `<RecallsTable>` (rows, pagination, sort, patient-panel) — **no new visual components**.

---

## 8. Testing

Follows the project's characterization-baseline approach (see `recall-map/BASELINE.md`).

### Django (`dentallyIntegration/tests.py`)
- **Config API:** `GET` returns default when no row; `PUT` upserts and round-trips; invalid `not_seen_months` → 400; practice isolation (can't read/write another practice's config).
- **Dormant list:** seed `recall_record` + `recall_appointment` rows and assert:
  - patient whose last completed visit is older than threshold → **included**;
  - patient seen within threshold → **excluded**;
  - patient with **no** completed appointment (`last_seen` null) → **included**;
  - boundary at exactly `not_seen_months`;
  - changing the config (24 → 6) shifts membership;
  - only `state='completed'` counts (a future `pending` appointment does not make a long-absent patient "recent");
  - practice isolation.

### Frontend (vitest)
- Dormant card loads the saved `not_seen_months` and Save issues the `PUT` with the edited value.
- Dormant tab renders rows returned by `useDormantRecalls` (mock the fetch).

---

## 9. Decisions & Edge Cases

- **Null `last_seen` = Dormant** (decided). Consequence: a brand-new patient registered recently with no completed visit yet will appear as Dormant. Acceptable for this slice because the exclusions that would filter them (upcoming-booked / eligibility) are deliberately deferred. Revisit when eligibility exclusions land.
- **`enabled` semantics (decided):** the `enabled` flag is persisted for forward-compatibility (the `SegmentCard` already has the toggle) but has **no behavioral effect in this slice** — the Dormant tab is always shown and the list endpoint always computes. Wiring `enabled` to tab visibility / send-audience inclusion is a future slice. This keeps slice 1 minimal (YAGNI).
- **Calendar months vs. days:** threshold uses calendar months (`relativedelta`), matching the admin UI wording ("not seen in over N months").

---

## 10. Files Touched (anticipated)

**Backend (`TreatmentPathBackend/TreatmentPath/dentallyIntegration/`)**
- `models.py` — add `RecallSegmentConfig`.
- `migrations/` — one new migration.
- `serializers.py` — `RecallSegmentConfigSerializer`.
- `views.py` — `RecallSegmentConfigViewSet`; `dormant` action on `RecallRecordViewSet`.
- `urls.py` — routes for both.
- `tests.py` — config + dormant-list tests.

**Frontend (`perfect-pixel-playground-project/src/pages/recalls/`)**
- `config/api.ts` — new endpoint entries.
- `components/administration/SegmentsSection.tsx` — wire Dormant card load/save.
- A `useSegmentConfig` hook (new) + a `useDormantRecalls` hook (new).
- `RecallsPage.tsx` — render Dormant tab from the dormant list.
- `components/RecallsTable.test.tsx` sibling / new test files for the above.

---

## 11. Future Slices (not now)
- Long-term cutoff + eligibility exclusions (needs a reliable Dentally **archived** flag — currently `is_active` is hardcoded `true`; archived patients aren't synced — so this needs data plumbing).
- The remaining 6 segments, each as a `segment_key` row with its own `params`.
- Optional precompute/materialized `last_seen` if live evaluation ever shows strain.
- Removal of `DormantSection` (user-led).

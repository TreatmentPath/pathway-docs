# Dormant Segment Config (Slice 1) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a practice admin define the Dormant rule (`not_seen_months`, default 24) in the Recalls Administration → Segments tab, persist it, and render the Dormant tab as a live list of patients whose last completed visit is older than that threshold (null last_seen = Dormant).

**Architecture:** New per-practice Django model `RecallSegmentConfig` (generalizable, JSON `params`); a config GET/PUT endpoint; a live `dormant` action on the existing `RecallRecordViewSet` that annotates `last_seen` from `recall_appointment` and filters by the saved threshold; frontend hooks wire the SegmentsSection Dormant card (save/load) and a new `DormantTab` (reusing `RecallsTable`).

**Tech Stack:** Django REST Framework + PostgreSQL (backend); React + TypeScript + vitest/happy-dom (frontend). Live query (no Go engine change).

**Spec:** `docs/superpowers/specs/2026-06-05-dormant-segment-config-design.md`

**⚠️ Git policy for this project:** The assistant/agent **never** runs git. Every "Checkpoint" step means *stop and hand the suggested commit message to the user* — the user commits.

---

## File Structure

**Backend** — `TreatmentPathBackend/TreatmentPath/dentallyIntegration/`
- `models.py` — add `RecallSegmentConfig` (Task 1).
- `migrations/00NN_recallsegmentconfig.py` — generated (Task 1).
- `views.py` — add `RecallSegmentConfigViewSet`; add `dormant` method to `RecallRecordViewSet` (Tasks 2–3).
- `urls.py` — add config + dormant routes (Tasks 2–3).
- `tests.py` — config + dormant tests (Tasks 2–3).

**Frontend** — `perfect-pixel-playground-project/src/`
- `config/api.ts` — add `RECALL_SEGMENT` + `RECALLS_DORMANT` endpoints (Task 4).
- `pages/recalls/useSegmentConfig.ts` — new hook (Task 4).
- `pages/recalls/components/administration/SegmentsSection.tsx` — wire Dormant card (Task 4).
- `pages/recalls/useSegmentConfig.test.ts` — hook test (Task 4).
- `pages/recalls/useDormantRecalls.ts` — new hook (Task 5).
- `pages/recalls/components/DormantTab.tsx` — new component (Task 5).
- `pages/recalls/RecallsPage.tsx` — branch the Dormant tab (Task 5).
- `pages/recalls/useDormantRecalls.test.ts` — hook test (Task 5).

---

## Task 1: `RecallSegmentConfig` model + migration

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/models.py` (append at end of file)
- Create: migration via `makemigrations`
- Test: `TreatmentPathBackend/TreatmentPath/dentallyIntegration/tests.py`

- [ ] **Step 1: Add the model**

Append to `models.py` (uses the existing `import uuid` and `from django.db import models` already at the top of the file):

```python
class RecallSegmentConfig(models.Model):
    """
    Admin-defined configuration for a recall segment, per practice.

    Generalizable: one row per (practice, segment_key). `params` is JSON so each
    segment carries its own rule shape without a schema change. Slice 1 wires only
    segment_key="dormant" with params {"not_seen_months": <int>}.
    """

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    practice = models.ForeignKey(
        "UserAuthentication.Practice",
        on_delete=models.CASCADE,
        related_name="recall_segment_configs",
    )
    segment_key = models.CharField(max_length=50, db_index=True)
    enabled = models.BooleanField(default=True)
    params = models.JSONField(default=dict, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = "recall_segment_config"
        unique_together = ("practice", "segment_key")
        indexes = [
            models.Index(
                fields=["practice", "segment_key"], name="idx_rsegcfg_practice_key"
            ),
        ]

    def __str__(self):
        return f"{self.segment_key} config for practice {self.practice_id}"
```

- [ ] **Step 2: Generate the migration**

Run:
```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py makemigrations dentallyIntegration
```
Expected: a new file `dentallyIntegration/migrations/00NN_recallsegmentconfig.py` is created (NN follows the current latest, e.g. 0078).

- [ ] **Step 3: Write the failing model test**

Append to `tests.py`. Add `RecallSegmentConfig` to the existing `from .models import (...)` block, then add:

```python
class RecallSegmentConfigModelTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(name="Segment Config Dental")

    def test_create_and_unique_per_practice_segment(self):
        cfg = RecallSegmentConfig.objects.create(
            practice=self.practice,
            segment_key="dormant",
            params={"not_seen_months": 24},
        )
        self.assertTrue(cfg.enabled)
        self.assertEqual(cfg.params["not_seen_months"], 24)
        with self.assertRaises(Exception):
            RecallSegmentConfig.objects.create(
                practice=self.practice, segment_key="dormant", params={}
            )
```

- [ ] **Step 4: Run the test to verify it passes** (migration makes the table exist)

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test dentallyIntegration.tests.RecallSegmentConfigModelTests --noinput
```
Expected: `Ran 1 test ... OK`.
(If a stale `test_treatmentpath` lock error appears, drop it as `mannie`: `PGPASSWORD=maniZolas1008 psql -U mannie -h localhost -d postgres -c "DROP DATABASE IF EXISTS test_treatmentpath;"`, then re-run.)

- [ ] **Step 5: Checkpoint — hand off for commit**

Tell the user: task complete. Suggested message:
`feat(recall): add RecallSegmentConfig model for admin-configurable segments`

---

## Task 2: Config API (GET/PUT the Dormant rule)

**Files:**
- Modify: `dentallyIntegration/views.py` (add viewset; near `RecallRecordViewSet`)
- Modify: `dentallyIntegration/urls.py` (import + route)
- Test: `dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing API tests**

Append to `tests.py`:

```python
class RecallSegmentConfigViewSetTests(TestCase):
    def setUp(self):
        self.factory = APIRequestFactory()
        self.get_view = RecallSegmentConfigViewSet.as_view({"get": "retrieve"})
        self.put_view = RecallSegmentConfigViewSet.as_view({"put": "update"})
        self.practice = Practice.objects.create(name="Seg Cfg API Dental")
        self.user = User.objects.create_user(
            email="segcfg@example.com", password="password123",
            current_practice=self.practice,
        )
        UserPracticeRelationship.objects.create(
            user=self.user, practice=self.practice, role="admin",
            is_practice_admin=True, practice_verified=True,
        )

    def _get(self, key="dormant"):
        request = self.factory.get(f"/api/backend/dentally/recall-segments/{key}/")
        force_authenticate(request, user=self.user)
        return self.get_view(request, segment_key=key)

    def _put(self, body, key="dormant"):
        request = self.factory.put(
            f"/api/backend/dentally/recall-segments/{key}/", body, format="json"
        )
        force_authenticate(request, user=self.user)
        return self.put_view(request, segment_key=key)

    def test_get_returns_default_when_no_row(self):
        res = self._get()
        self.assertEqual(res.status_code, 200)
        self.assertEqual(res.data["segment_key"], "dormant")
        self.assertTrue(res.data["enabled"])
        self.assertEqual(res.data["params"]["not_seen_months"], 24)

    def test_put_upserts_and_roundtrips(self):
        res = self._put({"enabled": True, "params": {"not_seen_months": 18}})
        self.assertEqual(res.status_code, 200)
        self.assertEqual(res.data["params"]["not_seen_months"], 18)
        again = self._get()
        self.assertEqual(again.data["params"]["not_seen_months"], 18)

    def test_put_rejects_non_positive_months(self):
        res = self._put({"params": {"not_seen_months": 0}})
        self.assertEqual(res.status_code, 400)

    def test_put_rejects_unknown_segment_key(self):
        res = self._put({"params": {"not_seen_months": 12}}, key="banana")
        self.assertEqual(res.status_code, 404)

    def test_practice_isolation(self):
        self._put({"params": {"not_seen_months": 9}})
        other = Practice.objects.create(name="Other Seg Cfg")
        other_user = User.objects.create_user(
            email="otherseg@example.com", password="password123",
            current_practice=other,
        )
        UserPracticeRelationship.objects.create(
            user=other_user, practice=other, role="admin",
            is_practice_admin=True, practice_verified=True,
        )
        request = self.factory.get("/api/backend/dentally/recall-segments/dormant/")
        force_authenticate(request, user=other_user)
        res = self.get_view(request, segment_key="dormant")
        self.assertEqual(res.data["params"]["not_seen_months"], 24)  # default, not 9
```

Add `RecallSegmentConfigViewSet` to the `from .views import (...)` block at the top of `tests.py`.

- [ ] **Step 2: Run the tests to verify they fail**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test dentallyIntegration.tests.RecallSegmentConfigViewSetTests --noinput
```
Expected: ImportError / cannot import `RecallSegmentConfigViewSet`.

- [ ] **Step 3: Implement the viewset**

In `views.py`, add (place directly after the `RecallRecordViewSet` class). The file already imports `viewsets`, `status`, `Response`, `IsAuthenticated`:

```python
ALLOWED_SEGMENT_KEYS = {"dormant"}
DEFAULT_SEGMENT_PARAMS = {"dormant": {"not_seen_months": 24}}


class RecallSegmentConfigViewSet(viewsets.ViewSet):
    """Read/save admin-defined recall segment configuration, per practice."""

    permission_classes = [IsAuthenticated]

    def _practice(self, request):
        return request.user.practice if hasattr(request.user, "practice") else None

    def retrieve(self, request, segment_key=None):
        from .models import RecallSegmentConfig

        if segment_key not in ALLOWED_SEGMENT_KEYS:
            return Response({"error": "Unknown segment"}, status=status.HTTP_404_NOT_FOUND)
        practice = self._practice(request)
        if not practice:
            return Response(
                {"error": "No practice associated with this user"},
                status=status.HTTP_400_BAD_REQUEST,
            )
        cfg = RecallSegmentConfig.objects.filter(
            practice=practice, segment_key=segment_key
        ).first()
        if cfg is None:
            return Response(
                {
                    "segment_key": segment_key,
                    "enabled": True,
                    "params": DEFAULT_SEGMENT_PARAMS[segment_key],
                }
            )
        return Response(
            {"segment_key": cfg.segment_key, "enabled": cfg.enabled, "params": cfg.params}
        )

    def update(self, request, segment_key=None):
        from .models import RecallSegmentConfig

        if segment_key not in ALLOWED_SEGMENT_KEYS:
            return Response({"error": "Unknown segment"}, status=status.HTTP_404_NOT_FOUND)
        practice = self._practice(request)
        if not practice:
            return Response(
                {"error": "No practice associated with this user"},
                status=status.HTTP_400_BAD_REQUEST,
            )

        params = request.data.get("params") or {}
        try:
            not_seen_months = int(params.get("not_seen_months"))
        except (TypeError, ValueError):
            return Response(
                {"error": "params.not_seen_months must be an integer"},
                status=status.HTTP_400_BAD_REQUEST,
            )
        if not_seen_months < 1:
            return Response(
                {"error": "params.not_seen_months must be >= 1"},
                status=status.HTTP_400_BAD_REQUEST,
            )

        enabled = bool(request.data.get("enabled", True))
        cfg, _ = RecallSegmentConfig.objects.update_or_create(
            practice=practice,
            segment_key=segment_key,
            defaults={"enabled": enabled, "params": {"not_seen_months": not_seen_months}},
        )
        return Response(
            {"segment_key": cfg.segment_key, "enabled": cfg.enabled, "params": cfg.params}
        )
```

- [ ] **Step 4: Wire the route**

In `urls.py`, add `RecallSegmentConfigViewSet` to the `from .views import (...)` block, then add this `path` inside `urlpatterns` (place it right after the existing `recalls/` block near line 214):

```python
    # Recall segment configuration (admin-configurable segments)
    path(
        "recall-segments/<str:segment_key>/",
        RecallSegmentConfigViewSet.as_view({"get": "retrieve", "put": "update"}),
        name="dentally-recall-segment-config",
    ),
```

- [ ] **Step 5: Run the tests to verify they pass**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test dentallyIntegration.tests.RecallSegmentConfigViewSetTests --noinput
```
Expected: `Ran 5 tests ... OK`.

- [ ] **Step 6: Checkpoint — hand off for commit**

Suggested message: `feat(recall): add RecallSegmentConfig GET/PUT API`

---

## Task 3: Dormant list endpoint (`dormant` action)

**Files:**
- Modify: `dentallyIntegration/views.py` (add `dormant` method to `RecallRecordViewSet`)
- Modify: `dentallyIntegration/urls.py` (route)
- Test: `dentallyIntegration/tests.py`

- [ ] **Step 1: Write the failing tests**

Append to `tests.py` (reuses `RecallPatient`, `RecallRecord` already imported; add `RecallAppointment` and `RecallSegmentConfig` to the models import block). Also ensure the datetime import at the top of `tests.py` reads `from datetime import date, datetime, timedelta`:

```python
class RecallDormantActionTests(TestCase):
    def setUp(self):
        self.factory = APIRequestFactory()
        self.view = RecallRecordViewSet.as_view({"get": "dormant"})
        self.practice = Practice.objects.create(name="Dormant Test Dental")
        self.user = User.objects.create_user(
            email="dormant@example.com", password="password123",
            current_practice=self.practice,
        )
        UserPracticeRelationship.objects.create(
            user=self.user, practice=self.practice, role="admin",
            is_practice_admin=True, practice_verified=True,
        )

    def _mk(self, dpid, last_completed_dt=None, state="completed"):
        patient = RecallPatient.objects.create(
            practice=self.practice, dentally_patient_id=dpid
        )
        RecallRecord.objects.create(
            practice=self.practice, recall_patient=patient,
            dentally_patient_id=dpid, patient_name=f"P{dpid}",
        )
        if last_completed_dt is not None:
            RecallAppointment.objects.create(
                practice=self.practice, recall_patient=patient,
                dentally_appointment_id=900000 + dpid, dentally_patient_id=dpid,
                start_time=last_completed_dt, state=state,
            )
        return patient

    def _get(self, **params):
        request = self.factory.get("/api/backend/dentally/recalls/dormant/", params)
        force_authenticate(request, user=self.user)
        return self.view(request)

    def test_old_visit_included_recent_excluded_null_included(self):
        old = timezone.now() - timedelta(days=30 * 30)   # ~30 months ago
        recent = timezone.now() - timedelta(days=30)     # ~1 month ago
        self._mk(1, old)        # dormant (older than 24mo)
        self._mk(2, recent)     # not dormant
        self._mk(3, None)       # null last_seen -> dormant
        res = self._get()
        self.assertEqual(res.status_code, 200)
        ids = {r["dentally_patient_id"] for r in res.data["recalls"]}
        self.assertEqual(ids, {1, 3})

    def test_only_completed_state_counts_as_seen(self):
        # A recent *pending* appointment must NOT make a long-absent patient "recent".
        recent_pending = timezone.now() - timedelta(days=10)
        self._mk(1, recent_pending, state="pending")
        res = self._get()
        ids = {r["dentally_patient_id"] for r in res.data["recalls"]}
        self.assertIn(1, ids)  # still dormant: no completed visit

    def test_config_change_shifts_membership(self):
        ten_months = timezone.now() - timedelta(days=30 * 10)
        self._mk(1, ten_months)
        # Default 24mo: patient seen 10mo ago is NOT dormant.
        self.assertEqual(self._get().data["count"], 0)
        # Lower threshold to 6 months: now dormant.
        RecallSegmentConfig.objects.create(
            practice=self.practice, segment_key="dormant",
            params={"not_seen_months": 6},
        )
        self.assertEqual(self._get().data["count"], 1)

    def test_practice_isolation(self):
        old = timezone.now() - timedelta(days=30 * 30)
        self._mk(1, old)
        other = Practice.objects.create(name="Other Dormant")
        other_user = User.objects.create_user(
            email="otherdormant@example.com", password="password123",
            current_practice=other,
        )
        UserPracticeRelationship.objects.create(
            user=other_user, practice=other, role="admin",
            is_practice_admin=True, practice_verified=True,
        )
        request = self.factory.get("/api/backend/dentally/recalls/dormant/")
        force_authenticate(request, user=other_user)
        res = self.view(request)
        self.assertEqual(res.data["count"], 0)
```

- [ ] **Step 2: Run the tests to verify they fail**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test dentallyIntegration.tests.RecallDormantActionTests --noinput
```
Expected: FAIL — `dormant` is not a valid action / AttributeError.

- [ ] **Step 3: Implement the `dormant` method**

Add this method inside the `RecallRecordViewSet` class in `views.py` (after `list`):

```python
    def dormant(self, request):
        from datetime import date

        from dateutil.relativedelta import relativedelta
        from django.db.models import F, OuterRef, Q, Subquery

        from .models import RecallAppointment, RecallRecord, RecallSegmentConfig
        from .serializers import RecallRecordSerializer

        practice = request.user.practice if hasattr(request.user, "practice") else None
        if not practice:
            return Response(
                {"error": "No practice associated with this user"},
                status=status.HTTP_400_BAD_REQUEST,
            )

        # Saved dormant threshold (fallback default 24 months).
        not_seen_months = 24
        cfg = RecallSegmentConfig.objects.filter(
            practice=practice, segment_key="dormant"
        ).first()
        if cfg and isinstance(cfg.params, dict):
            try:
                not_seen_months = int(cfg.params.get("not_seen_months", 24))
            except (TypeError, ValueError):
                not_seen_months = 24
        if not_seen_months < 1:
            not_seen_months = 24

        cutoff = date.today() - relativedelta(months=not_seen_months)

        last_completed = (
            RecallAppointment.objects.filter(
                practice=practice,
                dentally_patient_id=OuterRef("dentally_patient_id"),
                state="completed",
            )
            .order_by("-start_time")
            .values("start_time")[:1]
        )

        queryset = (
            RecallRecord.objects.filter(practice=practice)
            .annotate(last_seen=Subquery(last_completed))
            .filter(Q(last_seen__isnull=True) | Q(last_seen__date__lt=cutoff))
        )

        search = (request.query_params.get("search") or "").strip()
        if search:
            queryset = queryset.filter(patient_name__icontains=search)

        # Most dormant first: nulls (never seen), then oldest visits.
        queryset = queryset.order_by(F("last_seen").asc(nulls_first=True))

        page = int(request.query_params.get("page", 1))
        page_size = min(int(request.query_params.get("page_size", 50)), 100)
        total = queryset.count()
        start = (page - 1) * page_size
        recalls = queryset[start : start + page_size]

        data = RecallRecordSerializer(recalls, many=True).data
        return Response(
            {
                "count": total,
                "total_pages": (total + page_size - 1) // page_size,
                "page": page,
                "page_size": page_size,
                "recalls": data,
            }
        )
```

- [ ] **Step 4: Wire the route**

In `urls.py`, add this `path` immediately **before** the existing `recalls/` path (both are exact matches, but keep the more specific one first for clarity):

```python
    path(
        "recalls/dormant/",
        RecallRecordViewSet.as_view({"get": "dormant"}),
        name="dentally-recalls-dormant",
    ),
```

- [ ] **Step 5: Run the tests to verify they pass**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test dentallyIntegration.tests.RecallDormantActionTests --noinput
```
Expected: `Ran 4 tests ... OK`.

- [ ] **Step 6: Run the whole dentallyIntegration suite (no regressions)**

Run:
```bash
python manage.py test dentallyIntegration --noinput
```
Expected: all prior recall tests still pass; only the pre-existing `test_returns_signed_url` failure (unrelated URL-shortener assertion) remains.

- [ ] **Step 7: Checkpoint — hand off for commit**

Suggested message: `feat(recall): add live dormant patient list endpoint`

---

## Task 4: Frontend — config endpoints + Dormant card wiring

**Files:**
- Modify: `perfect-pixel-playground-project/src/config/api.ts:426` (DENTALLY block)
- Create: `perfect-pixel-playground-project/src/pages/recalls/useSegmentConfig.ts`
- Modify: `perfect-pixel-playground-project/src/pages/recalls/components/administration/SegmentsSection.tsx`
- Test: `perfect-pixel-playground-project/src/pages/recalls/useSegmentConfig.test.ts`

- [ ] **Step 1: Add API endpoints**

In `src/config/api.ts`, inside the `DENTALLY` block, add after the `RECALLS_CANCEL_SYNC` line (≈line 430):

```ts
    RECALL_SEGMENT: (key: string) => getApiUrl(`/dentally/recall-segments/${key}/`),
    RECALLS_DORMANT: (params?: string) => getApiUrl(params ? `/dentally/recalls/dormant/?${params}` : '/dentally/recalls/dormant/'),
```

- [ ] **Step 2: Write the failing hook test**

Create `src/pages/recalls/useSegmentConfig.test.ts`:

```ts
import { renderHook, act, waitFor } from '@testing-library/react';
import { describe, expect, it, vi, beforeEach } from 'vitest';

const fetchWithAuth = vi.fn();
vi.mock('@/lib/helpers', () => ({ useFetchWithAuth: () => fetchWithAuth }));

import { useSegmentConfig } from './useSegmentConfig';

function jsonResponse(body: unknown, ok = true) {
  return { ok, json: async () => body } as Response;
}

describe('useSegmentConfig', () => {
  beforeEach(() => fetchWithAuth.mockReset());

  it('GETs the config on mount', async () => {
    fetchWithAuth.mockResolvedValueOnce(
      jsonResponse({ segment_key: 'dormant', enabled: true, params: { not_seen_months: 18 } }),
    );
    const { result } = renderHook(() => useSegmentConfig('dormant'));
    await waitFor(() => expect(result.current.loading).toBe(false));
    expect(result.current.config?.params.not_seen_months).toBe(18);
    expect(fetchWithAuth).toHaveBeenCalledTimes(1);
  });

  it('save() issues a PUT with the new params', async () => {
    fetchWithAuth
      .mockResolvedValueOnce(jsonResponse({ segment_key: 'dormant', enabled: true, params: { not_seen_months: 24 } }))
      .mockResolvedValueOnce(jsonResponse({ segment_key: 'dormant', enabled: true, params: { not_seen_months: 12 } }));
    const { result } = renderHook(() => useSegmentConfig('dormant'));
    await waitFor(() => expect(result.current.loading).toBe(false));
    await act(async () => {
      await result.current.save({ enabled: true, params: { not_seen_months: 12 } });
    });
    const putCall = fetchWithAuth.mock.calls[1];
    expect(putCall[1]).toMatchObject({ method: 'PUT' });
    expect(JSON.parse(putCall[1].body)).toEqual({ enabled: true, params: { not_seen_months: 12 } });
    expect(result.current.config?.params.not_seen_months).toBe(12);
  });
});
```

- [ ] **Step 3: Run the test to verify it fails**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vitest run src/pages/recalls/useSegmentConfig.test.ts
```
Expected: FAIL — cannot resolve `./useSegmentConfig`.

- [ ] **Step 4: Implement the hook**

Create `src/pages/recalls/useSegmentConfig.ts`:

```ts
import { useCallback, useEffect, useState } from 'react';
import { API_ENDPOINTS } from '@/config/api';
import { useFetchWithAuth } from '@/lib/helpers';

export interface SegmentConfig {
  segment_key: string;
  enabled: boolean;
  params: { not_seen_months?: number } & Record<string, unknown>;
}

export interface SaveSegmentConfig {
  enabled?: boolean;
  params: Record<string, unknown>;
}

export function useSegmentConfig(segmentKey: string) {
  const fetchWithAuth = useFetchWithAuth();
  const [config, setConfig] = useState<SegmentConfig | null>(null);
  const [loading, setLoading] = useState(true);

  const load = useCallback(async () => {
    setLoading(true);
    try {
      const res = await fetchWithAuth(API_ENDPOINTS.DENTALLY.RECALL_SEGMENT(segmentKey));
      if (res.ok) setConfig((await res.json()) as SegmentConfig);
    } finally {
      setLoading(false);
    }
  }, [fetchWithAuth, segmentKey]);

  useEffect(() => {
    load();
  }, [load]);

  const save = useCallback(
    async (next: SaveSegmentConfig): Promise<boolean> => {
      const res = await fetchWithAuth(API_ENDPOINTS.DENTALLY.RECALL_SEGMENT(segmentKey), {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(next),
      });
      if (res.ok) setConfig((await res.json()) as SegmentConfig);
      return res.ok;
    },
    [fetchWithAuth, segmentKey],
  );

  return { config, loading, save, reload: load };
}
```

- [ ] **Step 5: Run the test to verify it passes**

Run:
```bash
npx vitest run src/pages/recalls/useSegmentConfig.test.ts
```
Expected: `2 passed`.

- [ ] **Step 6: Wire the Dormant card in `SegmentsSection.tsx`**

Make three edits to `src/pages/recalls/components/administration/SegmentsSection.tsx`:

(a) Add imports near the top (after the existing imports):
```ts
import { useEffect } from 'react';
import { useSegmentConfig } from '../../useSegmentConfig';
```
(Merge `useEffect` into the existing `import { useState, KeyboardEvent, ReactNode } from 'react';` line instead, so it reads `import { useState, useEffect, KeyboardEvent, ReactNode } from 'react';`.)

(b) Inside `export function SegmentsSection()`, after the existing `const [config, setConfig] = useState<SegmentConfig>(DEFAULT_CONFIG);`, add the hook and a sync effect:
```ts
  const { config: dormantCfg, save: saveDormant } = useSegmentConfig('dormant');

  useEffect(() => {
    const months = dormantCfg?.params?.not_seen_months;
    if (months != null) {
      setConfig((c) => ({ ...c, dormant: { ...c.dormant, notSeenMonths: String(months) } }));
    }
  }, [dormantCfg]);
```

(c) Replace the `onSave` handler on the `<AdminSection>` (currently `onSave={() => toast.success('Attendance segments saved')}`) with:
```ts
      onSave={async () => {
        const months = Number(config.dormant.notSeenMonths);
        if (!Number.isInteger(months) || months < 1) {
          toast.error('Dormant threshold must be a whole number of months (1 or more)');
          return;
        }
        const ok = await saveDormant({
          enabled: config.dormant.enabled,
          params: { not_seen_months: months },
        });
        toast[ok ? 'success' : 'error'](
          ok ? 'Attendance segments saved' : 'Could not save the Dormant rule',
        );
      }}
```

- [ ] **Step 7: Type-check the wiring**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx tsc --noEmit -p tsconfig.json
```
Expected: no new errors in `SegmentsSection.tsx` / `useSegmentConfig.ts`.
(If the project has no clean `tsc` baseline, instead run `npx vitest run src/pages/recalls/` and confirm no transform/type errors are reported for these files.)

- [ ] **Step 8: Checkpoint — hand off for commit**

Suggested message: `feat(recall-ui): persist Dormant segment config from Segments tab`

---

## Task 5: Frontend — Dormant tab rendering

**Files:**
- Create: `perfect-pixel-playground-project/src/pages/recalls/useDormantRecalls.ts`
- Create: `perfect-pixel-playground-project/src/pages/recalls/components/DormantTab.tsx`
- Modify: `perfect-pixel-playground-project/src/pages/recalls/RecallsPage.tsx`
- Test: `perfect-pixel-playground-project/src/pages/recalls/useDormantRecalls.test.ts`

- [ ] **Step 1: Write the failing hook test**

Create `src/pages/recalls/useDormantRecalls.test.ts`:

```ts
import { renderHook, waitFor } from '@testing-library/react';
import { describe, expect, it, vi, beforeEach } from 'vitest';

const fetchWithAuth = vi.fn();
vi.mock('@/lib/helpers', () => ({ useFetchWithAuth: () => fetchWithAuth }));

import { useDormantRecalls } from './useDormantRecalls';

function jsonResponse(body: unknown, ok = true) {
  return { ok, json: async () => body } as Response;
}

describe('useDormantRecalls', () => {
  beforeEach(() => fetchWithAuth.mockReset());

  it('fetches the dormant list and maps the envelope', async () => {
    fetchWithAuth.mockResolvedValueOnce(
      jsonResponse({ count: 2, total_pages: 1, page: 1, page_size: 50,
        recalls: [{ id: 'a', dentally_patient_id: 1, patient_name: 'A' }] }),
    );
    const { result } = renderHook(() => useDormantRecalls({ page: 1, pageSize: 50, search: '' }));
    await waitFor(() => expect(result.current.loading).toBe(false));
    expect(result.current.totalCount).toBe(2);
    expect(result.current.recalls).toHaveLength(1);
    const url = fetchWithAuth.mock.calls[0][0] as string;
    expect(url).toContain('/dentally/recalls/dormant/');
  });

  it('sets an error message on a failed response', async () => {
    fetchWithAuth.mockResolvedValueOnce(jsonResponse({ error: 'boom' }, false));
    const { result } = renderHook(() => useDormantRecalls({ page: 1, pageSize: 50, search: '' }));
    await waitFor(() => expect(result.current.loading).toBe(false));
    expect(result.current.error).toBe('boom');
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vitest run src/pages/recalls/useDormantRecalls.test.ts
```
Expected: FAIL — cannot resolve `./useDormantRecalls`.

- [ ] **Step 3: Implement the hook**

Create `src/pages/recalls/useDormantRecalls.ts`:

```ts
import { useCallback, useEffect, useState } from 'react';
import { API_ENDPOINTS } from '@/config/api';
import { useFetchWithAuth } from '@/lib/helpers';
import type { Recall, RecallsResponse } from './types';

interface UseDormantRecallsOptions {
  page: number;
  pageSize: number;
  search: string;
}

export function useDormantRecalls({ page, pageSize, search }: UseDormantRecallsOptions) {
  const fetchWithAuth = useFetchWithAuth();
  const [recalls, setRecalls] = useState<Recall[]>([]);
  const [totalCount, setTotalCount] = useState(0);
  const [totalPages, setTotalPages] = useState(0);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const load = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const params = new URLSearchParams();
      params.set('page', String(page));
      params.set('page_size', String(pageSize));
      if (search) params.set('search', search);
      const res = await fetchWithAuth(API_ENDPOINTS.DENTALLY.RECALLS_DORMANT(params.toString()));
      if (res.ok) {
        const data: RecallsResponse = await res.json();
        setRecalls(data.recalls);
        setTotalCount(data.count);
        setTotalPages(data.total_pages);
      } else {
        const e = await res.json();
        setError(e.error || 'Failed to load dormant patients');
      }
    } catch {
      setError('Network error. Please try again.');
    } finally {
      setLoading(false);
    }
  }, [fetchWithAuth, page, pageSize, search]);

  useEffect(() => {
    load();
  }, [load]);

  return { recalls, totalCount, totalPages, loading, error, reload: load };
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run:
```bash
npx vitest run src/pages/recalls/useDormantRecalls.test.ts
```
Expected: `2 passed`.

- [ ] **Step 5: Create the `DormantTab` component**

Create `src/pages/recalls/components/DormantTab.tsx`:

```tsx
import { useState } from 'react';
import { useDormantRecalls } from '../useDormantRecalls';
import { RecallsTable } from './RecallsTable';
import type { Recall } from '../types';

interface DormantTabProps {
  onOpenPatientPanel: (recall: Recall) => void;
}

export function DormantTab({ onOpenPatientPanel }: DormantTabProps) {
  const [page, setPage] = useState(1);
  const pageSize = 50;
  const { recalls, totalCount, totalPages, loading, error, reload } = useDormantRecalls({
    page,
    pageSize,
    search: '',
  });

  return (
    <div className="bg-[#faf9fe] p-6 flex-1 overflow-y-auto overflow-x-hidden max-w-[100vw]">
      <div className="w-full max-w-full h-full flex flex-col min-w-0">
        <RecallsTable
          recalls={recalls}
          totalCount={totalCount}
          totalPages={totalPages}
          page={page}
          pageSize={pageSize}
          loading={loading}
          error={error}
          sortBy=""
          sortDir="desc"
          onSort={() => {}}
          onPageChange={setPage}
          onRetry={reload}
          onOpenPatientPanel={onOpenPatientPanel}
        />
      </div>
    </div>
  );
}
```

- [ ] **Step 6: Branch the Dormant tab in `RecallsPage.tsx`**

(a) Add the import after the `RecallsAdministration` import (line 9):
```ts
import { DormantTab } from './components/DormantTab';
```

(b) Change the tab branch. Replace the opening of the conditional at line 151 (`{activeTab === 'administration' ? (`) so the render reads:
```tsx
      {activeTab === 'administration' ? (
        <RecallsAdministration />
      ) : activeTab === 'dormant' ? (
        <DormantTab onOpenPatientPanel={openPatientPanel} />
      ) : (
```
(The existing `<RecallsAdministration />`, the default recalls block, and the closing `)}` stay as-is — this only inserts the `dormant` branch.)

- [ ] **Step 7: Run the recalls frontend tests (no regressions)**

Run:
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vitest run src/pages/recalls/
```
Expected: the existing `RecallsTable.test.tsx` (10) plus the two new hook test files all pass.

- [ ] **Step 8: Checkpoint — hand off for commit**

Suggested message: `feat(recall-ui): wire Dormant tab to live dormant list`

---

## Final verification (after all tasks)

- [ ] **Backend full recall surface:**
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test dentallyIntegration --noinput
```
Expected: all recall tests pass (pre-existing `test_returns_signed_url` failure aside).

- [ ] **Frontend recalls:**
```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vitest run src/pages/recalls/
```
Expected: all pass.

- [ ] **Manual smoke (optional, real app):** open Recalls → Administration → Segments → set Dormant to e.g. 12 → Save; open the Dormant Patients tab and confirm the list reflects the threshold.

---

## Spec coverage check
- Data model (spec §4) → Task 1.
- Config API GET/PUT + validation + isolation (spec §5) → Task 2.
- Live dormant evaluation, null=Dormant, completed-only, pagination/search reuse (spec §6) → Task 3.
- SegmentsSection Dormant card load/save (spec §7) → Task 4.
- Dormant tab rendering via RecallsTable (spec §7) → Task 5.
- Tests (spec §8) → Tasks 2, 3, 4, 5.
- Out-of-scope items (spec §2) → not implemented, by design.

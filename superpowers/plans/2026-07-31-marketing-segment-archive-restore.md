# Marketing Segment Archive/Restore Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let staff see and restore archived marketing segments — today archiving a segment is a one-way trip with no way to view or undo it anywhere in the product.

**Architecture:** One backend fix (detail-level segment lookups currently 404 on archived rows — blocks restore working at all) plus one new backend action (`restore`), and a new frontend page + nav entry that lists archived segments with a restore button. No new models.

**Tech Stack:** Django REST Framework (backend), React + TypeScript + Vitest (frontend).

**Spec:** [`PRD/Broadcasts/specs/2026-07-31-marketing-segment-archive-restore-design.md`](../../../PRD/Broadcasts/specs/2026-07-31-marketing-segment-archive-restore-design.md)

---

### Task 1: Backend — fix detail-lookup filtering + add restore action

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/views/segment_views.py`
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/urls.py`
- Modify: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/tests.py`

- [ ] **Step 1: Write the failing tests**

In `tests.py`, find `test_delete_archives_instead_of_hard_deleting` (inside `MarketingSegmentAPITests`, currently uses the old `?include_archived=true` param) and replace it with the updated version below — the param is being renamed with different semantics (`archived=true` now means "list ONLY archived," not "include archived alongside active"):

```python
    def test_delete_archives_instead_of_hard_deleting(self):
        segment = MarketingSegment.objects.create(
            practice=self.practice, name="To archive", created_by=self.user
        )
        response = self.client.delete(f"/api/backend/marketing/segments/{segment.id}/")
        self.assertEqual(response.status_code, 204)
        segment.refresh_from_db()
        self.assertTrue(segment.is_archived)
        self.assertTrue(MarketingSegment.objects.filter(id=segment.id).exists())

        list_response = self.client.get("/api/backend/marketing/segments/")
        self.assertEqual(len(list_response.data), 0)  # excluded by default
        archived_response = self.client.get(
            "/api/backend/marketing/segments/?archived=true"
        )
        self.assertEqual(len(archived_response.data), 1)
```

Then add these new tests to the end of `MarketingSegmentAPITests` — insert them right after `test_preview_paginates_the_full_matching_list` (the last method in the class, ending just before the blank lines and `class BroadcastCampaignAPITests(TestCase):`):

```python
    def test_archived_list_excludes_active_segments(self):
        active = MarketingSegment.objects.create(
            practice=self.practice, name="Active", created_by=self.user
        )
        archived = MarketingSegment.objects.create(
            practice=self.practice, name="Archived", is_archived=True, created_by=self.user
        )
        response = self.client.get("/api/backend/marketing/segments/?archived=true")
        self.assertEqual(response.status_code, 200)
        ids = {row["id"] for row in response.data}
        self.assertEqual(ids, {archived.id})
        self.assertNotIn(active.id, ids)

    def test_retrieve_works_on_archived_segment(self):
        segment = MarketingSegment.objects.create(
            practice=self.practice, name="Archived", is_archived=True, created_by=self.user
        )
        response = self.client.get(f"/api/backend/marketing/segments/{segment.id}/")
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.data["id"], segment.id)

    def test_restore_unarchives_segment(self):
        segment = MarketingSegment.objects.create(
            practice=self.practice, name="Archived", is_archived=True, created_by=self.user
        )
        response = self.client.post(
            f"/api/backend/marketing/segments/{segment.id}/restore/"
        )
        self.assertEqual(response.status_code, 200)
        self.assertFalse(response.data["is_archived"])
        segment.refresh_from_db()
        self.assertFalse(segment.is_archived)

        list_response = self.client.get("/api/backend/marketing/segments/")
        self.assertEqual(len(list_response.data), 1)
        self.assertEqual(list_response.data[0]["id"], segment.id)

    def test_restore_is_idempotent_on_already_active_segment(self):
        segment = MarketingSegment.objects.create(
            practice=self.practice, name="Already active", created_by=self.user
        )
        response = self.client.post(
            f"/api/backend/marketing/segments/{segment.id}/restore/"
        )
        self.assertEqual(response.status_code, 200)
        self.assertFalse(response.data["is_archived"])

    def test_restore_scoped_to_practice(self):
        other_practice = Practice.objects.create(name="Other Restore Practice")
        other_segment = MarketingSegment.objects.create(
            practice=other_practice,
            name="Not mine",
            is_archived=True,
            created_by=User.objects.create_user(
                email="otherrestorepractice@example.com", password="password123"
            ),
        )
        response = self.client.post(
            f"/api/backend/marketing/segments/{other_segment.id}/restore/"
        )
        self.assertEqual(response.status_code, 404)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test marketingBroadcast.tests.MarketingSegmentAPITests --keepdb -v 2
```
Expected: `test_delete_archives_instead_of_hard_deleting` FAILs (old param semantics gone), the 5 new tests FAIL (retrieve/restore 404 — `get_queryset` still filters archived rows out of detail lookups; `restore` URL doesn't exist yet).

- [ ] **Step 3: Fix `get_queryset` and add the `restore` action**

In `segment_views.py`, replace:

```python
    def get_queryset(self):
        practice = self.get_user_practice()
        queryset = MarketingSegment.objects.filter(practice=practice)
        if self.request.query_params.get("include_archived") != "true":
            queryset = queryset.filter(is_archived=False)
        return queryset.order_by("-created_at")
```

with:

```python
    def get_queryset(self):
        practice = self.get_user_practice()
        queryset = MarketingSegment.objects.filter(practice=practice)
        if self.action == "list":
            show_archived = self.request.query_params.get("archived") == "true"
            queryset = queryset.filter(is_archived=show_archived)
        return queryset.order_by("-created_at")
```

Then add the `restore` action right after `perform_destroy`:

```python
    @action(detail=True, methods=["POST"])
    def restore(self, request, pk=None):
        segment = self.get_object()
        segment.is_archived = False
        segment.save(update_fields=["is_archived"])
        return Response(MarketingSegmentSerializer(segment).data)
```

- [ ] **Step 4: Wire the URL**

In `urls.py`, add this path right after the `segments/<int:pk>/preview/` entry:

```python
    path(
        "segments/<int:pk>/restore/",
        MarketingSegmentViewSet.as_view({"post": "restore"}),
        name="marketing-segment-restore",
    ),
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
python manage.py test marketingBroadcast.tests.MarketingSegmentAPITests --keepdb -v 2
```
Expected: `OK`.

- [ ] **Step 6: Run the full marketingBroadcast suite to check for regressions**

```bash
python manage.py test marketingBroadcast --keepdb
```
Expected: `OK` — pay particular attention to any other test that relied on `get_queryset` filtering non-list actions (e.g. `preview`, `partial_update`, `destroy` tests) still passing, since the filter now only applies to `list`.

- [ ] **Step 7: Commit**

```bash
git add TreatmentPathBackend/TreatmentPath/marketingBroadcast/views/segment_views.py TreatmentPathBackend/TreatmentPath/marketingBroadcast/urls.py TreatmentPathBackend/TreatmentPath/marketingBroadcast/tests.py
git commit -m "feat(marketing): add segment restore action, fix archived-segment detail lookups"
```

---

### Task 2: Frontend — API endpoints + archived segments page

**Files:**
- Modify: `perfect-pixel-playground-project/src/config/api.ts`
- Create: `perfect-pixel-playground-project/src/pages/MarketingArchivedSegments.tsx`
- Test: `perfect-pixel-playground-project/src/pages/MarketingArchivedSegments.test.tsx` (new)
- Modify: `perfect-pixel-playground-project/src/App.tsx`
- Modify: `perfect-pixel-playground-project/src/components/AppSidebarV2.tsx`

- [ ] **Step 1: Add the API endpoint builders**

In `api.ts`, find the `segments` block inside `marketing`:

```typescript
    segments: {
      list: () => getApiUrl('/marketing/segments/'),
      previewDraft: () => getApiUrl('/marketing/segments/preview/'),
      detail: (id: number | string) => getApiUrl(`/marketing/segments/${id}/`),
      preview: (id: number | string, page = 1, pageSize = 50) =>
        getApiUrl(`/marketing/segments/${id}/preview/?page=${page}&page_size=${pageSize}`),
    },
```

Replace it with:

```typescript
    segments: {
      list: () => getApiUrl('/marketing/segments/'),
      archivedList: () => getApiUrl('/marketing/segments/?archived=true'),
      previewDraft: () => getApiUrl('/marketing/segments/preview/'),
      detail: (id: number | string) => getApiUrl(`/marketing/segments/${id}/`),
      preview: (id: number | string, page = 1, pageSize = 50) =>
        getApiUrl(`/marketing/segments/${id}/preview/?page=${page}&page_size=${pageSize}`),
      restore: (id: number | string) => getApiUrl(`/marketing/segments/${id}/restore/`),
    },
```

- [ ] **Step 2: Write the failing test**

Create `MarketingArchivedSegments.test.tsx`:

```typescript
import { MemoryRouter } from "react-router-dom";
import { afterEach, describe, expect, it, vi } from "vitest";
import { fireEvent, render, screen, waitFor } from "@testing-library/react";

vi.mock("@/lib/helpers", () => ({
  useFetchWithAuth: () => mockFetchWithAuth,
}));

const mockFetchWithAuth = vi.fn();

import MarketingArchivedSegments from "./MarketingArchivedSegments";

function jsonResponse(body: unknown, ok = true) {
  return Promise.resolve({
    ok,
    json: () => Promise.resolve(body),
  } as Response);
}

function renderPage() {
  return render(
    <MemoryRouter>
      <MarketingArchivedSegments />
    </MemoryRouter>,
  );
}

describe("MarketingArchivedSegments", () => {
  afterEach(() => {
    vi.restoreAllMocks();
    mockFetchWithAuth.mockReset();
  });

  it("lists archived segments", async () => {
    mockFetchWithAuth.mockResolvedValue(
      jsonResponse([
        {
          id: 1,
          name: "Lapsed patients",
          match_mode: "all",
          rule_groups: [],
          is_archived: true,
          created_at: "2026-07-01T00:00:00Z",
          last_resolved_count: 42,
          last_resolved_at: null,
        },
      ]),
    );

    renderPage();

    expect(await screen.findByText("Lapsed patients")).toBeInTheDocument();
    expect(screen.getByText("42")).toBeInTheDocument();
  });

  it("shows an empty state when there are no archived segments", async () => {
    mockFetchWithAuth.mockResolvedValue(jsonResponse([]));

    renderPage();

    expect(await screen.findByText(/no archived segments/i)).toBeInTheDocument();
  });

  it("restores a segment and removes it from the list", async () => {
    mockFetchWithAuth.mockImplementation((url: string, init?: RequestInit) => {
      if (init?.method === "POST") {
        return jsonResponse({ id: 1, is_archived: false });
      }
      return jsonResponse([
        {
          id: 1,
          name: "Lapsed patients",
          match_mode: "all",
          rule_groups: [],
          is_archived: true,
          created_at: "2026-07-01T00:00:00Z",
          last_resolved_count: null,
          last_resolved_at: null,
        },
      ]);
    });

    renderPage();

    const restoreButton = await screen.findByRole("button", { name: /restore/i });
    fireEvent.click(restoreButton);

    await waitFor(() =>
      expect(mockFetchWithAuth).toHaveBeenCalledWith(
        expect.stringContaining("/segments/1/restore/"),
        expect.objectContaining({ method: "POST" }),
      ),
    );
    await waitFor(() => expect(screen.queryByText("Lapsed patients")).not.toBeInTheDocument());
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vitest run src/pages/MarketingArchivedSegments.test.tsx
```
Expected: FAIL — module doesn't exist yet.

- [ ] **Step 4: Create the page component**

Create `MarketingArchivedSegments.tsx`. This mirrors `MarketingSegments.tsx`'s table chrome (same header/card/table classes) but is simpler — no create dialog, no rule builder, one action per row:

```typescript
import { useCallback, useEffect, useState } from "react";
import { ArchiveRestore, Loader2 } from "lucide-react";
import { toast } from "sonner";
import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";
import { useFetchWithAuth } from "@/lib/helpers";
import { API_ENDPOINTS } from "@/config/api";
import type { RuleClause } from "@/components/marketing/segmentFields";
import { fieldDef } from "@/components/marketing/segmentFields";

interface MarketingSegment {
  id: number;
  name: string;
  match_mode: "all" | "any";
  rule_groups: RuleClause[];
  is_archived: boolean;
  created_at: string;
  last_resolved_count: number | null;
  last_resolved_at: string | null;
}

export default function MarketingArchivedSegments() {
  const fetchWithAuth = useFetchWithAuth();
  const [segments, setSegments] = useState<MarketingSegment[]>([]);
  const [loading, setLoading] = useState(true);
  const [restoringId, setRestoringId] = useState<number | null>(null);

  const loadArchivedSegments = useCallback(async () => {
    setLoading(true);
    try {
      const res = await fetchWithAuth(API_ENDPOINTS.marketing.segments.archivedList());
      if (!res.ok) throw new Error("Failed to load archived segments");
      const data = await res.json();
      setSegments(data);
    } catch {
      toast.error("Failed to load archived segments");
    } finally {
      setLoading(false);
    }
  }, [fetchWithAuth]);

  useEffect(() => {
    loadArchivedSegments();
  }, [loadArchivedSegments]);

  const handleRestore = async (id: number) => {
    setRestoringId(id);
    try {
      const res = await fetchWithAuth(API_ENDPOINTS.marketing.segments.restore(id), {
        method: "POST",
      });
      if (!res.ok) throw new Error("Failed to restore segment");
      toast.success("Segment restored");
      setSegments((prev) => prev.filter((s) => s.id !== id));
    } catch {
      toast.error("Failed to restore segment");
    } finally {
      setRestoringId(null);
    }
  };

  return (
    <div className="flex flex-col h-full w-full bg-background relative overflow-hidden">
      <div className="border-b border-border bg-card/50 backdrop-blur-sm sticky top-0 z-20">
        <div className="flex items-center justify-between px-6 py-4 max-w-full !h-16">
          <div className="flex items-center min-w-0 flex-1">
            <h1 className="text-lg font-semibold text-foreground">Archived Segments</h1>
          </div>
        </div>
      </div>

      <div className="bg-[#faf9fe] p-6 flex-1 overflow-y-auto overflow-x-hidden max-w-[100vw]">
        <div className="bg-white rounded-xl border border-gray-200 shadow-sm">
          <div className="border-b border-gray-200">
            <div className="px-6 py-4">
              <div className="flex items-center gap-2">
                <h2 className="text-lg font-semibold text-gray-900">Archived Segments</h2>
                <div className="bg-[#f9f5ff] flex items-center px-2 py-0.5 rounded-full border border-[#e9d7fe]">
                  <span className="font-medium text-[#6941c6] text-xs">{segments.length}</span>
                </div>
              </div>
              <p className="text-sm text-gray-500 mt-0.5">
                Segments hidden from new campaigns. Restore one to use it again.
              </p>
            </div>
          </div>

          <div className="overflow-x-auto">
            <table className="w-full min-w-full">
              <thead className="bg-[#FAF9FE] border-b border-gray-200">
                <tr>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Name</th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Rules</th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Last count</th>
                  <th className="w-[100px] min-w-[100px] px-2 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
                </tr>
              </thead>
              <tbody className="bg-white divide-y divide-gray-200">
                {loading ? (
                  <tr>
                    <td colSpan={4} className="px-6 py-12 text-center text-sm text-gray-400">
                      <Loader2 className="w-4 h-4 animate-spin inline mr-2" />
                      Loading...
                    </td>
                  </tr>
                ) : segments.length === 0 ? (
                  <tr>
                    <td colSpan={4} className="px-6 py-12 text-center text-sm text-gray-400">
                      No archived segments
                    </td>
                  </tr>
                ) : (
                  segments.map((segment) => (
                    <tr key={segment.id} className="hover:bg-gray-50">
                      <td className="px-6 py-4 text-sm font-medium text-gray-900">{segment.name}</td>
                      <td className="px-6 py-4">
                        <div className="flex flex-wrap gap-1">
                          {segment.rule_groups.length === 0 ? (
                            <span className="text-xs text-gray-400">All eligible patients</span>
                          ) : (
                            segment.rule_groups.map((clause, i) => (
                              <Badge key={i} variant="outline" className="text-xs font-normal">
                                {fieldDef(clause.field)?.label ?? clause.field}
                              </Badge>
                            ))
                          )}
                        </div>
                      </td>
                      <td className="px-6 py-4 text-sm text-gray-600">{segment.last_resolved_count ?? "—"}</td>
                      <td className="w-[100px] min-w-[100px] px-2 py-4 text-right">
                        <Button
                          size="sm"
                          variant="ghost"
                          onClick={() => handleRestore(segment.id)}
                          disabled={restoringId === segment.id}
                          className="text-xs text-primary hover:bg-primary/5 gap-1.5"
                        >
                          {restoringId === segment.id ? (
                            <Loader2 className="w-3.5 h-3.5 animate-spin" />
                          ) : (
                            <ArchiveRestore className="w-3.5 h-3.5" />
                          )}
                          Restore
                        </Button>
                      </td>
                    </tr>
                  ))
                )}
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
npx vitest run src/pages/MarketingArchivedSegments.test.tsx
```
Expected: PASS (3 tests).

- [ ] **Step 6: Add the route**

In `App.tsx`, add the import near the other Marketing imports:

```typescript
import MarketingArchivedSegments from "./pages/MarketingArchivedSegments";
```

(Place it right after `import MarketingSegmentPreview from "./pages/MarketingSegmentPreview";`.)

Add the route right after the existing `/marketing/segments` route (before `/marketing/segments/:id/preview`):

```tsx
                  <Route
                    path="/marketing/segments/archived"
                    element={
                      <ProtectedRouteWithContexts>
                        <FeatureRoute feature="marketing_broadcast" module="marketing_broadcast">
                          <Layout>
                            <MarketingArchivedSegments />
                          </Layout>
                        </FeatureRoute>
                      </ProtectedRouteWithContexts>
                    }
                  />
```

- [ ] **Step 7: Add the nav item**

In `AppSidebarV2.tsx`, find the marketing section's items array:

```typescript
    items: [
      { title: "Campaigns", url: "/marketing", icon: Megaphone },
      { title: "Segments", url: "/marketing/segments", icon: Filter },
    ],
```

Replace it with:

```typescript
    items: [
      { title: "Campaigns", url: "/marketing", icon: Megaphone },
      { title: "Segments", url: "/marketing/segments", icon: Filter },
      { title: "Archived", url: "/marketing/segments/archived", icon: Archive },
    ],
```

(`Archive` is already imported in this file — used elsewhere for the Clinical section's "Drafts" item — no new import needed.)

Then find the visibility filter for the marketing section:

```typescript
      if (section.id === "marketing") {
        return {
          ...section,
          items: filterVisibleItems(
            section.items,
            (item) =>
              ["Campaigns", "Segments"].includes(item.title) &&
              hasModule("marketing_broadcast") &&
              hasFeature("marketing_broadcast")
          ),
        };
      }
```

Change the title list to include the new item — otherwise it's silently filtered out regardless of feature access:

```typescript
      if (section.id === "marketing") {
        return {
          ...section,
          items: filterVisibleItems(
            section.items,
            (item) =>
              ["Campaigns", "Segments", "Archived"].includes(item.title) &&
              hasModule("marketing_broadcast") &&
              hasFeature("marketing_broadcast")
          ),
        };
      }
```

- [ ] **Step 8: Typecheck and lint**

```bash
npx eslint src/pages/MarketingArchivedSegments.tsx src/config/api.ts src/App.tsx src/components/AppSidebarV2.tsx
npm run typecheck 2>&1 | grep -iE "MarketingArchivedSegments|config/api.ts|App.tsx|AppSidebarV2"
```
Expected: 0 new eslint problems in the new file (the other three files may show their existing pre-existing errors — confirm any hits are ones that existed before this change, not new ones introduced by it).

- [ ] **Step 9: Commit**

```bash
git add src/config/api.ts src/pages/MarketingArchivedSegments.tsx src/pages/MarketingArchivedSegments.test.tsx src/App.tsx src/components/AppSidebarV2.tsx
git commit -m "feat(marketing): add archived segments view with restore action"
```

---

### Task 3: Manual verification

- [ ] **Step 1: Verify the nav item and empty state**

With both dev servers running, visit `/marketing/segments/archived` (or click the new "Archived" item in the Marketing nav). Confirm it loads and shows "No archived segments" if none exist yet.

- [ ] **Step 2: Verify archive → restore round-trip**

From the regular Segments page, archive a segment (existing flow). Confirm it disappears from the Segments list. Go to the Archived tab, confirm it appears there with its rule badges and last-count intact. Click Restore, confirm it disappears from the Archived list and reappears back on the Segments page.

---

## Self-Review Notes

- **Spec coverage:** §3 (backend fix + restore action) → Task 1. §4 (frontend nav/route/page) → Task 2. §5 (error handling — idempotent restore, practice-scoping, empty state) → covered by Task 1's `test_restore_is_idempotent_on_already_active_segment`/`test_restore_scoped_to_practice` and Task 2's empty-state test. §6 (testing) → covered by each task's own test step.
- **No placeholders:** all steps contain complete, runnable code.
- **Type consistency:** `MarketingArchivedSegments.tsx`'s `MarketingSegment` interface matches the shape `MarketingSegments.tsx` already uses and the backend serializer already returns — no new fields invented.
- **Existing test impact:** `test_delete_archives_instead_of_hard_deleting` is explicitly updated in Task 1 Step 1, not just added to — its old `?include_archived=true` assertion would otherwise silently break once the param is renamed.

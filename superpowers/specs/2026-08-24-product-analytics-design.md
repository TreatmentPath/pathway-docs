# Product Analytics ("Clarity for Pathway") — Design

**Date:** 2026-08-24
**Status:** Approved design, not yet implemented
**Scope:** New `productAnalytics` Django app + frontend autocapture SDK + superadmin dashboard

---

## 1. Problem

We do not know how the app is used. We cannot answer:

- Which parts of the app do practices actually spend time in?
- Which features have we built that nobody has ever opened?
- Where do users get stuck, click repeatedly, or click something that does nothing?
- Which practices are engaged, and which are quietly churning?

Existing systems do **not** answer this:

- `activityLog` records **writes to domain records** (patient created, status changed). It is blind to reading, browsing, navigating, and abandoning.
- `Statistics/staff_activity_views.py` reports **business outcomes** per staff member, not UI usage.

Neither sees a screen that is opened and abandoned, an admin panel nobody visits, or a feature shipped six months ago that has never been clicked.

## 2. Goal

A plug-and-play usage-analytics layer, in the spirit of Microsoft Clarity, that:

- **Self-registers.** New endpoints and new pages appear in the data with no instrumentation work, ever.
- **Auto-captures behaviour.** Page views, dwell time, clicks, rage clicks, dead clicks, scroll depth.
- **Is safe on patient data.** No screen contents, no free text, no patient identifiers — by construction, not by discipline.
- **Produces heatmaps** per practice and per user.

**Explicitly out of scope:** session replay, funnel builder, feature flags, practice-facing dashboards, A/B testing.

## 3. Decisions taken

| Decision | Choice | Rationale |
|---|---|---|
| Buy vs. build | Build, in-house | PostHog self-host is an unsupported single-machine "hobby" deploy; OpenReplay adds a heavy deployment whose replay value is destroyed by the text masking a patient system requires. |
| Session replay | **No** | Replay records the DOM, which on this app means patient names, clinical notes, and medical history. |
| Endpoint tracking vs. explicit events | **Endpoint middleware**, not hand-written events | Self-registering. Explicit events only cover what someone remembered to instrument. |
| Endpoint-only? | **No — plus frontend autocapture** | Endpoint counts measure machine traffic, not human attention (see §3.1). |
| Rollout | **All practices from day one** | With a per-practice kill switch retained for incident response. |
| Raw retention | **90 days**, then pruned; rollups kept indefinitely | Bounds table growth; dashboards read rollups anyway. |
| Audience | Superadmin only | Dashboard at `/admin/product-analytics`. |
| Identity granularity | `practice_id` + `user_id` | Top-level view is per-practice; drill-down is per-user. |

### 3.1 Why endpoint tracking alone is insufficient

This codebase's realtime convention is *payload-less signal + silent refetch*, and it uses singleton fetch caches and polling. Consequently a screen that silently refetches on every websocket nudge generates far more endpoint traffic than a feature a human deliberately opens each day. Endpoint hits measure **machine** traffic.

Endpoint data also cannot see anything client-side: tab switches, modal opens, filter changes, dental-chart interaction, or anything served from cache. It has no concept of dwell time.

The frontend layer supplies human intent; the endpoint layer supplies exhaustive, zero-maintenance coverage. Both are required.

---

## 4. Architecture

Two collectors, one store, one dashboard.

```
┌─────────────────────────┐        ┌──────────────────────────┐
│  Frontend SDK           │        │  ApiUsageMiddleware      │
│  (autocapture)          │        │  (Django, response phase)│
│  page.view / page.leave │        │  every API request       │
│  click / rage / dead    │        │  route pattern + status  │
│  scroll depth           │        │  + duration              │
└───────────┬─────────────┘        └────────────┬─────────────┘
            │ batched POST                      │ direct write
            │ (keepalive)                       │ (buffered)
            ▼                                   ▼
      ┌───────────────────────────────────────────────┐
      │  UsageEvent  (raw, 90-day retention)          │
      └───────────────────┬───────────────────────────┘
                          │ nightly Celery rollup
                          ▼
      ┌───────────────────────────────────────────────┐
      │  UsageDailyRollup  +  ElementClickRollup      │
      └───────────────────┬───────────────────────────┘
                          ▼
            /admin/product-analytics  (IsSuperAdminUser)
```

---

## 5. Collector A — `ApiUsageMiddleware`

New middleware in `productAnalytics/middleware.py`, appended to `MIDDLEWARE` after `PrometheusAfterMiddleware`.

**Captures:** `resolver_match.route` (the URL *pattern*, e.g. `patients/<int:pk>/`), HTTP method, response status, duration in ms, `practice_id`, `user_id`.

**Critical implementation note.** JWT authentication in this project runs through DRF (`DualAuthentication`), *not* `django.contrib.auth.middleware.AuthenticationMiddleware`. Reading `request.user` in `process_request` yields `AnonymousUser` for every API call. The middleware MUST read `request.user` on the **response** phase, after the view has authenticated. Getting this wrong fails silently — every event is attributed to nobody.

**Practice resolution:** from the authenticated user's active practice, server-side. Never from a request header or body.

**Excluded paths:** `/health/`, `/api/backend/prometheus/`, `/static/`, `/media/`, `/app/admin/`, `__debug__/`, and all internal service-to-service callbacks (`/api/backend/messaging/internal/`, `/api/backend/marketing/internal/`) — those are machine traffic with no human behind them.

**Cost control:** events are appended to an in-process buffer and flushed in bulk (`bulk_create`) rather than one INSERT per request. A failure in the middleware must never affect the response — the whole body is wrapped so analytics can never break a user request.

**Using the route pattern rather than the path is a privacy control as well as an aggregation one:** record IDs never enter the analytics store.

---

## 6. Collector B — frontend autocapture SDK

Location: `perfect-pixel-playground-project/src/lib/analytics/`. Integration is a single `<AnalyticsProvider>` in the app shell. No per-page or per-component work, now or later.

### 6.1 Captured events

| Event | Trigger | Payload |
|---|---|---|
| `page.view` | route change | normalized route pattern |
| `page.leave` | route change / tab close | `total_ms`, `active_ms` |
| `click` | any click on an interactive element | element descriptor, route |
| `click.rage` | 3+ clicks on same element within 1s | element descriptor, count |
| `click.dead` | click with no DOM mutation and no navigation within 500ms | element descriptor |
| `scroll.depth` | on page leave | max depth reached, bucketed |

`active_ms` pauses on `visibilitychange` (tab hidden) and after 60 seconds with no input. Without this, a tab left open overnight reports nine hours of "engagement" and every dwell metric becomes noise.

Clicks are captured by **one delegated capture-phase listener** on `document`, walking up to the nearest interactive element (`button`, `a`, `[role=button]`, `[role=tab]`, `[role=menuitem]`, `input`, `select`).

### 6.2 Route normalization

`/patients/8412` → `/patients/:id`. Derived from the React Router match, not by regex on the URL. Without this, every patient ID becomes its own row — useless for aggregation and a record-ID leak into analytics.

### 6.3 Element identity — the PHI-critical rule

Resolution order:

1. `data-track` attribute (explicit, human-readable)
2. `data-testid` attribute
3. Structural descriptor: tag + role + nth-child chain trimmed to ~3 levels, scoped to the normalized route

**We never send `innerText`. We never send `aria-label`.** The codebase contains 765 `aria-label` usages, and labels of the form `"Open patient John Smith"` are precisely how patient names would enter an analytics table. Structural descriptors are less readable but PHI-free by construction.

Current label coverage: 140 `data-testid` occurrences across 54 of 1,496 `.tsx` files. Most elements will therefore resolve to structural descriptors initially. They still aggregate correctly. The dashboard surfaces a "top unlabeled elements" list showing exactly where adding a `data-track` buys the most readability. Coverage improves incrementally; nothing blocks on it.

### 6.4 Transport

In-memory queue, flushed on whichever comes first: 20 events, 5 seconds, or `visibilitychange`. Sent via `fetch(..., { keepalive: true })` so the final batch survives navigation away. On failure the batch is dropped, never retried indefinitely — analytics must never degrade the app.

### 6.5 Practice-switch reset

The queue is a module-level singleton. This codebase has hit the same trap four separate times (`AuthContext`, `useFetchWithAuth`, `FeatureAccessContext`, `useTeamChat`): module-level caches and closures that are not re-scoped on `PracticeSwitcher`'s soft switch. The queue **must flush and reset** on practice switch, or events are attributed to the wrong practice. This is a required test case, not a nicety.

---

## 7. Storage

### `UsageEvent` (raw, 90-day retention)

| Field | Notes |
|---|---|
| `practice` | FK, indexed |
| `user` | FK, nullable |
| `session_id` | UUID, client-generated per tab session |
| `source` | `frontend` \| `api` |
| `event_name` | e.g. `page.view`, `click`, `api.request` |
| `route` | normalized route pattern or URL pattern |
| `element` | element descriptor, nullable |
| `duration_ms` | nullable |
| `status_code` | nullable, API events only |
| `occurred_at` | client/server timestamp |
| `received_at` | server timestamp |
| `props` | JSONB, scalars only |
| `app_version` | nullable |

Indexes: `(practice, occurred_at)`, `(event_name, occurred_at)`, `(route, occurred_at)`.

### Rollups (retained indefinitely)

- `UsageDailyRollup` — `(practice, user, event_name, route, day) → count, total_active_ms`
- `ElementClickRollup` — `(practice, route, element, day) → count, rage_count, dead_count`

A nightly Celery task builds rollups and prunes raw events older than 90 days. **Dashboards read rollups only** — never the raw table.

### Volume

Full autocapture runs roughly 500–2,000 events per active user per day, plus every API call. At ~50 active users that is on the order of 100k events/day — comfortable for Postgres given pruning and rollups.

Two controls are built from day one rather than retrofitted under load:

- **Per-practice kill switch** — disables collection for one practice without a deploy.
- **Sampling rate** — a settings-level percentage applied to high-frequency events (`click`, `scroll.depth`); `page.view` and API events are never sampled.

---

## 8. Ingest API

`POST /api/backend/product-analytics/events/`

- Accepts a batch array.
- Authenticated with the existing JWT.
- **`practice_id` and `user_id` are derived server-side from the token and never read from the request body.** No spoofing surface, no anonymous ingest path to secure.
- Server-side validation rejects: unknown event names, non-scalar `props` values, `props` exceeding a size cap, and batches exceeding a length cap.
- Returns 202 with a count. Partial batches are accepted — one bad event does not reject the batch; it is dropped and counted.

Registered at `path("api/backend/product-analytics/", include("productAnalytics.urls"))` in `TreatmentPath/urls.py`. Added to `SubscriptionMiddleware.EXCLUDED_PATHS` — analytics ingest must not depend on subscription state.

---

## 9. Dashboard

Route `/admin/product-analytics`, following the existing `src/routes/admin.routes.tsx` + `src/components/admin/AdminSidebar.tsx` pattern, `access: "admin"`. Backend guarded by `Admin.permissions.IsSuperAdminUser` — **not** DRF's `IsAdminUser`, which also passes for `staff`.

**Practices overview.** Table of practices: active users, sessions, total active time, top features, last seen. Plus a usage trend chart and a "features nobody uses" list (routes and endpoints with zero events in the period).

**Practice drill-down.** Users × features intensity heatmap. Per-user table: last seen, sessions, active time, top features. Dead-end list: highest rage-click and dead-click elements for this practice.

**User drill-down.** Route and feature breakdown over time for one user.

**Per-route click heat.** Element click counts for a given route, ranked. Plus a superadmin-only in-app **overlay mode** that paints those counts onto the real page — the Clarity feel, with nothing recorded.

---

## 10. Privacy summary

Every safeguard is structural rather than procedural:

1. Route **patterns**, never paths — record IDs cannot enter the store.
2. Element descriptors never include `innerText` or `aria-label` — patient names cannot enter the store.
3. `props` accepts scalars only, against a per-event allowlist, enforced **both** client-side and server-side.
4. Identity is derived server-side from the JWT — the client cannot assert who it is.
5. No DOM recording, no screenshots, no form values.

---

## 11. Testing

**Backend (pytest, `--keepdb`):**
- Middleware attributes events to the correct user and practice via the response phase (the regression test for the DRF-auth trap)
- Excluded paths produce no events
- A middleware exception never affects the response
- Ingest rejects body-supplied `practice_id`/`user_id`
- Ingest rejects non-scalar props, oversized props, oversized batches
- Partial-batch acceptance
- `IsSuperAdminUser` gate: `staff` is refused
- Rollup task correctness and 90-day prune boundary
- Kill switch suppresses collection for one practice only

**Frontend (vitest):**
- Queue flushes at 20 events, at 5 seconds, and on `visibilitychange`
- Route normalization maps `/patients/8412` → `/patients/:id`
- Element descriptor never contains `innerText` or `aria-label` content
- `active_ms` pauses on tab hide and on idle
- Rage-click and dead-click detection thresholds
- **Queue flushes and resets on practice switch**

---

## 12. Build order

1. Storage models + migrations (`productAnalytics` app)
2. `ApiUsageMiddleware` — ships value immediately with zero frontend work
3. Ingest API
4. Frontend SDK + `<AnalyticsProvider>`
5. Rollup Celery task + prune
6. Dashboard: practices overview → drill-downs
7. Per-route click heat + in-app overlay mode

Steps 1–2 are independently useful and can ship before anything else exists.

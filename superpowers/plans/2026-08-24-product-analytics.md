# Product Analytics Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give superadmins a plug-and-play view of how every practice and user actually uses the app — self-registering endpoint coverage plus frontend autocapture — with heatmaps, and with no patient data in the analytics store.

**Architecture:** Two collectors write to one `UsageEvent` table: a Django middleware that records every API request by URL *pattern*, and a frontend SDK that auto-captures page views, dwell time, clicks, rage/dead clicks, and scroll depth. A nightly Celery task rolls raw events into two aggregate tables and prunes raw rows past 90 days. A superadmin dashboard at `/admin/product-analytics` reads the rollups only.

**Tech Stack:** Django 4 + DRF + Celery + Postgres (backend); React 18 + TypeScript + Vite + react-router-dom 6.26 + TanStack Query (frontend); pytest-style `django.test.TestCase` and vitest for tests.

**Spec:** `docs/superpowers/specs/2026-08-24-product-analytics-design.md`

## Global Constraints

- **Never `git commit` or `git push`.** The user handles all VCS operations. Tasks end in a **Checkpoint** step (run the tests, report), not a commit.
- **Backend tests MUST use `--keepdb`. NEVER pass `--noinput`** — it destroys the persistent test DB and pre-existing duplicate migrations then block a fresh rebuild.
- Activate the venv before any Python command: `source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate`
- Backend root: `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath` (this is where `manage.py` lives).
- Frontend root: `/home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project`
- Frontend typecheck is `npx tsc -p tsconfig.app.json` — **not** bare `npx tsc --noEmit`, which checks nothing and exits 0. There are ~491 pre-existing errors; judge by *delta only*.
- **The analytics store must never receive:** `innerText`, `aria-label` contents, raw URL paths containing record IDs, patient identifiers, or form values. Every safeguard is structural, not procedural.
- **Identity is always derived server-side** from the authenticated user. `practice_id` and `user_id` are never read from a request body or header.
- Analytics must never break a user request or degrade the app: middleware body is exception-wrapped; frontend send failures drop the batch silently.
- Practice resolution idiom in this codebase is `user.current_practice` (single source of truth — see `utils/practice_mixins.py:35`).
- Superadmin gate is `Admin.permissions.IsSuperAdminUser` (checks `user_type == "superuser"`). **Never** DRF's `IsAdminUser` — it also passes for `staff`.

---

## File Structure

**Backend — new app `productAnalytics/`** (at `TreatmentPathBackend/TreatmentPath/productAnalytics/`)

| File | Responsibility |
|---|---|
| `__init__.py`, `apps.py` | App registration |
| `models.py` | `UsageEvent`, `UsageDailyRollup`, `ElementClickRollup`, `PracticeAnalyticsSetting` |
| `constants.py` | Allowed event names, excluded paths, prop validation limits |
| `middleware.py` | `ApiUsageMiddleware` — self-registering endpoint capture |
| `collector.py` | Shared buffered-write helper used by middleware and ingest |
| `serializers.py` | Ingest batch validation |
| `views.py` | Ingest endpoint |
| `dashboard_views.py` | Superadmin query endpoints (kept separate — `views.py` stays small) |
| `tasks.py` | `build_usage_rollups`, `prune_usage_events` |
| `urls.py` | Route table |
| `tests/` | `test_middleware.py`, `test_ingest.py`, `test_rollups.py`, `test_dashboard.py` |

**Backend — modified**

| File | Change |
|---|---|
| `TreatmentPath/settings.py` | Add `productAnalytics` to `INSTALLED_APPS`, middleware entry, beat schedule, sampling/retention settings |
| `TreatmentPath/urls.py` | Mount `api/backend/product-analytics/` |
| `payments/middleware.py` | Add ingest path to `EXCLUDED_PATHS` |

**Frontend — new `src/lib/analytics/`**

| File | Responsibility |
|---|---|
| `types.ts` | `AnalyticsEvent`, `EventName` |
| `session.ts` | Per-tab session id, app version |
| `routeName.ts` | URL → route *pattern* via `matchRoutes` |
| `elementId.ts` | Element → PHI-free descriptor |
| `queue.ts` | Batching, flush triggers, keepalive transport, reset |
| `activeTime.ts` | `total_ms` / `active_ms` with visibility + idle pausing |
| `clicks.ts` | Delegated listener, rage/dead detection |
| `scroll.ts` | Max scroll depth per route |
| `analytics.ts` | Public surface: `init`, `track`, `reset` |
| `AnalyticsProvider.tsx` | Mounts everything; resets on practice switch |

**Frontend — modified**

| File | Change |
|---|---|
| `src/App.tsx` | Mount `<AnalyticsProvider>` inside `<AuthProvider>` |
| `src/config/api.ts` | Add `productAnalytics` endpoints |
| `src/routes/admin.routes.tsx` | Add `/admin/product-analytics` route |
| `src/components/admin/AdminSidebar.tsx` | Add nav entry |
| `src/pages/admin/ProductAnalytics.tsx` (new) | Dashboard page |
| `src/pages/admin/product-analytics/` (new) | Dashboard sub-components + hook |

---

## Task 1: Backend app scaffold, models, migration

**Files:**
- Create: `productAnalytics/__init__.py`, `productAnalytics/apps.py`, `productAnalytics/constants.py`, `productAnalytics/models.py`, `productAnalytics/tests/__init__.py`
- Create: `productAnalytics/tests/test_models.py`
- Modify: `TreatmentPath/settings.py` (INSTALLED_APPS)

**Interfaces:**
- Consumes: `UserAuthentication.models.Practice`, `settings.AUTH_USER_MODEL`
- Produces: `UsageEvent`, `UsageDailyRollup`, `ElementClickRollup`, `PracticeAnalyticsSetting`; constants `FRONTEND_EVENT_NAMES`, `API_EVENT_NAME`, `SOURCE_FRONTEND`, `SOURCE_API`, `MAX_BATCH_SIZE`, `MAX_PROP_KEYS`, `MAX_PROP_STRING_LEN`, `EXCLUDED_PATH_PREFIXES`

- [ ] **Step 1: Create the app package and constants**

`productAnalytics/__init__.py` — empty file.

`productAnalytics/apps.py`:

```python
from django.apps import AppConfig


class ProductAnalyticsConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "productAnalytics"
```

`productAnalytics/constants.py`:

```python
"""
Shared constants for product analytics collection.

Event names are an allowlist, enforced server-side. An event name that is not
listed here is dropped at ingest — this is what stops an arbitrary client from
writing arbitrary rows into the analytics store.
"""

SOURCE_FRONTEND = "frontend"
SOURCE_API = "api"

# The single event name the Django middleware emits.
API_EVENT_NAME = "api.request"

# Everything the frontend SDK is permitted to send.
FRONTEND_EVENT_NAMES = frozenset(
    {
        "page.view",
        "page.leave",
        "click",
        "click.rage",
        "click.dead",
        "scroll.depth",
    }
)

# Events that are never sampled away — these carry the core "what is used"
# numbers and must stay exact even when sampling is dialled down.
NEVER_SAMPLED_EVENTS = frozenset({"page.view", "page.leave"})

# Ingest limits.
MAX_BATCH_SIZE = 200
MAX_PROP_KEYS = 20
MAX_PROP_STRING_LEN = 100
MAX_ROUTE_LEN = 255
MAX_ELEMENT_LEN = 255

# Paths the middleware never records. These are machine traffic (health checks,
# scrapes, static assets, service-to-service callbacks) with no human behind
# them; recording them would swamp real usage.
EXCLUDED_PATH_PREFIXES = (
    "/health/",
    "/api/backend/prometheus/",
    "/api/backend/product-analytics/",
    "/static/",
    "/media/",
    "/app/admin/",
    "/__debug__/",
    "/api/backend/messaging/internal/",
    "/api/backend/marketing/internal/",
)
```

- [ ] **Step 2: Write the failing model test**

`productAnalytics/tests/__init__.py` — empty file.

`productAnalytics/tests/test_models.py`:

```python
from django.utils import timezone
from django.test import TestCase

from productAnalytics.constants import SOURCE_API
from productAnalytics.models import (
    ElementClickRollup,
    PracticeAnalyticsSetting,
    UsageDailyRollup,
    UsageEvent,
)
from UserAuthentication.models import Practice, User


class UsageEventModelTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(
            name="Analytics Test Practice",
            slug="analytics-test-practice",
        )
        self.user = User.objects.create_user(
            email="analytics-user@example.com",
            password="testpass123",
        )

    def test_usage_event_stores_route_pattern_and_props(self):
        event = UsageEvent.objects.create(
            practice=self.practice,
            user=self.user,
            source=SOURCE_API,
            event_name="api.request",
            route="patients/<int:pk>/",
            method="GET",
            status_code=200,
            duration_ms=42,
            occurred_at=timezone.now(),
            props={"query_count": 3},
        )

        event.refresh_from_db()
        self.assertEqual(event.route, "patients/<int:pk>/")
        self.assertEqual(event.props["query_count"], 3)
        self.assertIsNotNone(event.received_at)

    def test_usage_event_survives_user_deletion(self):
        """Analytics history must not vanish when a user is removed."""
        event = UsageEvent.objects.create(
            practice=self.practice,
            user=self.user,
            source=SOURCE_API,
            event_name="api.request",
            route="patients/",
            occurred_at=timezone.now(),
        )
        self.user.delete()

        event.refresh_from_db()
        self.assertIsNone(event.user)
        self.assertEqual(event.practice, self.practice)

    def test_collection_is_enabled_by_default_for_a_practice(self):
        setting = PracticeAnalyticsSetting.objects.create(practice=self.practice)
        self.assertTrue(setting.collection_enabled)

    def test_daily_rollup_is_unique_per_dimension_set(self):
        today = timezone.now().date()
        kwargs = dict(
            practice=self.practice,
            user=self.user,
            day=today,
            source=SOURCE_API,
            event_name="api.request",
            route="patients/",
        )
        UsageDailyRollup.objects.create(count=5, total_active_ms=0, **kwargs)

        with self.assertRaises(Exception):
            UsageDailyRollup.objects.create(count=1, total_active_ms=0, **kwargs)

    def test_element_click_rollup_is_unique_per_dimension_set(self):
        today = timezone.now().date()
        kwargs = dict(
            practice=self.practice,
            day=today,
            route="/patients/:id",
            element="button#save-patient",
        )
        ElementClickRollup.objects.create(count=3, rage_count=0, dead_count=0, **kwargs)

        with self.assertRaises(Exception):
            ElementClickRollup.objects.create(
                count=1, rage_count=0, dead_count=0, **kwargs
            )
```

- [ ] **Step 3: Run the test to verify it fails**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test productAnalytics.tests.test_models --keepdb -v 2
```

Expected: FAIL — `ModuleNotFoundError: No module named 'productAnalytics.models'`.

- [ ] **Step 4: Write the models**

`productAnalytics/models.py`:

```python
"""
Product analytics storage.

Three layers:
  UsageEvent          raw events, 90-day retention, written by two collectors
  UsageDailyRollup    nightly aggregate — what the dashboard actually reads
  ElementClickRollup  nightly aggregate of click heat per route + element

PRIVACY: `route` always holds a URL *pattern* (`patients/<int:pk>/` from Django,
`/patients/:id` from the frontend), never a concrete path. `element` holds a
structural or data-track descriptor, never innerText or aria-label. `props`
accepts scalars only. See constants.py and serializers.py for enforcement.
"""

from django.conf import settings
from django.db import models

from UserAuthentication.models import Practice

from .constants import SOURCE_API, SOURCE_FRONTEND


SOURCE_CHOICES = [
    (SOURCE_FRONTEND, "Frontend"),
    (SOURCE_API, "API"),
]


class UsageEvent(models.Model):
    """One raw observed event. Pruned after PRODUCT_ANALYTICS_RETENTION_DAYS."""

    practice = models.ForeignKey(
        Practice, on_delete=models.CASCADE, related_name="usage_events"
    )
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name="usage_events",
    )
    session_id = models.CharField(max_length=64, blank=True, default="")
    source = models.CharField(max_length=10, choices=SOURCE_CHOICES)
    event_name = models.CharField(max_length=64)
    route = models.CharField(max_length=255, blank=True, default="")
    element = models.CharField(max_length=255, blank=True, default="")
    method = models.CharField(max_length=10, blank=True, default="")
    status_code = models.PositiveSmallIntegerField(null=True, blank=True)
    duration_ms = models.PositiveIntegerField(null=True, blank=True)
    occurred_at = models.DateTimeField()
    received_at = models.DateTimeField(auto_now_add=True)
    props = models.JSONField(default=dict, blank=True)
    app_version = models.CharField(max_length=32, blank=True, default="")

    class Meta:
        indexes = [
            models.Index(fields=["practice", "occurred_at"]),
            models.Index(fields=["event_name", "occurred_at"]),
            models.Index(fields=["route", "occurred_at"]),
        ]

    def __str__(self):
        return f"{self.event_name} {self.route} @ {self.occurred_at:%Y-%m-%d %H:%M}"


class UsageDailyRollup(models.Model):
    """Per-day aggregate. The dashboard reads this, never UsageEvent."""

    practice = models.ForeignKey(
        Practice, on_delete=models.CASCADE, related_name="usage_rollups"
    )
    user = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        null=True,
        blank=True,
        on_delete=models.SET_NULL,
        related_name="usage_rollups",
    )
    day = models.DateField()
    source = models.CharField(max_length=10, choices=SOURCE_CHOICES)
    event_name = models.CharField(max_length=64)
    route = models.CharField(max_length=255, blank=True, default="")
    count = models.PositiveIntegerField(default=0)
    total_active_ms = models.BigIntegerField(default=0)

    class Meta:
        unique_together = ("practice", "user", "day", "source", "event_name", "route")
        indexes = [
            models.Index(fields=["practice", "day"]),
            models.Index(fields=["day", "route"]),
        ]


class ElementClickRollup(models.Model):
    """Per-day click heat for one element on one route."""

    practice = models.ForeignKey(
        Practice, on_delete=models.CASCADE, related_name="element_click_rollups"
    )
    day = models.DateField()
    route = models.CharField(max_length=255)
    element = models.CharField(max_length=255)
    count = models.PositiveIntegerField(default=0)
    rage_count = models.PositiveIntegerField(default=0)
    dead_count = models.PositiveIntegerField(default=0)

    class Meta:
        unique_together = ("practice", "day", "route", "element")
        indexes = [models.Index(fields=["route", "day"])]


class PracticeAnalyticsSetting(models.Model):
    """
    Per-practice kill switch. Absence of a row means enabled — collection is on
    for all practices by default, and a row is only created to turn it off.
    """

    practice = models.OneToOneField(
        Practice, on_delete=models.CASCADE, related_name="analytics_setting"
    )
    collection_enabled = models.BooleanField(default=True)
    updated_at = models.DateTimeField(auto_now=True)
```

- [ ] **Step 5: Register the app**

In `TreatmentPath/settings.py`, add to `INSTALLED_APPS` immediately after the `"usage",` line (around line 112):

```python
    "productAnalytics",  # Product analytics — endpoint + frontend usage capture
```

Also add the settings block. Put it directly above the `CELERY_BEAT_SCHEDULE` definition (around line 1000):

```python
# ---------------------------------------------------------------------------
# Product analytics
# ---------------------------------------------------------------------------
# Raw UsageEvent rows older than this are pruned nightly. Rollups are kept
# indefinitely, and the dashboard only reads rollups, so shortening this does
# not lose dashboard history.
PRODUCT_ANALYTICS_RETENTION_DAYS = 90

# Fraction of high-frequency events (click, click.rage, click.dead,
# scroll.depth) that are kept. page.view / page.leave / api.request are never
# sampled — see constants.NEVER_SAMPLED_EVENTS.
PRODUCT_ANALYTICS_SAMPLE_RATE = 1.0

# Global off switch, for incident response.
PRODUCT_ANALYTICS_ENABLED = True
```

- [ ] **Step 6: Generate the migration**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py makemigrations productAnalytics
```

Expected: creates `productAnalytics/migrations/0001_initial.py` with four models. All tables are new, so `django_migration_linter` has nothing to flag.

- [ ] **Step 7: Run the tests to verify they pass**

```bash
python manage.py test productAnalytics.tests.test_models --keepdb -v 2
```

Expected: 5 tests PASS.

- [ ] **Step 8: Checkpoint**

Report: migration filename, test count, and confirm `productAnalytics` appears in `python manage.py showmigrations productAnalytics`. Do not commit.

---

## Task 2: `ApiUsageMiddleware` — self-registering endpoint capture

**Files:**
- Create: `productAnalytics/collector.py`, `productAnalytics/middleware.py`
- Create: `productAnalytics/tests/test_middleware.py`
- Modify: `TreatmentPath/settings.py` (MIDDLEWARE)

**Interfaces:**
- Consumes: `UsageEvent`, constants from Task 1
- Produces: `productAnalytics.collector.record_events(events: list[UsageEvent]) -> int`, `productAnalytics.collector.is_collection_enabled(practice_id: int) -> bool`, `ApiUsageMiddleware`

- [ ] **Step 1: Write the failing middleware test**

`productAnalytics/tests/test_middleware.py`:

```python
from django.test import TestCase, override_settings
from django.urls import reverse
from rest_framework.test import APIClient

from productAnalytics.constants import API_EVENT_NAME, SOURCE_API
from productAnalytics.models import PracticeAnalyticsSetting, UsageEvent
from UserAuthentication.models import Practice, User


class ApiUsageMiddlewareTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(
            name="Middleware Practice",
            slug="middleware-practice",
        )
        self.user = User.objects.create_user(
            email="middleware-user@example.com",
            password="testpass123",
        )
        self.user.current_practice = self.practice
        self.user.save(update_fields=["current_practice"])
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)

    def test_authenticated_api_call_is_attributed_to_user_and_practice(self):
        """
        The regression test for the DRF-auth trap: JWT auth runs inside the
        view, not in AuthenticationMiddleware, so reading request.user on the
        REQUEST phase yields AnonymousUser and every event lands unattributed.
        """
        self.client.get("/api/backend/auth/user/practices/")

        event = UsageEvent.objects.filter(source=SOURCE_API).first()
        self.assertIsNotNone(event, "middleware recorded no event")
        self.assertEqual(event.user, self.user)
        self.assertEqual(event.practice, self.practice)
        self.assertEqual(event.event_name, API_EVENT_NAME)
        self.assertEqual(event.method, "GET")

    def test_route_pattern_is_stored_not_the_concrete_path(self):
        """Record IDs must never enter the analytics store."""
        self.client.get("/api/backend/auth/user/practices/")

        event = UsageEvent.objects.filter(source=SOURCE_API).first()
        self.assertNotIn("/api/backend/", event.route)
        self.assertNotRegex(event.route, r"/\d+/")

    def test_excluded_paths_are_not_recorded(self):
        self.client.get("/health/")
        self.assertFalse(UsageEvent.objects.filter(route__contains="health").exists())

    def test_unauthenticated_request_records_nothing(self):
        anon = APIClient()
        anon.get("/api/backend/auth/user/practices/")
        self.assertEqual(UsageEvent.objects.count(), 0)

    def test_kill_switch_suppresses_one_practice_only(self):
        PracticeAnalyticsSetting.objects.create(
            practice=self.practice, collection_enabled=False
        )
        self.client.get("/api/backend/auth/user/practices/")
        self.assertEqual(UsageEvent.objects.count(), 0)

    @override_settings(PRODUCT_ANALYTICS_ENABLED=False)
    def test_global_switch_suppresses_all_collection(self):
        self.client.get("/api/backend/auth/user/practices/")
        self.assertEqual(UsageEvent.objects.count(), 0)

    def test_collector_failure_never_breaks_the_response(self):
        from unittest.mock import patch

        with patch(
            "productAnalytics.middleware.record_events",
            side_effect=RuntimeError("analytics exploded"),
        ):
            response = self.client.get("/api/backend/auth/user/practices/")

        self.assertLess(response.status_code, 500)
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
python manage.py test productAnalytics.tests.test_middleware --keepdb -v 2
```

Expected: FAIL — `ModuleNotFoundError: No module named 'productAnalytics.middleware'`.

- [ ] **Step 3: Write the collector**

`productAnalytics/collector.py`:

```python
"""
Shared write path for both collectors.

Everything here is defensive: analytics is a passenger, never a blocker. A
failure to record must never surface to the user, so callers wrap invocations
and this module swallows nothing silently that it cannot log.
"""

import logging
import random

from django.conf import settings

from .constants import NEVER_SAMPLED_EVENTS
from .models import PracticeAnalyticsSetting, UsageEvent


logger = logging.getLogger(__name__)


def is_collection_enabled(practice_id) -> bool:
    """
    Collection is ON by default. A PracticeAnalyticsSetting row exists only to
    turn a practice OFF, so absence of a row means enabled.
    """
    if not getattr(settings, "PRODUCT_ANALYTICS_ENABLED", True):
        return False
    if not practice_id:
        return False
    return not PracticeAnalyticsSetting.objects.filter(
        practice_id=practice_id, collection_enabled=False
    ).exists()


def should_sample(event_name: str) -> bool:
    """
    High-frequency events are sampled; the core 'what is used' events are not.
    """
    if event_name in NEVER_SAMPLED_EVENTS:
        return True
    rate = float(getattr(settings, "PRODUCT_ANALYTICS_SAMPLE_RATE", 1.0))
    if rate >= 1.0:
        return True
    return random.random() < rate


def record_events(events) -> int:
    """
    Bulk-insert a list of unsaved UsageEvent instances. Returns the number
    written. One INSERT per request would be an unacceptable per-request cost,
    hence bulk_create.
    """
    if not events:
        return 0
    UsageEvent.objects.bulk_create(events, batch_size=500)
    return len(events)
```

- [ ] **Step 4: Write the middleware**

`productAnalytics/middleware.py`:

```python
"""
ApiUsageMiddleware — self-registering endpoint usage capture.

A new endpoint appears in the analytics data the day it merges; nobody ever
instruments anything.

CRITICAL: JWT auth in this project runs through DRF (`DualAuthentication`),
NOT django.contrib.auth.middleware.AuthenticationMiddleware. `request.user` is
AnonymousUser during the request phase for every API call. We therefore read
identity on the RESPONSE phase, after the view has authenticated. Getting this
wrong fails silently — every event lands with no user and no practice.
"""

import logging
import time

from .collector import is_collection_enabled, record_events, should_sample
from .constants import API_EVENT_NAME, EXCLUDED_PATH_PREFIXES, SOURCE_API
from .models import UsageEvent


logger = logging.getLogger(__name__)


class ApiUsageMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        started = time.monotonic()
        response = self.get_response(request)

        try:
            self._record(request, response, started)
        except Exception:  # pragma: no cover - defensive
            # Analytics must never affect a user request.
            logger.exception("ApiUsageMiddleware failed to record an event")

        return response

    def _record(self, request, response, started):
        path = request.path
        if any(path.startswith(prefix) for prefix in EXCLUDED_PATH_PREFIXES):
            return

        # Response phase: DRF has now set request.user if the view authenticated.
        user = getattr(request, "user", None)
        if not user or not user.is_authenticated:
            return

        practice_id = getattr(user, "current_practice_id", None)
        if not is_collection_enabled(practice_id):
            return

        if not should_sample(API_EVENT_NAME):
            return

        # The URL PATTERN, not the concrete path — this is a privacy control as
        # much as an aggregation one: record IDs never enter the store.
        match = getattr(request, "resolver_match", None)
        route = getattr(match, "route", "") if match else ""
        if not route:
            return

        from django.utils import timezone

        event = UsageEvent(
            practice_id=practice_id,
            user=user,
            source=SOURCE_API,
            event_name=API_EVENT_NAME,
            route=route[:255],
            method=request.method[:10],
            status_code=getattr(response, "status_code", None),
            duration_ms=int((time.monotonic() - started) * 1000),
            occurred_at=timezone.now(),
        )
        record_events([event])
```

- [ ] **Step 5: Register the middleware**

In `TreatmentPath/settings.py`, append to `MIDDLEWARE` after `"django_prometheus.middleware.PrometheusAfterMiddleware",` (line 212):

```python
    "productAnalytics.middleware.ApiUsageMiddleware",  # Product analytics — endpoint capture
```

- [ ] **Step 6: Run the tests to verify they pass**

```bash
python manage.py test productAnalytics.tests.test_middleware --keepdb -v 2
```

Expected: 7 tests PASS.

`/api/backend/auth/user/practices/` is chosen deliberately as the probe: it is `UserPracticeSwitchView` (`UserAuthentication/views/user_practice_switch_views.py:23`), which has `permission_classes = [IsAuthenticated]` and a real `get()` handler, so DRF definitely runs authentication and the response phase definitely has a populated `request.user`. If it ever fails with `event is None`, substitute another authenticated GET endpoint rather than weakening the assertion.

- [ ] **Step 7: Checkpoint**

Run the whole app's tests so far: `python manage.py test productAnalytics --keepdb`. Report pass count. Do not commit.

---

## Task 3: Ingest API

**Files:**
- Create: `productAnalytics/serializers.py`, `productAnalytics/views.py`, `productAnalytics/urls.py`
- Create: `productAnalytics/tests/test_ingest.py`
- Modify: `TreatmentPath/urls.py`, `payments/middleware.py`

**Interfaces:**
- Consumes: `UsageEvent`, `record_events`, `is_collection_enabled`, `should_sample`, constants
- Produces: `POST /api/backend/product-analytics/events/` returning `{"accepted": int, "rejected": int}` with HTTP 202

- [ ] **Step 1: Write the failing ingest test**

`productAnalytics/tests/test_ingest.py`:

```python
from django.utils import timezone
from django.test import TestCase
from rest_framework.test import APIClient

from productAnalytics.models import UsageEvent
from UserAuthentication.models import Practice, User


INGEST_URL = "/api/backend/product-analytics/events/"


def _event(**overrides):
    payload = {
        "event_name": "page.view",
        "route": "/patients/:id",
        "occurred_at": timezone.now().isoformat(),
        "session_id": "sess-abc",
    }
    payload.update(overrides)
    return payload


class IngestTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(
            name="Ingest Practice", slug="ingest-practice"
        )
        self.other_practice = Practice.objects.create(
            name="Other Practice", slug="other-practice"
        )
        self.user = User.objects.create_user(
            email="ingest-user@example.com", password="testpass123"
        )
        self.user.current_practice = self.practice
        self.user.save(update_fields=["current_practice"])
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)

    def test_accepts_a_batch_and_stores_events(self):
        response = self.client.post(
            INGEST_URL, {"events": [_event(), _event(event_name="click")]}, format="json"
        )

        self.assertEqual(response.status_code, 202)
        self.assertEqual(response.data["accepted"], 2)
        self.assertEqual(UsageEvent.objects.count(), 2)

    def test_identity_is_server_derived_and_body_values_are_ignored(self):
        """A client must not be able to assert who it is."""
        response = self.client.post(
            INGEST_URL,
            {
                "events": [
                    _event(
                        practice=self.other_practice.id,
                        practice_id=self.other_practice.id,
                        user=999999,
                        user_id=999999,
                    )
                ]
            },
            format="json",
        )

        self.assertEqual(response.status_code, 202)
        event = UsageEvent.objects.get()
        self.assertEqual(event.practice, self.practice)
        self.assertEqual(event.user, self.user)

    def test_unknown_event_names_are_rejected_not_stored(self):
        response = self.client.post(
            INGEST_URL, {"events": [_event(event_name="evil.exfiltrate")]}, format="json"
        )

        self.assertEqual(response.status_code, 202)
        self.assertEqual(response.data["accepted"], 0)
        self.assertEqual(response.data["rejected"], 1)
        self.assertEqual(UsageEvent.objects.count(), 0)

    def test_non_scalar_props_are_rejected(self):
        response = self.client.post(
            INGEST_URL,
            {"events": [_event(props={"nested": {"patient_name": "John Smith"}})]},
            format="json",
        )

        self.assertEqual(response.data["accepted"], 0)
        self.assertEqual(UsageEvent.objects.count(), 0)

    def test_oversized_prop_strings_are_rejected(self):
        response = self.client.post(
            INGEST_URL, {"events": [_event(props={"note": "x" * 500})]}, format="json"
        )

        self.assertEqual(response.data["accepted"], 0)

    def test_partial_batch_is_accepted(self):
        """One bad event must not reject the whole batch."""
        response = self.client.post(
            INGEST_URL,
            {"events": [_event(), _event(event_name="not.allowed")]},
            format="json",
        )

        self.assertEqual(response.data["accepted"], 1)
        self.assertEqual(response.data["rejected"], 1)
        self.assertEqual(UsageEvent.objects.count(), 1)

    def test_oversized_batch_is_refused(self):
        response = self.client.post(
            INGEST_URL, {"events": [_event() for _ in range(500)]}, format="json"
        )

        self.assertEqual(response.status_code, 400)
        self.assertEqual(UsageEvent.objects.count(), 0)

    def test_anonymous_ingest_is_refused(self):
        anon = APIClient()
        response = anon.post(INGEST_URL, {"events": [_event()]}, format="json")

        self.assertIn(response.status_code, (401, 403))
        self.assertEqual(UsageEvent.objects.count(), 0)
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
python manage.py test productAnalytics.tests.test_ingest --keepdb -v 2
```

Expected: FAIL — 404 on the ingest URL.

- [ ] **Step 3: Write the serializer**

`productAnalytics/serializers.py`:

```python
"""
Ingest validation.

The client supplies WHAT happened. It never supplies WHO — practice and user
are attached server-side from the authenticated token in the view. Any
practice/user keys in the body are ignored by omission: they are simply not
declared as fields here.
"""

from rest_framework import serializers

from .constants import (
    FRONTEND_EVENT_NAMES,
    MAX_ELEMENT_LEN,
    MAX_PROP_KEYS,
    MAX_PROP_STRING_LEN,
    MAX_ROUTE_LEN,
)


SCALAR_TYPES = (str, int, float, bool)


class UsageEventInSerializer(serializers.Serializer):
    event_name = serializers.CharField(max_length=64)
    route = serializers.CharField(
        max_length=MAX_ROUTE_LEN, required=False, allow_blank=True, default=""
    )
    element = serializers.CharField(
        max_length=MAX_ELEMENT_LEN, required=False, allow_blank=True, default=""
    )
    session_id = serializers.CharField(
        max_length=64, required=False, allow_blank=True, default=""
    )
    duration_ms = serializers.IntegerField(required=False, allow_null=True, min_value=0)
    occurred_at = serializers.DateTimeField()
    app_version = serializers.CharField(
        max_length=32, required=False, allow_blank=True, default=""
    )
    props = serializers.JSONField(required=False, default=dict)

    def validate_event_name(self, value):
        if value not in FRONTEND_EVENT_NAMES:
            raise serializers.ValidationError(f"Unknown event name: {value}")
        return value

    def validate_props(self, value):
        if value in (None, ""):
            return {}
        if not isinstance(value, dict):
            raise serializers.ValidationError("props must be an object")
        if len(value) > MAX_PROP_KEYS:
            raise serializers.ValidationError(
                f"props may contain at most {MAX_PROP_KEYS} keys"
            )
        for key, item in value.items():
            if not isinstance(item, SCALAR_TYPES):
                raise serializers.ValidationError(
                    f"props['{key}'] must be a scalar — nested values are refused "
                    "because they are how free text (and patient data) leaks in"
                )
            if isinstance(item, str) and len(item) > MAX_PROP_STRING_LEN:
                raise serializers.ValidationError(
                    f"props['{key}'] exceeds {MAX_PROP_STRING_LEN} characters"
                )
        return value
```

- [ ] **Step 4: Write the ingest view**

`productAnalytics/views.py`:

```python
"""Ingest endpoint for the frontend autocapture SDK."""

import logging

from rest_framework import status
from rest_framework.permissions import IsAuthenticated
from rest_framework.response import Response
from rest_framework.views import APIView

from .collector import is_collection_enabled, record_events, should_sample
from .constants import MAX_BATCH_SIZE, SOURCE_FRONTEND
from .models import UsageEvent
from .serializers import UsageEventInSerializer


logger = logging.getLogger(__name__)


class UsageEventIngestView(APIView):
    """
    POST a batch of autocaptured events.

    Partial acceptance is deliberate: a single malformed event is dropped and
    counted rather than rejecting the whole batch, because a client-side bug in
    one event type must not blind us to every other event in the flush.
    """

    permission_classes = [IsAuthenticated]

    def post(self, request):
        events = request.data.get("events")
        if not isinstance(events, list):
            return Response(
                {"error": "events must be a list"}, status=status.HTTP_400_BAD_REQUEST
            )
        if len(events) > MAX_BATCH_SIZE:
            return Response(
                {"error": f"batch exceeds {MAX_BATCH_SIZE} events"},
                status=status.HTTP_400_BAD_REQUEST,
            )

        user = request.user
        practice_id = getattr(user, "current_practice_id", None)
        if not is_collection_enabled(practice_id):
            return Response(
                {"accepted": 0, "rejected": 0}, status=status.HTTP_202_ACCEPTED
            )

        to_write = []
        rejected = 0
        for raw in events:
            serializer = UsageEventInSerializer(data=raw)
            if not serializer.is_valid():
                rejected += 1
                continue
            data = serializer.validated_data
            if not should_sample(data["event_name"]):
                continue
            to_write.append(
                UsageEvent(
                    practice_id=practice_id,
                    user=user,
                    source=SOURCE_FRONTEND,
                    event_name=data["event_name"],
                    route=data.get("route", ""),
                    element=data.get("element", ""),
                    session_id=data.get("session_id", ""),
                    duration_ms=data.get("duration_ms"),
                    occurred_at=data["occurred_at"],
                    app_version=data.get("app_version", ""),
                    props=data.get("props") or {},
                )
            )

        accepted = record_events(to_write)
        return Response(
            {"accepted": accepted, "rejected": rejected},
            status=status.HTTP_202_ACCEPTED,
        )
```

`productAnalytics/urls.py`:

```python
from django.urls import path

from .views import UsageEventIngestView


urlpatterns = [
    path("events/", UsageEventIngestView.as_view(), name="product-analytics-ingest"),
]
```

- [ ] **Step 5: Mount the URLs and exempt from subscription checks**

In `TreatmentPath/urls.py`, add alongside the other includes (near the `usage` line, ~line 63):

```python
    path(
        "api/backend/product-analytics/",
        include("productAnalytics.urls"),
    ),  # Product analytics ingest + dashboard
```

In `payments/middleware.py`, add to `SubscriptionMiddleware.EXCLUDED_PATHS` (after the internal callbacks entries, ~line 44):

```python
        # Analytics ingest must not depend on subscription state — a lapsed
        # practice is exactly the one whose usage we most want to see.
        "/api/backend/product-analytics/",
```

- [ ] **Step 6: Run the tests to verify they pass**

```bash
python manage.py test productAnalytics.tests.test_ingest --keepdb -v 2
```

Expected: 8 tests PASS.

- [ ] **Step 7: Checkpoint**

`python manage.py test productAnalytics --keepdb`. Report pass count. Do not commit.

---

## Task 4: Rollup and prune tasks

**Files:**
- Create: `productAnalytics/tasks.py`
- Create: `productAnalytics/tests/test_rollups.py`
- Modify: `TreatmentPath/settings.py` (CELERY_BEAT_SCHEDULE)

**Interfaces:**
- Consumes: `UsageEvent`, `UsageDailyRollup`, `ElementClickRollup`, `settings.PRODUCT_ANALYTICS_RETENTION_DAYS`
- Produces: `build_usage_rollups(for_date: str | None = None) -> dict`, `prune_usage_events() -> dict`

- [ ] **Step 1: Write the failing rollup test**

`productAnalytics/tests/test_rollups.py`:

```python
from datetime import timedelta

from django.test import TestCase, override_settings
from django.utils import timezone

from productAnalytics.constants import SOURCE_API, SOURCE_FRONTEND
from productAnalytics.models import ElementClickRollup, UsageDailyRollup, UsageEvent
from productAnalytics.tasks import build_usage_rollups, prune_usage_events
from UserAuthentication.models import Practice, User


class RollupTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(
            name="Rollup Practice", slug="rollup-practice"
        )
        self.user = User.objects.create_user(
            email="rollup-user@example.com", password="testpass123"
        )
        self.yesterday = timezone.now() - timedelta(days=1)

    def _event(self, **overrides):
        payload = dict(
            practice=self.practice,
            user=self.user,
            source=SOURCE_FRONTEND,
            event_name="page.view",
            route="/patients/:id",
            occurred_at=self.yesterday,
        )
        payload.update(overrides)
        return UsageEvent.objects.create(**payload)

    def test_rollup_counts_events_per_dimension_set(self):
        self._event()
        self._event()
        self._event(route="/daylist")

        build_usage_rollups(for_date=self.yesterday.date().isoformat())

        patients = UsageDailyRollup.objects.get(route="/patients/:id")
        self.assertEqual(patients.count, 2)
        daylist = UsageDailyRollup.objects.get(route="/daylist")
        self.assertEqual(daylist.count, 1)

    def test_rollup_sums_active_time_from_page_leave(self):
        self._event(event_name="page.leave", duration_ms=5000)
        self._event(event_name="page.leave", duration_ms=3000)

        build_usage_rollups(for_date=self.yesterday.date().isoformat())

        rollup = UsageDailyRollup.objects.get(event_name="page.leave")
        self.assertEqual(rollup.total_active_ms, 8000)

    def test_rollup_is_idempotent(self):
        """Re-running for the same day must not double-count."""
        self._event()
        day = self.yesterday.date().isoformat()

        build_usage_rollups(for_date=day)
        build_usage_rollups(for_date=day)

        self.assertEqual(UsageDailyRollup.objects.get(event_name="page.view").count, 1)

    def test_click_rollup_separates_plain_rage_and_dead_clicks(self):
        self._event(event_name="click", element="button#save")
        self._event(event_name="click", element="button#save")
        self._event(event_name="click.rage", element="button#save")
        self._event(event_name="click.dead", element="button#save")

        build_usage_rollups(for_date=self.yesterday.date().isoformat())

        heat = ElementClickRollup.objects.get(element="button#save")
        self.assertEqual(heat.count, 2)
        self.assertEqual(heat.rage_count, 1)
        self.assertEqual(heat.dead_count, 1)

    def test_api_events_are_rolled_up_too(self):
        self._event(source=SOURCE_API, event_name="api.request", route="patients/")

        build_usage_rollups(for_date=self.yesterday.date().isoformat())

        self.assertTrue(
            UsageDailyRollup.objects.filter(source=SOURCE_API, route="patients/").exists()
        )

    @override_settings(PRODUCT_ANALYTICS_RETENTION_DAYS=90)
    def test_prune_removes_events_past_retention_and_keeps_the_boundary(self):
        old = self._event(occurred_at=timezone.now() - timedelta(days=91))
        keep = self._event(occurred_at=timezone.now() - timedelta(days=89))

        prune_usage_events()

        self.assertFalse(UsageEvent.objects.filter(pk=old.pk).exists())
        self.assertTrue(UsageEvent.objects.filter(pk=keep.pk).exists())

    @override_settings(PRODUCT_ANALYTICS_RETENTION_DAYS=90)
    def test_prune_leaves_rollups_intact(self):
        self._event(occurred_at=timezone.now() - timedelta(days=91))
        build_usage_rollups(
            for_date=(timezone.now() - timedelta(days=91)).date().isoformat()
        )

        prune_usage_events()

        self.assertTrue(UsageDailyRollup.objects.exists())
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
python manage.py test productAnalytics.tests.test_rollups --keepdb -v 2
```

Expected: FAIL — `ModuleNotFoundError: No module named 'productAnalytics.tasks'`.

- [ ] **Step 3: Write the tasks**

`productAnalytics/tasks.py`:

```python
"""
Nightly aggregation and retention.

The dashboard reads rollups exclusively. Raw UsageEvent rows exist only long
enough to be aggregated and to allow a short investigation window; after
PRODUCT_ANALYTICS_RETENTION_DAYS they are pruned. Rollups are kept forever, so
pruning does not lose dashboard history.
"""

import logging
from datetime import datetime, timedelta

from celery import shared_task
from django.conf import settings
from django.db.models import Count, Sum
from django.utils import timezone

from .constants import SOURCE_API, SOURCE_FRONTEND
from .models import ElementClickRollup, UsageDailyRollup, UsageEvent


logger = logging.getLogger(__name__)


@shared_task
def build_usage_rollups(for_date=None):
    """
    Aggregate one day of raw events into both rollup tables.

    Idempotent: existing rows for that day are replaced, so a re-run (or a
    retry after a partial failure) never double-counts.
    """
    if for_date:
        day = datetime.fromisoformat(for_date).date()
    else:
        day = (timezone.now() - timedelta(days=1)).date()

    events = UsageEvent.objects.filter(occurred_at__date=day)

    UsageDailyRollup.objects.filter(day=day).delete()
    ElementClickRollup.objects.filter(day=day).delete()

    daily = (
        events.values("practice_id", "user_id", "source", "event_name", "route")
        .annotate(count=Count("id"), total_active_ms=Sum("duration_ms"))
        .order_by()
    )
    UsageDailyRollup.objects.bulk_create(
        [
            UsageDailyRollup(
                practice_id=row["practice_id"],
                user_id=row["user_id"],
                day=day,
                source=row["source"],
                event_name=row["event_name"],
                route=row["route"] or "",
                count=row["count"],
                total_active_ms=row["total_active_ms"] or 0,
            )
            for row in daily
        ],
        batch_size=500,
    )

    # Click heat: three event names collapse into three columns of one row per
    # (route, element), because the dashboard always wants them side by side.
    heat = {}
    click_events = events.filter(
        event_name__in=("click", "click.rage", "click.dead")
    ).exclude(element="")
    for row in (
        click_events.values("practice_id", "route", "element", "event_name")
        .annotate(count=Count("id"))
        .order_by()
    ):
        key = (row["practice_id"], row["route"] or "", row["element"])
        entry = heat.setdefault(key, {"count": 0, "rage_count": 0, "dead_count": 0})
        if row["event_name"] == "click":
            entry["count"] += row["count"]
        elif row["event_name"] == "click.rage":
            entry["rage_count"] += row["count"]
        else:
            entry["dead_count"] += row["count"]

    ElementClickRollup.objects.bulk_create(
        [
            ElementClickRollup(
                practice_id=practice_id,
                day=day,
                route=route,
                element=element,
                count=values["count"],
                rage_count=values["rage_count"],
                dead_count=values["dead_count"],
            )
            for (practice_id, route, element), values in heat.items()
        ],
        batch_size=500,
    )

    result = {
        "day": day.isoformat(),
        "daily_rows": UsageDailyRollup.objects.filter(day=day).count(),
        "element_rows": ElementClickRollup.objects.filter(day=day).count(),
    }
    logger.info("build_usage_rollups %s", result)
    return result


@shared_task
def prune_usage_events():
    """Delete raw events past the retention window. Rollups are untouched."""
    days = int(getattr(settings, "PRODUCT_ANALYTICS_RETENTION_DAYS", 90))
    cutoff = timezone.now() - timedelta(days=days)
    deleted, _ = UsageEvent.objects.filter(occurred_at__lt=cutoff).delete()
    logger.info("prune_usage_events deleted=%s cutoff=%s", deleted, cutoff)
    return {"deleted": deleted, "cutoff": cutoff.isoformat()}
```

- [ ] **Step 4: Schedule the tasks**

In `TreatmentPath/settings.py`, add inside `CELERY_BEAT_SCHEDULE` (alongside the other 2am aggregations):

```python
    # Product analytics rollup — 3:30 AM, after the other nightly aggregations
    "build-product-analytics-rollups": {
        "task": "productAnalytics.tasks.build_usage_rollups",
        "schedule": crontab(hour=3, minute=30),
        "options": {"queue": "default"},
    },
    # Retention prune — 4:00 AM, strictly after the rollup has run
    "prune-product-analytics-events": {
        "task": "productAnalytics.tasks.prune_usage_events",
        "schedule": crontab(hour=4, minute=0),
        "options": {"queue": "default"},
    },
```

The prune runs *after* the rollup by design — pruning first would discard a day of events before they were aggregated.

- [ ] **Step 5: Run the tests to verify they pass**

```bash
python manage.py test productAnalytics.tests.test_rollups --keepdb -v 2
```

Expected: 7 tests PASS.

- [ ] **Step 6: Checkpoint**

`python manage.py test productAnalytics --keepdb`. Report pass count. Do not commit.

---

## Task 5: Superadmin dashboard query API

**Files:**
- Create: `productAnalytics/dashboard_views.py`
- Create: `productAnalytics/tests/test_dashboard.py`
- Modify: `productAnalytics/urls.py`

**Interfaces:**
- Consumes: `UsageDailyRollup`, `ElementClickRollup`, `Admin.permissions.IsSuperAdminUser`
- Produces these endpoints, all superadmin-only, all accepting `?days=N` (default 30):
  - `GET .../practices/` → `{"practices": [{"practice_id", "practice_name", "active_users", "events", "active_ms", "top_routes": [{"route", "count"}]}]}`
  - `GET .../practices/<int:practice_id>/users/` → `{"users": [{"user_id", "email", "events", "active_ms", "last_seen", "top_routes"}]}`
  - `GET .../practices/<int:practice_id>/features/` → `{"routes": [...], "users": [...], "matrix": [{"user_id", "route", "count"}]}`
  - `GET .../unused/` → `{"unused_endpoints": [str], "known_routes": [str]}` — endpoints are enumerated authoritatively from the URL resolver, so "unused" is exact for them; frontend routes are only knowable from events observed, hence `known_routes` rather than an unused list
  - `GET .../elements/?route=/patients/:id` → `{"elements": [{"element", "count", "rage_count", "dead_count"}]}`

- [ ] **Step 1: Write the failing dashboard test**

`productAnalytics/tests/test_dashboard.py`:

```python
from datetime import timedelta

from django.test import TestCase
from django.utils import timezone
from rest_framework.test import APIClient

from productAnalytics.constants import SOURCE_FRONTEND
from productAnalytics.models import ElementClickRollup, UsageDailyRollup
from UserAuthentication.models import Practice, User


class DashboardApiTests(TestCase):
    def setUp(self):
        self.practice = Practice.objects.create(
            name="Dashboard Practice", slug="dashboard-practice"
        )
        self.member = User.objects.create_user(
            email="dashboard-member@example.com", password="testpass123"
        )
        self.superadmin = User.objects.create_user(
            email="dashboard-super@example.com", password="testpass123"
        )
        self.superadmin.user_type = "superuser"
        self.superadmin.save(update_fields=["user_type"])

        self.staff = User.objects.create_user(
            email="dashboard-staff@example.com", password="testpass123"
        )
        self.staff.user_type = "staff"
        self.staff.is_staff = True
        self.staff.save(update_fields=["user_type", "is_staff"])

        today = timezone.now().date()
        UsageDailyRollup.objects.create(
            practice=self.practice,
            user=self.member,
            day=today,
            source=SOURCE_FRONTEND,
            event_name="page.view",
            route="/patients/:id",
            count=10,
            total_active_ms=60000,
        )
        UsageDailyRollup.objects.create(
            practice=self.practice,
            user=self.member,
            day=today,
            source=SOURCE_FRONTEND,
            event_name="page.view",
            route="/daylist",
            count=3,
            total_active_ms=15000,
        )
        ElementClickRollup.objects.create(
            practice=self.practice,
            day=today,
            route="/patients/:id",
            element="button#save",
            count=7,
            rage_count=2,
            dead_count=1,
        )

        self.client = APIClient()
        self.client.force_authenticate(user=self.superadmin)

    def test_practices_overview_ranks_routes_by_usage(self):
        response = self.client.get("/api/backend/product-analytics/practices/")

        self.assertEqual(response.status_code, 200)
        row = response.data["practices"][0]
        self.assertEqual(row["practice_id"], self.practice.id)
        self.assertEqual(row["active_users"], 1)
        self.assertEqual(row["top_routes"][0]["route"], "/patients/:id")

    def test_practice_users_drilldown(self):
        response = self.client.get(
            f"/api/backend/product-analytics/practices/{self.practice.id}/users/"
        )

        self.assertEqual(response.status_code, 200)
        user_row = response.data["users"][0]
        self.assertEqual(user_row["user_id"], self.member.id)
        self.assertEqual(user_row["events"], 13)

    def test_feature_matrix_returns_user_by_route_cells(self):
        response = self.client.get(
            f"/api/backend/product-analytics/practices/{self.practice.id}/features/"
        )

        self.assertEqual(response.status_code, 200)
        self.assertIn("/patients/:id", response.data["routes"])
        cell = next(
            c for c in response.data["matrix"] if c["route"] == "/patients/:id"
        )
        self.assertEqual(cell["count"], 10)

    def test_element_heat_for_one_route(self):
        response = self.client.get(
            "/api/backend/product-analytics/elements/", {"route": "/patients/:id"}
        )

        self.assertEqual(response.status_code, 200)
        element = response.data["elements"][0]
        self.assertEqual(element["element"], "button#save")
        self.assertEqual(element["rage_count"], 2)

    def test_staff_user_is_refused(self):
        """IsAdminUser would pass here — IsSuperAdminUser must not."""
        client = APIClient()
        client.force_authenticate(user=self.staff)

        response = client.get("/api/backend/product-analytics/practices/")

        self.assertEqual(response.status_code, 403)

    def test_anonymous_is_refused(self):
        response = APIClient().get("/api/backend/product-analytics/practices/")
        self.assertIn(response.status_code, (401, 403))

    def test_days_window_excludes_older_rollups(self):
        UsageDailyRollup.objects.update(day=timezone.now().date() - timedelta(days=60))

        response = self.client.get(
            "/api/backend/product-analytics/practices/", {"days": 30}
        )

        self.assertEqual(response.data["practices"], [])

    def test_unused_surfaces_lists_endpoints_with_no_recorded_usage(self):
        response = self.client.get("/api/backend/product-analytics/unused/")

        self.assertEqual(response.status_code, 200)
        # The app defines far more endpoints than this test exercised, so the
        # unused list must be non-empty and must not contain a route we DID use.
        self.assertGreater(len(response.data["unused_endpoints"]), 0)
        self.assertNotIn("/patients/:id", response.data["unused_endpoints"])
        self.assertIn("/patients/:id", response.data["known_routes"])
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
python manage.py test productAnalytics.tests.test_dashboard --keepdb -v 2
```

Expected: FAIL — 404 on every dashboard URL.

- [ ] **Step 3: Write the dashboard views**

`productAnalytics/dashboard_views.py`:

```python
"""
Superadmin query endpoints. These read ROLLUPS ONLY — never UsageEvent — so a
dashboard page load stays a handful of indexed aggregate queries regardless of
how much raw traffic the app is generating.
"""

from collections import defaultdict
from datetime import timedelta

from django.db.models import Count, Max, Sum
from django.utils import timezone
from rest_framework.response import Response
from rest_framework.views import APIView

from Admin.permissions import IsSuperAdminUser

from .constants import SOURCE_API, SOURCE_FRONTEND
from .models import ElementClickRollup, UsageDailyRollup


DEFAULT_DAYS = 30
MAX_DAYS = 365
TOP_ROUTES_PER_PRACTICE = 3


def _window_start(request):
    try:
        days = int(request.query_params.get("days", DEFAULT_DAYS))
    except (TypeError, ValueError):
        days = DEFAULT_DAYS
    days = max(1, min(days, MAX_DAYS))
    return timezone.now().date() - timedelta(days=days)


class PracticesOverviewView(APIView):
    permission_classes = [IsSuperAdminUser]

    def get(self, request):
        since = _window_start(request)
        rollups = UsageDailyRollup.objects.filter(day__gte=since)

        totals = (
            rollups.values("practice_id", "practice__name")
            .annotate(
                events=Sum("count"),
                active_ms=Sum("total_active_ms"),
                active_users=Count("user_id", distinct=True),
                last_seen=Max("day"),
            )
            .order_by("-events")
        )

        top_routes = defaultdict(list)
        route_totals = (
            rollups.filter(source=SOURCE_FRONTEND)
            .exclude(route="")
            .values("practice_id", "route")
            .annotate(count=Sum("count"))
            .order_by("practice_id", "-count")
        )
        for row in route_totals:
            bucket = top_routes[row["practice_id"]]
            if len(bucket) < TOP_ROUTES_PER_PRACTICE:
                bucket.append({"route": row["route"], "count": row["count"]})

        return Response(
            {
                "practices": [
                    {
                        "practice_id": row["practice_id"],
                        "practice_name": row["practice__name"],
                        "events": row["events"] or 0,
                        "active_ms": row["active_ms"] or 0,
                        "active_users": row["active_users"] or 0,
                        "last_seen": row["last_seen"],
                        "top_routes": top_routes.get(row["practice_id"], []),
                    }
                    for row in totals
                ]
            }
        )


class PracticeUsersView(APIView):
    permission_classes = [IsSuperAdminUser]

    def get(self, request, practice_id):
        since = _window_start(request)
        rollups = UsageDailyRollup.objects.filter(
            practice_id=practice_id, day__gte=since, user_id__isnull=False
        )

        totals = (
            rollups.values("user_id", "user__email")
            .annotate(
                events=Sum("count"),
                active_ms=Sum("total_active_ms"),
                last_seen=Max("day"),
            )
            .order_by("-events")
        )

        top_routes = defaultdict(list)
        for row in (
            rollups.filter(source=SOURCE_FRONTEND)
            .exclude(route="")
            .values("user_id", "route")
            .annotate(count=Sum("count"))
            .order_by("user_id", "-count")
        ):
            bucket = top_routes[row["user_id"]]
            if len(bucket) < TOP_ROUTES_PER_PRACTICE:
                bucket.append({"route": row["route"], "count": row["count"]})

        return Response(
            {
                "users": [
                    {
                        "user_id": row["user_id"],
                        "email": row["user__email"],
                        "events": row["events"] or 0,
                        "active_ms": row["active_ms"] or 0,
                        "last_seen": row["last_seen"],
                        "top_routes": top_routes.get(row["user_id"], []),
                    }
                    for row in totals
                ]
            }
        )


class PracticeFeatureMatrixView(APIView):
    """Users × routes intensity grid — the per-practice heatmap."""

    permission_classes = [IsSuperAdminUser]

    def get(self, request, practice_id):
        since = _window_start(request)
        cells = (
            UsageDailyRollup.objects.filter(
                practice_id=practice_id,
                day__gte=since,
                source=SOURCE_FRONTEND,
                user_id__isnull=False,
            )
            .exclude(route="")
            .values("user_id", "user__email", "route")
            .annotate(count=Sum("count"))
            .order_by("-count")
        )

        routes = []
        users = []
        matrix = []
        seen_users = set()
        for row in cells:
            if row["route"] not in routes:
                routes.append(row["route"])
            if row["user_id"] not in seen_users:
                seen_users.add(row["user_id"])
                users.append({"user_id": row["user_id"], "email": row["user__email"]})
            matrix.append(
                {
                    "user_id": row["user_id"],
                    "route": row["route"],
                    "count": row["count"],
                }
            )

        return Response({"routes": routes, "users": users, "matrix": matrix})


class UnusedSurfacesView(APIView):
    """
    Routes and API endpoints the app defines but which saw zero usage in the
    window. This is the 'what did we build that nobody opens' list.
    """

    permission_classes = [IsSuperAdminUser]

    def get(self, request):
        since = _window_start(request)
        used = set(
            UsageDailyRollup.objects.filter(day__gte=since)
            .exclude(route="")
            .values_list("route", flat=True)
            .distinct()
        )

        all_endpoints = self._all_api_routes()
        unused_endpoints = sorted(all_endpoints - used)

        # Frontend routes are only knowable from events we have seen, so this
        # list reports endpoints authoritatively and routes best-effort.
        return Response(
            {
                "unused_endpoints": unused_endpoints,
                "known_routes": sorted(used),
            }
        )

    def _all_api_routes(self):
        from django.urls import get_resolver

        routes = set()

        def walk(resolver, prefix=""):
            for pattern in resolver.url_patterns:
                if hasattr(pattern, "url_patterns"):
                    walk(pattern, prefix + str(pattern.pattern))
                else:
                    routes.add(prefix + str(pattern.pattern))

        walk(get_resolver())
        return {r for r in routes if r.startswith("api/backend/")}


class ElementHeatView(APIView):
    permission_classes = [IsSuperAdminUser]

    def get(self, request):
        route = request.query_params.get("route", "")
        if not route:
            return Response({"elements": []})

        since = _window_start(request)
        query = ElementClickRollup.objects.filter(route=route, day__gte=since)

        practice_id = request.query_params.get("practice_id")
        if practice_id:
            query = query.filter(practice_id=practice_id)

        rows = (
            query.values("element")
            .annotate(
                count=Sum("count"),
                rage_count=Sum("rage_count"),
                dead_count=Sum("dead_count"),
            )
            .order_by("-count")
        )

        return Response({"elements": list(rows)})
```

- [ ] **Step 4: Add the dashboard routes**

Replace `productAnalytics/urls.py` with:

```python
from django.urls import path

from .dashboard_views import (
    ElementHeatView,
    PracticeFeatureMatrixView,
    PracticeUsersView,
    PracticesOverviewView,
    UnusedSurfacesView,
)
from .views import UsageEventIngestView


urlpatterns = [
    path("events/", UsageEventIngestView.as_view(), name="product-analytics-ingest"),
    path(
        "practices/",
        PracticesOverviewView.as_view(),
        name="product-analytics-practices",
    ),
    path(
        "practices/<int:practice_id>/users/",
        PracticeUsersView.as_view(),
        name="product-analytics-practice-users",
    ),
    path(
        "practices/<int:practice_id>/features/",
        PracticeFeatureMatrixView.as_view(),
        name="product-analytics-practice-features",
    ),
    path("unused/", UnusedSurfacesView.as_view(), name="product-analytics-unused"),
    path("elements/", ElementHeatView.as_view(), name="product-analytics-elements"),
]
```

- [ ] **Step 5: Run the tests to verify they pass**

```bash
python manage.py test productAnalytics.tests.test_dashboard --keepdb -v 2
```

Expected: 7 tests PASS.

- [ ] **Step 6: Checkpoint**

`python manage.py test productAnalytics --keepdb`. Report total pass count across all four test modules. Do not commit.

---

## Task 6: Frontend SDK core — session, queue, transport

**Files:**
- Create: `src/lib/analytics/types.ts`, `src/lib/analytics/session.ts`, `src/lib/analytics/queue.ts`
- Create: `src/lib/analytics/queue.test.ts`
- Modify: `src/config/api.ts`

**Interfaces:**
- Consumes: `TokenManager.getToken()` from `@/contexts/AuthContext`, `API_ENDPOINTS.productAnalytics.ingest()`
- Produces:
  - `types.ts`: `type EventName`, `interface AnalyticsEvent { event_name: EventName; route?: string; element?: string; duration_ms?: number; occurred_at: string; session_id: string; app_version?: string; props?: Record<string, string | number | boolean>; }`
  - `session.ts`: `getSessionId(): string`, `getAppVersion(): string`
  - `queue.ts`: `enqueue(event: AnalyticsEvent): void`, `flush(): Promise<void>`, `resetQueue(): void`, `MAX_QUEUE_BEFORE_FLUSH = 20`, `FLUSH_INTERVAL_MS = 5000`

- [ ] **Step 1: Write the failing queue test**

`src/lib/analytics/queue.test.ts`:

```typescript
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";

import { enqueue, flush, resetQueue, MAX_QUEUE_BEFORE_FLUSH } from "./queue";
import type { AnalyticsEvent } from "./types";

const anEvent = (overrides: Partial<AnalyticsEvent> = {}): AnalyticsEvent => ({
  event_name: "page.view",
  route: "/patients/:id",
  occurred_at: new Date().toISOString(),
  session_id: "sess-test",
  ...overrides,
});

describe("analytics queue", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    vi.useFakeTimers();
    resetQueue();
    fetchMock = vi.fn().mockResolvedValue({ ok: true });
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => {
    vi.useRealTimers();
    vi.unstubAllGlobals();
  });

  it("does not send until a flush trigger fires", () => {
    enqueue(anEvent());
    expect(fetchMock).not.toHaveBeenCalled();
  });

  it("auto-flushes once the queue reaches the batch size", async () => {
    for (let i = 0; i < MAX_QUEUE_BEFORE_FLUSH; i += 1) enqueue(anEvent());
    await vi.runAllTimersAsync();

    expect(fetchMock).toHaveBeenCalledTimes(1);
    const body = JSON.parse(fetchMock.mock.calls[0][1].body);
    expect(body.events).toHaveLength(MAX_QUEUE_BEFORE_FLUSH);
  });

  it("sends with keepalive so the last batch survives navigation away", async () => {
    enqueue(anEvent());
    await flush();

    expect(fetchMock.mock.calls[0][1].keepalive).toBe(true);
  });

  it("never sends identity fields — the server derives them", async () => {
    enqueue(anEvent());
    await flush();

    const raw = fetchMock.mock.calls[0][1].body as string;
    expect(raw).not.toContain("practice_id");
    expect(raw).not.toContain("user_id");
  });

  it("drops the batch on failure rather than retrying forever", async () => {
    fetchMock.mockRejectedValueOnce(new Error("network down"));
    enqueue(anEvent());
    await flush();
    await flush();

    expect(fetchMock).toHaveBeenCalledTimes(1);
  });

  it("resetQueue discards pending events without sending them", async () => {
    enqueue(anEvent());
    resetQueue();
    await flush();

    expect(fetchMock).not.toHaveBeenCalled();
  });

  it("flush is a no-op when the queue is empty", async () => {
    await flush();
    expect(fetchMock).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npx vitest run src/lib/analytics/queue.test.ts
```

Expected: FAIL — cannot resolve `./queue`.

- [ ] **Step 3: Write types and session**

`src/lib/analytics/types.ts`:

```typescript
/**
 * The event shape the SDK sends. Deliberately narrow.
 *
 * There is no practice or user field: identity is derived server-side from the
 * JWT. A client that could assert who it is would be a spoofing surface, and
 * one that could send arbitrary strings would be a PHI leak.
 */
export type EventName =
  | "page.view"
  | "page.leave"
  | "click"
  | "click.rage"
  | "click.dead"
  | "scroll.depth";

export interface AnalyticsEvent {
  event_name: EventName;
  /** Route PATTERN (`/patients/:id`), never a concrete path. */
  route?: string;
  /** Structural or data-track descriptor. Never innerText or aria-label. */
  element?: string;
  duration_ms?: number;
  occurred_at: string;
  session_id: string;
  app_version?: string;
  props?: Record<string, string | number | boolean>;
}
```

`src/lib/analytics/session.ts`:

```typescript
/**
 * Per-tab session identity. Stored in sessionStorage so a reload keeps the
 * same session but a new tab starts a fresh one — which is what "session"
 * means for dwell-time purposes.
 */

const SESSION_KEY = "tp_analytics_session";

let cachedSessionId: string | null = null;

const newId = (): string => {
  if (typeof crypto !== "undefined" && "randomUUID" in crypto) {
    return crypto.randomUUID();
  }
  return `sess-${Date.now()}-${Math.random().toString(36).slice(2, 10)}`;
};

export const getSessionId = (): string => {
  if (cachedSessionId) return cachedSessionId;
  try {
    const stored = sessionStorage.getItem(SESSION_KEY);
    if (stored) {
      cachedSessionId = stored;
      return stored;
    }
    const created = newId();
    sessionStorage.setItem(SESSION_KEY, created);
    cachedSessionId = created;
    return created;
  } catch {
    // Private mode / storage disabled: fall back to an in-memory id.
    cachedSessionId = newId();
    return cachedSessionId;
  }
};

export const resetSession = (): void => {
  cachedSessionId = null;
  try {
    sessionStorage.removeItem(SESSION_KEY);
  } catch {
    // no-op
  }
};

export const getAppVersion = (): string =>
  (import.meta.env.VITE_APP_VERSION as string | undefined) ?? "";
```

- [ ] **Step 4: Write the queue**

`src/lib/analytics/queue.ts`:

```typescript
/**
 * Batching + transport.
 *
 * Analytics is a passenger: a send failure drops the batch and moves on. It
 * never retries indefinitely, never blocks, and never surfaces an error.
 */

import { API_ENDPOINTS } from "@/config/api";
import { TokenManager } from "@/contexts/AuthContext";

import type { AnalyticsEvent } from "./types";

export const MAX_QUEUE_BEFORE_FLUSH = 20;
export const FLUSH_INTERVAL_MS = 5000;

let queue: AnalyticsEvent[] = [];
let timer: ReturnType<typeof setTimeout> | null = null;

const clearTimer = () => {
  if (timer !== null) {
    clearTimeout(timer);
    timer = null;
  }
};

export const flush = async (): Promise<void> => {
  clearTimer();
  if (queue.length === 0) return;

  const batch = queue;
  queue = [];

  const token = TokenManager.getToken();
  if (!token) return;

  try {
    await fetch(API_ENDPOINTS.productAnalytics.ingest(), {
      method: "POST",
      keepalive: true,
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${token}`,
      },
      body: JSON.stringify({ events: batch }),
    });
  } catch {
    // Dropped deliberately. Retrying would grow unbounded on a flaky network
    // and analytics must never compete with the app for bandwidth.
  }
};

export const enqueue = (event: AnalyticsEvent): void => {
  queue.push(event);

  if (queue.length >= MAX_QUEUE_BEFORE_FLUSH) {
    void flush();
    return;
  }

  if (timer === null) {
    timer = setTimeout(() => {
      void flush();
    }, FLUSH_INTERVAL_MS);
  }
};

/**
 * Discard everything pending WITHOUT sending.
 *
 * Used on practice switch: events collected under the previous practice must
 * not be flushed with the new practice's token, or they land attributed to the
 * wrong practice.
 */
export const resetQueue = (): void => {
  clearTimer();
  queue = [];
};

export const pendingCount = (): number => queue.length;
```

- [ ] **Step 5: Add the API endpoint definitions**

In `src/config/api.ts`, add a new top-level key inside `API_ENDPOINTS`:

```typescript
  // Product analytics — usage capture ingest + superadmin dashboard queries
  productAnalytics: {
    ingest: () => getApiUrl("/product-analytics/events/"),
    practices: (days = 30) => getApiUrl(`/product-analytics/practices/?days=${days}`),
    practiceUsers: (practiceId: number, days = 30) =>
      getApiUrl(`/product-analytics/practices/${practiceId}/users/?days=${days}`),
    practiceFeatures: (practiceId: number, days = 30) =>
      getApiUrl(`/product-analytics/practices/${practiceId}/features/?days=${days}`),
    unused: (days = 30) => getApiUrl(`/product-analytics/unused/?days=${days}`),
    elements: (route: string, days = 30) =>
      getApiUrl(
        `/product-analytics/elements/?route=${encodeURIComponent(route)}&days=${days}`,
      ),
  },
```

Confirm that `getApiUrl` prefixes `/api/backend` — check an existing entry such as `individualauth.login` which produces `/auth/login/`. If `getApiUrl` does **not** add the `/api/backend` prefix, include it in the paths above to match whatever the neighbouring entries do.

- [ ] **Step 6: Run the tests to verify they pass**

```bash
npx vitest run src/lib/analytics/queue.test.ts
```

Expected: 7 tests PASS.

- [ ] **Step 7: Checkpoint**

Run `npx tsc -p tsconfig.app.json 2>&1 | grep -c "error TS"` and compare to the pre-existing baseline (~491). Report the delta — it must be zero. Do not commit.

---

## Task 7: Frontend autocapture — route names, dwell, clicks, scroll

**Files:**
- Create: `src/lib/analytics/routeName.ts`, `src/lib/analytics/elementId.ts`, `src/lib/analytics/activeTime.ts`, `src/lib/analytics/clicks.ts`, `src/lib/analytics/scroll.ts`, `src/lib/analytics/analytics.ts`
- Create: `src/lib/analytics/routeName.test.ts`, `src/lib/analytics/elementId.test.ts`, `src/lib/analytics/activeTime.test.ts`, `src/lib/analytics/clicks.test.ts`

**Interfaces:**
- Consumes: `appRoutes` from `@/routes`, `matchRoutes` from `react-router-dom`, `enqueue` / `flush` / `resetQueue` from `./queue`, `getSessionId` / `getAppVersion` from `./session`
- Produces:
  - `routeName.ts`: `normalizeRoute(pathname: string): string`
  - `elementId.ts`: `describeElement(el: Element): string`, `findInteractive(target: EventTarget | null): Element | null`
  - `activeTime.ts`: `class ActiveTimer { start(): void; stop(): { totalMs: number; activeMs: number }; }`
  - `clicks.ts`: `installClickCapture(getRoute: () => string): () => void`
  - `scroll.ts`: `installScrollTracking(): { maxDepth(): number; reset(): void; teardown(): void }`
  - `analytics.ts`: `track(name: EventName, fields?: Partial<AnalyticsEvent>): void`, `trackPageView(route: string): void`, `trackPageLeave(route, totalMs, activeMs): void`

- [ ] **Step 1: Write the failing route-normalization test**

`src/lib/analytics/routeName.test.ts`:

```typescript
import { describe, expect, it } from "vitest";

import { normalizeRoute } from "./routeName";

describe("normalizeRoute", () => {
  it("returns the route PATTERN, not the concrete path", () => {
    // Any real app route with a param works; /patients/:id is declared in
    // workspace.routes.tsx. Assert on the shape, not on a hardcoded string,
    // so this test survives a route rename.
    const result = normalizeRoute("/patients/8412");
    expect(result).not.toContain("8412");
  });

  it("never leaks a numeric record id", () => {
    expect(normalizeRoute("/patients/8412")).not.toMatch(/\d{3,}/);
  });

  it("returns the path unchanged for a static route", () => {
    expect(normalizeRoute("/dashboard")).toBe("/dashboard");
  });

  it("falls back to a safe placeholder for an unmatched path", () => {
    expect(normalizeRoute("/no/such/route/12345")).toBe("unmatched");
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

```bash
npx vitest run src/lib/analytics/routeName.test.ts
```

Expected: FAIL — cannot resolve `./routeName`.

- [ ] **Step 3: Write route normalization**

`src/lib/analytics/routeName.ts`:

```typescript
/**
 * URL → route PATTERN.
 *
 * `/patients/8412` becomes `/patients/:id`. This is the single most important
 * privacy control in the SDK: without it every record id becomes its own row,
 * which is both useless for aggregation and puts record ids in the analytics
 * store.
 *
 * We match against `appRoutes` (the flat route table in src/routes/index.ts)
 * rather than `routeObjects`, because the flat table already carries exactly
 * the `path` patterns we want and none of the wrapper nesting.
 */

import { matchRoutes } from "react-router-dom";

import { appRoutes } from "@/routes";

const PATTERNS = appRoutes
  .filter((route) => Boolean(route.path))
  .map((route) => ({ path: route.path }));

export const normalizeRoute = (pathname: string): string => {
  try {
    const matches = matchRoutes(PATTERNS, pathname);
    if (!matches || matches.length === 0) return "unmatched";
    const pattern = matches[matches.length - 1]?.route?.path;
    if (!pattern || pattern === "*") return "unmatched";
    return pattern;
  } catch {
    return "unmatched";
  }
};
```

- [ ] **Step 4: Write the failing element-descriptor test**

`src/lib/analytics/elementId.test.ts`:

```typescript
import { beforeEach, describe, expect, it } from "vitest";

import { describeElement, findInteractive } from "./elementId";

describe("describeElement", () => {
  beforeEach(() => {
    document.body.innerHTML = "";
  });

  it("prefers an explicit data-track name", () => {
    document.body.innerHTML = `<button data-track="save-patient">Save</button>`;
    const el = document.querySelector("button")!;
    expect(describeElement(el)).toBe("save-patient");
  });

  it("falls back to data-testid", () => {
    document.body.innerHTML = `<button data-testid="save-btn">Save</button>`;
    const el = document.querySelector("button")!;
    expect(describeElement(el)).toBe("save-btn");
  });

  it("NEVER includes innerText", () => {
    document.body.innerHTML = `<button>Open patient John Smith</button>`;
    const el = document.querySelector("button")!;
    const descriptor = describeElement(el);

    expect(descriptor).not.toContain("John");
    expect(descriptor).not.toContain("Smith");
  });

  it("NEVER includes aria-label", () => {
    document.body.innerHTML = `<button aria-label="Call Jane Doe on 07700900000">x</button>`;
    const el = document.querySelector("button")!;
    const descriptor = describeElement(el);

    expect(descriptor).not.toContain("Jane");
    expect(descriptor).not.toContain("07700900000");
  });

  it("produces a stable structural descriptor for an unlabeled element", () => {
    document.body.innerHTML = `<div><button>a</button><button>b</button></div>`;
    const [first, second] = Array.from(document.querySelectorAll("button"));

    expect(describeElement(first)).toBe(describeElement(first));
    expect(describeElement(first)).not.toBe(describeElement(second));
  });
});

describe("findInteractive", () => {
  beforeEach(() => {
    document.body.innerHTML = "";
  });

  it("walks up from an inner span to the enclosing button", () => {
    document.body.innerHTML = `<button data-track="save"><span id="inner">Save</span></button>`;
    const inner = document.querySelector("#inner")!;

    expect(findInteractive(inner)?.tagName).toBe("BUTTON");
  });

  it("returns null for a click on inert text", () => {
    document.body.innerHTML = `<p id="text">Some notes</p>`;
    expect(findInteractive(document.querySelector("#text"))).toBeNull();
  });
});
```

- [ ] **Step 5: Write the element descriptor**

`src/lib/analytics/elementId.ts`:

```typescript
/**
 * Element → PHI-free descriptor.
 *
 * HARD RULE: we never read innerText and never read aria-label. This codebase
 * has 765 aria-labels, and labels of the form "Open patient John Smith" are
 * exactly how a patient name would end up in an analytics table. Structural
 * descriptors are less readable but safe by construction.
 *
 * Readability improves over time by adding `data-track` to elements the
 * dashboard flags as high-traffic-but-unlabeled.
 */

const INTERACTIVE_SELECTOR = [
  "button",
  "a",
  "input",
  "select",
  "textarea",
  '[role="button"]',
  '[role="tab"]',
  '[role="menuitem"]',
  '[role="option"]',
  '[role="switch"]',
].join(",");

const MAX_DEPTH = 3;

export const findInteractive = (target: EventTarget | null): Element | null => {
  if (!target || !(target instanceof Element)) return null;
  return target.closest(INTERACTIVE_SELECTOR);
};

const indexAmongSiblings = (el: Element): number => {
  const parent = el.parentElement;
  if (!parent) return 0;
  return Array.from(parent.children).indexOf(el);
};

export const describeElement = (el: Element): string => {
  const track = el.getAttribute("data-track");
  if (track) return track.slice(0, 200);

  const testId = el.getAttribute("data-testid");
  if (testId) return testId.slice(0, 200);

  // Structural fallback: tag + role + position, up to MAX_DEPTH ancestors.
  const segments: string[] = [];
  let node: Element | null = el;
  let depth = 0;

  while (node && depth < MAX_DEPTH) {
    const tag = node.tagName.toLowerCase();
    const role = node.getAttribute("role");
    const position = indexAmongSiblings(node);
    segments.unshift(role ? `${tag}[${role}]:${position}` : `${tag}:${position}`);
    node = node.parentElement;
    depth += 1;
  }

  return segments.join(">").slice(0, 200);
};
```

- [ ] **Step 6: Write the failing active-time test**

`src/lib/analytics/activeTime.test.ts`:

```typescript
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";

import { ActiveTimer, IDLE_AFTER_MS } from "./activeTime";

describe("ActiveTimer", () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.useRealTimers();
  });

  it("counts elapsed wall time as total", () => {
    const timer = new ActiveTimer();
    timer.start();
    vi.advanceTimersByTime(10_000);

    expect(timer.stop().totalMs).toBeGreaterThanOrEqual(10_000);
  });

  it("stops accruing active time once idle", () => {
    const timer = new ActiveTimer();
    timer.start();

    vi.advanceTimersByTime(IDLE_AFTER_MS + 60_000);
    const { totalMs, activeMs } = timer.stop();

    expect(totalMs).toBeGreaterThan(activeMs);
    expect(activeMs).toBeLessThanOrEqual(IDLE_AFTER_MS + 1000);
  });

  it("resumes accruing active time after user input", () => {
    const timer = new ActiveTimer();
    timer.start();

    vi.advanceTimersByTime(IDLE_AFTER_MS + 5_000);
    document.dispatchEvent(new Event("keydown"));
    vi.advanceTimersByTime(3_000);

    const { activeMs } = timer.stop();
    expect(activeMs).toBeGreaterThan(IDLE_AFTER_MS);
  });

  it("a tab left open overnight does not report hours of engagement", () => {
    const timer = new ActiveTimer();
    timer.start();

    vi.advanceTimersByTime(9 * 60 * 60 * 1000);
    const { activeMs } = timer.stop();

    expect(activeMs).toBeLessThan(5 * 60 * 1000);
  });
});
```

- [ ] **Step 7: Write the active timer**

`src/lib/analytics/activeTime.ts`:

```typescript
/**
 * Dwell time, split into total (wall clock) and active (user actually there).
 *
 * Without the idle and visibility pauses, a tab left open overnight reports
 * nine hours of "engagement" and every dwell metric in the dashboard becomes
 * noise. This is the difference between a usable time-on-page number and a
 * meaningless one.
 */

export const IDLE_AFTER_MS = 60_000;

const ACTIVITY_EVENTS = ["keydown", "mousedown", "mousemove", "scroll", "touchstart"];

export class ActiveTimer {
  private startedAt = 0;
  private activeMs = 0;
  private lastTick = 0;
  private lastActivity = 0;
  private interval: ReturnType<typeof setInterval> | null = null;
  private onActivity = () => {
    this.lastActivity = Date.now();
  };

  start(): void {
    const now = Date.now();
    this.startedAt = now;
    this.lastTick = now;
    this.lastActivity = now;
    this.activeMs = 0;

    ACTIVITY_EVENTS.forEach((name) =>
      document.addEventListener(name, this.onActivity, { passive: true }),
    );

    this.interval = setInterval(() => this.accrue(), 1000);
  }

  private accrue(): void {
    const now = Date.now();
    const sinceTick = now - this.lastTick;
    this.lastTick = now;

    const idle = now - this.lastActivity > IDLE_AFTER_MS;
    const hidden = typeof document !== "undefined" && document.hidden;

    if (!idle && !hidden) {
      this.activeMs += sinceTick;
    }
  }

  stop(): { totalMs: number; activeMs: number } {
    this.accrue();

    if (this.interval !== null) {
      clearInterval(this.interval);
      this.interval = null;
    }
    ACTIVITY_EVENTS.forEach((name) =>
      document.removeEventListener(name, this.onActivity),
    );

    return {
      totalMs: Date.now() - this.startedAt,
      activeMs: this.activeMs,
    };
  }
}
```

- [ ] **Step 8: Write the failing click-capture test**

`src/lib/analytics/clicks.test.ts`:

```typescript
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";

import { installClickCapture, RAGE_CLICK_THRESHOLD, RAGE_WINDOW_MS } from "./clicks";
import * as queue from "./queue";

describe("click autocapture", () => {
  let enqueueSpy: ReturnType<typeof vi.spyOn>;
  let teardown: () => void;

  beforeEach(() => {
    vi.useFakeTimers();
    document.body.innerHTML = "";
    enqueueSpy = vi.spyOn(queue, "enqueue").mockImplementation(() => undefined);
    teardown = installClickCapture(() => "/patients/:id");
  });

  afterEach(() => {
    teardown();
    vi.useRealTimers();
    vi.restoreAllMocks();
  });

  it("captures a click on an interactive element", () => {
    document.body.innerHTML = `<button data-track="save">Save</button>`;
    document.querySelector("button")!.click();

    expect(enqueueSpy).toHaveBeenCalled();
    const event = enqueueSpy.mock.calls[0][0];
    expect(event.event_name).toBe("click");
    expect(event.element).toBe("save");
    expect(event.route).toBe("/patients/:id");
  });

  it("ignores clicks on inert text", () => {
    document.body.innerHTML = `<p id="t">notes</p>`;
    (document.querySelector("#t") as HTMLElement).click();

    expect(enqueueSpy).not.toHaveBeenCalled();
  });

  it("emits click.rage after repeated fast clicks on the same element", () => {
    document.body.innerHTML = `<button data-track="stuck">x</button>`;
    const button = document.querySelector("button")!;

    for (let i = 0; i < RAGE_CLICK_THRESHOLD; i += 1) button.click();

    const names = enqueueSpy.mock.calls.map((call) => call[0].event_name);
    expect(names).toContain("click.rage");
  });

  it("does not emit click.rage when clicks are spread out", () => {
    document.body.innerHTML = `<button data-track="calm">x</button>`;
    const button = document.querySelector("button")!;

    for (let i = 0; i < RAGE_CLICK_THRESHOLD; i += 1) {
      button.click();
      vi.advanceTimersByTime(RAGE_WINDOW_MS + 100);
    }

    const names = enqueueSpy.mock.calls.map((call) => call[0].event_name);
    expect(names).not.toContain("click.rage");
  });

  it("never sends the element's visible text", () => {
    document.body.innerHTML = `<button>Call Jane Doe</button>`;
    document.querySelector("button")!.click();

    const serialized = JSON.stringify(enqueueSpy.mock.calls[0][0]);
    expect(serialized).not.toContain("Jane");
  });
});
```

- [ ] **Step 9: Write click capture**

`src/lib/analytics/clicks.ts`:

```typescript
/**
 * One delegated capture-phase listener for the whole app.
 *
 * Rage clicks (repeated fast clicks on one element) and dead clicks (a click
 * that changes nothing) are where broken and confusing UI shows up, so they get
 * their own event names rather than being buried in the click count.
 */

import { describeElement, findInteractive } from "./elementId";
import { enqueue } from "./queue";
import { getAppVersion, getSessionId } from "./session";

export const RAGE_CLICK_THRESHOLD = 3;
export const RAGE_WINDOW_MS = 1000;
export const DEAD_CLICK_WINDOW_MS = 500;

export const installClickCapture = (getRoute: () => string): (() => void) => {
  let lastElement = "";
  let lastClickAt = 0;
  let repeatCount = 0;

  const emit = (
    eventName: "click" | "click.rage" | "click.dead",
    element: string,
    props?: Record<string, string | number | boolean>,
  ) => {
    enqueue({
      event_name: eventName,
      route: getRoute(),
      element,
      occurred_at: new Date().toISOString(),
      session_id: getSessionId(),
      app_version: getAppVersion(),
      props,
    });
  };

  const onClick = (event: MouseEvent) => {
    const target = findInteractive(event.target);
    if (!target) return;

    const element = describeElement(target);
    const now = Date.now();

    emit("click", element);

    // Rage detection
    if (element === lastElement && now - lastClickAt <= RAGE_WINDOW_MS) {
      repeatCount += 1;
      if (repeatCount >= RAGE_CLICK_THRESHOLD) {
        emit("click.rage", element, { repeats: repeatCount });
        repeatCount = 0;
      }
    } else {
      repeatCount = 1;
    }
    lastElement = element;
    lastClickAt = now;

    // Dead-click detection: did anything change?
    const pathBefore = window.location.pathname;
    let mutated = false;
    const observer = new MutationObserver(() => {
      mutated = true;
    });
    observer.observe(document.body, { childList: true, subtree: true, attributes: true });

    setTimeout(() => {
      observer.disconnect();
      const navigated = window.location.pathname !== pathBefore;
      if (!mutated && !navigated) {
        emit("click.dead", element);
      }
    }, DEAD_CLICK_WINDOW_MS);
  };

  document.addEventListener("click", onClick, true);

  return () => {
    document.removeEventListener("click", onClick, true);
  };
};
```

- [ ] **Step 10: Write scroll tracking and the public surface**

`src/lib/analytics/scroll.ts`:

```typescript
/**
 * Max scroll depth reached on the current route, bucketed to 25% steps.
 * Bucketing keeps the cardinality low — exact pixel depths would produce a
 * distinct value per viewport size and aggregate to nothing useful.
 */

const bucket = (fraction: number): number => {
  if (fraction >= 0.95) return 100;
  if (fraction >= 0.75) return 75;
  if (fraction >= 0.5) return 50;
  if (fraction >= 0.25) return 25;
  return 0;
};

export const installScrollTracking = () => {
  let maxFraction = 0;

  const onScroll = () => {
    const scrollable = document.documentElement.scrollHeight - window.innerHeight;
    if (scrollable <= 0) {
      maxFraction = 1;
      return;
    }
    const fraction = window.scrollY / scrollable;
    if (fraction > maxFraction) maxFraction = fraction;
  };

  window.addEventListener("scroll", onScroll, { passive: true });

  return {
    maxDepth: () => bucket(maxFraction),
    reset: () => {
      maxFraction = 0;
    },
    teardown: () => window.removeEventListener("scroll", onScroll),
  };
};
```

`src/lib/analytics/analytics.ts`:

```typescript
/**
 * Public surface of the analytics SDK.
 *
 * Application code does not need to call any of this — AnalyticsProvider wires
 * up autocapture. `track` exists so that a future high-value action can be
 * named explicitly without reworking the pipeline.
 */

import { enqueue, flush, resetQueue } from "./queue";
import { getAppVersion, getSessionId, resetSession } from "./session";
import type { AnalyticsEvent, EventName } from "./types";

export const track = (
  eventName: EventName,
  fields: Partial<AnalyticsEvent> = {},
): void => {
  enqueue({
    event_name: eventName,
    occurred_at: new Date().toISOString(),
    session_id: getSessionId(),
    app_version: getAppVersion(),
    ...fields,
  });
};

export const trackPageView = (route: string): void => {
  track("page.view", { route });
};

export const trackPageLeave = (
  route: string,
  totalMs: number,
  activeMs: number,
): void => {
  track("page.leave", {
    route,
    duration_ms: activeMs,
    props: { total_ms: totalMs },
  });
};

export const trackScrollDepth = (route: string, depth: number): void => {
  track("scroll.depth", { route, props: { depth } });
};

/** Discard pending events and start a fresh session. Used on practice switch. */
export const resetAnalytics = (): void => {
  resetQueue();
  resetSession();
};

export { flush };
```

- [ ] **Step 11: Run all SDK tests**

```bash
npx vitest run src/lib/analytics
```

Expected: all tests across `queue.test.ts`, `routeName.test.ts`, `elementId.test.ts`, `activeTime.test.ts`, `clicks.test.ts` PASS.

- [ ] **Step 12: Checkpoint**

Report per-file pass counts and the `tsc -p tsconfig.app.json` error delta (must be zero). Do not commit.

---

## Task 8: `AnalyticsProvider` — wire autocapture into the app

**Files:**
- Create: `src/lib/analytics/AnalyticsProvider.tsx`, `src/lib/analytics/AnalyticsProvider.test.tsx`
- Modify: `src/App.tsx`

**Interfaces:**
- Consumes: `useAuth` from `@/contexts/AuthContext`, `useLocation` from `react-router-dom`, everything from Tasks 6–7
- Produces: `<AnalyticsProvider>{children}</AnalyticsProvider>`

- [ ] **Step 1: Write the failing provider test**

`src/lib/analytics/AnalyticsProvider.test.tsx`:

```typescript
import { render } from "@testing-library/react";
import { MemoryRouter } from "react-router-dom";
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";

import { AnalyticsProvider } from "./AnalyticsProvider";
import * as analytics from "./analytics";

const mockAuth = { user: null as { practiceId?: string } | null };

vi.mock("@/contexts/AuthContext", async () => {
  const actual = await vi.importActual<typeof import("@/contexts/AuthContext")>(
    "@/contexts/AuthContext",
  );
  return {
    ...actual,
    useAuth: () => mockAuth,
  };
});

describe("AnalyticsProvider", () => {
  beforeEach(() => {
    mockAuth.user = { practiceId: "13" };
    vi.restoreAllMocks();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it("emits a page.view on mount", () => {
    const spy = vi.spyOn(analytics, "trackPageView");

    render(
      <MemoryRouter initialEntries={["/dashboard"]}>
        <AnalyticsProvider>
          <div>app</div>
        </AnalyticsProvider>
      </MemoryRouter>,
    );

    expect(spy).toHaveBeenCalledWith("/dashboard");
  });

  it("emits page.leave when the route changes", () => {
    const spy = vi.spyOn(analytics, "trackPageLeave");

    const { rerender } = render(
      <MemoryRouter initialEntries={["/dashboard"]}>
        <AnalyticsProvider>
          <div>app</div>
        </AnalyticsProvider>
      </MemoryRouter>,
    );

    rerender(
      <MemoryRouter initialEntries={["/daylist"]}>
        <AnalyticsProvider>
          <div>app</div>
        </AnalyticsProvider>
      </MemoryRouter>,
    );

    expect(spy).toHaveBeenCalled();
  });

  it("resets the queue when the practice changes", () => {
    /**
     * The regression test for this codebase's recurring trap: module-level
     * singletons that are not re-scoped on PracticeSwitcher's SOFT switch.
     * AuthContext, useFetchWithAuth, FeatureAccessContext and useTeamChat have
     * all shipped this bug. Without this reset, events collected under practice
     * A get flushed with practice B's token and land on the wrong practice.
     */
    const spy = vi.spyOn(analytics, "resetAnalytics");

    const { rerender } = render(
      <MemoryRouter>
        <AnalyticsProvider>
          <div>app</div>
        </AnalyticsProvider>
      </MemoryRouter>,
    );

    mockAuth.user = { practiceId: "16" };
    rerender(
      <MemoryRouter>
        <AnalyticsProvider>
          <div>app</div>
        </AnalyticsProvider>
      </MemoryRouter>,
    );

    expect(spy).toHaveBeenCalled();
  });

  it("collects nothing when no user is signed in", () => {
    mockAuth.user = null;
    const spy = vi.spyOn(analytics, "trackPageView");

    render(
      <MemoryRouter initialEntries={["/login"]}>
        <AnalyticsProvider>
          <div>app</div>
        </AnalyticsProvider>
      </MemoryRouter>,
    );

    expect(spy).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

```bash
npx vitest run src/lib/analytics/AnalyticsProvider.test.tsx
```

Expected: FAIL — cannot resolve `./AnalyticsProvider`.

- [ ] **Step 3: Write the provider**

`src/lib/analytics/AnalyticsProvider.tsx`:

```typescript
/**
 * The whole integration surface of the analytics SDK: mount this once, inside
 * AuthProvider and inside the router, and autocapture runs forever. There is
 * deliberately no per-page or per-component work to do.
 */

import { useEffect, useRef, type ReactNode } from "react";
import { useLocation } from "react-router-dom";

import { useAuth } from "@/contexts/AuthContext";

import { ActiveTimer } from "./activeTime";
import {
  flush,
  resetAnalytics,
  trackPageLeave,
  trackPageView,
  trackScrollDepth,
} from "./analytics";
import { installClickCapture } from "./clicks";
import { normalizeRoute } from "./routeName";
import { installScrollTracking } from "./scroll";

export function AnalyticsProvider({ children }: { children: ReactNode }) {
  const { user } = useAuth();
  const location = useLocation();

  const routeRef = useRef<string>("");
  const timerRef = useRef<ActiveTimer | null>(null);
  const scrollRef = useRef<ReturnType<typeof installScrollTracking> | null>(null);
  const practiceRef = useRef<string | undefined>(undefined);

  const signedIn = Boolean(user);

  // Practice switch: PracticeSwitcher's SOFT switch does not reload the app, so
  // this module-level queue must be flushed and reset by hand or events land on
  // the wrong practice. Four other singletons in this codebase have shipped
  // exactly this bug.
  useEffect(() => {
    const practiceId = user?.practiceId;
    if (practiceRef.current !== undefined && practiceRef.current !== practiceId) {
      resetAnalytics();
    }
    practiceRef.current = practiceId;
  }, [user?.practiceId]);

  // Click autocapture — installed once, reads the current route through a ref
  // so it never needs reinstalling on navigation.
  useEffect(() => {
    if (!signedIn) return undefined;
    const teardownClicks = installClickCapture(() => routeRef.current);
    scrollRef.current = installScrollTracking();
    return () => {
      teardownClicks();
      scrollRef.current?.teardown();
      scrollRef.current = null;
    };
  }, [signedIn]);

  // Page views + dwell.
  useEffect(() => {
    if (!signedIn) return undefined;

    const route = normalizeRoute(location.pathname);

    const closePrevious = () => {
      if (!routeRef.current || !timerRef.current) return;
      const { totalMs, activeMs } = timerRef.current.stop();
      trackPageLeave(routeRef.current, totalMs, activeMs);
      const depth = scrollRef.current?.maxDepth() ?? 0;
      if (depth > 0) trackScrollDepth(routeRef.current, depth);
      scrollRef.current?.reset();
    };

    closePrevious();

    routeRef.current = route;
    trackPageView(route);
    timerRef.current = new ActiveTimer();
    timerRef.current.start();

    return closePrevious;
  }, [location.pathname, signedIn]);

  // Flush on tab hide — this is the batch that would otherwise be lost when
  // someone closes the tab or switches away.
  useEffect(() => {
    const onHide = () => {
      if (document.hidden) void flush();
    };
    document.addEventListener("visibilitychange", onHide);
    return () => document.removeEventListener("visibilitychange", onHide);
  }, []);

  return <>{children}</>;
}
```

- [ ] **Step 4: Mount it in the app shell**

In `src/App.tsx`, add the import alongside the other context imports:

```typescript
import { AnalyticsProvider } from "@/lib/analytics/AnalyticsProvider";
```

Then wrap the existing tree. It must sit **inside** `<AuthProvider>` (it reads `useAuth`) and **inside** `<BrowserRouter>` (it reads `useLocation`). Place it immediately inside `<AuthProvider>`, wrapping the existing children:

```tsx
            <AuthProvider>
              <AnalyticsProvider>
                <ChromeAlertsManager />
                <SaveDeviceTrustPrompt />
                <PracticeSwitchOverlay />
                <FeatureAccessProvider>
                  {/* ...existing tree unchanged... */}
                </FeatureAccessProvider>
              </AnalyticsProvider>
            </AuthProvider>
```

Note the `ConfirmationApp` branch (the patient-facing `confirm.dental` page) is returned earlier and deliberately gets **no** providers — leave it untouched. Patient-facing pages are not tracked.

- [ ] **Step 5: Run the tests to verify they pass**

```bash
npx vitest run src/lib/analytics
```

Expected: all SDK tests plus the 4 provider tests PASS.

- [ ] **Step 6: Verify end to end against the running app**

Start the backend and frontend, sign in, click around three or four pages, wait 10 seconds, then confirm rows landed:

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py shell -c "
from productAnalytics.models import UsageEvent
print('total:', UsageEvent.objects.count())
for row in UsageEvent.objects.values('source','event_name','route').distinct()[:20]:
    print(row)
"
```

Expected: both `api` and `frontend` sources present; every `route` is a pattern (`/patients/:id`, `patients/<int:pk>/`) with no numeric ids; `element` values contain no human names.

- [ ] **Step 7: Checkpoint**

Report the distinct route list from Step 6 verbatim — it is the direct evidence that no record ids leaked. Report the `tsc` delta. Do not commit.

---

## Task 9: Superadmin dashboard page

**Files:**
- Create: `src/pages/admin/ProductAnalytics.tsx`
- Create: `src/pages/admin/product-analytics/useProductAnalytics.ts`
- Create: `src/pages/admin/product-analytics/PracticesTable.tsx`
- Create: `src/pages/admin/product-analytics/PracticeDrilldown.tsx`
- Create: `src/pages/admin/product-analytics/FeatureHeatmap.tsx`
- Create: `src/pages/admin/product-analytics/UnusedSurfaces.tsx`
- Create: `src/pages/admin/product-analytics/useProductAnalytics.test.tsx`
- Modify: `src/routes/admin.routes.tsx`, `src/components/admin/AdminSidebar.tsx`

**Interfaces:**
- Consumes: `API_ENDPOINTS.productAnalytics.*` from Task 6, the Task 5 endpoints
- Produces: route `/admin/product-analytics`; hook `useProductAnalytics(days: number)` returning `{ practices, isLoading, error }`; `usePracticeDrilldown(practiceId: number, days: number)` returning `{ users, matrix, routes, isLoading }`

- [ ] **Step 1: Write the failing hook test**

`src/pages/admin/product-analytics/useProductAnalytics.test.tsx`:

```typescript
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { renderHook, waitFor } from "@testing-library/react";
import type { ReactNode } from "react";
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";

import { useProductAnalytics } from "./useProductAnalytics";

const wrapper = ({ children }: { children: ReactNode }) => {
  const client = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
};

describe("useProductAnalytics", () => {
  beforeEach(() => {
    vi.stubGlobal(
      "fetch",
      vi.fn().mockResolvedValue({
        ok: true,
        json: async () => ({
          practices: [
            {
              practice_id: 13,
              practice_name: "Practice Mannie",
              events: 120,
              active_ms: 600000,
              active_users: 4,
              last_seen: "2026-08-24",
              top_routes: [{ route: "/patients/:id", count: 80 }],
            },
          ],
        }),
      }),
    );
  });

  afterEach(() => {
    vi.unstubAllGlobals();
  });

  it("returns practices ranked by usage", async () => {
    const { result } = renderHook(() => useProductAnalytics(30), { wrapper });

    await waitFor(() => expect(result.current.isLoading).toBe(false));
    expect(result.current.practices[0].practice_name).toBe("Practice Mannie");
    expect(result.current.practices[0].top_routes[0].route).toBe("/patients/:id");
  });

  it("requests the window it was given", async () => {
    renderHook(() => useProductAnalytics(7), { wrapper });

    await waitFor(() =>
      expect((globalThis.fetch as ReturnType<typeof vi.fn>).mock.calls[0][0]).toContain(
        "days=7",
      ),
    );
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

```bash
npx vitest run src/pages/admin/product-analytics/useProductAnalytics.test.tsx
```

Expected: FAIL — cannot resolve `./useProductAnalytics`.

- [ ] **Step 3: Write the data hook**

`src/pages/admin/product-analytics/useProductAnalytics.ts`:

```typescript
import { useQuery } from "@tanstack/react-query";

import { API_ENDPOINTS } from "@/config/api";
import { TokenManager } from "@/contexts/AuthContext";

export interface TopRoute {
  route: string;
  count: number;
}

export interface PracticeUsageRow {
  practice_id: number;
  practice_name: string;
  events: number;
  active_ms: number;
  active_users: number;
  last_seen: string | null;
  top_routes: TopRoute[];
}

export interface UserUsageRow {
  user_id: number;
  email: string;
  events: number;
  active_ms: number;
  last_seen: string | null;
  top_routes: TopRoute[];
}

export interface MatrixCell {
  user_id: number;
  route: string;
  count: number;
}

const authedGet = async <T>(url: string): Promise<T> => {
  const token = TokenManager.getToken();
  const response = await fetch(url, {
    headers: { Authorization: `Bearer ${token}` },
  });
  if (!response.ok) {
    throw new Error(`Request failed: ${response.status}`);
  }
  return (await response.json()) as T;
};

export const useProductAnalytics = (days: number) => {
  const query = useQuery({
    queryKey: ["product-analytics", "practices", days],
    queryFn: () =>
      authedGet<{ practices: PracticeUsageRow[] }>(
        API_ENDPOINTS.productAnalytics.practices(days),
      ),
  });

  return {
    practices: query.data?.practices ?? [],
    isLoading: query.isLoading,
    error: query.error,
  };
};

export const usePracticeDrilldown = (practiceId: number | null, days: number) => {
  const users = useQuery({
    queryKey: ["product-analytics", "practice-users", practiceId, days],
    enabled: practiceId !== null,
    queryFn: () =>
      authedGet<{ users: UserUsageRow[] }>(
        API_ENDPOINTS.productAnalytics.practiceUsers(practiceId as number, days),
      ),
  });

  const features = useQuery({
    queryKey: ["product-analytics", "practice-features", practiceId, days],
    enabled: practiceId !== null,
    queryFn: () =>
      authedGet<{
        routes: string[];
        users: { user_id: number; email: string }[];
        matrix: MatrixCell[];
      }>(API_ENDPOINTS.productAnalytics.practiceFeatures(practiceId as number, days)),
  });

  return {
    users: users.data?.users ?? [],
    routes: features.data?.routes ?? [],
    matrixUsers: features.data?.users ?? [],
    matrix: features.data?.matrix ?? [],
    isLoading: users.isLoading || features.isLoading,
  };
};
```

- [ ] **Step 4: Write the practices table**

`src/pages/admin/product-analytics/PracticesTable.tsx`:

```typescript
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";

import type { PracticeUsageRow } from "./useProductAnalytics";

const formatMinutes = (ms: number): string => `${Math.round(ms / 60000)}m`;

export function PracticesTable({
  practices,
  onSelect,
}: {
  practices: PracticeUsageRow[];
  onSelect: (practiceId: number) => void;
}) {
  if (practices.length === 0) {
    return (
      <p className="text-sm text-muted-foreground">
        No usage recorded in this window.
      </p>
    );
  }

  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead>Practice</TableHead>
          <TableHead className="text-right">Active users</TableHead>
          <TableHead className="text-right">Events</TableHead>
          <TableHead className="text-right">Active time</TableHead>
          <TableHead>Top features</TableHead>
          <TableHead>Last seen</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {practices.map((row) => (
          <TableRow
            key={row.practice_id}
            className="cursor-pointer"
            onClick={() => onSelect(row.practice_id)}
          >
            <TableCell className="font-medium">{row.practice_name}</TableCell>
            <TableCell className="text-right">{row.active_users}</TableCell>
            <TableCell className="text-right">{row.events}</TableCell>
            <TableCell className="text-right">{formatMinutes(row.active_ms)}</TableCell>
            <TableCell className="text-xs text-muted-foreground">
              {row.top_routes.map((route) => route.route).join(", ") || "—"}
            </TableCell>
            <TableCell>{row.last_seen ?? "—"}</TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  );
}
```

- [ ] **Step 5: Write the heatmap and drilldown**

`src/pages/admin/product-analytics/FeatureHeatmap.tsx`:

```typescript
import type { MatrixCell } from "./useProductAnalytics";

/**
 * Users × routes intensity grid. Colour is a single hue at varying opacity so
 * the grid reads as one scale rather than a rainbow, and stays legible in both
 * light and dark themes.
 */
export function FeatureHeatmap({
  routes,
  users,
  matrix,
}: {
  routes: string[];
  users: { user_id: number; email: string }[];
  matrix: MatrixCell[];
}) {
  if (routes.length === 0 || users.length === 0) {
    return <p className="text-sm text-muted-foreground">No feature usage recorded.</p>;
  }

  const lookup = new Map<string, number>();
  let max = 0;
  matrix.forEach((cell) => {
    lookup.set(`${cell.user_id}|${cell.route}`, cell.count);
    if (cell.count > max) max = cell.count;
  });

  return (
    <div className="overflow-x-auto">
      <table className="border-separate border-spacing-1 text-xs">
        <thead>
          <tr>
            <th className="text-left font-medium">User</th>
            {routes.map((route) => (
              <th key={route} className="max-w-24 truncate text-left font-normal">
                {route}
              </th>
            ))}
          </tr>
        </thead>
        <tbody>
          {users.map((user) => (
            <tr key={user.user_id}>
              <td className="whitespace-nowrap pr-2">{user.email}</td>
              {routes.map((route) => {
                const count = lookup.get(`${user.user_id}|${route}`) ?? 0;
                const intensity = max > 0 ? count / max : 0;
                return (
                  <td
                    key={route}
                    title={`${user.email} — ${route}: ${count}`}
                    className="h-6 w-10 rounded text-center"
                    style={{
                      backgroundColor: `rgb(37 99 235 / ${intensity.toFixed(2)})`,
                    }}
                  >
                    {count || ""}
                  </td>
                );
              })}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

`src/pages/admin/product-analytics/PracticeDrilldown.tsx`:

```typescript
import { Button } from "@/components/ui/button";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";

import { FeatureHeatmap } from "./FeatureHeatmap";
import { usePracticeDrilldown } from "./useProductAnalytics";

const formatMinutes = (ms: number): string => `${Math.round(ms / 60000)}m`;

export function PracticeDrilldown({
  practiceId,
  practiceName,
  days,
  onBack,
}: {
  practiceId: number;
  practiceName: string;
  days: number;
  onBack: () => void;
}) {
  const { users, routes, matrixUsers, matrix, isLoading } = usePracticeDrilldown(
    practiceId,
    days,
  );

  return (
    <div className="space-y-6">
      <div className="flex items-center gap-3">
        <Button variant="outline" size="sm" onClick={onBack}>
          Back
        </Button>
        <h2 className="text-lg font-semibold">{practiceName}</h2>
      </div>

      {isLoading ? (
        <p className="text-sm text-muted-foreground">Loading…</p>
      ) : (
        <>
          <section className="space-y-2">
            <h3 className="text-sm font-medium">Users</h3>
            <Table>
              <TableHeader>
                <TableRow>
                  <TableHead>User</TableHead>
                  <TableHead className="text-right">Events</TableHead>
                  <TableHead className="text-right">Active time</TableHead>
                  <TableHead>Top features</TableHead>
                  <TableHead>Last seen</TableHead>
                </TableRow>
              </TableHeader>
              <TableBody>
                {users.map((user) => (
                  <TableRow key={user.user_id}>
                    <TableCell>{user.email}</TableCell>
                    <TableCell className="text-right">{user.events}</TableCell>
                    <TableCell className="text-right">
                      {formatMinutes(user.active_ms)}
                    </TableCell>
                    <TableCell className="text-xs text-muted-foreground">
                      {user.top_routes.map((route) => route.route).join(", ") || "—"}
                    </TableCell>
                    <TableCell>{user.last_seen ?? "—"}</TableCell>
                  </TableRow>
                ))}
              </TableBody>
            </Table>
          </section>

          <section className="space-y-2">
            <h3 className="text-sm font-medium">Feature heatmap</h3>
            <FeatureHeatmap routes={routes} users={matrixUsers} matrix={matrix} />
          </section>
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 6: Write the unused-surfaces panel and the page**

`src/pages/admin/product-analytics/UnusedSurfaces.tsx` — the spec's "what did we build that nobody opens" list:

```typescript
import { useQuery } from "@tanstack/react-query";

import { API_ENDPOINTS } from "@/config/api";
import { TokenManager } from "@/contexts/AuthContext";

export function UnusedSurfaces({ days }: { days: number }) {
  const query = useQuery({
    queryKey: ["product-analytics", "unused", days],
    queryFn: async () => {
      const token = TokenManager.getToken();
      const response = await fetch(API_ENDPOINTS.productAnalytics.unused(days), {
        headers: { Authorization: `Bearer ${token}` },
      });
      if (!response.ok) throw new Error(`Request failed: ${response.status}`);
      return (await response.json()) as {
        unused_endpoints: string[];
        known_routes: string[];
      };
    },
  });

  const unused = query.data?.unused_endpoints ?? [];

  if (query.isLoading) {
    return <p className="text-sm text-muted-foreground">Loading…</p>;
  }

  return (
    <div className="space-y-2">
      <p className="text-sm text-muted-foreground">
        {unused.length} API endpoints saw no usage in this window.
      </p>
      <ul className="max-h-64 overflow-y-auto rounded border p-2 font-mono text-xs">
        {unused.map((endpoint) => (
          <li key={endpoint} className="py-0.5">
            {endpoint}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

`src/pages/admin/ProductAnalytics.tsx`:

```typescript
import { useState } from "react";

import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";

import { PracticeDrilldown } from "./product-analytics/PracticeDrilldown";
import { PracticesTable } from "./product-analytics/PracticesTable";
import { UnusedSurfaces } from "./product-analytics/UnusedSurfaces";
import { useProductAnalytics } from "./product-analytics/useProductAnalytics";

export default function ProductAnalytics() {
  const [days, setDays] = useState(30);
  const [selected, setSelected] = useState<number | null>(null);

  const { practices, isLoading, error } = useProductAnalytics(days);
  const selectedPractice = practices.find((p) => p.practice_id === selected);

  return (
    <div className="space-y-6 p-6">
      <div className="flex items-center justify-between">
        <div>
          <h1 className="text-xl font-semibold">Product analytics</h1>
          <p className="text-sm text-muted-foreground">
            How practices and users actually use the app.
          </p>
        </div>
        <Select value={String(days)} onValueChange={(value) => setDays(Number(value))}>
          <SelectTrigger className="w-36">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="7">Last 7 days</SelectItem>
            <SelectItem value="30">Last 30 days</SelectItem>
            <SelectItem value="90">Last 90 days</SelectItem>
          </SelectContent>
        </Select>
      </div>

      {error ? (
        <p className="text-sm text-destructive">Could not load analytics.</p>
      ) : isLoading ? (
        <p className="text-sm text-muted-foreground">Loading…</p>
      ) : selected !== null && selectedPractice ? (
        <PracticeDrilldown
          practiceId={selected}
          practiceName={selectedPractice.practice_name}
          days={days}
          onBack={() => setSelected(null)}
        />
      ) : (
        <>
          <PracticesTable practices={practices} onSelect={setSelected} />
          <section className="space-y-2">
            <h3 className="text-sm font-medium">Features nobody uses</h3>
            <UnusedSurfaces days={days} />
          </section>
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 7: Register the route and nav entry**

In `src/routes/admin.routes.tsx`, add the lazy import alongside the others (keep alphabetical placement — after `PracticeManagementDetail`):

```typescript
const ProductAnalytics = lazyPage("ProductAnalytics", () => import("../pages/admin/ProductAnalytics"));
```

And add the route object alongside the others:

```typescript
  {
    path: "/admin/product-analytics",
    component: ProductAnalytics,
    access: "admin",
  },
```

In `src/components/admin/AdminSidebar.tsx`, add `BarChart3` to the existing `lucide-react` import block (it already imports `LayoutDashboard`, `ShieldAlert`, etc. from that one block near the top of the file). Then insert this section immediately after the Security Logs section, which ends at line 618:

```tsx
        {/* Product Analytics Section */}
        <div className="space-y-1">
          <Link
            to="/admin/product-analytics"
            className={cn(
              "flex items-center gap-3 px-3 py-2 rounded-lg text-sm font-medium transition-colors",
              isRouteActive("/admin/product-analytics")
                ? "bg-[#6C5ACF] text-white"
                : "text-gray-700 hover:bg-gray-100"
            )}
          >
            <BarChart3 className="w-5 h-5" />
            Product Analytics
          </Link>
        </div>
```

- [ ] **Step 8: Run the tests to verify they pass**

```bash
npx vitest run src/pages/admin/product-analytics
```

Expected: 2 tests PASS.

- [ ] **Step 9: Verify in the browser**

Sign in as a superadmin, visit `/admin/product-analytics`, confirm the practices table renders and clicking a row opens the drilldown with the heatmap. Note: rollups run nightly, so a freshly-collected day shows nothing until `build_usage_rollups` has run. To see data immediately:

```bash
python manage.py shell -c "
from django.utils import timezone
from productAnalytics.tasks import build_usage_rollups
print(build_usage_rollups(for_date=timezone.now().date().isoformat()))
"
```

- [ ] **Step 10: Checkpoint**

Report test counts, the `tsc` delta, and confirmation the page rendered with real data. Do not commit.

---

## Task 10: Per-route click heat and in-app overlay

**Files:**
- Create: `src/pages/admin/product-analytics/ElementHeatPanel.tsx`
- Create: `src/lib/analytics/HeatOverlay.tsx`
- Create: `src/lib/analytics/HeatOverlay.test.tsx`
- Modify: `src/pages/admin/ProductAnalytics.tsx`, `src/lib/analytics/AnalyticsProvider.tsx`

**Interfaces:**
- Consumes: `API_ENDPOINTS.productAnalytics.elements`, `describeElement` from Task 7
- Produces: `<ElementHeatPanel route={string} days={number} />`; `<HeatOverlay />` mounted by `AnalyticsProvider` for superadmins only, toggled by `localStorage.tp_heat_overlay === "1"`

- [ ] **Step 1: Write the failing overlay test**

`src/lib/analytics/HeatOverlay.test.tsx`:

```typescript
import { render, waitFor } from "@testing-library/react";
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";

import { HeatOverlay } from "./HeatOverlay";

describe("HeatOverlay", () => {
  beforeEach(() => {
    document.body.innerHTML = `<button data-track="save">Save</button>`;
    localStorage.setItem("tp_heat_overlay", "1");
    vi.stubGlobal(
      "fetch",
      vi.fn().mockResolvedValue({
        ok: true,
        json: async () => ({
          elements: [
            { element: "save", count: 42, rage_count: 3, dead_count: 0 },
          ],
        }),
      }),
    );
  });

  afterEach(() => {
    localStorage.removeItem("tp_heat_overlay");
    vi.unstubAllGlobals();
  });

  it("paints a badge onto the matching element", async () => {
    render(<HeatOverlay route="/patients/:id" />);

    await waitFor(() =>
      expect(document.querySelector("[data-heat-badge]")).not.toBeNull(),
    );
    expect(document.querySelector("[data-heat-badge]")!.textContent).toContain("42");
  });

  it("renders nothing when the overlay is off", async () => {
    localStorage.setItem("tp_heat_overlay", "0");
    render(<HeatOverlay route="/patients/:id" />);

    await waitFor(() =>
      expect(document.querySelector("[data-heat-badge]")).toBeNull(),
    );
  });
});
```

- [ ] **Step 2: Run it to verify it fails**

```bash
npx vitest run src/lib/analytics/HeatOverlay.test.tsx
```

Expected: FAIL — cannot resolve `./HeatOverlay`.

- [ ] **Step 3: Write the overlay**

`src/lib/analytics/HeatOverlay.tsx`:

```typescript
/**
 * In-app click heat, painted onto the real page.
 *
 * This is the Clarity feel without recording anything: we fetch aggregate
 * click counts for the current route and position a badge over each element
 * whose descriptor matches. Superadmin-only, opt-in via localStorage, and it
 * never mutates the page's own DOM — badges live in their own fixed layer.
 */

import { useEffect, useState } from "react";

import { API_ENDPOINTS } from "@/config/api";
import { TokenManager } from "@/contexts/AuthContext";

import { describeElement } from "./elementId";

const OVERLAY_KEY = "tp_heat_overlay";

interface ElementHeat {
  element: string;
  count: number;
  rage_count: number;
  dead_count: number;
}

interface Badge {
  key: string;
  top: number;
  left: number;
  heat: ElementHeat;
}

const overlayEnabled = (): boolean => {
  try {
    return localStorage.getItem(OVERLAY_KEY) === "1";
  } catch {
    return false;
  }
};

export function HeatOverlay({ route }: { route: string }) {
  const [badges, setBadges] = useState<Badge[]>([]);

  useEffect(() => {
    if (!overlayEnabled() || !route) {
      setBadges([]);
      return undefined;
    }

    let cancelled = false;

    const load = async () => {
      try {
        const token = TokenManager.getToken();
        const response = await fetch(API_ENDPOINTS.productAnalytics.elements(route), {
          headers: { Authorization: `Bearer ${token}` },
        });
        if (!response.ok) return;
        const data = (await response.json()) as { elements: ElementHeat[] };
        if (cancelled) return;

        const byDescriptor = new Map(data.elements.map((e) => [e.element, e]));
        const found: Badge[] = [];

        document
          .querySelectorAll("button,a,[role='button'],[role='tab'],[role='menuitem']")
          .forEach((el, index) => {
            const heat = byDescriptor.get(describeElement(el));
            if (!heat) return;
            const rect = el.getBoundingClientRect();
            found.push({
              key: `${heat.element}-${index}`,
              top: rect.top + window.scrollY,
              left: rect.left + window.scrollX,
              heat,
            });
          });

        setBadges(found);
      } catch {
        // Overlay is a diagnostic. Never surface its failures.
      }
    };

    void load();
    return () => {
      cancelled = true;
    };
  }, [route]);

  if (badges.length === 0) return null;

  return (
    <div className="pointer-events-none fixed inset-0 z-[9999]">
      {badges.map((badge) => (
        <span
          key={badge.key}
          data-heat-badge
          className="absolute rounded bg-blue-600 px-1 text-[10px] font-medium text-white"
          style={{ top: badge.top, left: badge.left }}
          title={`${badge.heat.count} clicks · ${badge.heat.rage_count} rage · ${badge.heat.dead_count} dead`}
        >
          {badge.heat.count}
          {badge.heat.rage_count > 0 ? ` ⚠${badge.heat.rage_count}` : ""}
        </span>
      ))}
    </div>
  );
}
```

- [ ] **Step 4: Mount the overlay for superadmins**

In `src/lib/analytics/AnalyticsProvider.tsx`, add the import:

```typescript
import { HeatOverlay } from "./HeatOverlay";
```

And change the return statement from `return <>{children}</>;` to:

```typescript
  return (
    <>
      {children}
      {user?.userType === "superuser" ? <HeatOverlay route={routeRef.current} /> : null}
    </>
  );
```

Because `routeRef` is a ref, the overlay re-reads it whenever the provider re-renders on navigation — which the `location.pathname` effect guarantees.

- [ ] **Step 5: Write the dashboard element panel**

`src/pages/admin/product-analytics/ElementHeatPanel.tsx`:

```typescript
import { useQuery } from "@tanstack/react-query";

import { API_ENDPOINTS } from "@/config/api";
import { TokenManager } from "@/contexts/AuthContext";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";

interface ElementHeat {
  element: string;
  count: number;
  rage_count: number;
  dead_count: number;
}

export function ElementHeatPanel({ route, days }: { route: string; days: number }) {
  const query = useQuery({
    queryKey: ["product-analytics", "elements", route, days],
    enabled: Boolean(route),
    queryFn: async () => {
      const token = TokenManager.getToken();
      const response = await fetch(
        API_ENDPOINTS.productAnalytics.elements(route, days),
        { headers: { Authorization: `Bearer ${token}` } },
      );
      if (!response.ok) throw new Error(`Request failed: ${response.status}`);
      return (await response.json()) as { elements: ElementHeat[] };
    },
  });

  const elements = query.data?.elements ?? [];

  if (query.isLoading) {
    return <p className="text-sm text-muted-foreground">Loading…</p>;
  }

  if (elements.length === 0) {
    return (
      <p className="text-sm text-muted-foreground">
        No click data for {route} in this window.
      </p>
    );
  }

  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead>Element</TableHead>
          <TableHead className="text-right">Clicks</TableHead>
          <TableHead className="text-right">Rage</TableHead>
          <TableHead className="text-right">Dead</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {elements.map((element) => (
          <TableRow key={element.element}>
            <TableCell className="font-mono text-xs">{element.element}</TableCell>
            <TableCell className="text-right">{element.count}</TableCell>
            <TableCell className="text-right">{element.rage_count || ""}</TableCell>
            <TableCell className="text-right">{element.dead_count || ""}</TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  );
}
```

- [ ] **Step 6: Add a route picker to the dashboard page**

In `src/pages/admin/ProductAnalytics.tsx`, add the import and a state hook:

```typescript
import { ElementHeatPanel } from "./product-analytics/ElementHeatPanel";
```

```typescript
  const [heatRoute, setHeatRoute] = useState<string>("");
```

Then add this section immediately before the closing `</div>` of the page body:

```tsx
      <section className="space-y-2">
        <h3 className="text-sm font-medium">Click heat by route</h3>
        <Select value={heatRoute} onValueChange={setHeatRoute}>
          <SelectTrigger className="w-72">
            <SelectValue placeholder="Choose a route" />
          </SelectTrigger>
          <SelectContent>
            {Array.from(
              new Set(
                practices.flatMap((practice) =>
                  practice.top_routes.map((route) => route.route),
                ),
              ),
            ).map((route) => (
              <SelectItem key={route} value={route}>
                {route}
              </SelectItem>
            ))}
          </SelectContent>
        </Select>
        {heatRoute ? <ElementHeatPanel route={heatRoute} days={days} /> : null}
      </section>
```

- [ ] **Step 7: Run all frontend tests**

```bash
npx vitest run src/lib/analytics src/pages/admin/product-analytics
```

Expected: every test in both directories PASS.

- [ ] **Step 8: Run the full backend suite for the app**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test productAnalytics --keepdb -v 2
```

Expected: all tests across the five backend test modules PASS.

- [ ] **Step 9: Final checkpoint**

Report:
1. Backend test count and result.
2. Frontend test count and result.
3. `npx tsc -p tsconfig.app.json` error count vs the ~491 baseline (delta must be zero).
4. The distinct `route` and `element` values in `UsageEvent` — direct evidence that no record ids and no human names entered the store.
5. Confirm the overlay toggle works: set `localStorage.tp_heat_overlay = "1"` as a superadmin, reload a page, see badges.

Do not commit — the user handles all VCS operations.

---

## Post-implementation: production steps

These do not happen on deploy and must be run by hand. Add them to the to-run-in-prod list:

1. `python manage.py migrate productAnalytics` — creates the four tables.
2. Verify the two new Celery Beat entries appear in `django_celery_beat` after the worker restarts (this project uses `DatabaseScheduler`, so settings-defined schedules need a beat restart to register).
3. Optionally backfill nothing — collection starts from the deploy forward; there is no historical data to import.
4. Watch `UsageEvent` row count for the first week. If growth exceeds expectations, lower `PRODUCT_ANALYTICS_SAMPLE_RATE` — `page.view`, `page.leave`, and `api.request` stay exact regardless.

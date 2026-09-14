# dentallyIntegration Duplication Audit

## G1 — _get_service implementation (6 sites)

**Category:** viewset boilerplate  
**Belongs in:** Base viewset mixin or a shared service accessor  
**Sites:**
- `dentallyIntegration/views/dentally_views.py:158` — `DentallyPatientsViewSet._get_service(self)`
- `dentallyIntegration/views/dentally_views.py:1142` — `DentallyAppointmentsViewSet._get_service(self)`
- `dentallyIntegration/views/dentally_views.py:1319` — `DentallyTreatmentsViewSet._get_service(self)`
- `dentallyIntegration/views/dentally_views.py:1397` — `DentallyPractitionersViewSet._get_service(self)`
- `dentallyIntegration/views/dentally_views.py:1452` — `DentallySitesViewSet._get_service(self)`
- `dentallyIntegration/views/dentally_views.py:1504` — `DentallyPaymentPlansViewSet._get_service(self)`

**Identical or divergent?** IDENTICAL — All 6 follow the exact same pattern:
```python
def _get_service(self):
    """Get Dentally API service for current user's practice"""
    user = self.request.user
    if not hasattr(user, "practice") or not user.practice:
        raise DentallyAPIError("User is not associated with a practice")
    return DentallyAPIService(user.practice)
```

**Consolidation note:** Extract into a base viewset mixin (`DentallyAPIViewSetMixin`) with this single method, then inherit across all 6 viewsets.

---

## G2 — clean_phone helper (2 sites)

**Category:** request parsing / data cleaning  
**Belongs in:** shared `dentallyIntegration/utils/` module  
**Sites:**
- `dentallyIntegration/tasks.py:690-694` — inside `migrate_all_dentally_patients` task
- `dentallyIntegration/views/dentally_views.py:812-816` — inside `bulk_import_patients` action

**Identical or divergent?** IDENTICAL:
```python
def clean_phone(value):
    """Convert 'None' string, None, and empty strings to None"""
    if value is None or value == "" or value == "None":
        return None
    return value
```

**Consolidation note:** Move to `dentallyIntegration/utils/phone_utils.py` and import both sites.

---

## G3 — Redis client initialization (4 sites)

**Category:** sync / API-client plumbing  
**Belongs in:** shared `dentallyIntegration/utils/redis_utils.py` helper function  
**Sites:**
- `dentallyIntegration/views/dentally_views.py:348-357` — in `start_migration` action
- `dentallyIntegration/views/dentally_views.py:525-534` — in `migration_status` action
- `dentallyIntegration/views/dentally_views.py:603-612` — in `cancel_migration` action
- `dentallyIntegration/views/dentally_views.py:724-731` — in `clear_migration_status` action

**Identical or divergent?** IDENTICAL across all 4:
```python
import os
import pickle
import redis

redis_host = os.environ.get("REDIS_HOST", "127.0.0.1")
redis_port = int(os.environ.get("REDIS_PORT", 6379))
r = redis.Redis(host=redis_host, port=redis_port, db=1)
redis_key = f":1:{cache_key}"
raw_data = r.get(redis_key)
existing_progress = None
if raw_data:
    existing_progress = pickle.loads(raw_data)
```

**Consolidation note:** Create `get_redis_client()` and `get_redis_data(cache_key)` helpers in `dentallyIntegration/utils/redis_utils.py`.

---

## G4 — use_cache parameter parsing (9 sites)

**Category:** request parsing  
**Belongs in:** shared parsing utility or a decorator  
**Sites:**
- `dentallyIntegration/views/dentally_views.py:186` — `DentallyPatientsViewSet.list()` (default "false")
- `dentallyIntegration/views/dentally_views.py:282` — `DentallyPatientsViewSet.retrieve()` (default "false")
- `dentallyIntegration/views/dentally_views.py:1171` — `DentallyAppointmentsViewSet.list()` (default "false")
- `dentallyIntegration/views/dentally_views.py:1205` — `DentallyAppointmentsViewSet.retrieve()` (default "false")
- `dentallyIntegration/views/dentally_views.py:1344` — `DentallyTreatmentsViewSet.list()` (default "false")
- `dentallyIntegration/views/dentally_views.py:1420` — `DentallyPractitionersViewSet.list()` (default "true")
- `dentallyIntegration/views/dentally_views.py:1470` — `DentallySitesViewSet.list()` (default "true")
- `dentallyIntegration/views/dentally_views.py:1529` — `DentallyPaymentPlansViewSet.list()` (default "false")
- `dentallyIntegration/views/dentally_views.py:1554` — `DentallyPaymentPlansViewSet.retrieve()` (default "false")

**Identical or divergent?** IDENTICAL pattern (with 2 different default values):
```python
use_cache = request.query_params.get("use_cache", "false").lower() == "true"
# OR
use_cache = request.query_params.get("use_cache", "true").lower() == "true"
```

**Consolidation note:** Create helper `parse_bool_param(request, key, default=False)` in `dentallyIntegration/utils/parsing_utils.py`.

---

## G5 — practice extraction pattern (37 sites)

**Category:** request parsing  
**Belongs in:** a mixin or helper method  
**Sites:**
- `dentallyIntegration/views/dentally_views.py:336, 510, 587, 714, 799, 2235` (6 sites)
- `dentallyIntegration/views/opportunity_config_views.py:233, 278` (2 sites, in helper method `_practice`)
- `dentallyIntegration/views/recall_config_views.py:66, 240, 369, 422, 497, 635, 793, 894, 969, 1065, 1168, 1282, 1337, 1412` (14 sites, in `_practice` helper)
- `dentallyIntegration/views/recall_views.py:492, 757, 776, 1009, 1023, 1044, 1108, 1654, 1964, 1993, 2013, 2371, 2429, 3691` (14 sites)

**Identical or divergent?** IDENTICAL:
```python
practice = request.user.practice if hasattr(request.user, "practice") else None
```

**Consolidation note:** Extract as a mixin method `get_user_practice(self)` in a base viewset, or use Django's `HasPractice` permission class. Total sites: **37 calls**, but only **3 unique implementations** (some are already in a `_practice()` helper). Consolidate remaining inline calls.

---

## G6 — Practice is None guard clause + error response (25+ sites)

**Category:** viewset boilerplate / practice gate  
**Belongs in:** a mixin or decorator  
**Sites (sampling):**
- `dentallyIntegration/views/dentally_views.py:339-342` — HTTP_400_BAD_REQUEST, `{"error": "User must be associated with a practice"}`
- `dentallyIntegration/views/dentally_views.py:513-517` — HTTP_400_BAD_REQUEST with logger.warning
- `dentallyIntegration/views/dentally_views.py:590-594` — HTTP_400_BAD_REQUEST with logger.warning
- `dentallyIntegration/views/dentally_views.py:717-721` — HTTP_400_BAD_REQUEST
- `dentallyIntegration/views/dentally_views.py:802-806` — HTTP_400_BAD_REQUEST
- `dentallyIntegration/views/opportunity_config_views.py:239-242` — `{"error": "No practice associated with this user"}`
- `dentallyIntegration/views/appointment_confirm_views.py:72-76` — `{"error": "User not associated with a practice"}`

**Identical or divergent?** DIVERGENT in error message text, but all use the same structure:
```python
if not practice:
    return Response({"error": "<message>"}, status=status.HTTP_400_BAD_REQUEST)
```

Error message variants seen:
- `"User must be associated with a practice"` (most common)
- `"No practice associated with this user"` 
- `"User not associated with a practice"`

**Consolidation note:** Create a mixin method that encapsulates this check and raises a standard exception, or use a permission/decorator to eliminate the inline guard.

---

## G7 — DentallyAuthenticationError exception handler (42+ sites)

**Category:** response shaping  
**Belongs in:** a mixin or decorator  
**Sites:**
- Repeated throughout `dentallyIntegration/views/dentally_views.py` in every CRUD action across 6 viewsets (lines 265-266, 287-288, 1071-1072, 1093-1094, 1114-1115, 1189-1190, 1210-1211, 1231-1232, 1252-1253, 1273-1274, 1294-1295, 1351-1352, 1373-1374, 1428-1429, 1476-1477, 1537-1538, 1559-1560, 1580-1581, 1601-1602, 1622-1623)
- Total count: **42 instances** of the pattern (DentallyAuthenticationError, DentallyAPIError, generic Exception handlers)

**Identical or divergent?** IDENTICAL except for one outlier at line 130-133:
```python
except DentallyAuthenticationError as e:
    return Response({"success": False, "message": str(e)},
                    status=status.HTTP_401_UNAUTHORIZED)  # test_connection only
```

All others use:
```python
except DentallyAuthenticationError as e:
    return Response({"error": str(e)}, status=status.HTTP_401_UNAUTHORIZED)
```

Plus repeated pairs:
```python
except DentallyAPIError as e:
    return Response({"error": str(e)}, status=status.HTTP_400_BAD_REQUEST)

except Exception as e:
    logger.error(f"Error <action>: {str(e)}")
    return Response({"error": "An unexpected error occurred"},
                    status=status.HTTP_500_INTERNAL_SERVER_ERROR)
```

**Consolidation note:** Wrap viewset actions in a decorator `@handle_dentally_errors` that catches these exceptions and returns the standard response. Unify error key names (one view uses `"success": False, "message"` while all others use `"error"`).

---

## G8 — Integer parameter parsing (10+ sites)

**Category:** request parsing  
**Belongs in:** shared parsing utility  
**Sites:**
- `dentallyIntegration/views/dentally_views.py:183-184` — page, per_page parsing
- `dentallyIntegration/views/dentally_views.py:1169-1170` — page, per_page parsing
- `dentallyIntegration/views/dentally_views.py:1342-1343` — page, per_page parsing
- `dentallyIntegration/views/dentally_views.py:1418-1419` — page, per_page parsing
- `dentallyIntegration/views/dentally_views.py:1527-1528` — page, per_page parsing
- `dentallyIntegration/views/dentally_views.py:1777` — days parameter
- `dentallyIntegration/views/recall_views.py:2635, 3492-3493, 3565, 3779-3780, 3826` (various page, page_size, recall_lookback)

**Identical or divergent?** IDENTICAL:
```python
page = int(request.query_params.get("page", 1))
per_page = int(request.query_params.get("per_page", 100))
```

**Consolidation note:** Create `parse_int_param(request, key, default)` in parsing utils.

---

## G9 — Date parsing pattern (2 sites)

**Category:** request parsing  
**Belongs in:** shared parsing utility  
**Sites:**
- `dentallyIntegration/views/dentally_views.py:1174-1175` — `parse_date(start_date) if start_date else None`
- `dentallyIntegration/tasks.py:567` — custom `parse_date` defined locally

**Identical or divergent?** DIVERGENT — Views use Django's `parse_date` (imported at line 9), tasks.py redefines it locally (lines 567-578).

**Consolidation note:** Use Django's `parse_date` consistently; remove the local redefinition in tasks.py if it's identical, or extract task-specific logic if it differs.

---

## G10 — Migration progress cache key pattern (2 sites)

**Category:** caching  
**Belongs in:** constants or a cache-key builder  
**Sites:**
- `dentallyIntegration/views/dentally_views.py:345, 519, 597, 722` — `f"dentally_migration_{practice.id}"`

**Identical or divergent?** IDENTICAL (one pattern reused in 4 actions within the same viewset).

**Consolidation note:** Not a multi-file duplication (all in one viewset), but consolidate as a class constant or method.

---

## G11 — Pagination response building (multiple sites in list actions)

**Category:** response shaping  
**Belongs in:** a mixin or serializer method  
**Sites:**
- `dentallyIntegration/views/dentally_views.py:96-100` — `DentallyIntegrationViewSet.list()` returns `{"results": [...]}` override
- `dentallyIntegration/views/dentally_views.py:233-261` — `DentallyPatientsViewSet.list()` builds `{"pagination": {...}}` envelope

**Identical or divergent?** DIVERGENT — Different response shapes (one uses `"results"`, other adds `"pagination"`).

**Consolidation note:** Establish a single pagination response wrapper across all list views, or document the convention if intentionally divergent.

---

## Summary

| Finding | Count | Category | Impact |
|---------|-------|----------|--------|
| G1: _get_service | 6 | viewset boilerplate | HIGH — all identical, consolidate to mixin |
| G2: clean_phone | 2 | parsing | MEDIUM — move to utils, import both |
| G3: Redis init | 4 | sync plumbing | HIGH — extract helper function |
| G4: use_cache parsing | 9 | parsing | MEDIUM — create parse_bool_param helper |
| G5: practice extraction | 37 | parsing | HIGH — 37 call sites, some already in helpers; mixin consolidates remaining |
| G6: practice guard + error | 25+ | practice gate | HIGH — 65+ total implementations (inline + helpers); divergent error messages |
| G7: exception handlers | 42+ | response shaping | CRITICAL — 42 repetitions across 6 viewsets; 1 divergent response shape (test_connection) |
| G8: int parsing | 10+ | parsing | MEDIUM — parse_int_param helper |
| G9: date parsing | 2 | parsing | LOW — task redefines Django's parse_date; consolidate |
| G10: cache keys | 4 | caching | LOW — single viewset, extract to constant |
| G11: pagination response | 2 | response shaping | LOW — two divergent shapes, establish convention |

**Total duplicated definitions:** ~100 (counting all repetitions across viewsets and tasks).  
**Consolidation priority:** G7 (exception handlers) → G3 (Redis) → G1 (_get_service) → G5 (practice extraction) → G4,G8 (parsing helpers).

**Top 5 by site count:**
1. **G7: Exception handlers** — 42+ sites
2. **G5: Practice extraction** — 37 sites
3. **G4: use_cache parsing** — 9 sites
4. **G8: Integer parsing** — 10+ sites
5. **G6: Practice guard + error** — 25+ sites

**Divergent findings:**
- G6: Practice error messages inconsistent (3 variants)
- G7: test_connection uses `{"success": False, "message"}` while all others use `{"error"}`
- G9: tasks.py redefines parse_date locally instead of using Django's
- G11: Different pagination response shapes

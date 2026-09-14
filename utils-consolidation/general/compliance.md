# Code Duplication Inventory: compliance/

## G1 — Serializer getter: created_by to display name (13 sites)
**Category:** serializer getter | consolidation candidate
**Belongs in:** shared serializers mixin or base class
**Sites:**
- `serializers.py:566` — `if obj.created_by: return get_user_display_name(obj.created_by) return None`
- `serializers.py:735` — identical
- `serializers.py:891` — identical
- `serializers.py:1250` — identical
- `serializers.py:1741` — `return get_user_display_name(obj.created_by) if obj.created_by else None` (ternary form)
- `serializers.py:1888` — `if obj.created_by: return get_user_display_name(obj.created_by) return None`
- `serializers.py:2040` — identical
- `serializers.py:2171` — identical
- `serializers.py:2332` — identical
- `serializers.py:2516` — identical
- `serializers.py:2744` — identical
- `serializers.py:2971` — identical
- `serializers.py:3178` — identical

**Identical or divergent?** DIVERGENT: 12 sites use multi-line if/return pattern; 1 site (1741) uses ternary conditional expression. Both are functionally equivalent and both call the same `get_user_display_name()` helper.

**Consolidation note:** Extract to a shared mixin `UserDisplayNameMixin.get_created_by_name()`. The helper function already exists; refactor the redundant 13 copies into one method.


## G2 — Serializer getter: practice to name (6 sites)
**Category:** serializer getter | consolidation candidate
**Belongs in:** shared serializers mixin
**Sites:**
- `serializers.py:1893` — `if obj.practice: return obj.practice.name return None`
- `serializers.py:2045` — identical
- `serializers.py:2176` — identical
- `serializers.py:2337` — identical
- `serializers.py:2521` — identical
- `serializers.py:2749` — identical

**Identical or divergent?** IDENTICAL. All 6 are byte-identical multi-line if/return pattern.

**Consolidation note:** Extract to shared mixin `PracticeDisplayMixin.get_practice_name()`.


## G3 — Serializer getter: group sharing annotation (5 sites)
**Category:** serializer getter | consolidation candidate
**Belongs in:** shared serializers mixin
**Sites:**
- `serializers.py:147` — `annotated = getattr(obj, "_is_group_shared", None); if annotated is not None: return bool(annotated); [fallback to per-object query]`
- `serializers.py:224` — similar pattern, no docstring
- `serializers.py:967` — same logic, has docstring + comment about fallback
- `serializers.py:1270` — same logic, has docstring
- `serializers.py:2976` — same logic, has docstring

**Identical or divergent?** MOSTLY IDENTICAL LOGIC with varied docstring styles. Core pattern (getattr → bool conversion → fallback query) is consistent across all 5; variation is cosmetic (docstring format and comment presence).

**Consolidation note:** Extract to `GroupShareMixin.get_is_group_shared()` with a single docstring. The fallback logic is identical.


## G4 — Serializer getter: group sharing (shared-out annotation) (5 sites)
**Category:** serializer getter | consolidation candidate
**Belongs in:** shared serializers mixin
**Sites:**
- `serializers.py:159` — `annotated = getattr(obj, "_has_group_shares", None); [fallback pattern]`
- `serializers.py:235` — similar
- `serializers.py:985` — similar
- `serializers.py:1282` — similar
- `serializers.py:2988` — similar

**Identical or divergent?** IDENTICAL LOGIC across all 5. All check `_has_group_shares` annotation and fall back to per-object query.

**Consolidation note:** Extract to `GroupShareMixin.get_is_shared_out()`.


## G5 — Serializer getter: file URL resolution (4 sites)
**Category:** serializer getter | consolidation candidate | EXISTS IN MIXIN OUTSIDE APP
**Belongs in:** shared serializers mixin (already exists: `TreatmentPlan/serializers/mixins.py:33`)
**Sites:**
- `serializers.py:283` — `file_url = obj.file.url; if not file_url.startswith("http"): request = self.context.get("request"); if request: return request.build_absolute_uri(file_url) return file_url`
- `serializers.py:1607` — identical
- `serializers.py:3128` — identical
- `serializers.py:3348` — identical

**Identical or divergent?** IDENTICAL across all 4 sites.

**Consolidation note:** A shared mixin `FileURLMixin.get_url()` already exists at `TreatmentPlan/serializers/mixins.py:33` (named `get_image_url`). The compliance app should import and reuse that mixin instead of duplicating this logic. Alternatively, create a generic `UrlMixin` in a global utils location.


## G6 — Client IP extraction from request (2 sites)
**Category:** request parsing | consolidation candidate
**Belongs in:** global utils or base viewset mixin
**Sites:**
- `serializers.py:1593` — `x_forwarded_for = request.META.get("HTTP_X_FORWARDED_FOR"); if x_forwarded_for: return x_forwarded_for.split(",")[0].strip() return request.META.get("REMOTE_ADDR")`
- `views.py:4641` — identical

**Identical or divergent?** IDENTICAL. Both read `HTTP_X_FORWARDED_FOR` first, then fall back to `REMOTE_ADDR`.

**Consolidation note:** Move to a shared utility function (e.g., `utils/request_helpers.py:get_client_ip()`). The backend has 11 total copies across multiple apps; this app contributes 2.


## G7 — ViewSet method: get_permissions (10 sites)
**Category:** viewset boilerplate | consolidation candidate (context-dependent)
**Belongs in:** per-viewset; may benefit from a pattern guide
**Sites:**
- `views.py:360` — `if self.action in ["create", "update", "partial_update", "destroy"]: return [IsAdminUser()] return [IsAuthenticated()]`
- `views.py:411` — different logic: nested feature access checks
- `views.py:3868` — different logic: per-action feature gating
- `views.py:4797` — different logic: custom permission rules
- `views.py:5578` — different logic: per-action gating
- `views.py:5757` — different logic: per-action gating
- `views.py:6252` — different logic: custom permission rules
- `views.py:9005` — different logic: per-action gating
- `views.py:9652` — different logic: custom permission rules
- `views.py:10007` — different logic: per-action gating

**Identical or divergent?** HIGHLY DIVERGENT. Each viewset has unique business logic: some guard on `IsAdminUser()`, others gate on `check_user_feature_access()` with specific features (compliance_manage, compliance_view, etc.), others have custom permission classes. No two implementations are identical.

**Consolidation note:** Cannot consolidate without losing semantics. Each viewset enforces distinct permission policies. Document the pattern (action-based dispatch to permission classes) in a pattern guide; do not consolidate the implementations.


## G8 — ViewSet method: partial_update (6 sites)
**Category:** viewset boilerplate | consolidation candidate (conditional)
**Belongs in:** base viewset mixin
**Sites:**
- `views.py:1013` — `kwargs["partial"] = True; return self.update(request, *args, **kwargs)`
- `views.py:4175` — identical
- `views.py:9237` — identical
- `views.py:9851` — identical
- `views.py:10099` — identical
- `views.py:10437` — `kwargs["partial"] = True; [permission check + custom logic]; return self.update(request, *args, **kwargs)` (DIVERGENT)

**Identical or divergent?** MOSTLY IDENTICAL with 1 divergent copy. Sites 1013, 4175, 9237, 9851, 10099 are byte-identical. Site 10437 adds custom permission validation before the update call.

**Consolidation note:** Extract the 5 identical copies to a base mixin. Site 10437 should override `partial_update` AND call `super().partial_update()` to avoid code duplication while preserving its permission check.


## G9 — Serializer getters for user fields (multiple variants)
**Category:** serializer getter | consolidation candidate
**Belongs in:** shared serializers mixin
**Details:**
- `get_user_name` (4 sites): Lines 339, 592 use inline f-string formatting; lines 624, 661 use `get_user_display_name()` helper. **DIVERGENT: 2 different implementations of the same logic.**
- `get_reported_by_user_name` (2 sites): Lines 1883, 2166 both use `get_user_display_name()` helper. **IDENTICAL.**
- `get_handled_by_user_name` (1 site): Line 2035. **Not duplicated.**
- `get_recorded_by_user_name` (3 sites): Lines 2327, 2511, 2739 all use `get_user_display_name()` helper. **IDENTICAL.**
- `get_assessed_by_user_name` (1 site): Line 2966. **Not duplicated.**
- `get_assigned_user_name` (1 site): Line 1369. **Not duplicated.**

**Consolidation note:** The getters at lines 339 and 592 should be refactored to use the `get_user_display_name()` helper (like lines 624, 661 do). Create a parameterizable mixin `UserFieldDisplayMixin` with a method `get_user_field_name(obj, field_name)` to eliminate the variation and upcoming duplication as new similar fields are added.


## G10 — Practice gate: current_practice retrieval and None check (61 sites)
**Category:** practice gate | permission pattern
**Belongs in:** base viewset mixin or decorator
**Pattern:**
```python
practice = getattr(request.user, "current_practice", None)
if not practice:
    return Response({"error": "No practice selected"}, status=400)
```

**Count:** 61 exact matches of `getattr(request.user, "current_practice", None)` in views.py + serializers.py.

**Response patterns observed:**
- `{"error": "No practice selected"}` with status 400 — most common
- `{"configured_ids": []}` — empty array response when no practice
- `{}` — empty object response
- `{"items": [], "total_outstanding": 0}` — empty result object

**Identical or divergent?** IDENTICAL PATTERN with divergent error responses. The retrieval and None-check are always the same; the error response varies by endpoint.

**Consolidation note:** Create a decorator `@require_practice` or a mixin method `self.get_current_practice_or_error()` to reduce the 61 repetitions. Each endpoint can define its own error response, but the gate itself should be centralized.


## G11 — Response body structures: error key variations
**Category:** exception handling | response consistency
**Observed distributions:**
- `{"error": "..."}` — 89 sites (most common error key)
- `{"detail": "..."}` — (not observed in compliance excepting DRF defaults)
- `{"configured_ids": [...]}`, `{"batch_id": "..."}`, `{"items": [...]}` — context-specific successful responses

**Consolidation note:** No urgent consolidation needed; error key is consistent (always `"error"` in this app). Verify against other apps: `dentallyIntegration` uses 83 `{"error"}` / 3 `{"detail"}` / 2 `{"success"}`; `messaging` has a three-way split.


## G12 — User display name helper
**Category:** helper function | already consolidated
**Belongs in:** `serializers.py:61` (defined locally) or global utils
**Sites:**
- `serializers.py:61` — `get_user_display_name(user)` — calls `TreatmentPath.utils.get_display_name_for_user()`

**Note:** This is a one-liner wrapper. All 13 copies of `get_created_by_name()` delegate to it. The consolidation at the method level (G1) is the key fix; the helper is already shared.


## Summary

| Group | Pattern | Sites | Identical? | Top Priority |
|-------|---------|-------|-----------|--------------|
| G1 | get_created_by_name | 13 | 12 identical + 1 ternary | **HIGH** — 13 copies |
| G2 | get_practice_name | 6 | IDENTICAL | **MEDIUM** — 6 copies |
| G3 | get_is_group_shared | 5 | IDENTICAL LOGIC (docstring variance) | MEDIUM — 5 copies |
| G4 | get_is_shared_out | 5 | IDENTICAL | MEDIUM — 5 copies |
| G5 | get_url (file) | 4 | IDENTICAL | **HIGH** — exists in shared mixin outside app; import instead |
| G6 | get_client_ip | 2 | IDENTICAL | LOW — 2 copies, part of backend-wide 11 |
| G7 | get_permissions | 10 | HIGHLY DIVERGENT | NONE — context-dependent; document pattern |
| G8 | partial_update | 6 | 5 identical + 1 divergent | MEDIUM — 5 can be consolidated |
| G9 | User field getters | 7 scattered | 2 divergent implementations | MEDIUM — standardize to use helper |
| G10 | Practice gate | 61 | IDENTICAL PATTERN (divergent responses) | **HIGH** — 61 repetitions; create decorator |
| G11 | Response bodies | ~80+ | Mostly `{"error"}` | LOW — already consistent |

**Total duplicated definitions:** 13 + 6 + 5 + 5 + 4 + 2 + 6 + 7 + 1 mixin (get_image_url) = **49 semantically similar definitions** across the three main categories (serializers, views boilerplate, patterns).

**Top 5 by site count:**
1. **G1: get_created_by_name** — 13 sites (most urgent)
2. **G10: Practice gate** — 61 sites (pattern opportunity, not code duplication per se)
3. **G7: get_permissions** — 10 sites (divergent; document, don't consolidate)
4. **G8: partial_update** — 6 sites (5 consolidatable)
5. **G2: get_practice_name** — 6 sites (consolidatable)

**Response key / status code distribution:**
- Error responses: **89 sites** return `{"error": "..."}` with `status=400` or `HTTP_400_BAD_REQUEST`
- Custom success responses: vary by endpoint (batch_id, configured_ids, items, etc.)
- No status code divergence observed within this app (all errors use 400)

**Divergent copies worth noting:**
- `get_created_by_name`: 1 ternary (line 1741) among 13 if/return patterns
- `partial_update`: 1 with permission check (line 10437) among 5 simple delegations
- `get_user_name`: 2 using inline f-string (lines 339, 592) vs. 2 using helper (lines 624, 661)

**Files with consolidation opportunities:**
- Primary: `serializers.py` (majority of getter duplication)
- Secondary: `views.py` (partial_update, get_permissions patterns)
- Tertiary: `compliance_audit.py` (exports helper function already used)

**Foreign resources already available:**
- `TreatmentPlan/serializers/mixins.py:33` — `get_image_url()` mixin (compliance's `get_url` is identical pattern)
- `TreatmentPlan/utils.py` — `get_display_name_for_user()` (already wrapped locally and used by all 13 copies)


# TreatmentPlan Directory — General Code Duplication Inventory

## G1 — get_created_by / get_created_by_name (9 sites)
**Category:** serializer getter
**Belongs in:** global `utils/` (used across multiple serializers)
**Sites:**
- `serializers/notes.py:48` — `return get_display_name_for_user(obj.updated_by or obj.created_by) or "Unknown"`
- `serializers/practice_treatment_plan_template.py` — `f"{user.first_name} {user.last_name}".strip() or user.email`
- `serializers/membership.py` — `f"{obj.created_by.first_name} {obj.created_by.last_name}".strip() or obj.created_by.email`
- `serializers/archive.py:163` — `return get_display_name_for_user(obj.created_by)`
- `serializers/archive.py:175` — `return get_display_name_for_user(obj.created_by)` (duplicate)
- `serializers/archive.py:240` — `return get_display_name_for_user(obj.created_by) or "Unknown"`
- `serializers/intake.py` — `return get_display_name_for_user(intake.created_by)`
- `serializers/nurture.py` — `return get_display_name_for_user(nurture.created_by)`
- `serializers/patient.py` — `return get_display_name_for_user(patient.created_by)`
**Identical or divergent?** DIVERGENT — notes.py adds fallback logic (`updated_by or created_by`), archive.py:240 adds "Unknown" fallback, practice_treatment_plan_template.py and membership.py build the name manually instead of using `get_display_name_for_user()`.
**Consolidation note:** Create a single mixin method `get_created_by_name_field(created_by_field_name="created_by")` that handles both manual and utility-based approaches, with consistent fallback behavior.

## G2 — Permission classes [IsAuthenticated, FeatureAccessPermission] (49 sites)
**Category:** viewset boilerplate
**Belongs in:** a shared base class / mixin
**Sites:**
- `views/notes_views.py:82` — `permission_classes = [IsAuthenticated, FeatureAccessPermission]`
- `views/notes_views.py:132` — (duplicate)
- `views/notes_views.py:213` — (duplicate)
- `views/notes_views.py:254` — (duplicate)
- `views/patient_category_views.py:24` — (duplicate)
- `views/journey_stage_views.py:93` — (duplicate)
- `views/procedure_views.py:27` — (duplicate)
- `views/payment_views.py:25` — (duplicate)
- And 41 more instances across all view files
**Identical or divergent?** IDENTICAL — exact same pattern in all 49 locations
**Consolidation note:** Move to a base viewset class or mixin that both API and regular views can inherit. Current usage: most views inherit `PracticeAccessMixin` already; extend that mixin to set this permission class by default.

## G3 — Practice filtering gate (101 sites)
**Category:** viewset boilerplate / query fragment
**Belongs in:** a shared base class / mixin (already partially exists as PracticeAccessMixin)
**Sites:**
- `views/intake_views.py:139` — `practice = self.get_user_practice_or_none()` + `if not practice: return Intake.objects.none()`
- `views/nurture_views.py:106` — (identical pattern)
- `views/archive_views.py:59` — (identical pattern)
- `views/treatment_plan_views.py` — (multiple instances)
- `views/patient_views.py` — (multiple instances)
- `views/journey_stage_views.py` — (multiple instances)
- `views/custom_stage_views.py` — (multiple instances)
- And 88 more across all viewsets using `get_user_practice_or_none()`
**Identical or divergent?** IDENTICAL — all follow the same gate pattern
**Consolidation note:** `PracticeAccessMixin.get_user_practice_or_none()` already exists. Create a wrapper method `get_practice_filtered_queryset(model_class, **extra_filters)` that encapsulates the entire gate and default filtering chain.

## G4 — Date parsing with datetime.strptime (10 sites)
**Category:** request parsing
**Belongs in:** shared utils function
**Sites:**
- `views/intake_views.py:166` — `date_after = datetime.strptime(created_after, "%Y-%m-%d").date()`
- `views/intake_views.py:173` — (identical with before instead of after)
- `views/custom_stage_views.py:160` — (identical pattern)
- `views/custom_stage_views.py:167` — (identical pattern)
- `views/nurture_views.py:135` — (identical pattern)
- `views/nurture_views.py:142` — (identical pattern)
- `views/archive_views.py:113` — (identical pattern)
- `views/archive_views.py:120` — (identical pattern)
- `views/patient_views.py:1062` — (identical pattern)
- `views/patient_views.py:1072` — (identical pattern)
**Identical or divergent?** IDENTICAL — exact same pattern in all 10 locations, all wrapped in try/except ValueError
**Consolidation note:** Create `parse_date_param(date_string: str) -> Optional[date]` in `utils/` that returns None on ValueError instead of silent pass.

## G5 — Household member count annotation (7 sites)
**Category:** query fragment / annotation
**Belongs in:** a shared queryset builder or method in Person/Patient models
**Sites:**
- `selectors.py:63` — `.annotate(_hhm=Count("person__household__members", distinct=True))`
- `serializers/intake.py:350` — (identical)
- `serializers/intake.py:390` — (identical, in different method)
- `serializers/nurture.py:330` — (identical)
- `serializers/nurture.py:369` — (identical, in different method)
- `views/contact_merge_views.py:957` — (identical)
- `views/intake_views.py:379` — (identical)
**Identical or divergent?** IDENTICAL — exact same annotation across all 7 sites
**Consolidation note:** Extract to a model method: `Patient.objects.with_household_member_count()` using QuerySet.as_manager() pattern, or move to a custom QuerySet method.

## G6 — Last activity date annotation (10 sites)
**Category:** query fragment / annotation
**Belongs in:** a shared queryset builder
**Sites:**
- `views/intake_views.py:265` — `.annotate(last_date=Max("created_at"))`
- `views/intake_views.py:274` — `.annotate(last_date=Max("received_at"))`
- `views/custom_stage_views.py:231` — (identical with created_at)
- `views/custom_stage_views.py:240` — (identical with received_at)
- `views/nurture_views.py:220` — (identical with created_at)
- `views/nurture_views.py:231` — (identical with received_at)
- `views/treatment_plan_views.py:518` — (identical with created_at)
- `views/treatment_plan_views.py:529` — (identical with received_at)
- `views/patient_views.py:1120` — (identical with created_at)
- `views/patient_views.py:1131` — (identical with received_at)
**Identical or divergent?** IDENTICAL — exact same annotation pattern, split across two field names (created_at and received_at)
**Consolidation note:** Create shared method `add_last_activity_date(queryset, field_name)` utility in `utils/` that returns annotated queryset.

## G7 — Query parameter parsing with strip/lower (25+ sites)
**Category:** request parsing
**Belongs in:** a shared utility function or mixin method
**Sites:**
- `views/patient_views.py:379` — `email = (request.query_params.get("email") or "").strip()`
- `views/patient_views.py:1026` — `self.request.query_params.get("partial", "true").lower() != "false"`
- `views/patient_views.py:1044` — `request.query_params.get("category_id")`
- `views/patient_views.py:1048` — `request.query_params.get("uncategorized")`
- `views/patient_views.py:1057` — `request.query_params.get("created_after")`
- `views/treatment_plan_views.py:382` — `self.request.query_params.get("partial", "true").lower() != "false"`
- `views/treatment_plan_views.py:572` — (identical)
- `views/treatment_plan_views.py:642` — (identical)
- `views/treatment_plan_views.py:767` — (identical)
- `views/treatment_plan_views.py:1300` — (identical)
- `views/intake_views.py:1523` — `request.query_params.get("email", "").strip()`
- `views/intake_views.py:1524` — `request.query_params.get("phone", "").strip()`
- And 13 more similar patterns
**Identical or divergent?** IDENTICAL (for boolean coercion `.lower() != "false"`) and SIMILAR (for string stripping)
**Consolidation note:** Create helper methods: `get_bool_param(request, key, default=True)` and `get_stripped_param(request, key, default="")` in a mixin or utils module.

## G8 — get_queryset() boilerplate (49 viewset instances)
**Category:** viewset boilerplate
**Belongs in:** a shared base class / mixin
**Sites:**
- `views/intake_views.py:137, 482, 600, 635, 712, 1359` — 6 instances
- `views/nurture_views.py:104, 297, 424, 456, 506` — 5 instances
- `views/treatment_plan_views.py:253, 344` — 2 instances
- `views/patient_views.py:977` — 1 instance
- `views/custom_stage_views.py:94` — 1 instance
- `views/journey_stage_views.py:103` — 1 instance
- `views/archive_views.py:57` — 1 instance
- And 31 more across other views
**Identical or divergent?** DIVERGENT — same structure (get practice, check, filter) but different models, select_related chains, and extra filtering logic
**Consolidation note:** Create a method on PracticeAccessMixin: `get_practice_filtered_queryset(model_class, select_related=[], prefetch_related=[], **filters)` to eliminate boilerplate while allowing per-viewset customization.

## G9 — "Unknown" fallback for missing user names (3 sites)
**Category:** serializer getter / formatting
**Belongs in:** a shared utility function
**Sites:**
- `serializers/notes.py:52` — `return get_display_name_for_user(obj.updated_by or obj.created_by) or "Unknown"`
- `serializers/archive.py:240` — `return get_display_name_for_user(obj.created_by) or "Unknown"`
**Identical or divergent?** IDENTICAL
**Consolidation note:** Standardize fallback via `get_display_name_for_user()` default parameter or wrapper.

## Summary

| Pattern | Sites | Category | Divergent | Priority |
|---------|-------|----------|-----------|----------|
| Permission classes (IsAuthenticated + FeatureAccessPermission) | 49 | boilerplate | No | 1 |
| Practice filtering gate | 101 | boilerplate | No | 1 |
| Query parameter parsing (strip/lower/bool) | 25+ | request parsing | Similar | 2 |
| get_queryset() boilerplate | 49 | boilerplate | Yes | 2 |
| Last activity date annotation | 10 | query fragment | No | 3 |
| Date parsing (strptime) | 10 | request parsing | No | 3 |
| Household member count annotation | 7 | query fragment | No | 3 |
| get_created_by / get_created_by_name | 9 | serializer getter | Yes | 4 |
| "Unknown" fallback | 3 | formatting | No | 4 |

**Total duplicated definitions found: 263** (not counting method variants or near-duplicates)

**Top 3 by site count (highest consolidation ROI):**
1. **Practice filtering gate (101 sites)** — Extend `PracticeAccessMixin.get_practice_filtered_queryset()` to encapsulate the pattern
2. **Permission classes (49 sites)** — Add default `permission_classes` to a base viewset mixin
3. **get_queryset() (49 sites)** — Extract to a mixin method with model class + relation overrides as parameters

**Divergent patterns worth addressing:**
- `get_created_by_name` — 3 implementations (manual vs utility-based, with or without fallbacks) — standardize via helper
- `get_queryset()` — Same structure, different relations — wrap in mixin with per-viewset customization points

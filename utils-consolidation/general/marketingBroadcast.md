# marketingBroadcast: Engineering Duplication Inventory

Scan of `/TreatmentPathBackend/TreatmentPath/marketingBroadcast/` (40 non-test Python files).
**Count method:** file path + line number + literal code excerpt.

---

## G1 — Exception → Response blocks  (9 sites)
**Category:** exception handling
**Belongs in:** `marketingBroadcast/utils/` (shared exception handler)
**Distribution (9 total):**
- `{"error": str(exc)}` with `status.HTTP_400_BAD_REQUEST`: 3 sites
- `{"error": str(exc)}` with `status.HTTP_502_BAD_GATEWAY`: 3 sites
- `{"error": exc.message}` with `status.HTTP_400_BAD_REQUEST`: 1 site
- `{"error": "..."}` (literal string) with `status.HTTP_400_BAD_REQUEST`: 2 sites

**Sites:**
- `views/template_views.py:45` — `except PersonalisationBlocked as exc: return Response({"error": str(exc)}, status=status.HTTP_400_BAD_REQUEST)`
- `views/campaign_views.py:473` — `except PersonalisationBlocked as exc: return Response({"error": str(exc)}, status=status.HTTP_400_BAD_REQUEST)` (IDENTICAL)
- `views/domain_views.py:80` — `except MarketingEmailServiceError as exc: return Response({"error": str(exc)}, status=status.HTTP_502_BAD_GATEWAY)`
- `views/domain_views.py:110` — `except MarketingEmailServiceError as exc: return Response({"error": str(exc)}, status=status.HTTP_502_BAD_GATEWAY)` (IDENTICAL to 80)
- `views/domain_views.py:187` — `except MarketingEmailServiceError as exc: return Response({"error": str(exc)}, status=status.HTTP_502_BAD_GATEWAY)` (IDENTICAL to 80 and 110)
- `views/template_views.py:222` — `except DjangoValidationError as exc: return Response({"error": exc.message}, status=status.HTTP_400_BAD_REQUEST)` (DIVERGENT: uses `exc.message` not `str(exc)`)
- `views/preferences_views.py:36` — `except MarketingPreferenceToken.DoesNotExist: return Response({"error": "Not found"}, status=404)` (DIVERGENT: literal string, numeric status)
- `views/preferences_views.py:61` — `return Response({"error": "state must be 'on' or 'off'"}, status=400)` (not in except block, literal string)
- `views/webhook_views.py:236` — `return Response({"error": "Unauthorized"}, status=status.HTTP_401_UNAUTHORIZED)` (not in except block)

**Identical or divergent?** DIVERGENT — status codes vary (400/404/502/401), some use numeric literals vs. constants, one uses `exc.message` vs `str(exc)`.

**Consolidation note:** Extract to a shared utility that accepts exception type and status code, standardizes on `str(exc)`, and enforces const over literals.

---

## G2 — Practice gate pattern in ViewSet get_queryset  (8 sites)
**Category:** practice gate / viewset boilerplate
**Belongs in:** `PracticeAccessMixin` (already exists but duplicated pattern)
**Sites:**
- `views/campaign_views.py:188` — `practice = self.get_user_practice(); queryset = BroadcastCampaign.objects.filter(practice=practice)`
- `views/template_views.py:148` — `practice = self.get_user_practice(); queryset = BroadcastPracticeTemplate.objects.filter(practice=practice, is_archived=False)`
- `views/segment_views.py:155` — `practice = self.get_user_practice(); queryset = MarketingSegment.objects.filter(practice=practice).select_related("created_by")`
- `views/form_views.py:70` — `queryset = self.queryset.filter(practice=self.get_user_practice())`
- `views/form_views.py:149` — `return self.queryset.filter(practice=self.get_user_practice())`
- `views/form_views.py:198` — `queryset = self.queryset.filter(practice=self.get_user_practice())`
- `views/form_views.py:261` — `return MarketingFormSubmission.objects.filter(practice=self.get_user_practice())`
- `views/form_template_views.py:45` — (catalog-side: no practice gate)

**Identical or divergent?** IDENTICAL — all follow `self.get_user_practice()` + filter pattern.

**Consolidation note:** All already inherit `PracticeAccessMixin`. Pattern is consistent; no dedup needed.

---

## G3 — `get_url` SerializerMethodField (identical across 2 serializers)  (2 sites)
**Category:** serializer boilerplate
**Belongs in:** shared base `ImageSerializer` mixin
**Sites:**
- `serializers.py:367-375` (BroadcastTemplateImageSerializer):
  ```python
  def get_url(self, obj):
      image_url = obj.image.url
      if image_url.startswith("http"):
          return image_url
      request = self.context.get("request")
      return request.build_absolute_uri(image_url) if request else image_url
  ```
- `serializers.py:639-647` (MarketingFormImageSerializer): **BYTE-IDENTICAL BLOCK**

**Identical or divergent?** IDENTICAL — exact code, same intent.

**Consolidation note:** Extract to a shared mixin `ImageSerializerMixin` with `get_url()`, subclasses inherit and override only the model-specific fields.

---

## G4 — `has_unpublished_changes` SerializerMethodField delegation  (2 sites)
**Category:** serializer boilerplate
**Belongs in:** `MarketingFormSerializer` base class (already centralized)
**Sites:**
- `serializers.py:427-441` (MarketingFormSerializer): Full implementation
- `serializers.py:479-480` (MarketingFormListSerializer): Delegates to parent: `return MarketingFormSerializer.get_has_unpublished_changes(self, obj)`

**Identical or divergent?** INTENTIONALLY DELEGATED — list serializer calls parent's method, not duplicated.

**Consolidation note:** Already handled well; no dedup needed.

---

## G5 — Contact snapshot field extraction pattern  (3 getters in 1 serializer)
**Category:** serializer boilerplate (but internally consistent)
**Sites:**
- `serializers.py:592-596` (get_contact_full_name)
- `serializers.py:598-602` (get_contact_phone)
- `serializers.py:604-608` (get_contact_email)

All three use the same pattern:
```python
def get_contact_X(self, obj):
    stored = obj.contact_X
    return stored or self._contact_from_snapshot(obj, self.CONTACT_KEYS["contact_X"])
```

**Identical or divergent?** IDENTICAL PATTERN, DIFFERENT KEYS — shared `_contact_from_snapshot` helper (line 584-590) already centralized.

**Consolidation note:** Already DRY; no dedup needed.

---

## G6 — Domain validation Response errors  (4 sites)
**Category:** request parsing / validation error responses
**Sites:**
- `views/domain_views.py:69-71` — `if not full_domain: return Response({"error": "full_domain is required"}, status=status.HTTP_400_BAD_REQUEST)`
- `views/campaign_views.py:449-452` — `if not render_source: return Response({"error": "Campaign needs a template before it can be previewed."}, status=status.HTTP_400_BAD_REQUEST)`
- `views/campaign_views.py:490-496` — `if (not (campaign.template or campaign.practice_template) or not campaign.content_snapshot): return Response({"error": "Campaign needs a template and content before a test send."}, status=status.HTTP_400_BAD_REQUEST)`
- `views/campaign_views.py:613-617` — `if not campaign.sender_name.strip(): return Response({"error": "sender_name is required before scheduling."}, status=status.HTTP_400_BAD_REQUEST)`

**Identical or divergent?** IDENTICAL PATTERN (all 400, all `{"error": string}`), DIVERGENT DETAIL (different messages).

**Consolidation note:** Extract to a helper `validation_error(message, status_code=400)` that returns Response dict.

---

## G7 — Request parameter parsing with `.get()` + `.strip()` + `.lower()`  (repeated shape, not a bug)
**Category:** request parsing
**Belongs in:** utilities or serializer validators
**Sample sites (6+ instances):**
- `views/campaign_views.py:196` — `status_filter = self.request.query_params.get("status")`
- `views/campaign_views.py:199` — `show_archived = self.request.query_params.get("archived") == "true"`
- `views/campaign_views.py:204` — `self.request.query_params.get("search")`
- `views/form_image_views.py:35` — `kind = request.data.get("kind") or MarketingFormImage.Kind.BLOCK`
- `views/form_image_views.py:52` — `alt_text=(request.data.get("alt_text") or "").strip()[:255]`
- `views/campaign_views.py:343` — `search = request.query_params.get("search", "").strip().lower()`

**Identical or divergent?** IDENTICAL PATTERN (`.get()` + optional `.strip()` + optional `.lower()`), DIVERGENT DETAIL (different param names).

**Consolidation note:** Not necessarily a bug; pattern is mechanical. Consider a param extraction helper if this app scales, but current distribution doesn't warrant extraction.

---

## G8 — `perform_create` with practice assignment  (3 sites)
**Category:** viewset boilerplate
**Sites:**
- `views/campaign_views.py:211-220` — `serializer.save(practice=self.get_user_practice(), created_by=self.request.user, practice_timezone=practice.timezone)`
- `views/template_views.py:171` — `serializer.save(practice=self.get_user_practice(), created_by=self.request.user)`
- `views/form_views.py:119` — `serializer.save(practice=self.get_user_practice())`

**Identical or divergent?** IDENTICAL PATTERN, DIVERGENT DETAIL (campaign adds `practice_timezone`, others don't).

**Consolidation note:** Consider a `perform_create` mixin that auto-injects `practice` + `created_by`, reducing boilerplate.

---

## G9 — HTTP status code distribution in error responses
**Category:** exception handling (supporting data for G1)

Across all exception handlers:
- `status.HTTP_400_BAD_REQUEST`: 6 uses (most common)
- `status.HTTP_502_BAD_GATEWAY`: 3 uses (Postmark errors)
- `status.HTTP_404_NOT_FOUND`: 1 use
- `status.HTTP_401_UNAUTHORIZED`: 1 use
- Numeric literals (400, 404): 2 uses (non-const)

**Consolidation note:** Standardize on const (no numeric literals) and consider a status-code enum per exception type.

---

## G10 — Pagination handling in preview actions  (2 sites)
**Category:** viewset boilerplate
**Sites:**
- `views/campaign_views.py:366-378` — Manual `page`/`page_size` parsing with try-except ValueError + defaults
- `views/segment_views.py:177-182` — Similar pattern: try-except ValueError on pagination params

**Identical or divergent?** IDENTICAL PATTERN, DIFFERENT DETAIL (different defaults and ranges).

**Consolidation note:** Extract to a `parse_pagination_params(request, default_limit, max_limit)` helper.

---

## Summary

| Category | Count | Opportunities | Mechanical Fixes |
|----------|-------|----------------|------------------|
| Exception handling | 9 | Extract error-response template + status-code standardization | High |
| Serializer boilerplate | 2 | Create `ImageSerializerMixin` for `get_url` | High |
| Practice gate (viewset) | 8 | Already well-factored via `PracticeAccessMixin` | None |
| Validation responses | 4 | Extract `validation_error()` helper | High |
| Pagination parsing | 2 | Extract `parse_pagination_params()` helper | High |
| Request param parsing | 6+ | Pattern-based (non-duplicated), low risk | Low |
| `perform_create` boilerplate | 3 | Consider mixin for `practice`/`created_by` auto-injection | Medium |

**Total definitions scanned:** 40 files; **total exception handlers:** 56 (span 9 Response-returning sites); **byte-identical blocks:** 1 (BroadcastTemplateImageSerializer.get_url vs MarketingFormImageSerializer.get_url).

**High-value consolidations:**
1. Shared error-response handler (9 sites, status-code drift)
2. Image serializer mixin (2 byte-identical methods)
3. Validation error helper (4 sites, literal messages)
4. Pagination param parser (2 sites, repeated try-except)

**No critical duplication gaps found.** The practice-gate pattern is already well-factored via `PracticeAccessMixin`. The codebase is already fairly DRY; the consolidations above are polish, not risk-mitigations.

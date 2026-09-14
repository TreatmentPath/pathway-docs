# Documents App: Duplicate Logic Inventory

Scan of `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/Documents/` for duplicate patterns and consolidation opportunities. Consent and signing logic is medico-legally sensitive — consistency here prevents the exact class of bugs that produced 90+ patient-identity issues elsewhere.

---

## B1 — Latest consent status mapping (2 sites)

**Canonical helper:** NONE — would need creating, or standardise on one

**Belongs in:** App `Documents/utils/` (rename one, delete other)

**Sites:**
- `Documents/views/signing_views.py:725-738` — `patient_signing_summary()` action
  ```python
  if latest.status == "signed":
      latest_consent_status = "signed"
  elif latest.status in ("sent", "viewed", "pending"):
      latest_consent_status = "pending"
  elif latest.status in ("expired",):
      latest_consent_status = "expired"
  elif latest.status == "declined":
      latest_consent_status = "declined"
  else:
      latest_consent_status = "not_required"
  ```

- `Documents/views/signing_views.py:886-894` — `signing_summary()` action on `PatientSigningDocumentsView`
  ```python
  status_map = {
      "signed": "signed",
      "sent": "pending",
      "viewed": "pending",
      "pending": "pending",
      "expired": "expired",
      "declined": "declined",
  }
  latest_consent_status = status_map.get(latest.status, "not_required")
  ```

**Do they actually agree?** FUNCTIONALLY IDENTICAL. Method differs: if/elif vs dict.get(). Both default to "not_required" on unmapped status.

**Risk:** Maintenance burden — updating consent status logic in one place risks leaving the other stale, leading to inconsistent summary responses from the two endpoints even though they compute the same value. The logic belongs in PRD; if PRD changes (e.g. "pending" becomes "awaiting_signature"), both must update together.

---

## B2 — _get_client_ip() (2 sites, identical)

**Canonical helper:** NONE — would create in shared location

**Belongs in:** Global `TreatmentPath/utils/` or app-scoped `Documents/utils/`

**Sites:**
- `Documents/views/public_views.py:42-49`
  ```python
  def _get_client_ip(request):
      x_forwarded = request.META.get("HTTP_X_FORWARDED_FOR")
      if x_forwarded:
          return x_forwarded.split(",")[0].strip()
      return request.META.get("REMOTE_ADDR")
  ```

- `Documents/views/signing_views.py:145-149` — identical code

**Do they actually agree?** IDENTICAL — exact same implementation.

**Risk:** If a bug is found (e.g., X-Forwarded-For parsing under IPv6), both must be fixed. Separate definitions = duplication = missed fixes. This is one of the exact patterns that caused identity bugs elsewhere (same lookup, two places, one path updated).

**Sibling miss:** `_get_device_info()` is defined only in public_views.py (line 49-50), not replicated in signing_views.py, so signing_views.py builds device_info inline instead (signing_views.py:212-215). That asymmetry compounds the fragmentation.

---

## B3 — _get_practice() ViewSet method (4 sites, mostly identical)

**Canonical helper:** Already exists in codebase (PracticeAccessMixin) — NONE these should be using it

**Belongs in:** Mixin (`TreatmentPath/utils/practice_mixins.py` — use PracticeAccessMixin.get_current_practice or similar)

**Sites:**
- `Documents/views/bundle_views.py:19-20`
  ```python
  def _get_practice(self):
      return getattr(self.request.user, "current_practice", None)
  ```

- `Documents/views/template_views.py:29-33`
  ```python
  def _get_practice(self):
      practice = getattr(self.request.user, "current_practice", None)
      if not practice:
          return None
      return practice
  ```

- `Documents/views/signing_views.py:387-388` (SigningRequestViewSet)
  ```python
  def _get_practice(self):
      return getattr(self.request.user, "current_practice", None)
  ```

- `Documents/views/signing_views.py:845-846` (PatientSigningDocumentsView)
  ```python
  def _get_practice(self):
      return getattr(self.request.user, "current_practice", None)
  ```

**Do they actually agree?** MOSTLY IDENTICAL. Template_views has an extra None-check that returns None regardless of condition (lines 31-33 are a tautology — `if not practice: return None` followed by `return practice` means both branches return the same thing).

**Risk:** Low on correctness (all return the current practice or None), but high on code duplication. Each ViewSet redefines the same 1-3 line method. Should inherit from a mixin or call a shared helper.

---

## B4 — Name joining via f-string instead of full_name() helper (3 sites)

**Canonical helper:** `TreatmentPlan.utils.names.full_name()`

**Belongs in:** Use the canonical helper everywhere; delete inline implementations

**Sites:**
- `Documents/models.py:474-478` — `SigningRequest.recipient_display_name` property for patient
  ```python
  if self.patient_id:
      first = getattr(self.patient, "first_name", "") or ""
      last = getattr(self.patient, "last_name", "") or ""
      return (
          f"{first} {last}".strip()
          or getattr(self.patient, "email", "")
          or "Patient"
      )
  ```

- `Documents/models.py:483-487` — same property for recipient_user
  ```python
  if self.recipient_user_id:
      user = self.recipient_user
      first = getattr(user, "first_name", "") or ""
      last = getattr(user, "last_name", "") or ""
      return (
          f"{first} {last}".strip() or getattr(user, "email", "") or "Recipient"
      )
  ```

- `Documents/serializers.py:26-32` — `_user_display()` helper
  ```python
  def _user_display(user):
      if user is None:
          return None
      first = getattr(user, "first_name", "") or ""
      last = getattr(user, "last_name", "") or ""
      full = f"{first} {last}".strip()
      return full or getattr(user, "email", None)
  ```

**Do they actually agree?** FUNCTIONALLY EQUIVALENT. All three:
  1. Guard against None first_name/last_name with getattr + "or" trick
  2. Collapse internal whitespace with `.strip()`
  3. Fall back to email or default

However, `full_name()` is the canonical, centrally-maintained implementation that handles all edge cases (internal multiple spaces, Unicode, None values, etc.). Using it everywhere prevents drift.

**Risk:** If `full_name()` is updated (e.g., to handle internal spaces more aggressively), these three inline implementations miss the fix — names display differently across the app. For consent documents, this is a clinical record — incorrect recipient name is a data-quality issue.

---

## B5 — Audit event creation with slightly different signatures (2 sites, similar intent)

**Canonical helper:** NONE — would need creating or standardising

**Belongs in:** App `Documents/utils/`

**Sites:**
- `Documents/views/signing_views.py:206-225` — `_create_audit_event()`
  ```python
  def _create_audit_event(
      signing_request, event_type, request=None, channel=None, metadata=None
  ):
      """Append an audit event to a signing request."""
      actor = request.user if request and request.user.is_authenticated else None
      actor_ip = _get_client_ip(request) if request else None
      device_info = {}
      if request:
          ua = request.META.get("HTTP_USER_AGENT", "")
          device_info = {"user_agent": ua}
      SigningAuditEvent.objects.create(
          signing_request=signing_request,
          event_type=event_type,
          channel=channel,
          actor=actor,
          actor_ip=actor_ip,
          device_info=device_info,
          metadata=metadata or {},
      )
  ```
  — Handles None request; builds device_info inline.

- `Documents/views/public_views.py:53-63` — `_append_audit()`
  ```python
  def _append_audit(signing_request, event_type, request, channel=None, metadata=None):
      actor = request.user if request.user and request.user.is_authenticated else None
      SigningAuditEvent.objects.create(
          signing_request=signing_request,
          event_type=event_type,
          channel=channel,
          actor=actor,
          actor_ip=_get_client_ip(request),
          device_info=_get_device_info(request),
          metadata=metadata or {},
      )
  ```
  — Requires request (no None guard); delegates device_info to helper.

**Do they actually agree?** SIMILAR INTENT, DIFFERENT DETAILS:
- `_create_audit_event` signature allows `request=None`; `_append_audit` requires request.
- `_create_audit_event` builds device_info inline with just user_agent; `_append_audit` delegates to `_get_device_info()`.

No functional disagreement (both create SigningAuditEvent with same fields), but signatures don't match, so callers must remember which to use where.

**Risk:** Low on data correctness (both work), but high on maintainability and API clarity. Should be one audit-creation function with consistent signature.

**Sibling miss:** `_get_device_info()` exists only in public_views (not in signing_views), so signing_views.py manually rebuilds it at line 212-215.

---

## B6 — Public-link authorization stubs (8 call sites, 1 definition)

**Canonical helper:** `Documents/views/public_views.py:111-120` — `_check_recipient_access()`

**Belongs in:** Same location (intentionally a no-op stub; design decision documented)

**Sites (call only):**
- `Documents/views/public_views.py:288` (PublicSigningSessionView.get)
- `Documents/views/public_views.py:314` (PublicMarkViewedView.post)
- `Documents/views/public_views.py:357` (PublicConfirmDocumentView.post)
- `Documents/views/public_views.py:462` (PublicDeclareDocumentView.post)
- `Documents/views/public_views.py:631` (PublicSubmitSignatureView.post)
- `Documents/views/public_views.py:818` (PublicSubmitFieldValuesView.post)
- `Documents/views/public_views.py:967` (PublicPdfProxyView.get)
- Plus comment reference at line 536 and test reference at line 748

**Do they actually agree?** N/A — single definition, called 8 times. The function is a no-op stub:
```python
def _check_recipient_access(req, request):
    """
    FR9 (staff/compliance signing requires a matching login) has been
    reverted — every signing request (patient or staff/compliance) is now
    fully anonymous via the link alone, matching the original design.
    Kept as a no-op function rather than removed at each of its 8 call
    sites, so re-enabling later only touches this one place. Always
    returns None (access allowed).
    """
    return None
```

**Risk:** NONE. This is intentional: a disabled feature left as a no-op placeholder so re-enabling touches one place. Design decision explicitly documented in docstring.

---

## Summary

| Behaviour | Sites | Agree? | Proposed Home |
|-----------|-------|--------|---------------|
| Latest consent status mapping (if/elif vs dict) | 2 | Functionally identical, method differs | `Documents/utils/` (create `map_signing_status()`) |
| _get_client_ip() | 2 | Identical | `TreatmentPath/utils/` or `Documents/utils/` |
| _get_practice() method | 4 | Mostly identical (one has tautology) | Mixin or shared method |
| Name joining (f-string vs helper) | 3 | Equivalent but not using canonical | Use `TreatmentPlan.utils.names.full_name()` everywhere; delete inline |
| Audit event creation | 2 | Similar intent, different signatures | Standardise one function, consolidate signatures |
| Public-link auth (no-op stubs) | 8 call sites | Single definition, intentional | Keep as-is (design decision documented) |

### Key Findings

1. **Code duplication count:** 4 distinct duplicate patterns (not counting the intentional no-op auth stubs).

2. **Author-membership gate scan:** No author-membership gates found (`user__practices=` pattern). All practice scoping gates on the RECORD's practice field itself (correct pattern).

3. **Sibling misses:** `_get_device_info()` defined only in public_views.py; signing_views.py duplicates its logic inline. Not a high-risk miss but fragments helpers.

4. **Non-deterministic .first():** All `.first()` calls are ordered (order_by before them) or on unique/constrained querysets (pk=), so no risk of non-determinism.

5. **Unordered querysets:** None found in staff-facing or public views.

6. **Cross-file consistency:** `_check_recipient_access` calls are spread across 8 sites but all call the same (intentional) no-op; no drift risk. Same for `_resolve_request` and `_check_expired` helpers.

### Recommended Immediate Actions

1. **Consolidate consent status mapping:** Extract to `Documents/utils/` as `map_signing_request_status(status: str) -> str` and call from both endpoints. Reduces update burden and ensures PRD changes apply uniformly.

2. **Replace inline name-joining with full_name():** Replace lines 474-478 and 483-487 in models.py and 26-32 in serializers.py with calls to the canonical `full_name()` helper. Low-risk refactor; centralises one of the exact patterns that produced name-joining bugs elsewhere.

3. **Extract shared audit-event function:** Create `Documents/utils/audit.py` with single `create_audit_event()` function, compatible signature for both staff and public paths, and call it from both views.

4. **Extract _get_client_ip():** Move to `Documents/utils/` (or reuse if one exists globally); remove from signing_views.py.

5. **Unify _get_practice() methods:** Either create a mixin, or call a shared helper. Four ViewSets duplicating 2 lines each is unnecessary.

---

**Scan completeness:** All non-migration, non-test `.py` files under Documents/ scanned. No logic in admin.py, no omitted subdirectories.

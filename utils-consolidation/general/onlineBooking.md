# onlineBooking Duplication Inventory

## G1 — OnlineBookingProfile get_or_create boilerplate  (2 sites)
**Category:** viewset boilerplate
**Belongs in:** a shared method or base class mixin
**Sites:**
- `views.py:92-95` — `profile, _ = OnlineBookingProfile.objects.get_or_create(practice=self.get_practice(), defaults={"public_slug": self.get_practice().slug})`
- `views.py:99-102` — identical, in patch() of same view

**Identical or divergent?** IDENTICAL

**Consolidation note:** Extract to helper method; avoid duplication between get/patch in a single view class.

---

## G2 — PracticeStripeAccount get_or_create boilerplate  (2 sites)
**Category:** viewset boilerplate
**Belongs in:** a shared method or base class mixin
**Sites:**
- `views.py:155-159` — `account, _ = PracticeStripeAccount.objects.get_or_create(practice=practice, defaults={"created_by": request.user})`
- `views.py:163-166` — identical, in patch() of same view

**Identical or divergent?** IDENTICAL

**Consolidation note:** Extract to helper method in PracticeStripeSettingsView or shared base.

---

## G3 — Public hold lookup with profile slug validation  (6 sites)
**Category:** public-endpoint guard
**Belongs in:** a shared base mixin for public hold endpoints
**Sites:**
- `views.py:401-405` — OnlineBookingPublicConfirmUnpaidView.post()
- `views.py:434-437` — OnlineBookingPublicCheckoutView.post()
- `views.py:493-496` — OnlineBookingPublicHoldStatusView.get()
- `views.py:512-516` — OnlineBookingPublicHoldReleaseView.post()
- `views.py:969-972` — OnlineBookingPublicSessionUpdateView.patch()
- `views.py:1001-1005` — OnlineBookingPublicSessionCompleteView.post()

**Code pattern:**
```python
hold = get_object_or_404(
    OnlineBookingHold,
    id=hold_id,
    profile__public_slug=public_slug,
)
```

**Identical or divergent?** IDENTICAL across all 6 sites

**Consolidation note:** Move to PublicProfileMixin or new PublicHoldMixin with get_hold() method.

---

## G4 — Device token encryption/payload building  (2 sites, divergent)
**Category:** public-endpoint plumbing — device-token encrypt/decrypt
**Belongs in:** a shared utils function or mixin method
**Sites:**
- `views.py:610-617` — OnlineBookingPublicEmailVerifyConfirmView (initial issuance)
- `views.py:752-759` — OnlineBookingPublicEmailVerifyDeviceSkipView (renewal)

**Code pattern (both):**
```python
device_token = encrypt_otp_payload({
    "type": "booking_device_trust",
    "email": email,
    "public_slug": public_slug,
    "verified_at": timezone.now().isoformat(),
})
```

**Identical or divergent?** IDENTICAL payload structure; only variable name differs (device_token vs new_device_token)

**Consolidation note:** Extract to helper function _issue_device_token(email, public_slug) to avoid duplication and ensure consistency.

---

## G5 — Practitioner name formatting  (3 sites, divergent)
**Category:** serializer getter / response building
**Belongs in:** a shared utility function or base serializer
**Sites:**
- `views.py:480` — OnlineBookingPublicServicePractitionerListView: `f"{p.first_name or ''} {p.last_name or ''}".strip() or p.email`
- `views.py:684` — _build_verified_response(): `f"{h.practitioner.first_name} {h.practitioner.last_name}".strip() or h.practitioner.email`
- `serializers.py:421-424` — OnlineBookingServicePractitionerSerializer.get_name(): `(f"{first} {last}".strip()) or obj.practitioner.email`

**Identical or divergent?** DIVERGENT: views.py:480 adds `or ''` to guard against None on first/last name; views.py:684 does not (could crash if first_name is None). Serializer uses intermediate variables but is logically identical to 480.

**Consolidation note:** One copy treats None gracefully; others don't. Consolidate with guards, or create a Practitioner.display_name property on User model or a utility function in a shared location.

---

## G6 — Canonical email extraction from request  (3 sites)
**Category:** request parsing
**Belongs in:** a shared utils function or middleware
**Sites:**
- `views.py:535` — OnlineBookingPublicEmailVerifySendView: `email = canonical_email(request.data.get("email")) or ""`
- `views.py:583` — OnlineBookingPublicEmailVerifyConfirmView: `email = canonical_email(request.data.get("email")) or ""`
- `views.py:712` — OnlineBookingPublicEmailVerifyDeviceSkipView: `email = canonical_email(request.data.get("email")) or ""`

**Identical or divergent?** IDENTICAL

**Consolidation note:** Extract to a helper or create a mixin for OTP/email verification views. The pattern is request.data.get → canonical_email → validate → use.

---

## G7 — Exception → Response blocks  (19 sites; distributed)
**Category:** exception handling
**Belongs in:** standardized error response format
**Distribution of response keys and status codes:**

- `{"detail": "..."}` + status.HTTP_400_BAD_REQUEST: **12 sites** (lines 175, 187, 196, 233, 249, 328, 341, 389, 407, 518, 537, 584)
- `{"detail": "..."}` + status.HTTP_401_UNAUTHORIZED: **3 sites** (lines 723, 733, 743)
- `{"detail": "..."}` + status.HTTP_409_CONFLICT: **1 site** (line 390)
- `{"detail": "..."}` + status.HTTP_500_INTERNAL_SERVER_ERROR: **1 site** (line 568)
- Other response keys (no "detail"): `{"results": ...}`, `{"hold_id": ..., "status": ...}`, etc.

**Identical or divergent?** CONSISTENT use of "detail" key; status codes match exception semantics. No divergence.

**Consolidation note:** Already well-standardized on `{"detail": "message"}` format. No consolidation needed.

---

## G8 — URL building for profile logos / avatar images  (2 sites)
**Category:** URL / image builders
**Belongs in:** a shared method or mixin
**Sites:**
- `serializers.py:66` — OnlineBookingProfileSerializer.get_logo_url(): 
  ```python
  if request:
      return request.build_absolute_uri(obj.practice.logo.url)
  else:
      return obj.practice.logo.url
  ```
- `views.py:471-475` — OnlineBookingPublicServicePractitionerListView:
  ```python
  if getattr(p, "profile_picture", None):
      url = p.profile_picture.url
      if url.startswith("/"):
          url = request.build_absolute_uri(url)
      avatar_url = url
  ```

**Identical or divergent?** DIVERGENT: serializer always uses build_absolute_uri when request is present; view only calls it for relative URLs (startswith "/"?). View's pattern is safer (avoids double-qualifying absolute URLs).

**Consolidation note:** Create a shared utility `make_absolute_url(url, request)` that checks for leading "/" before calling build_absolute_uri.

---

## G9 — Session field update with repetitive if-checks  (1 site; pattern)
**Category:** viewset boilerplate
**Belongs in:** a mixin or serializer update method
**Sites:**
- `views.py:978-993` — OnlineBookingPublicSessionUpdateView.patch():
  ```python
  if "current_step" in d:
      session.current_step = d["current_step"]
  if "service_id" in d:
      session.service_id = d["service_id"]
  # ... 6 more identical if-blocks
  session.save()
  ```

**Identical or divergent?** SINGLE SITE but represents a mechanical pattern that could be generalized.

**Consolidation note:** Use a loop over validated_data keys or a dict of field mappings to reduce boilerplate. This is low duplication now but becomes a problem if replicated in other session/model updates.

---

## G10 — Hold/appointment ID serialization  (4 sites)
**Category:** response building
**Belongs in:** serializer method field or utility
**Sites:**
- `views.py:426` — `"appointment_id": str(appointment.id)`
- `views.py:450` — `"payment_id": str(payment.id)`
- `views.py:500` — `"hold_id": str(hold.id)`
- `views.py:524` — `"hold_id": str(hold.id)` (repeated)
- Plus list serialization at `views.py:678` (hold ID in upcoming array)

**Identical or divergent?** IDENTICAL pattern: convert UUID to string for response JSON.

**Consolidation note:** Serializer method field pattern; no consolidation needed. String conversion is inevitable for JSON.

---

## Summary

| Finding | Category | Sites | Divergent? | Consolidation Priority |
|---------|----------|-------|-----------|------------------------|
| G1: OnlineBookingProfile get_or_create | viewset boilerplate | 2 | No | HIGH |
| G2: PracticeStripeAccount get_or_create | viewset boilerplate | 2 | No | HIGH |
| G3: Public hold lookup (profile slug) | public-endpoint guard | 6 | No | HIGH |
| G4: Device token encryption | public-endpoint plumbing | 2 | No | MEDIUM |
| G5: Practitioner name formatting | serializer getter | 3 | Yes* | MEDIUM |
| G6: Canonical email extraction | request parsing | 3 | No | MEDIUM |
| G7: Exception → Response blocks | exception handling | 19 | No | LOW (already consistent) |
| G8: URL building (logo/avatar) | URL builders | 2 | Yes** | LOW |
| G9: Session field updates | viewset boilerplate | 1 | No | LOW (not yet duplicated) |
| G10: ID serialization | response building | 5+ | No | LOW (inevitable) |

**Total distinct definitions**: 10 categories, **19 sites with exception handlers (already consistent)**, **27 sites for other duplications**.

**Divergent findings:**
- G5: Practitioner name formatting has guards (vs. unguarded) — consolidate with safe version.
- G8: URL building differs on when to call build_absolute_uri — consolidate with the safer relative-check version.

**No divergent public-endpoint guards found** — G3 (hold lookup) is IDENTICAL across all 6 sites.

---

## Notes

1. Exception handling is already well-standardized on `{"detail": "message"}` + appropriate HTTP status codes. No drift.
2. G3 is the biggest consolidation opportunity: 6 identical guard patterns across public endpoints. A PublicHoldMixin with get_hold(public_slug, hold_id) would eliminate all 6.
3. G1 and G2 can be resolved by extracting get/patch-shared get_or_create calls into a single method (_get_or_create_profile, etc.).
4. G5 (practitioner name) needs a safety audit first — views.py:684 will crash if practitioner.first_name is None; the other two guard against it.

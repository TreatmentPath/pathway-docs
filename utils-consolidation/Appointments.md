# Appointments Directory — Duplicate Logic Inventory

Scanned: `/home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath/Appointments/`  
Date: 2026-09-07

## B1 — Email Comparison (2 sites, DIFFER)

**Canonical helper:** `TreatmentPlan/utils/contact_keys.py::canonical_email()` — would normalize both before comparison

**Belongs in:** global `utils/` comparison wrapper OR existing public_booking_views.py pattern consistency

**Sites:**
- `public_booking_views.py:313` — Anti-spam filter uses raw email from validated_data without normalization:
  ```python
  pending_count = Appointment.objects.filter(
      patient_email=data["patient_email"],
      status=Appointment.Status.PENDING,
      created_at__gte=timezone.now() - timedelta(hours=1),
  ).count()
  ```
- `public_booking_views.py:526` — Cancel endpoint uses `.lower()` for comparison:
  ```python
  if appointment.patient_email.lower() != email.lower():
      return Response({"error": "Email does not match booking"}, ...)
  ```

**Do they actually agree?** NO. Filter 1 searches raw email field; Filter 2 compares lowercase. If an email was stored as "John@example.com" and someone calls the cancel API with "john@example.com", the cancel check passes but the earlier anti-spam filter would miss prior pending bookings under "John@example.com".

**Risk:** Email case-sensitivity confusion; potential double-booking or lost spam prevention on re-attempts with different casing.

**Reachability:** YES — common real-world scenario (user typos, different form sources).

---

## B2 — Patient Full Name Building (2 sites, SIBLING MISSES)

**Canonical helper:** `TreatmentPlan/utils/names.py::full_name(first, last)` — the ONE display join

**Belongs in:** already correct in Display Mixin; fix sibling in ShortNoticePatientSerializer

**Sites:**
- `serializers.py:50-60` (`_PatientDisplayNameMixin.get_patient_display_name()`) — Uses canonical `full_name()` helper:
  ```python
  from TreatmentPlan.utils.names import full_name
  if obj.patient:
      name = full_name(
          getattr(obj.patient, "first_name", ""),
          getattr(obj.patient, "last_name", ""),
      )
  ```
- `serializers.py:914-921` (`ShortNoticePatientSerializer.get_patient_name()`) — Manual f-string, no collapse:
  ```python
  first = getattr(obj.patient, "first_name", "") or ""
  last = getattr(obj.patient, "last_name", "") or ""
  name = f"{first} {last}".strip()
  return name if name else str(obj.patient)
  ```

**Do they actually agree?** IDENTICAL OUTPUT on normal data, but ShortNotice does not collapse internal whitespace (e.g., `"John  Doe"` stays as-is; full_name collapses to `"John Doe"`).

**Risk:** Display inconsistency across appointment and short-notice views if a patient record has internal whitespace in name fields. Low probability (data entry validation likely prevents this), but the duplication invites divergence.

**Reachability:** Only if Patient.first_name or last_name contain multiple spaces.

---

## B3 — Phone Number Retrieval from Patient (2 sites, DIFFER IN APPROACH)

**Canonical helper:** `TreatmentPlan/utils/phones.py::canonical_phone_e164()` — normalizes E.164 format; already used in ShortNotice

**Belongs in:** already correct in ShortNoticePatientSerializer; AppointmentListSerializer needs same logic

**Sites:**
- `serializers.py:125-131` (`AppointmentListSerializer.get_patient_phone()`) — Returns raw phone from patient, no normalization:
  ```python
  def get_patient_phone(self, obj):
      """Return appointment-level phone; fall back to linked patient record."""
      if obj.patient_phone:
          return obj.patient_phone
      if obj.patient:
          return getattr(obj.patient, "phone_number", None) or ""
      return ""
  ```
- `serializers.py:923-947` (`ShortNoticePatientSerializer.get_patient_phone()`) — Uses `canonical_phone_e164()`:
  ```python
  canonical = canonical_phone_e164(phone, country_code)
  if canonical and str(canonical).startswith("+"):
      return str(canonical)
  return phone
  ```

**Do they actually agree?** NO. Appointment returns raw, possibly unnormalized phone; ShortNotice normalizes to E.164 with country code handling (including ISO→dial-code conversion).

**Risk:** Appointment diary shows `"+GB4473781631659"` for ISO code "GB"; ShortNotice shows `"+447378..."` for the same patient. Inconsistent display; more importantly, if code ever matches phones across the two views, it will fail.

**Reachability:** YES — production has 3,167 ISO "GB" codes in country_code field (documented in memory as country_code_is_not_a_dial_code.md).

---

## B4 — Patient Email Fallback (2 sites, BOTH CORRECT, DIFFERENT CONTEXT)

**Canonical helper:** None needed; fallback pattern is intentional per business logic

**Belongs in:** both are correct as-is (document why, if not obvious)

**Sites:**
- `serializers.py:117-123` (`AppointmentListSerializer.get_patient_email()`) — Returns appointment email if set, else patient email:
  ```python
  def get_patient_email(self, obj):
      if obj.patient_email:
          return obj.patient_email
      if obj.patient:
          return getattr(obj.patient, "email", None) or ""
      return ""
  ```
- `serializers.py:949-953` (`ShortNoticePatientSerializer.get_patient_email()`) — Always returns patient email (ShortNotice is always linked):
  ```python
  def get_patient_email(self, obj):
      if not obj.patient:
          return None
      return getattr(obj.patient, "email", None)
  ```

**Do they actually agree?** YES. Both are intentional: Appointment might be unlinked (agent/online booking with loose email), so it checks appointment-level email first. ShortNotice is always linked to a patient, so it only needs patient email.

**Risk:** None — design is correct.

---

## B5 — Unordered `.first()` on User Lookup (1 site)

**Canonical helper:** `TreatmentPlan/utils/sole_patient.py::sole_*` family for patients; no equivalent for User, but `order_by()` is required for determinism

**Belongs in:** global pattern — any `.first()` without `order_by()` on ambiguous querysets must be flagged

**Sites:**
- `management/commands/seed_diary.py:202` — No order_by on admin/staff user lookup:
  ```python
  created_by = User.objects.filter(
      practices=practice,
      is_active=True,
  ).exclude(user_type="dentist").first() or clinicians[0]
  ```

**Do they actually agree?** N/A — single site, but violates determinism rule.

**Risk:** Seed command assigns appointments to a random staff user (undefined order) each run. Not a production risk (seed data only), but a maintenance trap if this pattern is copied to real code.

**Reachability:** Only in seed_diary; fallback to `clinicians[0]` masks the issue on small datasets.

---

## B6 — Signing Request Lookup (2 sites, BOTH CORRECT, ORDERED)

**Canonical helper:** None needed; consent is a child of appointment/patient, lookup is unambiguous

**Belongs in:** already correct as-is

**Sites:**
- `serializers.py:154` — Appointment-linked signing request with order_by:
  ```python
  qs = (
      SigningRequest.objects.select_related("artifact")
      .exclude(status__in=["cancelled"])
      .order_by("-created_at")
  )
  req = qs.filter(appointment_id=obj.id).first()
  ```
- `serializers.py:160` — Patient-linked signing request fallback with same order:
  ```python
  if not req:
      req = qs.filter(patient_id=obj.patient_id).first()
  ```

**Do they actually agree?** YES. Both use the same queryset with `order_by("-created_at")`, so `.first()` is deterministic. Pattern is correct.

**Risk:** None.

---

## B7 — Practice Scoping in Serializer FKs (1 site, ALREADY FIXED)

**Canonical helper:** `TreatmentPath/utils/practice_mixins.py` validators

**Belongs in:** already correct; documented in test_appointment_patient_scoping.py

**Sites:**
- `serializers.py:276-302` (`PracticeScopedPatientClinicianMixin`) — Both `patient` and `clinician` FKs are validated:
  ```python
  def validate_patient(self, patient):
      if patient is None:
          return patient
      practice = self._scope_practice()
      if practice is None or patient.practice_id != practice.id:
          raise serializers.ValidationError("Patient is not in this practice.")
      if getattr(patient, "archived_at", None) is not None:
          raise serializers.ValidationError("Patient is archived.")
      return patient

  def validate_clinician(self, clinician):
      ...
      practice = self._scope_practice()
      if ... not UserPracticeRelationship.objects.filter(
          user=clinician, practice=practice
      ).exists():
          raise serializers.ValidationError("Clinician is not a member of this practice.")
      return clinician
  ```

**Do they actually agree?** YES. Both validate correctly. Audit #24 finding is resolved.

**Risk:** None — pattern is correct and enforced by test `test_appointment_patient_scoping.py`.

---

## B8 — Name Comparison for Mismatch Detection (1 site)

**Canonical helper:** `TreatmentPlan/utils/contact_keys.py::canonical_full_name_key()`

**Belongs in:** already correct

**Sites:**
- `serializers.py:62-75` (`_PatientDisplayNameMixin.get_booked_as()`) — Detects mismatch between typed name and linked patient:
  ```python
  def get_booked_as(self, obj):
      if not obj.patient_name or not obj.patient:
          return None
      typed = canonical_full_name_key(obj.patient_name, "")
      linked = canonical_full_name_key(
          getattr(obj.patient, "first_name", ""),
          getattr(obj.patient, "last_name", ""),
      )
      return obj.patient_name if typed != linked else None
  ```

**Do they actually agree?** N/A — single site, and it's correct. Uses canonical helper to compare.

**Risk:** None.

---

## Summary

| Behaviour | Sites | Agree? | Proposed Home | Priority |
|-----------|-------|--------|---------------|----------|
| Email comparison (anti-spam vs. cancel) | 2 | NO | Normalize both before comparison in public_booking_views.py or extract to `canonical_email()` wrapper | HIGH |
| Full name building (Display vs. ShortNotice) | 2 | MOSTLY (no collapse in ShortNotice) | Use `full_name()` in ShortNoticePatientSerializer.get_patient_name() | MEDIUM |
| Phone retrieval (Appointment vs. ShortNotice) | 2 | NO (normalization) | Use `canonical_phone_e164()` in AppointmentListSerializer.get_patient_phone() | HIGH |
| Patient email fallback | 2 | YES (intentional) | Already correct | — |
| Unordered `.first()` on User | 1 | N/A | Add `order_by()` to seed_diary.py line 202 | LOW (seed only) |
| Signing request lookup | 2 | YES (ordered) | Already correct | — |
| Practice scoping FKs | 1 | YES | Already correct (test verifies) | — |
| Name comparison for mismatches | 1 | YES (canonical) | Already correct | — |

---

## Key Findings

- **Email case-sensitivity trap** (B1): Anti-spam and cancel logic disagree on case sensitivity. This is reachable and could allow email reuse across different casing.
- **Phone normalization gap** (B3): Appointment list does not normalize phone numbers; production has 3,167 patients with ISO codes that need conversion. ShortNotice already does this correctly.
- **Sibling name building** (B2): ShortNotice builds names manually; Display Mixin uses canonical helper. No immediate breakage, but whitespace handling differs.

---

## No findings for:
- Matching loose bookings to patients (none found in this directory; logic likely exists elsewhere)
- `.iexact` vs exact email lookups (none found; `.lower()` comparison is inline)
- Phone normalization at input time (happens in utils, not checked here)

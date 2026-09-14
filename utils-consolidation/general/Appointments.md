# Appointments Module — Duplication Inventory

## G1 — Serializer method field getters (get_clinician_name)  (3 sites)
**Category:** serializer getter

**Belongs in:** base serializer mixin or shared utility

**Sites:**
- `Appointments/serializers.py:114` — `def get_clinician_name(self, obj): return get_user_full_name(obj.clinician)`
- `Appointments/serializers.py:248` — `def get_clinician_name(self, obj): return get_user_full_name(obj.clinician)`
- `Appointments/serializers.py:430` — `def get_clinician_name(self, obj): return get_user_full_name(obj.clinician)`

**Identical or divergent?** IDENTICAL

**Consolidation note:** All three are exact duplicates. Could move to a mixin or base serializer class.

---

## G2 — Serializer method field getters (get_practitioner_name)  (4 sites)
**Category:** serializer getter

**Belongs in:** base serializer mixin or shared utility

**Sites:**
- `Appointments/serializers.py:486` — `def get_practitioner_name(self, obj): return get_user_full_name(obj.practitioner)`
- `Appointments/serializers.py:582` — `def get_practitioner_name(self, obj): return get_user_full_name(obj.practitioner)`
- `Appointments/serializers.py:634` — `def get_practitioner_name(self, obj): return get_user_full_name(obj.practitioner)`
- `Appointments/serializers.py:767` — `def get_practitioner_name(self, obj): return get_user_full_name(obj.practitioner)`

**Identical or divergent?** IDENTICAL

**Consolidation note:** All four are exact duplicates. Same mixin candidate as G1.

---

## G3 — Appointment active status list  (3 sites)
**Category:** state constants

**Belongs in:** `Appointments/models.py` as a class constant or `Appointments/utils/constants.py`

**Sites:**
- `Appointments/serializers.py:383-384` — `[Appointment.Status.PENDING, Appointment.Status.CONFIRMED]` (in filter)
- `Appointments/public_booking_serializers.py:220-222` — `[Appointment.Status.PENDING, Appointment.Status.CONFIRMED]` (in status__in filter)
- `Appointments/public_booking_views.py:239-240` — `[Appointment.Status.PENDING, Appointment.Status.CONFIRMED]` (in status__in filter)

**Identical or divergent?** IDENTICAL (same list, different formatting across 2-3 lines vs inline)

**Consolidation note:** Could be defined as `ACTIVE_STATUSES = [Status.PENDING, Status.CONFIRMED]` on the model, imported globally.

---

## G4 — Timedelta policy calculations for slot generation  (2 sites)
**Category:** time maths

**Belongs in:** `Appointments/utils/slot_generation.py` or base class

**Sites:**
- `Appointments/public_booking_serializers.py:179-180` — `min_notice_delta = timedelta(minutes=policy.min_notice_minutes)` + `max_advance_delta = timedelta(days=policy.max_advance_days)`
- `Appointments/public_booking_views.py:139-140` — `min_notice_delta = timedelta(minutes=policy.min_notice_minutes)` + `max_advance_delta = timedelta(days=policy.max_advance_days)`

**Identical or divergent?** IDENTICAL (same variable names, same policy fields)

**Consolidation note:** This calculation appears in two slot-generation methods (one in serializer, one in viewset). Extract to a shared utility.

---

## G5 — Buffer time calculations  (2 sites)
**Category:** time maths

**Belongs in:** `Appointments/utils/slot_generation.py`

**Sites:**
- `Appointments/public_booking_serializers.py:210-211` — `buffer_before = timedelta(minutes=config.buffer_before_minutes)` + `buffer_after = timedelta(minutes=config.buffer_after_minutes)`
- `Appointments/public_booking_views.py:145-146` — `buffer_before = timedelta(minutes=config.buffer_before_minutes)` + `buffer_after = timedelta(minutes=config.buffer_after_minutes)`

**Identical or divergent?** IDENTICAL

**Consolidation note:** Same extraction candidate as G4. Both are part of the same slot-generation logic duplicated across two classes.

---

## G6 — Break time parsing (time format conversion)  (2 sites)
**Category:** time maths / parsing

**Belongs in:** `Appointments/utils/slot_generation.py` as a helper function

**Sites:**
- `Appointments/public_booking_serializers.py:261-262` — `datetime.strptime(br["start"], "%H:%M").time()` + `datetime.strptime(br["end"], "%H:%M").time()` (inline in validation loop)
- `Appointments/public_booking_views.py:267-268` — `datetime.strptime(br["start"], "%H:%M").time()` + `datetime.strptime(br["end"], "%H:%M").time()` (inline in _is_during_break method)

**Identical or divergent?** IDENTICAL (same format string, same transformation)

**Consolidation note:** Extract to `parse_time_str(s)` helper or move logic to a shared break validator.

---

## G7 — Appointment conflict detection with buffer times  (2 sites)
**Category:** time maths / state handling

**Belongs in:** `Appointments/utils/conflict_detection.py` or base validation mixin

**Sites:**
- `Appointments/public_booking_serializers.py:216-224` — `Appointment.objects.filter(clinician=clinician, start_time__lt=buffered_end, end_time__gt=buffered_start, status__in=[Status.PENDING, Status.CONFIRMED]).exists()`
- `Appointments/public_booking_views.py:234-242` — `Appointment.objects.filter(clinician=clinician, start_time__lt=buffered_end, end_time__gt=buffered_start, status__in=[Status.PENDING, Status.CONFIRMED]).exists()`

**Identical or divergent?** IDENTICAL (same query logic, same status filtering)

**Consolidation note:** Extract to a shared utility method `check_clinician_conflicts(clinician, buffered_start, buffered_end)`.

---

## G8 — Practice retrieval and null-check gate  (6 sites)
**Category:** practice gate

**Belongs in:** authentication middleware or base viewset mixin

**Sites:**
- `Appointments/views.py:411-413` — `practice = getattr(request.user, "current_practice", None)` + `if not practice: return Response({"error": "No practice associated with user"}, status=HTTP_400_BAD_REQUEST)`
- `Appointments/views.py:464-465` — `practice = getattr(request.user, "current_practice", None)` + `if not practice:` + `return Response(...)`
- `Appointments/views.py:523-524` — `practice = getattr(self.request.user, "current_practice", None)` + `if not practice:` + `return Response(...)`
- `Appointments/views.py:550` — `practice = getattr(self.request.user, "current_practice", None)` (without immediate check but pattern is consistent)
- `Appointments/views.py:558` — `practice = getattr(request.user, "current_practice", None)` (similar context)
- `Appointments/views.py:1076` — `practice = getattr(self.request.user, "current_practice", None)` (same pattern)

**Identical or divergent?** IDENTICAL (all use same attribute access, all followed by None check and identical error response)

**Consolidation note:** Extract to mixin method `get_practice_or_error()` that returns practice or Response. Would eliminate 6 repetitions.

---

## G9 — Response exception handling (47 total sites)
**Category:** exception handling

**Belongs in:** shared exception handler or response formatter

**Distribution:**
- `views.py`: 30 Response returns
  - "error" key: 12+ occurrences
  - "skipped" key: 3 occurrences
  - "weeks" key: 1 occurrence
  - "payments" key: 1 occurrence
  - "dates" key: 1 occurrence
  - Data responses (no error key): ~12 occurrences
- `public_booking_views.py`: 17 Response returns
  - "error" key: 14+ occurrences
  - Data responses: 3 occurrences

**Sites:** Too numerous to list all; sampling:
- `Appointments/views.py:322-324` — `return Response({"error": "Cannot cancel an appointment that is {appointment.status}"}, status=HTTP_400_BAD_REQUEST)`
- `Appointments/views.py:659-661` — `return Response({"skipped": True, "reason": "Practitioner not found"}, status=HTTP_200_OK)`
- `Appointments/public_booking_views.py:413-414` — `return Response({"error": "Appointment not found"}, status=HTTP_404_NOT_FOUND)`
- `Appointments/public_booking_views.py:461-466` — `return Response({"error": "Too many verification attempts. ..."}, status=HTTP_429_TOO_MANY_REQUESTS)`

**Identical or divergent?** DIVERGENT — error keys are consistent, but response bodies vary. Status codes show some divergence (400 vs 404 vs 429).

**Consolidation note:** No single fix; this is normal variation. Worth audit for consistency of status codes per error type (e.g., all "not found" should be 404, not 400). Exception handling itself is healthy (no broad `except Exception` returning 500 silently without logging).

---

## G10 — ViewSet get_queryset boilerplate  (6 sites)
**Category:** viewset boilerplate

**Belongs in:** base viewset mixin

**Sites:**
- `Appointments/views.py:166` — `def get_queryset(self): practice = ...; return self.queryset.filter(practice=practice)`
- `Appointments/views.py:522` — similar pattern
- `Appointments/views.py:549` — similar pattern
- `Appointments/views.py:608` — similar pattern
- `Appointments/views.py:780` — similar pattern
- `Appointments/views.py:888` — similar pattern

**Identical or divergent?** DIVERGENT — some filter on practice alone, others add status filters or exclusions. Pattern is consistent but payloads differ per viewset.

**Consolidation note:** Not a consolidation target; filtering is intentionally different per viewset. The practice-gate pattern is the shared part (see G8).

---

## Summary

| Pattern | Sites | Category | Effort | Impact |
|---------|-------|----------|--------|--------|
| get_clinician_name duplication | 3 | serializer getter | Low | Clarifies 6 lines |
| get_practitioner_name duplication | 4 | serializer getter | Low | Clarifies 8 lines |
| Appointment active status list | 3 | constant | Low | DRY state definition |
| Timedelta policy calculations | 2 | time maths | Medium | Shared slot-gen logic |
| Buffer time calculations | 2 | time maths | Medium | Shared slot-gen logic |
| Break time parsing | 2 | parsing | Medium | Shared utility function |
| Conflict detection query | 2 | time maths/state | Medium | Shared validator |
| Practice null-check gate | 6 | practice gate | Medium | Shared mixin method |
| Response exception handling | 47 | exception handling | Low | No action needed; divergence is intentional |
| ViewSet get_queryset | 6 | viewset boilerplate | None | Divergence is intentional |

**Total unique patterns:** 10  
**Total duplicated definitions:** ~30 code blocks  
**High-confidence consolidation targets:** G1, G2, G3, G4, G5, G6, G7, G8  
**Known divergence (acceptable):** G9, G10

---

## Detailed Consolidation Path

### Phase 1: Shared constants (lowest effort, immediate win)
- Extract `ACTIVE_STATUSES = [Status.PENDING, Status.CONFIRMED]` to models.py:Appointment
- Create `Appointments/constants.py` if needed

### Phase 2: Time utilities (medium effort, high reuse)
- Create `Appointments/utils/slot_generation.py`
  - `build_policy_timedeltas(policy)` → returns `(min_notice_delta, max_advance_delta)`
  - `build_buffer_timedeltas(config)` → returns `(buffer_before, buffer_after)`
  - `parse_time_str(s: str, fmt="%H:%M") → time`
  - `check_clinician_conflicts(clinician, buffered_start, buffered_end, practice) → bool`

### Phase 3: Serializer mixins (low effort, high clarity)
- Create `Appointments/serializer_mixins.py`
  - `ClinicianNameMixin` with `get_clinician_name(self, obj)`
  - `PractitionerNameMixin` with `get_practitioner_name(self, obj)`

### Phase 4: ViewSet mixins (medium effort, architectural)
- Create `Appointments/viewset_mixins.py`
  - `PracticeGateMixin` with `get_practice_or_error()` method
  - Refactor 6 sites in views.py to use mixin

---

## Notes

- No unguarded f-string name joins on nullable columns found in this scan.
- Exception handling follows good patterns (no broad catches silently logging 500s).
- ScopedRateThrottle usage (PublicBookingRateThrottle) is correctly scoped and only used in public endpoints.
- Anti-spam email check is case-insensitive (good); no divergence from cancel-check pattern noted.

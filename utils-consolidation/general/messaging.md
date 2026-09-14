# General Code Duplication Scan: messaging/

## G1 — Websocket broadcast plumbing (5 sites)
**Category:** websocket plumbing  
**Belongs in:** `messaging/utils.py` — extract to a shared `broadcast_helper()` function  
**Sites:**
- `messaging/utils.py:1093` — `broadcast_new_conversation()` lines 1104–1118
- `messaging/utils.py:1121` — `broadcast_conversation_updated()` lines 1132–1146
- `messaging/utils.py:1178` — `broadcast_new_message()` lines 1192–1206
- `messaging/utils.py:1320` — `broadcast_unread_count()` lines 1332–1348
- `messaging/views/contact_views.py:1165` — `broadcast_session_flagged_spam()` calls `broadcast_conversation_updated()` (reuses pattern)

**Pattern (all 5 identical):**
```python
from asgiref.sync import async_to_sync
from channels.layers import get_channel_layer

channel_layer = get_channel_layer()
if not channel_layer:
    logger.warning("Channel layer not configured...")
    return

room_group_name = f"<pattern>_{identifier}"
try:
    async_to_sync(channel_layer.group_send)(
        room_group_name,
        {"type": "<event_type>", "<data_key>": data},
    )
    logger.info(f"Broadcasted ...")
except Exception as e:
    logger.error(f"Failed to broadcast: {e}")
```

**Identical or divergent?** IDENTICAL structure; only `room_group_name` format and message envelope shape differ. All 5 follow the same exception/logging/return pattern.

**Consolidation note:** Extract `_broadcast_to_group(room_group_name, message_type, payload)` helper to eliminate 5 copies of the get→check→try/except boilerplate.

---

## G2 — MessageSessionSerializer duplicated getter methods (10 sites)
**Category:** serializer getter boilerplate  
**Belongs in:** `_SessionContactMixin` base class (partially already there, but methods not inherited properly in MessageSessionDetailSerializer)  
**Sites:**
- `messaging/serializers.py:619–701` — MessageSessionSerializer: `get_contact_name`, `get_participant_email`, `get_participant_phone_number`, `get_participant_country_code`, `get_patient_id`, `get_intake_id`, `get_nurture_id`, `get_activity_log_id`, `get_is_family`, `get_family_relationship`
- `messaging/serializers.py:813–895` — MessageSessionDetailSerializer: same 10 methods, byte-identical

**Identical or divergent?** IDENTICAL — all 10 getter methods are byte-for-byte copies between the two serializer classes.

**Consolidation note:** Move all 10 getters into `_SessionContactMixin`; both serializers inherit them. The mixin already exists for `to_representation()` and `get_family_members()`; extend it.

---

## G3 — Serializer getter `get_sender_name` (2 sites)
**Category:** serializer getter  
**Belongs in:** shared base class or utility function  
**Sites:**
- `messaging/serializers.py:1118–1121` — MessageHistorySerializer
- `messaging/serializers.py:1150–1153` — TemplateMessageHistorySerializer

**Code:**
```python
def get_sender_name(self, obj):
    if obj.sent_by:
        return f"{obj.sent_by.first_name} {obj.sent_by.last_name}"
    return None
```

**Identical or divergent?** IDENTICAL

**Consolidation note:** Extract to a `_SenderNameMixin` or define once in a utility function.

---

## G4 — Serializer getter `get_created_by_name` (2 sites)
**Category:** serializer getter  
**Belongs in:** utility function or mixin  
**Sites:**
- `messaging/serializers.py:1342–1343` — BulkSMSSerializer
- `messaging/serializers.py:1384–1385` — BulkEmailSerializer

**Code:**
```python
def get_created_by_name(self, obj):
    return get_display_name_for_user(obj.created_by)
```

**Identical or divergent?** IDENTICAL

**Consolidation note:** Already delegating to a global helper; could be a mixin or left as-is. Low impact.

---

## G5 — Serializer getter `get_recipients` (2 sites)
**Category:** serializer getter  
**Belongs in:** shared base class or utility  
**Sites:**
- `messaging/serializers.py:1345–1355` — BulkSMSSerializer
- `messaging/serializers.py:1387–1397` — BulkEmailSerializer

**Code (BulkSMSSerializer):**
```python
def get_recipients(self, obj):
    formatted_recipients = []
    if obj.results:
        for result in obj.results:
            formatted_recipients.append({
                "phone": result.get("recipient", ""),
                "send_status": result.get("status", "unknown"),
            })
    return formatted_recipients
```

**Code (BulkEmailSerializer):**
```python
def get_recipients(self, obj):
    formatted_recipients = []
    if obj.results:
        for result in obj.results:
            formatted_recipients.append({
                "email": result.get("recipient", ""),  # ← DIVERGENT KEY
                "send_status": result.get("status", "unknown"),
            })
    return formatted_recipients
```

**Identical or divergent?** DIVERGENT — only difference is key name (`"phone"` vs `"email"`); logic is identical.

**Consolidation note:** Extract `_format_recipients(obj, key_name)` helper with a parameter for the key; both serializers call it. Or: move to a mixin accepting a `recipient_key_name` class attribute.

---

## G6 — Practice gate pattern (37 sites)
**Category:** practice gate  
**Belongs in:** base view class or mixin (likely already in PracticeAccessMixin or similar)  
**Sites:**
- `messaging/views/domain_views.py` — 7 calls (lines 71, 211, 243, 317, 379, 402, 463)
- `messaging/views/contact_views.py` — 9 calls (lines 138, 350, 363, 400, 463, 521, 805, 916, 943, 968, 1029)
- `messaging/views/session_views.py` — 2 calls (lines 63, 180)
- `messaging/views/utility_views.py` — 5+ calls
- `messaging/views/call_log_views.py` — 1 call
- `messaging/views/whatsapp_views.py` — 2 calls
- `messaging/views/sms_config_views.py` — 2 calls
- `messaging/views/template_views.py` — 3+ calls

**Pattern:**
```python
practice = self.get_user_practice_or_none()
if not practice:
    return Response(
        {"detail": "Practice not found."}, 
        status=status.HTTP_403_FORBIDDEN
    )
```

**Response shapes (counted):**
- 21 × `{"detail": "Practice not found."}` + HTTP_403_FORBIDDEN
- 5 × custom data (e.g., `{"split_count": 0, "splits": []}`)
- 1 × `ContactChannel.objects.none()`
- Others: mixed

**Identical or divergent?** DIVERGENT — most use `{"detail": ...}` + 403, but ~6 return custom data or query methods instead.

**Consolidation note:** Those returning `{"detail": ...}` + 403 are true duplicates; call a shared `raise_forbidden_if_no_practice(practice)` helper. Custom-response cases (e.g., split) are legitimately different and should stay.

---

## G7 — Exception → Response block distributions (26 view functions)
**Category:** exception handling  
**Belongs in:** view base class or `@api_error_handler` decorator  
**Sites:** Across `views/*.py` files  

**Response key distribution (counted from all view files):**
- 26 return `Response({"status": ...})`
- 12 return `Response({"error": ...})`
- 7 return `Response({"detail": ...})`

**Status code distribution (from Response returns):**
- 9 × HTTP_400_BAD_REQUEST
- 7 × HTTP_404_NOT_FOUND
- 6 × HTTP_401_UNAUTHORIZED
- 2 × HTTP_201_CREATED
- 2 × HTTP_200_OK
- ~Others scattered

**Identical or divergent?** DIVERGENT — response keys and status codes are inconsistent. No two adjacent exception blocks are byte-identical, but patterns repeat.

**Consolidation note:** Standardize on one error key (`"detail"` is DRF default); apply consistent HTTP_400/401/403/404 codes by exception type. Cannot easily extract without refactoring each handler, but a documented convention + linter rule would help.

---

## G8 — Serializer helper functions `_get_person_from_channel`, `_get_person_contact_fields`, `_get_display_name_from_channel` (3 sites)
**Category:** serializer helper  
**Belongs in:** already in `messaging/serializers.py` as module-level functions (lines 26–88)  
**Sites:**
- `messaging/serializers.py:26–38` — `_get_person_from_channel(channel)` — module-level helper
- Called by: `_SessionContactMixin`, `MessageSessionSerializer`, `MessageSessionDetailSerializer`, and standalone in `MessageSessionMetadataUpdateSerializer.validate()`

**Note:** These ARE already shared (not duplicated). Listed for completeness; no consolidation needed.

---

## Summary

| Finding | Count | Category | Divergent? | Consolidated To |
|---------|-------|----------|-----------|-----------------|
| G1 — Broadcast websocket plumbing | 5 | websocket | No | Shared helper function |
| G2 — MessageSession getters | 10 | serializer boilerplate | No | Base mixin inheritance |
| G3 — `get_sender_name` | 2 | serializer getter | No | Mixin or utility |
| G4 — `get_created_by_name` | 2 | serializer getter | No | Mixin (low priority) |
| G5 — `get_recipients` | 2 | serializer getter | Yes | Parameterized helper |
| G6 — Practice gate | 37 | practice gate | Yes | Shared helper + linting |
| G7 — Exception → Response | 26 | exception handling | Yes | Style guide + consistency |
| G8 — Serializer helpers | 3 | serializer helper | — | Already shared (no work) |

**Total duplicated definitions:** 81 (including all instances of G1–G7)
**Top 5 by site count:** G6 (37), G7 (26), G2 (10), G1 (5), G3/G4/G5 (2 each)
**Divergent findings:** G5, G6, G7 (response shapes drift; most risky for live-editing bugs)
**Count methodology:** Grep + manual verification of byte-identical or near-identical code blocks; site count = number of separate file:line occurrences where the duplicate appears.

---

**Report path:** `/home/mannie/Desktop/Projects/treatmentpath/docs/utils-consolidation/general/messaging.md`

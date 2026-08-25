# Marketing Universal Search + Unsubscribed-Flag Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every marketing list can be searched by any parameter (name, email, phone) the way Journeys already can, and the campaign preview's "Unsubscribed" column stops disagreeing with its own sort order.

**Architecture:** Reuse `TreatmentPlan.utils.build_tokenized_search_query` — the same tokenizer Journeys uses — rather than hand-rolling `Q()` chains per view. One new module `marketingBroadcast/search.py` holds two helpers: a person-backed filter (name + any contact channel, used by the three per-person lists) and a thin passthrough for flat text lists (campaigns/segments/templates). Cross-app import from `TreatmentPlan` is already established precedent in this app (`consent_ledger.py:139`, `segment_engine.py:262`, `webhook_views.py:16`).

**Tech Stack:** Django 5 / DRF, React + TypeScript + Tailwind, vitest, pytest (Django `manage.py test`).

## Global Constraints

- Search param is **`search`** everywhere, matching Journeys (`self.request.query_params.get("search")`).
- Empty/whitespace-only `search` is a no-op — never filter to zero rows on a blank string.
- Person-backed searches MUST `.distinct()` — joining `person_channels` fans out one row per channel.
- Do NOT change existing default orderings except where Task 5 explicitly says so.
- Run backend tests with `--keepdb`; NEVER `--noinput` (destroys the persistent test DB).
- Activate the venv first: `source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate`

---

## File Structure

| File | Responsibility |
|---|---|
| `marketingBroadcast/search.py` (create) | The two shared search-filter builders. Single source of truth. |
| `marketingBroadcast/tests/test_search.py` (create) | Unit tests for the helpers + endpoint-level search tests. |
| `marketingBroadcast/views/campaign_views.py` (modify) | Campaign list search; preview search upgrade; unsubscribed-flag fix. |
| `marketingBroadcast/views/segment_views.py` (modify) | Segment list search; both segment previews. |
| `marketingBroadcast/views/template_views.py` (modify) | Global + practice template list search. |
| `marketingBroadcast/views/archive_views.py` (modify) | Archived items search. |
| Frontend marketing list components (modify) | Search inputs wired to `?search=`. |

---

### Task 1: The shared search helpers

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/search.py`
- Test: `TreatmentPathBackend/TreatmentPath/marketingBroadcast/tests/test_search.py`

**Interfaces:**
- Consumes: `TreatmentPlan.utils.build_tokenized_search_query(search_query, field_lookups, partial=True) -> Q`
- Produces:
  - `person_search_filter(search: str, prefix: str = "person") -> Q | None`
  - `text_search_filter(search: str, fields: list[str]) -> Q | None`
  Both return `None` for blank input so callers can skip filtering entirely.

- [ ] **Step 1: Write the failing tests**

```python
from django.test import SimpleTestCase
from marketingBroadcast.search import person_search_filter, text_search_filter


class SearchHelperTests(SimpleTestCase):
    def test_blank_search_returns_none(self):
        for blank in ["", "   ", None]:
            self.assertIsNone(person_search_filter(blank))
            self.assertIsNone(text_search_filter(blank, ["name"]))

    def test_person_filter_covers_name_and_channel_value(self):
        q = person_search_filter("smith")
        rendered = str(q)
        self.assertIn("person__first_name__icontains", rendered)
        self.assertIn("person__last_name__icontains", rendered)
        self.assertIn("person__person_channels__channel__canonical_value__icontains", rendered)

    def test_person_filter_honours_prefix(self):
        q = person_search_filter("smith", prefix="")
        rendered = str(q)
        self.assertIn("first_name__icontains", rendered)
        self.assertNotIn("person__first_name", rendered)

    def test_phone_search_strips_spaces_and_dashes(self):
        # build_tokenized_search_query ORs in a space/dash-stripped variant;
        # that is what makes "07700 900-123" match a stored E.164 value.
        rendered = str(person_search_filter("07700 900-123"))
        self.assertIn("07700900123", rendered)

    def test_text_filter_uses_given_fields_only(self):
        rendered = str(text_search_filter("promo", ["name", "subject"]))
        self.assertIn("name__icontains", rendered)
        self.assertIn("subject__icontains", rendered)
        self.assertNotIn("person__", rendered)
```

- [ ] **Step 2: Run to verify they fail**

Run: `python manage.py test marketingBroadcast.tests.test_search --keepdb -v 2`
Expected: FAIL — `ModuleNotFoundError: No module named 'marketingBroadcast.search'`

- [ ] **Step 3: Implement**

```python
"""Shared search-filter builders for the marketing app.

Every marketing list uses the SAME tokenizer Journeys uses
(``TreatmentPlan.utils.build_tokenized_search_query``) so that a phone number
typed with spaces or dashes, or a two-token name like "jane smith", behaves
identically wherever a user types it. Do not hand-roll Q() chains in a view —
add fields here instead.
"""

from TreatmentPlan.utils import build_tokenized_search_query

# Name lives on Person; email AND phone both live on ContactChannel.canonical_value
# reached through the PersonChannel through-table. canonical_value is deliberately
# not filtered by channel kind, so one box searches both.
_PERSON_FIELDS = [
    "first_name",
    "last_name",
    "person_channels__channel__canonical_value",
]


def person_search_filter(search, prefix="person"):
    """Q matching a person by name, email or phone. None when `search` is blank.

    `prefix` is the lookup path from the queryset's model to Person
    ("person" for MarketingPatientProfile, "" when querying Person directly).
    Callers MUST apply .distinct() — joining person_channels fans rows out.
    """
    cleaned = (search or "").strip()
    if not cleaned:
        return None
    head = f"{prefix}__" if prefix else ""
    return build_tokenized_search_query(
        cleaned, [f"{head}{field}" for field in _PERSON_FIELDS]
    )


def text_search_filter(search, fields):
    """Q matching flat text columns (campaign/segment/template names). None when blank."""
    cleaned = (search or "").strip()
    if not cleaned or not fields:
        return None
    return build_tokenized_search_query(cleaned, list(fields))
```

- [ ] **Step 4: Run to verify they pass**

Run: `python manage.py test marketingBroadcast.tests.test_search --keepdb -v 2`
Expected: PASS (5 tests)

- [ ] **Step 5: Commit**

```bash
git add marketingBroadcast/search.py marketingBroadcast/tests/test_search.py
git commit -m "feat(marketing): add shared tokenized search filters"
```

---

### Task 2: Campaign preview — upgrade search + fix the unsubscribed flag

**Files:**
- Modify: `marketingBroadcast/views/campaign_views.py:95-120` (`_serialize_preview_patient`), `:318-326` (search block)
- Test: `marketingBroadcast/tests/test_search.py`

**Interfaces:**
- Consumes: `person_search_filter` from Task 1.
- Produces: preview rows whose `is_unsubscribed` is true iff `MarketingConsent.state == "off"`.

**Why the flag is wrong today.** The row flag is derived from reason *strings*:
```python
"is_unsubscribed": "unsubscribed" in reasons
or any("unsubscribed" in (reason or "").lower() for reason in reasons),
```
but `segment_engine.resolve_cannot_be_sent` emits `block_reason` instead of `"unsubscribed"` when one is set (`hard_bounce`, `spam_complaint`, `staff_block`, …). Meanwhile the ORDERING uses `Exists(MarketingConsent state="off")`, which ignores `block_reason`. Net effect: a bounce-suppressed patient sorts to the top of the preview while its Unsubscribed column reads false. Fix = both read the same source.

- [ ] **Step 1: Write the failing test**

```python
def test_bounce_blocked_person_is_flagged_unsubscribed(self):
    # consent off with a block_reason: previously sorted first but displayed false
    MarketingConsent.objects.create(
        practice=self.practice, person=self.person, state="off",
        block_reason="hard_bounce",
    )
    row = _serialize_preview_patient(
        self.profile_with_flag(True), cannot_be_sent_reasons={self.person.id: ["hard_bounce"]}
    )
    self.assertTrue(row["is_unsubscribed"])
```

- [ ] **Step 2: Run to verify it fails**

Run: `python manage.py test marketingBroadcast.tests.test_search --keepdb -v 2`
Expected: FAIL — `False is not true`

- [ ] **Step 3: Implement — prefer the annotation, keep the reason fallback**

```python
    # Prefer the queryset annotation (single source of truth, same expression the
    # ordering uses). Fall back to reason-sniffing for callers that pass an
    # unannotated profile.
    annotated_unsubscribed = getattr(profile, "is_unsubscribed_flag", None)
    if annotated_unsubscribed is None:
        annotated_unsubscribed = "unsubscribed" in reasons or any(
            "unsubscribed" in (reason or "").lower() for reason in reasons
        )
```
and in the returned dict: `"is_unsubscribed": bool(annotated_unsubscribed),`

- [ ] **Step 4: Replace the hand-rolled preview search**

```python
        person_filter = person_search_filter(request.query_params.get("search"))
        if person_filter is not None:
            qs = qs.filter(person_filter).distinct()
```
(delete the old `if search:` block and the now-unused lowercased `search` local)

- [ ] **Step 5: Run tests**

Run: `python manage.py test marketingBroadcast --keepdb -v 2`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git commit -am "fix(marketing): unsubscribed flag matches sort source; tokenized preview search"
```

---

### Task 3: Campaign + segment list search

**Files:**
- Modify: `marketingBroadcast/views/campaign_views.py:129-146` (`get_queryset`)
- Modify: `marketingBroadcast/views/segment_views.py:94-102` (`get_queryset`), `:75-86` (`_paginated_preview`), `:60-73` (`_preview_count_and_patients`)

**Interfaces:** Consumes `text_search_filter`, `person_search_filter` from Task 1.

- [ ] **Step 1: Write failing endpoint tests** — assert `GET /marketing/campaigns/?search=<name fragment>` returns only the matching campaign, and `?search=` (blank) returns all.
- [ ] **Step 2: Run to verify they fail** (blank-vs-filter assertions fail because search is ignored today)
- [ ] **Step 3: Implement in campaign `get_queryset`**, inside the `if self.action == "list":` block:

```python
            search_filter = text_search_filter(
                self.request.query_params.get("search"), ["name", "subject"]
            )
            if search_filter is not None:
                queryset = queryset.filter(search_filter)
```

- [ ] **Step 4: Implement in segment `get_queryset`** with fields `["name", "description"]`.
- [ ] **Step 5: Implement in both segment previews** using `person_search_filter(...)` + `.distinct()`, reading `search` from `request.query_params` (saved preview) and from the POST body or query params (draft preview — match whichever the existing page/page_size uses).
- [ ] **Step 6: Run tests, commit.**

---

### Task 4: Template + archived list search

**Files:**
- Modify: `marketingBroadcast/views/template_views.py:48-49` (global), `:136-140` (practice)
- Modify: `marketingBroadcast/views/archive_views.py:17-56`

- [ ] **Step 1: Write failing tests** for each of the three lists.
- [ ] **Step 2: Run to verify they fail.**
- [ ] **Step 3: Implement.** Templates: `text_search_filter(search, ["name", "subject"])` — confirm `subject` exists on both template models first; drop it from the list if not. Archived: apply `text_search_filter(search, ["name"])` to the segment queryset AND the campaign queryset **before** they are merged and Python-sorted at `archive_views.py:53`.
- [ ] **Step 4: Run tests, commit.**

---

### Task 5: Frontend search inputs

**Files:**
- Modify: the marketing list components (campaigns, segments, templates, archived) and `src/config/api.ts` endpoint builders to forward `search`.
- Modify: `src/pages/MarketingCampaignPreview.tsx` — placeholder text only.
- Test: extend the nearest existing vitest suite per component.

- [ ] **Step 1:** Locate each list component and its API builder; confirm whether a debounced search input pattern already exists in the marketing pages (reuse it; do not invent a second one).
- [ ] **Step 2:** Write a failing test per component asserting the fetch URL contains `search=<typed value>` after debounce.
- [ ] **Step 3:** Run to verify failure.
- [ ] **Step 4:** Implement, forwarding `search` only when non-empty (match the existing `send_status`/`reason` convention at `api.ts:1526-1531`).
- [ ] **Step 5:** Update the campaign preview placeholder from "Search by name…" to "Search by name, email or phone…" — it already searches all three; the copy under-sells it.
- [ ] **Step 6:** Run `npx vitest run <suites>`, `npx tsc --noEmit`, `npx eslint <files>`, commit.

---

### Task 6: Verify end-to-end

- [ ] **Step 1:** Run the full marketing backend suite: `python manage.py test marketingBroadcast --keepdb -v 2`
- [ ] **Step 2:** Run the frontend marketing suites.
- [ ] **Step 3:** Drive the real app: search a marketing list by an email fragment and by a phone number with spaces; confirm non-zero results and that a blank search restores the full list.
- [ ] **Step 4:** Confirm in the campaign preview that a consent-`off`-with-`block_reason` patient now shows Unsubscribed = true AND sorts first.

---

## Self-Review

**Spec coverage.** "Unsubscribers seen first after send" → already implemented at `campaign_views.py:330-338`; the real defect is the flag/sort disagreement, covered by Task 2. "Search by email, name, phone like Journeys" → Tasks 1–4 (backend, all 8 lists) + Task 5 (frontend). "All marketing searches" → the enumerated list is campaigns, segments, global templates, practice templates, archived, segment preview (saved), segment preview (draft), campaign preview. No marketing list is left without search.

**Placeholders.** None — every code step carries real code. Task 4's `subject` field and Task 5's component paths are flagged as verify-then-implement rather than assumed, because the recon did not confirm them.

**Type consistency.** `person_search_filter` / `text_search_filter` names and their `Q | None` return contract are used identically in Tasks 2, 3 and 4. The `None`-means-blank convention is stated once in Global Constraints and relied on by every call site.

## Known risk

`person_search_filter` joins `person_channels`, so every person-backed search needs `.distinct()`. Forgetting it silently duplicates rows and inflates the `count` shown in pagination. Task 1's docstring says so; the endpoint tests in Task 3 should assert the count is not inflated for a person with both a phone and an email channel.

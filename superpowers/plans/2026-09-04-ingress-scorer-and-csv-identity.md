# Ingress Scorer Keying + CSV Identity Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the ingress-test-engine's merge verdicts falsifiable by keying each scenario ref to the actual DB row its delivery created, and stop `import_csv` discarding a row's contact details when it dedups onto an existing patient.

**Architecture:** Today `scorer.ref_to_person` maps a ref to a Person by looking its *email* up in the snapshot. When two refs share an email — which is the case in 7 of 14 scenarios, all of them merge assertions — both refs collapse onto one row and `classify` always emits `correct_merge`, even when the product created two distinct Persons. The fix threads a per-ref row identity `(table, id)` from each driver's own response through the runner into the scorer, and falls back to a *capacity-limited* email match (K refs sharing an email can claim at most the K rows that actually exist) instead of the current unlimited one. The backend half fixes the behaviour this exposed: when a CSV row matches an existing patient by phone, `import_csv` currently `continue`s and drops the row's email entirely, so the same human's new address is never attached to their Person and a later ingest under it mints a duplicate.

**Tech Stack:** Python 3.12, pytest, psycopg2, Django 4.x, DRF.

**Spec:** `docs/superpowers/specs/2026-09-03-ingress-test-engine-design.md`

## Global Constraints

- The engine's DB access stays **SELECT-only**; `dev_dsn` points at a read-only role. No new write path in `engine/`.
- Drivers stay **no-retry**; the runner owns retry policy. Do not add retries.
- `engine/db.py` SQL must keep quoting table names exactly as Django creates them (`"TreatmentPlan_intake"`, `"UserAuthentication_practice"`).
- Backend identity writes go through `ContactChannel.get_or_create_channel` — the declared single chokepoint for a channel write. Never construct a `ContactChannel` directly and never invent a canonical key.
- Practice-scoped throughout: never link a channel or Person across practices.
- A channel already linked to a **different** Person is a conflict to report, never to auto-weld. Welding two Persons is out of scope for an import.
- Backend tests run with `--keepdb`, never `--noinput`.
- No commits — the user handles all VCS operations. Ignore the `git commit` steps in the standard task template.

---

### Task 1: Snapshot carries row ids and can fetch rows by id

**Files:**
- Modify: `ingress-test-engine/engine/db.py:16-76`
- Test: `ingress-test-engine/tests/test_db.py`

**Interfaces:**
- Produces: `snapshot(dsn, practice_slug, email_prefix, row_ids=None) -> dict`. `row_ids` is `{table_name: [int, ...]}` for the three record tables. Each entry in `out["records"]` gains an `"id"` key (int). Rows are returned if their email matches the prefix **or** their id is in `row_ids[table]`.

- [ ] **Step 1: Write the failing test**

`snapshot` needs a live DB, so test the SQL-building seam instead. Extract the WHERE construction into a pure helper and test that.

```python
from engine.db import _records_query


def test_records_query_without_ids_filters_on_email_only():
    sql, params = _records_query("intake", "qa-slug", "qa+s1-", None)
    assert '"TreatmentPlan_intake"' in sql
    assert "r.id = ANY" not in sql
    assert params == ("qa-slug", "qa+s1-%")


def test_records_query_with_ids_adds_an_or_branch():
    sql, params = _records_query("patient", "qa-slug", "qa+s1-", [7, 9])
    # The id branch must be OR'd with the email branch, not AND'd — a row
    # matched by id will NOT have the run's email marker.
    assert "OR r.id = ANY(%s)" in sql
    assert params == ("qa-slug", "qa+s1-%", [7, 9])


def test_records_query_ignores_an_empty_id_list():
    sql, params = _records_query("intake", "qa-slug", "qa+s1-", [])
    assert "r.id = ANY" not in sql
    assert params == ("qa-slug", "qa+s1-%")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ingress-test-engine && ./venv/bin/python -m pytest tests/test_db.py -v -k records_query`
Expected: FAIL with `ImportError: cannot import name '_records_query'`

- [ ] **Step 3: Write minimal implementation**

Replace the `_RECORDS_SQL` constant and its use in `snapshot`:

```python
_RECORDS_SQL = """
    SELECT r.id, r.person_id, r.email, r.phone_number
      FROM "TreatmentPlan_{table}" r
     WHERE r.practice_id = (
               SELECT p.id FROM "UserAuthentication_practice" p WHERE p.slug = %s
           )
       AND ({match})
"""


def _records_query(table, practice_slug, email_prefix, ids):
    """Build (sql, params) for one record table. Rows are selected by the
    run's email marker OR by explicit id — the id branch exists because a
    lane that DEDUPS onto a pre-existing row lands on a row whose email is
    the OLD address and so can never match the marker."""
    params = [practice_slug, email_prefix + "%"]
    match = "r.email LIKE %s"
    if ids:
        match += " OR r.id = ANY(%s)"
        params.append(list(ids))
    return _RECORDS_SQL.format(table=table, match=match), tuple(params)
```

And in `snapshot`:

```python
def snapshot(dsn, practice_slug, email_prefix, row_ids=None):
    row_ids = row_ids or {}
    out: dict = {"records": [], "persons": {}, "channels": []}
    with psycopg2.connect(dsn) as conn, conn.cursor() as cur:
        for table in _RECORD_TABLES:
            sql, params = _records_query(table, practice_slug, email_prefix,
                                         row_ids.get(table))
            cur.execute(sql, params)
            for row_id, person_id, email, phone in cur.fetchall():
                out["records"].append({
                    "table": table,
                    "id": row_id,
                    "email": email,
                    "phone": phone,
                    "person_id": person_id,
                })
        ...
```

Update the module docstring's schema note to mention `id`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd ingress-test-engine && ./venv/bin/python -m pytest tests/test_db.py -v`
Expected: PASS (all pre-existing db tests still green)

---

### Task 2: Drivers report the row each delivery created

**Files:**
- Modify: `ingress-test-engine/engine/drivers/base.py:18-35`
- Modify: `ingress-test-engine/engine/drivers/api.py:22-52,55-69,72-89,92-107`
- Modify: `ingress-test-engine/engine/drivers/webhooks.py:196-246`
- Modify: `ingress-test-engine/engine/drivers/shim.py:344-374`
- Test: `ingress-test-engine/tests/test_drivers_registry.py`

**Interfaces:**
- Consumes: nothing from Task 1.
- Produces: `BaseDriver.row_table: str | None` class attribute, and `BaseDriver._row(body) -> dict | None` returning `{"table": str, "id": int}`. `fire()` return dicts gain an optional `"row"` key of that shape. Lanes that create no row (`merge_unweld`) and lanes that raise `DriverError` produce no `"row"`.

- [ ] **Step 1: Write the failing test**

```python
from engine.drivers.base import BaseDriver


class _Fake(BaseDriver):
    name = "fake"
    row_table = "intake"


def test_row_reads_the_plain_id_key():
    assert _Fake(None)._row({"id": 41, "first_name": "X"}) == {
        "table": "intake", "id": 41}


def test_row_reads_a_table_prefixed_id_key():
    # Webhook lanes answer {"success": true, "intake_id": 4146}
    assert _Fake(None)._row({"success": True, "intake_id": 4146}) == {
        "table": "intake", "id": 4146}


def test_row_prefers_an_explicit_table_from_the_body():
    # The shim reports its own table; it fires several lanes.
    class _Shim(BaseDriver):
        name = "s"
        row_table = None
    assert _Shim(None)._row({"id": 8, "table": "patient"}) == {
        "table": "patient", "id": 8}


def test_row_is_none_when_there_is_no_usable_id():
    assert _Fake(None)._row({"id": None, "table": "patient"}) is None
    assert _Fake(None)._row({"detail": "throttled"}) is None
    assert _Fake(None)._row(None) is None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ingress-test-engine && ./venv/bin/python -m pytest tests/test_drivers_registry.py -v -k row`
Expected: FAIL with `AttributeError: '_Fake' object has no attribute '_row'`

- [ ] **Step 3: Write minimal implementation**

In `base.py`, add to `BaseDriver`:

```python
    # Which record table this lane writes. None means "the response says".
    row_table = None

    def _row(self, body):
        """The DB row this delivery created, as {"table", "id"}, or None.

        The scorer keys a ref to its Person through THIS row. Matching on the
        email instead is what made shared-email scenarios unfalsifiable: two
        records with one email collapse onto a single row and every pair then
        scores correct_merge whatever the product did.
        """
        if not isinstance(body, dict):
            return None
        table = body.get("table") or self.row_table
        if not table:
            return None
        raw = body.get("id")
        if raw is None:
            raw = body.get(f"{table}_id")
        try:
            return {"table": table, "id": int(raw)}
        except (TypeError, ValueError):
            return None
```

In `api.py`, `_Api.call` already parses `body`; add the row to its return:

```python
        out = {"status": r.status_code, "response": r.text[:500], "json": body}
        row = self._row(body)
        if row:
            out["row"] = row
        return out
```

Set `row_table` on each API lane: `AddPatient.row_table = "intake"`;
`PatientsCrud.row_table = "patient"`; `FamilyMember.row_table = "patient"`;
`MoveConvert.row_table = "patient"`; `MergeUnweld.row_table = None`.

In `webhooks.py`, `LegacyWebhook.fire` must parse the body it currently
discards (`CustomWebhook` inherits this):

```python
class LegacyWebhook(_Webhook):
    ...
    row_table = "intake"

    def fire(self, record):
        ...
        out = {"status": r.status_code, "response": r.text[:500]}
        try:
            body = r.json()
        except ValueError:
            body = None
        row = self._row(body)
        if row:
            out["row"] = row
        return out
```

Also give `_Webhook.post` the same treatment (`row_table = "intake"` on the
class) so any lane routed through it reports its row.

In `shim.py`, the shim already returns `{"id", "table"}`:

```python
        out_dict = {"status": "shim", "response": json.dumps(out), "json": out}
        row = self._row(out)
        if row:
            out_dict["row"] = row
        return out_dict
```

`RubbishPhone` delegates to another lane and returns that lane's dict
unchanged, so it inherits the row with no edit.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd ingress-test-engine && ./venv/bin/python -m pytest tests/test_drivers_registry.py tests/test_drivers_shim.py -v`
Expected: PASS

---

### Task 3: Scorer keys refs by delivery row, with a capacity-limited email fallback

**Files:**
- Modify: `ingress-test-engine/engine/scorer.py:35-51`
- Modify: `ingress-test-engine/engine/runner.py:198-218,231-236`
- Test: `ingress-test-engine/tests/test_scorer.py`

**Interfaces:**
- Consumes: `snapshot` records carrying `"id"` (Task 1); delivery dicts carrying `"row"` (Task 2).
- Produces: `ref_to_person(snapshot_records, ref_emails, ref_rows=None) -> dict` mapping ref -> `person_id | None`. `ref_rows` is `{ref: {"table", "id"}}`. Also `delivery_rows(deliveries) -> dict` and `snapshot_row_ids(ref_rows) -> dict` in `engine/runner.py`.

**The rule:** a ref resolves through its own delivery row when one is known. Otherwise it falls back to the email match — but K refs sharing one email may claim at most the K rows that actually carry that email, assigned in id order. A ref with no row left to claim resolves to `None`. That is what makes a missed merge visible: you cannot observe more distinct records than exist.

- [ ] **Step 1: Write the failing test**

```python
from engine.scorer import classify, ref_to_person


def test_two_rows_one_email_two_persons_is_not_a_merge():
    """The tautology this whole change exists to kill. Two records share an
    email; the product filed them under DIFFERENT Persons — a real duplicate.
    Keyed on email alone both refs collapse onto one row and score
    correct_merge."""
    rows = [{"table": "intake", "id": 1, "email": "a@x.com", "person_id": 111},
            {"table": "intake", "id": 2, "email": "a@x.com", "person_id": 222}]
    actual = ref_to_person(rows, {"r1": "a@x.com", "r2": "a@x.com"})
    assert actual["r1"] != actual["r2"]
    assert classify({"r1": "p", "r2": "p"}, actual)["verdicts"] == {
        ("r1", "r2"): "missed_merge"}


def test_two_rows_one_email_one_person_is_a_real_merge():
    rows = [{"table": "intake", "id": 1, "email": "a@x.com", "person_id": 111},
            {"table": "intake", "id": 2, "email": "a@x.com", "person_id": 111}]
    actual = ref_to_person(rows, {"r1": "a@x.com", "r2": "a@x.com"})
    assert classify({"r1": "p", "r2": "p"}, actual)["verdicts"] == {
        ("r1", "r2"): "correct_merge"}


def test_a_ref_with_no_row_of_its_own_resolves_to_none():
    """csv-import-lane: r1's lane never landed a row, r2's did. r1 must NOT
    borrow r2's row just because they share an email."""
    rows = [{"table": "intake", "id": 2, "email": "a@x.com", "person_id": 222}]
    actual = ref_to_person(rows, {"r1": "a@x.com", "r2": "a@x.com"})
    assert list(sorted(actual.values(), key=lambda v: v is not None)) == [None, 222]
    assert classify({"r1": "p", "r2": "p"}, actual)["verdicts"] == {
        ("r1", "r2"): "missed_merge"}


def test_delivery_row_beats_the_email_match():
    """Distinct rows, same email: the ref's own delivery row decides which
    Person is attributed to it, so the mapping is exact rather than ordered."""
    rows = [{"table": "intake", "id": 1, "email": "a@x.com", "person_id": 111},
            {"table": "patient", "id": 1, "email": "a@x.com", "person_id": 222}]
    actual = ref_to_person(
        rows, {"r1": "a@x.com", "r2": "a@x.com"},
        {"r1": {"table": "patient", "id": 1}, "r2": {"table": "intake", "id": 1}})
    assert actual == {"r1": 222, "r2": 111}


def test_a_delivery_row_missing_from_the_snapshot_falls_back_to_email():
    rows = [{"table": "intake", "id": 5, "email": "a@x.com", "person_id": 111}]
    actual = ref_to_person(rows, {"r1": "a@x.com"},
                           {"r1": {"table": "patient", "id": 99}})
    assert actual == {"r1": 111}


def test_email_match_is_case_and_whitespace_tolerant():
    rows = [{"table": "intake", "id": 1, "email": "a@x.com", "person_id": 111}]
    assert ref_to_person(rows, {"r1": "  A@X.COM "}) == {"r1": 111}


def test_a_row_claimed_by_delivery_id_is_not_reused_by_the_email_fallback():
    rows = [{"table": "intake", "id": 1, "email": "a@x.com", "person_id": 111}]
    actual = ref_to_person(rows, {"r1": "a@x.com", "r2": "a@x.com"},
                           {"r1": {"table": "intake", "id": 1}})
    assert actual == {"r1": 111, "r2": None}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd ingress-test-engine && ./venv/bin/python -m pytest tests/test_scorer.py -v`
Expected: FAIL — `test_two_rows_one_email_two_persons_is_not_a_merge` reports `correct_merge`, and `test_a_ref_with_no_row_of_its_own_resolves_to_none` reports `correct_merge`. Both are the live bug.

- [ ] **Step 3: Write minimal implementation**

Replace `ref_to_person` in `scorer.py`:

```python
def _norm(value):
    return (value or "").strip().lower()


def ref_to_person(snapshot_records, ref_emails, ref_rows=None):
    """Map ref -> person_id (or None) for the pairwise scorer.

    A ref is keyed to the DB row ITS OWN delivery created, passed in
    `ref_rows` as {ref: {"table", "id"}}. When that is unavailable the ref
    falls back to matching its email against the snapshot — but only up to
    the number of rows that actually carry that email: K refs sharing an
    address may claim at most the K rows that exist, in id order, and any
    ref left over resolves to None.

    That cap is the whole point. Keyed on email alone, two records sharing an
    address always collapse onto one row, so `classify` emitted
    `correct_merge` for them no matter what the product did — including when
    it had filed them under two different Persons, the exact duplicate the
    suite exists to catch.
    """
    ref_rows = ref_rows or {}
    by_row = {(r["table"], r["id"]): r for r in snapshot_records}

    out, claimed = {}, set()
    pending = []
    for ref, email in ref_emails.items():
        key = ref_rows.get(ref)
        row = by_row.get((key["table"], key["id"])) if key else None
        if row is None:
            pending.append(ref)
            continue
        out[ref] = row["person_id"]
        claimed.add((row["table"], row["id"]))

    by_email = {}
    for r in sorted(snapshot_records, key=lambda r: (r["table"], r["id"])):
        if (r["table"], r["id"]) in claimed:
            continue
        by_email.setdefault(_norm(r["email"]), []).append(r)

    for ref in sorted(pending):
        rows = by_email.get(_norm(ref_emails[ref]))
        out[ref] = rows.pop(0)["person_id"] if rows else None
    return out
```

In `runner.py`, add the two wiring helpers and use them:

```python
def delivery_rows(deliveries: list) -> dict:
    """{ref: {"table", "id"}} for deliveries whose driver reported a row."""
    return {d["ref"]: d["row"] for d in deliveries if d.get("row")}


def snapshot_row_ids(ref_rows: dict) -> dict:
    """{table: [id, ...]} so db.snapshot can pull rows a lane DEDUPED onto —
    those carry a pre-existing email and never match the run's marker."""
    out: dict = {}
    for row in ref_rows.values():
        out.setdefault(row["table"], []).append(row["id"])
    return out
```

and in `run_scenario`, replace the snapshot/scoring block:

```python
    ref_rows = delivery_rows(deliveries)
    row_ids = snapshot_row_ids(ref_rows)
    ref_emails = {r.ref: r.email for r in records}
    fired_ok = {d["ref"] for d in deliveries
                if isinstance(d.get("status"), int) and 200 <= d["status"] < 300}
    deadline = time.monotonic() + max(settle_seconds, 6.0) + wait_timeout
    expected = len(fired_ok)
    snap = snapshot(cfg.dev_dsn, cfg.qa_practice_slug, marker, row_ids)
    while len(_visible_refs(snap, ref_emails, ref_rows)) < expected and \
            time.monotonic() < deadline:
        time.sleep(2.0)
        snap = snapshot(cfg.dev_dsn, cfg.qa_practice_slug, marker, row_ids)
    result.snapshot = snap
    pids = ref_to_person(snap["records"], ref_emails, ref_rows)
    actual = {ref: resolve_person(snap, pid) for ref, pid in pids.items()}
    result.classification = classify(scenario.truth.persons, actual)
```

and make the settle-poll agree with the scorer instead of keeping its own rule:

```python
def _visible_refs(snap: dict, ref_emails: dict, ref_rows: dict) -> set:
    """Refs the scorer can currently attribute to a row — same resolution as
    ref_to_person, so the poll never waits for something scoring ignores (or
    stops early on something it needs)."""
    resolved = ref_to_person(snap.get("records", []), ref_emails, ref_rows)
    return {ref for ref, pid in resolved.items() if pid is not None}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd ingress-test-engine && ./venv/bin/python -m pytest tests/ -v`
Expected: PASS for the whole engine suite. `tests/test_runner.py` calls `_visible_refs`; update its call sites to the new signature.

- [ ] **Step 5: Re-run the live suite and record which scenarios are genuinely green**

Run: `cd ingress-test-engine && ./venv/bin/ingress-engine run`
Expected: the 7 previously-unfalsifiable scenarios now report a verdict earned from per-ref rows. Any that turn red are real product findings — record them, do not adjust the scorer to re-green them.

---

### Task 4: `import_csv` keeps a deduped row's contact details

**Files:**
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/contact/channel_resolution.py` (append)
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/views/patient_views.py:1283-1291,1438-1462`
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/management/commands/qa_ingress_shim.py:53-75`
- Test: `TreatmentPathBackend/TreatmentPath/TreatmentPlan/tests/test_csv_import_identity.py` (create)

**Interfaces:**
- Consumes: nothing from Tasks 1-3.
- Produces: `attach_ingested_channel(practice, person, kind, raw, country_code=None) -> (channel_or_None, outcome)` in `contact/channel_resolution.py`, with new outcome constants `OUTCOME_ALREADY_LINKED = "already_linked"` and `OUTCOME_FOREIGN_CHANNEL = "channel_belongs_to_another_person"`.

**The bug:** at `patient_views.py:1438`, a CSV row that matches an existing patient (by phone, or by secondary phone) with no categories to add and no secondary to fill takes `skipped_count += 1; continue`. The row's email — a *new* address for a known human — is discarded. Nothing ever links it to that Person, so the next ingest arriving under that address matches nothing and mints a second Person. The importer's own dedup is what creates the later duplicate.

- [ ] **Step 1: Write the failing test**

```python
from django.test import TestCase

from TreatmentPlan.models import ContactChannel, Patient, Person, PersonChannel
from UserAuthentication.models import Practice


class CsvImportKeepsDedupedContactDetails(TestCase):
    """A CSV row that dedups onto an existing patient must still contribute
    its contact details to that patient's Person. Dropping them is what makes
    the NEXT ingest under the new address mint a duplicate Person."""

    def setUp(self):
        self.practice = Practice.objects.create(
            name="CSV QA", slug="csv-qa", default_country_code="+44")
        self.existing = Patient.objects.create(
            practice=self.practice, first_name="Victor", last_name="Csvlane",
            email="old@x.com", phone_number="7911125503", country_code="+44")

    def _import(self, csv_text):
        from TreatmentPlan.views.patient_views import PatientViewSet
        from django.core.files.uploadedfile import SimpleUploadedFile
        from types import SimpleNamespace

        view = PatientViewSet()
        view.get_user_practice_or_none = lambda: self.practice
        return view.import_csv(SimpleNamespace(
            FILES={"file": SimpleUploadedFile("qa.csv", csv_text.encode())},
            data={}))

    def test_new_email_on_a_phone_matched_row_is_linked_to_the_person(self):
        self._import(
            "first_name,last_name,email,phone_number\n"
            "Victor,Csvlane,new@x.com,+447911125503\n")

        person = Patient.objects.get(pk=self.existing.pk).person
        self.assertIsNotNone(person, "existing patient must have a Person")
        linked = {
            pc.channel.canonical_value
            for pc in PersonChannel.objects.filter(
                person=person).select_related("channel")
            if pc.channel.kind == ContactChannel.EMAIL
        }
        self.assertIn("new@x.com", linked)

    def test_the_new_email_is_not_promoted_over_the_existing_primary(self):
        self._import(
            "first_name,last_name,email,phone_number\n"
            "Victor,Csvlane,new@x.com,+447911125503\n")

        person = Patient.objects.get(pk=self.existing.pk).person
        primaries = [
            pc.channel.canonical_value
            for pc in PersonChannel.objects.filter(
                person=person, is_primary=True).select_related("channel")
            if pc.channel.kind == ContactChannel.EMAIL
        ]
        self.assertNotIn("new@x.com", primaries)

    def test_no_second_patient_row_is_created(self):
        self._import(
            "first_name,last_name,email,phone_number\n"
            "Victor,Csvlane,new@x.com,+447911125503\n")
        self.assertEqual(
            Patient.objects.filter(practice=self.practice).count(), 1)

    def test_an_email_owned_by_another_person_is_reported_not_welded(self):
        other = Patient.objects.create(
            practice=self.practice, first_name="Someone", last_name="Else",
            email="taken@x.com", phone_number="7900000001", country_code="+44")
        response = self._import(
            "first_name,last_name,email,phone_number\n"
            "Victor,Csvlane,taken@x.com,+447911125503\n")

        person = Patient.objects.get(pk=self.existing.pk).person
        linked = {
            pc.channel.canonical_value
            for pc in PersonChannel.objects.filter(
                person=person).select_related("channel")
            if pc.channel.kind == ContactChannel.EMAIL
        }
        self.assertNotIn(
            "taken@x.com", linked,
            "an address owned by another Person must not be welded on")
        self.assertNotEqual(
            person.pk, Patient.objects.get(pk=other.pk).person.pk)
        self.assertTrue(any(
            "another" in str(f.get("error", "")).lower()
            for f in response.data.get("failed_imports", [])))
```

- [ ] **Step 2: Run test to verify it fails**

Run:
```bash
source TreatmentPathBackend/venv/bin/activate
cd TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPlan.tests.test_csv_import_identity --keepdb -v 2
```
Expected: FAIL — `new@x.com` is not among the Person's linked channels, because the skip branch discarded it.

- [ ] **Step 3: Write minimal implementation**

Append to `contact/channel_resolution.py`:

```python
OUTCOME_ALREADY_LINKED = "already_linked"
OUTCOME_FOREIGN_CHANNEL = "channel_belongs_to_another_person"


def attach_ingested_channel(practice, person, kind, raw, country_code=None):
    """Link a contact detail carried by an ingested record onto `person`.

    Returns (channel, outcome); (None, None) when `raw` holds nothing that can
    be canonicalised, and (channel, OUTCOME_FOREIGN_CHANNEL) — WITHOUT linking
    — when the address already belongs to a different Person. Two Persons
    sharing an address is a duplicate for a human to adjudicate; silently
    welding them here would merge two people on the strength of a spreadsheet
    cell.

    Never promotes to primary: this is a detail arriving alongside an existing
    record, not a statement that it supersedes the person's chosen address.
    `link_channels` only promotes a kind that has NO primary yet, which is the
    behaviour we want and the reason this does not set the flag itself.
    """
    from TreatmentPlan.models import ContactChannel, PersonChannel

    if person is None:
        return None, None

    channel = ContactChannel.get_or_create_channel(practice, kind, raw,
                                                   country_code)
    if channel is None:
        return None, None

    owners = set(
        PersonChannel.objects.filter(channel=channel)
        .values_list("person_id", flat=True)
    )
    if person.id in owners:
        return channel, OUTCOME_ALREADY_LINKED
    if owners:
        logger.info(
            "attach_ingested_channel: refused to link channel %s to person %s "
            "— already held by %s",
            channel.id, person.id, sorted(owners),
        )
        return channel, OUTCOME_FOREIGN_CHANNEL

    person.link_channels([channel.id])
    return channel, OUTCOME_LINKED
```

In `patient_views.py`, add `"person_id"` to the prefetch at line ~1283:

```python
            existing_patients = list(
                Patient.objects.filter(practice=practice).values(
                    "id",
                    "person_id",
                    "email",
                    "phone_number",
                    "secondary_phone_number",
                    "country_code",
                )
            )
```

and, inside `if existing_patient:` before the `skipped_count` branch, contribute
the row's details to the matched Person:

```python
                    if existing_patient:
                        updated = False

                        # The row dedups onto a patient we already hold — but
                        # it may carry contact details that patient does NOT
                        # have. Dropping them is how the importer's own dedup
                        # creates the NEXT duplicate: nothing links the new
                        # address to this Person, so a later ingest under it
                        # matches nothing and mints a second one.
                        conflicts = _attach_row_contacts_to_person(
                            practice, existing_patient["person_id"],
                            email, phone_number, country_code,
                        )
                        if conflicts:
                            failed_imports.append({
                                "row": row_num,
                                "error": (
                                    "Row matched an existing patient, but "
                                    f"{', '.join(conflicts)} already belongs to "
                                    "another person — not linked; review the "
                                    "duplicate by hand"
                                ),
                                "data": row,
                            })

                        if secondary_phone_number and not existing_patient.get(
                            "secondary_phone_number"
                        ):
                            ...
```

with the helper defined at module level next to the other import helpers:

```python
def _attach_row_contacts_to_person(practice, person_id, email, phone_number,
                                   country_code):
    """Link a deduped CSV row's email/phone onto the matched patient's Person.
    Returns the values that could NOT be linked because another Person already
    holds them (a duplicate for a human to adjudicate, never an auto-merge)."""
    from ..contact.channel_resolution import (
        OUTCOME_FOREIGN_CHANNEL,
        attach_ingested_channel,
    )
    from ..models import ContactChannel, Person as PersonModel

    if not person_id:
        return []
    person = PersonModel.objects.filter(pk=person_id).first()
    if person is None:
        return []

    conflicts = []
    for kind, raw, cc in (
        (ContactChannel.EMAIL, email, None),
        (ContactChannel.PHONE, phone_number, country_code),
    ):
        if not raw:
            continue
        _, outcome = attach_ingested_channel(practice, person, kind, raw, cc)
        if outcome == OUTCOME_FOREIGN_CHANNEL:
            conflicts.append(raw)
    return conflicts
```

- [ ] **Step 4: Run test to verify it passes**

Run:
```bash
source TreatmentPathBackend/venv/bin/activate
cd TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPlan.tests.test_csv_import_identity --keepdb -v 2
```
Expected: PASS, all four cases.

- [ ] **Step 5: Make the QA shim report the patient a deduped import landed on**

The shim looks the patient up by the row's email, so a deduped row reports
`{"id": null}` and the harness cannot attribute the ref to any row. Resolve
through the identity graph instead — which now works precisely because Step 3
links the address.

In `qa_ingress_shim.py`, replace the `created = Patient.objects.filter(...)`
lookup with:

```python
            view.import_csv(request)
            out = {"id": _csv_landed_patient_id(practice, payload),
                   "table": "patient"}
```

and add, at module level:

```python
def _csv_landed_patient_id(practice, payload):
    """The Patient a csv_import row landed on — the one it created, or the
    pre-existing one it deduped onto. The dedup case cannot be found by the
    row's email (that patient carries its OLD address), so fall back to the
    Person the email channel now hangs off."""
    from TreatmentPlan.contact.sync import normalize_email
    from TreatmentPlan.models import ContactChannel, Patient, PersonChannel

    email = normalize_email(payload.get("email"))
    if not email:
        return None

    direct = (
        Patient.objects.filter(practice=practice, email=email)
        .order_by("-id")
        .first()
    )
    if direct is not None:
        return direct.id

    channel = ContactChannel.find_channel(practice, ContactChannel.EMAIL, email)
    if channel is None:
        return None
    person_ids = list(
        PersonChannel.objects.filter(channel=channel)
        .values_list("person_id", flat=True)
    )
    landed = (
        Patient.objects.filter(practice=practice, person_id__in=person_ids)
        .order_by("-id")
        .first()
    )
    return landed.id if landed is not None else None
```

- [ ] **Step 6: Run the full backend contact/identity suites**

Run:
```bash
source TreatmentPathBackend/venv/bin/activate
cd TreatmentPathBackend/TreatmentPath
python manage.py test TreatmentPlan.tests.test_csv_import_identity \
  TreatmentPlan.tests.test_normalization_unification \
  TreatmentPlan.tests.test_channel_resolution --keepdb -v 2
```
Expected: PASS, no regressions in the pre-existing channel-resolution suite.

- [ ] **Step 7: Re-run the live scenario suite**

Run: `cd ingress-test-engine && ./venv/bin/ingress-engine run`
Expected: `csv-import-lane` resolves r1 to the deduped patient's Person and
scores an earned verdict rather than `{"id": null}`.

---

## Self-Review

**Spec coverage:** the plan covers both halves the user asked for — the scorer keying (Tasks 1-3) and the product logic it exposed (Task 4). The `person_count` expectation in scenarios still reads `actual`, which Task 3 leaves populated for every ref, so no scenario loses its count assertion.

**Placeholder scan:** no TBDs; every code step carries the literal code.

**Type consistency:** `ref_rows` is `{ref: {"table": str, "id": int}}` in Tasks 2 and 3; `row_ids` is `{table: [int]}` in Tasks 1 and 3; `snapshot()` gains the same fourth positional parameter in both. `attach_ingested_channel(practice, person, kind, raw, country_code=None)` takes a Person **instance**, and `_attach_row_contacts_to_person` resolves the id to one before calling it.

---

### Task 5: Namespace phones per run (added during execution)

**Files:**
- Modify: `ingress-test-engine/engine/runner.py` (`namespace_phone`, `_namespace_records`)
- Test: `ingress-test-engine/tests/test_runner.py`

**Why this was added:** with the scorer fixed, `unification-crosslane` went red — but not
for a product reason. Its `patients_crud` POST returned `409 patient_exists` pointing at
patient 140760, a row left behind by an EARLIER run (email namespace `178850349025`)
carrying the same phone. Emails were namespaced per run; phones were not. The record never
landed, so the scorer honestly reported a missed merge for what was really harness
collision — a false alarm, which costs as much trust as the false green did.

**The rule:** shift the last six digits of the phone by a per-run offset, in place,
leaving every non-digit character alone. This preserves the three properties the scenarios
depend on: equal numbers stay equal (archetype-5/6 share a household phone); the shift is
injective, so numbers differing only in their last six digits stay distinct
(`+447911124401` and `+447911125503` are different scenarios and must not fuse); and
format variants of one number still canonicalise together, since they share those last six
digits (`+447708870413` and `07708 870413` are one human in archetype-3). Values with
fewer than 10 digits are left exactly as written — those are the deliberately malformed
adversarial inputs.

- [ ] **Step 1-4:** TDD as above; seven tests in `tests/test_runner.py` covering sharing,
  injectivity, format-variant agreement, whitespace preservation, malformed passthrough,
  cross-run isolation, and `_namespace_records` wiring.

- [ ] **Step 5: Verify injectivity on the real corpus**

Rewrite all 24 distinct phone spellings in `scenarios/` under several run namespaces and
assert zero canonical collisions.

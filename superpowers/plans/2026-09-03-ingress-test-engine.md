# Ingress Test Engine — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Standalone fixture-driven harness that fires synthetic identity data (with known ground truth) at the real DEV entry points and scores how many duplicate/unique verdicts the system got right.

**Architecture:** Python package + FastAPI web UI. Scenarios are YAML (records + ground-truth `truth` block + `expect` assertions). Drivers fire payloads (HTTP or `manage.py shell` shim); the runner snapshots the DEV Postgres graph with a read-only role; the scorer compares actual Person groupings to truth and classifies every ref pair (correct merge / missed merge / wrong merge / correct unique).

**Tech Stack:** Python 3.12, requests, psycopg2, PyYAML, PyJWT, click, fastapi + uvicorn, pytest. New git repo at `treatmentpath/ingress-test-engine/`.

**Spec:** `docs/superpowers/specs/2026-09-03-ingress-test-engine-design.md`

**Environment facts (already verified in this workspace):**
- Django app lives at `TreatmentPathBackend/TreatmentPath/`, venv at `TreatmentPathBackend/venv`.
- DEV Postgres: `localhost:5432`. JWTs are simplejwt HS256 AccessTokens carrying `practice_id` + `practice_slug` claims (see `UserAuthentication/authentication.py`); mint them with the backend `SECRET_KEY`.
- Identity tables: `TreatmentPlan_person` (`household_id`, `merged_into_id`), `TreatmentPlan_contactchannel` (`practice_id`, `kind`, `canonical_value`), `TreatmentPlan_personchannel` (`person_id`, `channel_id`, `is_primary`), plus `TreatmentPlan_intake` / `_patient` / `_nurture` record rows holding `email`/`phone_number`. **Verify exact table/column names with `\dt` against DEV before Task 4** — if they differ, adjust `SQL` dict in `engine/db.py` only.
- Webhook/auth endpoints: confirm exact paths from `TreatmentPath/urls.py` when writing drivers; the plan marks them `VERIFY`.

---

### Task 1: Scaffold project

**Files:**
- Create: `ingress-test-engine/pyproject.toml`, `ingress-test-engine/engine/__init__.py`, `engine/cli.py`, `engine/config.py`, `ingress-test-engine/tests/__init__.py`, `ingress-test-engine/.env.example`, `ingress-test-engine/README.md`

- [ ] **Step 1: Create directory and init git**

```bash
mkdir -p ~/Desktop/Projects/treatmentpath/ingress-test-engine/{engine,tests,web,scenarios,reports}
cd ~/Desktop/Projects/treatmentpath/ingress-test-engine && git init
```

- [ ] **Step 2: Write `pyproject.toml`**

```toml
[project]
name = "ingress-test-engine"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["requests>=2.31", "psycopg2-binary>=2.9", "pyyaml>=6", "PyJWT>=2.8", "click>=8.1", "fastapi>=0.110", "uvicorn>=0.29", "phonenumbers>=8.13"]

[project.optional-dependencies]
dev = ["pytest>=8"]

[project.scripts]
ingress-engine = "engine.cli:cli"

[tool.setuptools.packages.find]
include = ["engine*"]

[build-system]
requires = ["setuptools>=68"]
build-backend = "setuptools.build_meta"
```

- [ ] **Step 3: Write `engine/config.py`**

```python
import os
from dataclasses import dataclass, field

@dataclass
class Config:
    api_base: str = "http://localhost:8000"
    dev_dsn: str = "postgresql://sim_readonly@localhost:5432/treatmentpath_db"
    backend_dir: str = "../TreatmentPathBackend/TreatmentPath"
    backend_venv_python: str = "../TreatmentPathBackend/venv/bin/python"
    secret_key: str = ""            # Django SECRET_KEY, for JWT minting
    qa_practice_id: int = 0
    qa_practice_slug: str = ""
    qa_user_id: int = 0
    run_marker: str = "qa-ingress"  # namespace prefix for synthetic contacts

    @classmethod
    def load(cls, path: str | None = None) -> "Config":
        path = path or os.environ.get("INGRESS_ENGINE_CONFIG", ".env")
        cfg = cls()
        if os.path.exists(path):
            for line in open(path):
                line = line.strip()
                if not line or line.startswith("#") or "=" not in line:
                    continue
                k, v = line.split("=", 1)
                if hasattr(cfg, k.strip()):
                    attr = k.strip()
                    val = v.strip()
                    setattr(cfg, attr, int(val) if attr.endswith(("_id",)) and val.isdigit() else val)
        return cfg
```

- [ ] **Step 4: Write `.env.example`**

```dotenv
API_BASE=http://localhost:8000
DEV_DSN=postgresql://sim_readonly@localhost:5432/treatmentpath_db
BACKEND_DIR=../TreatmentPathBackend/TreatmentPath
BACKEND_VENV_PYTHON=../TreatmentPathBackend/venv/bin/python
SECRET_KEY=<copy from TreatmentPathBackend .env — never commit the real .env>
QA_PRACTICE_ID=0
QA_PRACTICE_SLUG=qa-ingress-engine
QA_USER_ID=0
RUN_MARKER=qa-ingress
```

- [ ] **Step 5: Write `engine/cli.py` (stub for now)**

```python
import click

@click.group()
def cli():
    """Ingress Test Engine — identity regression harness."""

if __name__ == "__main__":
    cli()
```

- [ ] **Step 6: Install + verify**

```bash
cd ~/Desktop/Projects/treatmentpath/ingress-test-engine
python3.12 -m venv venv && source venv/bin/activate
pip install -e ".[dev]" && ingress-engine --help
```
Expected: help text prints. Copy `.env.example` to `.env` and fill `SECRET_KEY` + `DEV_DSN` (read the DSN/secret from the backend's `.env`).

- [ ] **Step 7: Commit**

```bash
git add -A && git commit -m "chore: scaffold ingress-test-engine"
```

---

### Task 2: Core models + YAML scenario (de)serialization

**Files:**
- Create: `engine/models.py`, `tests/test_models.py`

- [ ] **Step 1: Write failing tests (`tests/test_models.py`)**

```python
from engine.models import Record, Truth, Scenario

def test_scenario_roundtrip():
    s = Scenario(
        name="t", order="sequential",
        records=[Record(ref="A", person_key="p1", first_name="Alice", last_name="Shaw",
                        email="qa+r1-a@qa.example.com", phone="+447700900001",
                        country_code="GB", entry_point="legacy_webhook")],
        truth=Truth(persons={"A": "p1"}, shared_channels=[], households=[]),
        expect={"person_count": 1},
    )
    yaml_text = s.to_yaml()
    s2 = Scenario.from_yaml(yaml_text)
    assert s2 == s
```

- [ ] **Step 2: Run — expect ImportError**

```bash
cd ~/Desktop/Projects/treatmentpath/ingress-test-engine && source venv/bin/activate
python -m pytest tests/test_models.py -v
```

- [ ] **Step 3: Implement `engine/models.py`**

```python
from dataclasses import dataclass, field, asdict
import yaml

@dataclass
class Record:
    ref: str
    person_key: str          # ground-truth synthetic person; refs sharing key = same human
    first_name: str
    last_name: str
    email: str
    phone: str               # exactly as the lane should receive it (noise variants allowed)
    country_code: str = "GB"
    entry_point: str = "legacy_webhook"

@dataclass
class Truth:
    persons: dict            # ref -> person_key
    shared_channels: list    # [{"a": refA, "b": refB, "kind": "phone"}]
    households: list         # [[ref, ref, ...]]

@dataclass
class Scenario:
    name: str
    order: str               # sequential | reverse | concurrent
    records: list            # [Record]
    truth: Truth
    expect: dict = field(default_factory=dict)

    def to_yaml(self) -> str:
        return yaml.safe_dump({
            "scenario": self.name, "order": self.order,
            "records": [asdict(r) for r in self.records],
            "truth": asdict(self.truth), "expect": self.expect,
        }, sort_keys=False)

    @staticmethod
    def from_yaml(text: str) -> "Scenario":
        d = yaml.safe_load(text)
        return Scenario(
            name=d["scenario"], order=d.get("order", "sequential"),
            records=[Record(**r) for r in d["records"]],
            truth=Truth(**d["truth"]), expect=d.get("expect", {}),
        )
```

- [ ] **Step 4: Run test — expect PASS** (`python -m pytest tests/test_models.py -v`)
- [ ] **Step 5: Commit** — `git add -A && git commit -m "feat: scenario models + yaml roundtrip"`

---

### Task 3: Phone number factory (`phonenumbers`-based)

**Files:**
- Create: `engine/phones.py`, `tests/test_phones.py`

- [ ] **Step 1: Write failing tests**

```python
import phonenumbers
from engine.phones import valid_phone, format_variant, rubbish_phone

def test_valid_phone_for_any_region_passes_libphonenumber():
    for region in ["GB", "US", "FR", "NG", "DE", "AU", "IN"]:
        raw, cc = valid_phone(region)
        parsed = phonenumbers.parse(raw)
        assert phonenumbers.is_valid_number(parsed), (region, raw)
        assert cc == region

def test_format_variant_preserves_number():
    e164, _ = valid_phone("GB")
    national = format_variant(e164, "NATIONAL")       # '07700 900123' style
    reparsed = phonenumbers.parse(national, "GB")
    assert phonenumbers.is_valid_number(reparsed)

def test_rubbish_phone_is_not_valid():
    for raw in [rubbish_phone("GB", kind="truncated"),
                rubbish_phone("GB", kind="letters"),
                rubbish_phone("GB", kind="transpose"),
                rubbish_phone("GB", kind="plus_middle"),
                rubbish_phone("GB", kind="empty")]:
        parsed = phonenumbers.parse(raw, "GB")
        assert not phonenumbers.is_valid_number(parsed), raw
```

- [ ] **Step 2: Run — expect ImportError**

- [ ] **Step 3: Implement `engine/phones.py`**

```python
"""Phone factory. Valid numbers come from libphonenumber (via the phonenumbers
package) so ANY region works; rubbish numbers violate it in specific, named ways."""
import random
import phonenumbers
from phonenumbers import NumberParseException
from phonenumbers.phonenumberutil import example_number_for_type, PhoneNumberType

def valid_phone(region: str, rng: random.Random | None = None) -> tuple[str, str]:
    """A random VALID number for the region, in E.164. Returns (e164, region).
    Derives candidates from the region's example number, then perturbs the
    subscriber digits until is_valid_number passes — keeps us inside the
    region's real numbering plan for any country code."""
    rng = rng or random.Random()
    base = example_number_for_type(region, PhoneNumberType.MOBILE) or \
           example_number_for_type(region, PhoneNumberType.FIXED_LINE)
    if base is None:
        raise ValueError(f"no numbering plan known for region {region}")
    digits = list(str(base.national_number))
    for _ in range(200):                      # bounded search; plans differ in shape
        cand = digits[:]
        for i in range(1, len(cand)):         # leave the first digit (trunk rules) alone
            cand[i] = str(rng.randrange(10))
        candidate = phonenumbers.PhoneNumber(country_code=base.country_code,
                                             national_number=int("".join(cand)))
        if phonenumbers.is_valid_number(candidate):
            return phonenumbers.format_number(candidate, phonenumbers.PhoneNumberFormat.E164), region
    # fall back to the example number itself — always valid
    return phonenumbers.format_number(base, phonenumbers.PhoneNumberFormat.E164), region

def format_variant(e164: str, style: str, region: str = "GB") -> str:
    n = phonenumbers.parse(e164, region)
    if style == "E164":
        return phonenumbers.format_number(n, phonenumbers.PhoneNumberFormat.E164)
    if style == "NATIONAL":
        return phonenumbers.format_number(n, phonenumbers.PhoneNumberFormat.NATIONAL)
    if style == "SPACES":                     # national + aggressive spacing
        nat = phonenumbers.format_number(n, phonenumbers.PhoneNumberFormat.NATIONAL)
        return nat.replace(" ", "").replace("-", " ")  # hmm — keep grouping noisy
    if style == "INTERNATIONAL":
        return phonenumbers.format_number(n, phonenumbers.PhoneNumberFormat.INTERNATIONAL)
    if style == "PADDED":
        return "  " + phonenumbers.format_number(n, phonenumbers.PhoneNumberFormat.E164) + " "
    raise ValueError(f"unknown style {style}")

def rubbish_phone(region: str, kind: str, rng: random.Random | None = None) -> str:
    """A number that must NEVER validate. Each kind is a distinct failure mode
    the lanes should survive (adversarial family A1)."""
    rng = rng or random.Random()
    good, _ = valid_phone(region, rng)
    if kind == "truncated":
        return good[: max(3, len(good) - 4)]
    if kind == "letters":
        return good[:3] + "Q" + good[4:]
    if kind == "transpose":
        return good[:-2] + good[-1] + good[-2]
    if kind == "plus_middle":
        return good[:2] + "+" + good[3:]
    if kind == "empty":
        return ""
    if kind == "garbage":
        return "".join(rng.choice("+-() 0123456789abc") for _ in range(rng.randrange(1, 25)))
    raise ValueError(f"unknown kind {kind}")
```

Note: `transpose`/`plus_middle` can *occasionally* still be valid for some plans — the test asserts invalidity for the specific example drawn; if a flake appears, the test draws with a fixed seed per region so it's deterministic.

- [ ] **Step 4: Run tests — PASS** (`python -m pytest tests/test_phones.py -v`)
- [ ] **Step 5: Commit** — `git commit -am "feat: libphonenumber phone factory — valid-for-any-region + named rubbish modes"`

---

### Task 4: Archetypes + generator

**Files:**
- Create: `engine/archetypes.py`, `engine/generator.py`, `tests/test_generator.py`

- [ ] **Step 1: Write failing tests**

```python
from engine.generator import generate

def test_determinism():
    a = generate(persons=3, families=1, family_size=3,
                 duplicates={"reformatted_phone": 2}, seed=42)
    b = generate(persons=3, families=1, family_size=3,
                 duplicates={"reformatted_phone": 2}, seed=42)
    assert a.to_yaml() == b.to_yaml()

def test_unique_person_truth():
    s = generate(persons=2, seed=1)
    assert len({s.truth.persons[r.ref] for r in s.records}) == 2

def test_family_shares_phone():
    s = generate(families=1, family_size=3, seed=1)
    assert s.truth.shared_channels and s.truth.households

def test_region_knob_produces_matching_country_codes():
    s = generate(persons=3, regions=["US", "FR", "NG"], seed=5)
    codes = {r.country_code for r in s.records}
    assert codes == {"US", "FR", "NG"}

def test_invalid_phones_knob_adds_rubbish_records():
    s = generate(persons=2, invalid_phones=3, seed=9)
    rubbish = [r for r in s.records if r.entry_point == "rubbish_phone"]
    assert len(rubbish) == 3
```

- [ ] **Step 2: Run — expect ImportError**

- [ ] **Step 3: Implement `engine/archetypes.py`**

```python
"""Archetype record factories. Each returns extra Record(s) + truth mutations.

Phones: produced by engine.phones.valid_phone (libphonenumber) for ANY region —
never a hardcoded pool. Emails: qa+<run>-<ref>N@qa.example.com.
"""
import random
from .phones import valid_phone, format_variant

FIRST = ["Alice", "Bob", "Chioma", "Dev", "Ewa", "Farid", "Grace", "Hugo"]
LAST = ["Shaw", "Okafor", "Chen", "Patel", "Novak", "Diop"]

def _new_person(rng, seq, run_id, region="GB"):
    ref = f"r{seq}"
    e164, _region = valid_phone(region, rng)
    return {
        "ref": ref,
        "first_name": rng.choice(FIRST),
        "last_name": rng.choice(LAST),
        "email": f"qa+{run_id}-{ref}@qa.example.com",
        "phone": e164,
        "country_code": region,
    }

ARCHETYPES = {}

def archetype(name):
    def deco(fn):
        ARCHETYPES[name] = fn
        return fn
    return deco

@archetype("reformatted_phone")
def reformatted_phone(base, rng, run_id, seq):
    """Same human, national format instead of E.164 — must canonicalize to SAME channel."""
    r = dict(base)
    r["ref"] = f"r{seq}"
    r["phone"] = format_variant(base["phone"], "NATIONAL", base["country_code"])
    return r, {"same_person_as": base["ref"]}

@archetype("padded_phone")
def padded_phone(base, rng, run_id, seq):
    """Same human, whitespace-padded E.164 — must canonicalize to SAME channel."""
    r = dict(base)
    r["ref"] = f"r{seq}"
    r["phone"] = format_variant(base["phone"], "PADDED", base["country_code"])
    return r, {"same_person_as": base["ref"]}

@archetype("new_email_same_person")
def new_email_same_person(base, rng, run_id, seq):
    r = dict(base)
    r["ref"] = f"r{seq}"
    r["email"] = f"qa+{run_id}-{ref2alt(base['ref'])}@qa.example.com"
    return r, {"same_person_as": base["ref"]}

@archetype("name_variant")
def name_variant(base, rng, run_id, seq):
    r = dict(base)
    r["ref"] = f"r{seq}"
    r["last_name"] = r["last_name"] + "-" + rng.choice(LAST)   # married-name style
    return r, {"same_person_as": base["ref"]}

@archetype("email_noise")
def email_noise(base, rng, run_id, seq):
    """Same email, case + whitespace noise — must dedup, not create a channel."""
    r = dict(base)
    r["ref"] = f"r{seq}"
    r["email"] = "  " + base["email"].upper() + " "
    return r, {"same_person_as": base["ref"]}

@archetype("merged_person_returns")
def merged_person_returns(base, rng, run_id, seq):
    """Contact value of a previously merged-away Person re-arrives."""
    r = dict(base)
    r["ref"] = f"r{seq}"
    r["first_name"] = rng.choice(FIRST)                # stale name from before merge
    return r, {"same_person_as": base["ref"]}

def ref2alt(ref):
    return ref + "alt"
```

- [ ] **Step 4: Implement `engine/generator.py`**

```python
import random
from .models import Record, Truth, Scenario
from .archetypes import _new_person, ARCHETYPES, FIRST, LAST
from .phones import rubbish_phone

RUBBISH_KINDS = ["truncated", "letters", "transpose", "plus_middle", "garbage"]

def generate(persons=0, families=0, family_size=3,
             duplicates=None, entry_points=None, order="sequential", seed=0,
             regions=None, invalid_phones=0) -> Scenario:
    """Build a Scenario. `duplicates` maps archetype-name -> count (applied to
    random distinct base persons). `entry_points` optionally maps ref -> lane;
    otherwise lanes cycle through the default set. `regions` cycles valid-number
    regions (libphonenumber); `invalid_phones` adds N deliberately-rubbish
    records (entry_point=rubbish_phone) as isolated truth-persons."""
    duplicates = dict(duplicates or {})
    entry_points = dict(entry_points or {})
    regions = regions or ["GB"]
    rng = random.Random(seed)
    run_id = f"s{seed}"
    records, truth_map, households, shared = [], {}, [], []
    lanes = ["legacy_webhook", "custom_webhook", "add_patient", "ai_email"]
    seq = 0
    bases = []
    for i in range(persons):
        seq += 1
        b = _new_person(rng, seq, run_id, regions[i % len(regions)])
        bases.append(b)
        records.append(Record(entry_point=entry_points.get(b["ref"], lanes[seq % len(lanes)]),
                              **{k: b[k] for k in ("ref", "first_name", "last_name", "email", "phone", "country_code")}))
        truth_map[b["ref"]] = b["ref"]
    for _ in range(families):
        family_refs = []
        last = rng.choice(LAST)
        region = regions[seq % len(regions)]
        for _i in range(family_size):
            seq += 1
            b = _new_person(rng, seq, run_id, region)
            b["last_name"] = last
            bases.append(b)
            records.append(Record(entry_point=entry_points.get(b["ref"], lanes[seq % len(lanes)]),
                                  **{k: b[k] for k in ("ref", "first_name", "last_name", "email", "phone", "country_code")}))
            truth_map[b["ref"]] = b["ref"]
            family_refs.append(b["ref"])
        shared.append({"a": family_refs[0], "b": family_refs[1], "kind": "phone"})
        # family phone: later members deliver member 0's number, format-noised
        head = next(r for r in records if r.ref == family_refs[0])
        style = "NATIONAL" if rng.random() < 0.5 else "PADDED"
        for ref in family_refs[1:]:
            next(r for r in records if r.ref == ref).phone = format_variant(head.phone, style, head.country_code)
        households.append(family_refs)
    for arch, count in duplicates.items():
        for _ in range(count):
            base = rng.choice(bases)
            seq += 1
            dup, note = ARCHETYPES[arch](base, rng, run_id, seq)
            truth_map[dup["ref"]] = truth_map[note["same_person_as"]]
            records.append(Record(entry_point=entry_points.get(dup["ref"], lanes[seq % len(lanes)]),
                                  **{k: dup[k] for k in ("ref", "first_name", "last_name", "email", "phone", "country_code")}))
    for i in range(invalid_phones):
        seq += 1
        region = regions[i % len(regions)]
        ref = f"r{seq}"
        records.append(Record(ref=ref, person_key=ref,
                              first_name=rng.choice(FIRST), last_name=rng.choice(LAST),
                              email=f"qa+{run_id}-{ref}@qa.example.com",
                              phone=rubbish_phone(region, RUBBISH_KINDS[i % len(RUBBISH_KINDS)], rng),
                              country_code=region, entry_point="rubbish_phone"))
        truth_map[ref] = ref
    return Scenario(name=f"gen-{seed}", order=order, records=records,
                    truth=Truth(persons=truth_map, shared_channels=shared, households=households),
                    expect={})
```

- [ ] **Step 5: Run tests — expect PASS** (`python -m pytest tests/test_generator.py -v`)
- [ ] **Step 6: Commit** — `git commit -am "feat: archetypes + deterministic generator"`

---

### Task 5: DB snapshot (`engine/db.py`)

**Files:**
- Create: `engine/db.py`, `tests/test_db.py`

- [ ] **Step 0 (VERIFY first):** Against DEV `psql`, confirm table/column names:

```bash
psql "$DEV_DSN" -c "\dt TreatmentPlan.*" -c "\d TreatmentPlan_person" | head -40
```
Adjust the `SQL` dict below to the real names before writing the test.

- [ ] **Step 1: Implement `engine/db.py`**

```python
"""Read-only snapshot of the identity graph for one run (rows matched by the
run's unique email marker). One responsibility: DB -> plain dicts."""
import psycopg2

SQL_RECORDS = """
SELECT r.email, r.phone_number, r.person_id
  FROM TreatmentPlan_intake r
  JOIN TreatmentPlan_practice p ON p.id = r.practice_id
 WHERE p.slug = %(slug)s AND r.email LIKE %(marker)s
 UNION ALL
 SELECT r.email, r.phone_number, r.person_id
  FROM TreatmentPlan_patient r
  JOIN TreatmentPlan_practice p ON p.id = r.practice_id
 WHERE p.slug = %(slug)s AND r.email LIKE %(marker)s
 UNION ALL
 SELECT r.email, r.phone_number, r.person_id
  FROM TreatmentPlan_nurture r
  JOIN TreatmentPlan_practice p ON p.id = r.practice_id
 WHERE p.slug = %(slug)s AND r.email LIKE %(marker)s
"""
SQL_PERSONS = """
SELECT per.id, per.merged_into_id, per.household_id
  FROM TreatmentPlan_person per
  JOIN TreatmentPlan_practice p ON p.id = per.practice_id
 WHERE p.slug = %(slug)s AND per.id IN (SELECT person_id FROM tmp_run_persons)
"""
SQL_CHANNELS = """
SELECT c.id, c.kind, c.canonical_value FROM TreatmentPlan_contactchannel c
  JOIN TreatmentPlan_practice p ON p.id = c.practice_id
 WHERE p.slug = %(slug)s AND c.canonical_value LIKE %(marker)s
"""

def snapshot(dsn: str, slug: str, marker: str) -> dict:
    """marker is a LIKE pattern, e.g. 'qa+s42-%@qa.example.com'."""
    out = {"records": [], "persons": {}, "channels": []}
    with psycopg2.connect(dsn) as conn, conn.cursor() as cur:
        cur.execute(SQL_RECORDS, {"slug": slug, "marker": marker})
        out["records"] = [
            {"email": e, "phone": ph, "person_id": pid} for e, ph, pid in cur.fetchall()
        ]
        pids = [r["person_id"] for r in out["records"] if r["person_id"]]
        if pids:
            cur.execute(
                "SELECT id, merged_into_id, household_id FROM TreatmentPlan_person WHERE id = ANY(%s)",
                (pids,),
            )
            out["persons"] = {pid: {"merged_into": m, "household_id": h} for pid, m, h in cur.fetchall()}
        cur.execute(SQL_CHANNELS, {"slug": slug, "marker": marker})
        out["channels"] = [{"kind": k, "canonical": v} for _i, k, v in cur.fetchall()]
    return out

def resolve_person(snapshot: dict, person_id: int | None) -> int | None:
    """Follow merged_into chains to the survivor."""
    seen = set()
    while person_id and person_id in snapshot.get("persons", {}) and person_id not in seen:
        seen.add(person_id)
        person_id = snapshot["persons"][person_id]["merged_into"] or None
    return person_id
```

- [ ] **Step 2: Unit-test `resolve_person` pure logic (`tests/test_db.py`)**

```python
from engine.db import resolve_person

def test_merge_chain_resolves_to_survivor():
    snap = {"persons": {1: {"merged_into": 2, "household_id": None},
                        2: {"merged_into": None, "household_id": 9}}}
    assert resolve_person(snap, 1) == 2
    assert resolve_person(snap, None) is None
```

Run: `python -m pytest tests/test_db.py -v` — expect PASS.
- [ ] **Step 3: Live smoke against DEV (integration, not CI):**

```bash
python -c "from engine.config import Config; from engine.db import snapshot; print(snapshot(Config.load().dev_dsn, Config.load().qa_practice_slug, 'qa+none-%') )"
```
Expected: `{'records': [...], 'persons': {}, 'channels': [...]}` — runs without error.
- [ ] **Step 4: Commit** — `git commit -am "feat: read-only DB snapshot + merge-chain resolver"`

---

### Task 6: Scorer

**Files:**
- Create: `engine/scorer.py`, `tests/test_scorer.py`

- [ ] **Step 1: Write failing tests**

```python
from engine.scorer import classify

def _snap(mapping):  # ref -> person_id or None
    return {"ref_person": mapping, "channels": []}

def test_correct_merge_and_unique():
    s = {"truth": {"A": "p1", "B": "p1", "C": "p2"},
         "actual": {"A": 10, "B": 10, "C": 20}}
    v = classify(s["truth"], s["actual"])
    assert v["verdicts"] == {("A", "B"): "correct_merge",
                             ("A", "C"): "correct_unique",
                             ("B", "C"): "correct_unique"}

def test_missed_and_wrong_merge():
    s = {"truth": {"A": "p1", "B": "p1", "C": "p2"}}
    v = classify(s["truth"], {"A": 10, "B": 11, "C": 10})
    assert v["verdicts"][("A", "B")] == "missed_merge"
    assert v["verdicts"][("A", "C")] == "wrong_merge"

def test_scores():
    v = classify({"A": "p1", "B": "p1", "C": "p2"}, {"A": 1, "B": 1, "C": 2})
    assert v["precision"] == 1.0 and v["recall"] == 1.0 and v["accuracy"] == 1.0
```

- [ ] **Step 2: Run — expect FAIL**
- [ ] **Step 3: Implement `engine/scorer.py`**

```python
"""Pairwise classification of ground truth vs the actual Person groupings.

Truth values are synthetic person keys; actual values are DB person ids
(already merge-chain resolved). A ref the system never linked to a Person
(actual None) forms its own group — that counts as a missed merge if truth
says it shares a person with anyone.
"""
from itertools import combinations

def classify(truth: dict, actual: dict) -> dict:
    verdicts = {}
    tp = fp = fn = tn = 0
    for a, b in combinations(sorted(truth), 2):
        same_truth = truth[a] == truth[b]
        aa, ab = actual.get(a), actual.get(b)
        same_actual = aa is not None and aa == ab
        if same_truth and same_actual:
            v, tp = "correct_merge", tp + 1
        elif same_truth:
            v, fn = "missed_merge", fn + 1
        elif same_actual:
            v, fp = "wrong_merge", fp + 1
        else:
            v, tn = "correct_unique", tn + 1
        verdicts[(a, b)] = v
    precision = tp / (tp + fp) if tp + fp else 1.0
    recall = tp / (tp + fn) if tp + fn else 1.0
    total = tp + fp + fn + tn
    return {"verdicts": verdicts, "true_merge": tp, "wrong_merge": fp,
            "missed_merge": fn, "true_unique": tn,
            "precision": precision, "recall": recall,
            "accuracy": (tp + tn) / total if total else 1.0}

def score_scenario(scenario_truth, snapshot_records, marker_emails_by_ref) -> dict:
    """Bridge: map each ref to the person_id of the record row whose email matches."""
    by_email = {r["email"]: r["person_id"] for r in snapshot_records}
    actual = {}
    for ref, email in marker_emails_by_ref.items():
        actual[ref] = by_email.get(email)          # exact match on the namespaced email
    return classify(scenario_truth, actual)
```

Note: `email_noise` archetype intentionally changes casing/whitespace — the *record row* stores whatever the lane stored, so `score_scenario` must match refs by the run marker **prefix** when exact match fails. Implement the fallback:

```python
def _match_ref(email, by_email):
    if email in by_email:
        return by_email[email]
    norm = email.strip().lower()
    for e, pid in by_email.items():
        if e.strip().lower() == norm:
            return pid
    return None
```
…and use `_match_ref(email, by_email)` instead of `by_email.get(email)`. Re-run tests.

- [ ] **Step 4: Add conformance test — generator + scorer agree by construction**

```python
def test_generator_scores_perfect_on_identity_mapping():
    from engine.generator import generate
    s = generate(persons=2, families=1, family_size=2, duplicates={"email_noise": 1}, seed=7)
    # pretend the system grouped refs exactly per truth
    fake_ids = {"p1": 100, "p2": 101, "p3": 102}
    actual = {ref: fake_ids[key] for ref, key in s.truth.persons.items()}
    v = classify(s.truth.persons, actual)
    assert v["accuracy"] == 1.0
```

- [ ] **Step 5: Run full suite — PASS. Commit** — `git commit -am "feat: pairwise scorer + precision/recall"`

---

### Task 7: JWT auth helper

**Files:**
- Create: `engine/auth.py`, `tests/test_auth.py`

- [ ] **Step 1: Implement + test roundtrip decode**

```python
# engine/auth.py
import time
import jwt

def mint_access_token(secret_key: str, user_id: int, practice_id: int, practice_slug: str, lifetime=3600) -> str:
    now = int(time.time())
    return jwt.encode(
        {"user_id": user_id, "practice_id": practice_id, "practice_slug": practice_slug,
         "jti": f"qa-{now}", "exp": now + lifetime, "iat": now, "token_type": "access"},
        secret_key, algorithm="HS256",
    )
```

```python
# tests/test_auth.py
import jwt as pyjwt
from engine.auth import mint_access_token

def test_token_carries_practice_claims():
    tok = mint_access_token("s3cret", user_id=7, practice_id=13, practice_slug="qa")
    claims = pyjwt.decode(tok, "s3cret", algorithms=["HS256"])
    assert claims["practice_id"] == 13 and claims["token_type"] == "access"
```

- [ ] **Step 2: Run — PASS. Live check:** mint a token and `curl` a DRF endpoint that requires auth (e.g. `GET {API_BASE}/api/<a known practice-scoped list>/`) with `Authorization: Bearer <tok>` — expect 200, not 401. If simplejwt rejects (aud/iss claims configured), mirror whatever `SIMPLE_JWT` sets in the backend settings.
- [ ] **Step 3: Commit** — `git commit -am "feat: practice-scoped JWT minting"`

---

### Task 8: Driver framework + HTTP drivers

**Files:**
- Create: `engine/drivers/__init__.py`, `engine/drivers/base.py`, `engine/drivers/webhooks.py`, `engine/drivers/api.py`, `tests/test_drivers_registry.py`

- [ ] **Step 1: `engine/drivers/base.py`**

```python
REGISTRY: dict[str, "BaseDriver"] = {}

class BaseDriver:
    name = "?"
    def __init__(self, cfg, token: str | None = None):
        self.cfg, self.token = cfg, token
    def fire(self, record) -> dict:
        """Deliver one record; returns {'status': int|'shim', 'response': ...}.
        Raises DriverError on transport failure. Never retries (A2 tests own that)."""
        raise NotImplementedError

def register(cls):
    REGISTRY[cls.name] = cls
    return cls

class DriverError(RuntimeError):
    pass
```

- [ ] **Step 2: `engine/drivers/webhooks.py`** — endpoint paths `VERIFY` against `TreatmentPath/urls.py` first:

```python
import requests
from .base import BaseDriver, register, DriverError

class _Webhook(BaseDriver):
    def post(self, path, payload):
        try:
            r = requests.post(self.cfg.api_base + path, json=payload, timeout=30)
        except requests.RequestException as e:
            raise DriverError(str(e))
        return {"status": r.status_code, "response": r.text[:500]}

@register
class LegacyWebhook(_Webhook):
    name = "legacy_webhook"
    def fire(self, record):
        return self.post("/webhooks/intake/", {          # VERIFY path
            "first_name": record.first_name, "last_name": record.last_name,
            "email": record.email, "phone_number": record.phone,
            "country_code": record.country_code,
        })

@register
class CustomWebhook(_Webhook):
    name = "custom_webhook"
    def fire(self, record):
        return self.post("/webhooks/custom-journey/", {  # VERIFY path + required contract fields
            "first_name": record.first_name, "last_name": record.last_name,
            "email": record.email, "phone_number": record.phone,
            "country_code": record.country_code, "stage": "intake",
        })

@register
class FormHandoff(_Webhook):
    name = "form_handoff"
    def fire(self, record):
        return self.post("/api/marketing/form-handoff/", {  # VERIFY path
            "first_name": record.first_name, "last_name": record.last_name,
            "email": record.email, "phone_number": record.phone,
        })

@register
class MessageSession(_Webhook):
    name = "message_session"
    def fire(self, record):
        return self.post("/webhooks/inbound-sms/", {     # VERIFY path (Twilio-shaped)
            "From": record.phone, "Body": "qa ingress probe", "MessageSid": f"SM{record.ref}",
        })
```

- [ ] **Step 3: `engine/drivers/api.py`** — JWT-authenticated DRF lanes:

```python
import requests
from .base import BaseDriver, register, DriverError

class _Api(BaseDriver):
    def call(self, method, path, payload=None, params=None):
        try:
            r = requests.request(method, self.cfg.api_base + path, json=payload, params=params,
                                 headers={"Authorization": f"Bearer {self.token}"}, timeout=30)
        except requests.RequestException as e:
            raise DriverError(str(e))
        return {"status": r.status_code, "response": r.text[:500], "json": _safe_json(r)}

def _safe_json(r):
    try:
        return r.json()
    except ValueError:
        return None

@register
class AddPatient(_Api):
    name = "add_patient"
    def fire(self, record):
        return self.call("POST", "/api/intakes/", {          # VERIFY path
            "first_name": record.first_name, "last_name": record.last_name,
            "email": record.email, "phone_number": record.phone,
            "country_code": record.country_code, "type": "intake",
        })

@register
class PatientsCrud(_Api):
    name = "patients_crud"
    def fire(self, record):
        return self.call("POST", "/api/patients/", {         # VERIFY path
            "first_name": record.first_name, "last_name": record.last_name,
            "email": record.email, "phone_number": record.phone,
            "country_code": record.country_code,
        })

@register
class FamilyMember(_Api):
    name = "family_member"
    def fire(self, record):
        return self.call("POST", "/api/patients/add-family-member/", {  # VERIFY path
            "first_name": record.first_name, "last_name": record.last_name,
            "email": record.email, "phone_number": record.phone,
            "country_code": record.country_code,
        })

@register
class MoveConvert(_Api):
    name = "move_convert"
    def fire(self, record):
        # fire = create an intake via add_patient first is the runner's job;
        # this driver only CONVERTS the record created for the same ref.
        return self.call("POST", f"/api/intakes/{record.converted_id}/convert-to-patient/",
                         {"create_new": True})               # VERIFY path/contract

@register
class Automations(_Api):
    name = "automations"
    def fire(self, record):
        return self.call("POST", "/api/automations/test-run/", {  # VERIFY: action entry
            "action": "create_intake", "payload": record.__dict__,
        })

@register
class DqImport(_Api):
    name = "dq_import"
    def fire(self, record):
        return self.call("POST", "/api/data-quality/force-import/", record.__dict__)  # VERIFY

@register
class MergeUnweld(_Api):
    name = "merge_unweld"
    def fire(self, record):
        return self.call("POST", "/api/contacts/merge/", record.__dict__)             # VERIFY
@register
class RubbishPhone(_Api):
    """A rubbish phone delivered through a real lane. Success = the lane does
    NOT 500 and does not create a junk duplicate (checked by expect blocks)."""
    name = "rubbish_phone"
    VIA = ["legacy_webhook", "add_patient", "form_handoff"]
    def fire(self, record):
        lane = REGISTRY[self.VIA[hash(record.ref) % len(self.VIA)]]
        out = lane(self.cfg, self.token).fire(record)
        if isinstance(out.get("status"), int) and out["status"] >= 500:
            out["rubbish_violation"] = "lane 500'd on rubbish input"
        return out
```

- [ ] **Step 4: Registry test**

```python
# tests/test_drivers_registry.py
def test_v1_drivers_registered():
    from engine.drivers import REGISTRY
    for name in ["legacy_webhook", "custom_webhook", "form_handoff", "message_session",
                 "add_patient", "patients_crud", "family_member", "move_convert",
                 "automations", "dq_import", "merge_unweld"]:
        assert name in REGISTRY, name
```
Add `engine/drivers/__init__.py` that imports the modules so registration side-effects run:
```python
from . import webhooks, api  # noqa: F401
```
- [ ] **Step 5: Run tests — PASS. Commit** — `git commit -am "feat: driver framework + HTTP/API drivers"`

---

### Task 9: Shell-shim drivers (no clean HTTP trigger)

**Files:**
- Create: `engine/drivers/shim.py`, `shim/ingress_shim.py` (lives in the **Django** repo, committed to `TreatmentPathBackend/TreatmentPath/TreatmentPlan/management/` — see Step 1)

- [ ] **Step 1: Write the Django-side shim command** at
  `TreatmentPathBackend/TreatmentPath/TreatmentPlan/management/commands/qa_ingress_shim.py`:

```python
"""QA Ingress Engine shim: runs one ingest function inside Django against the
QA practice. Reads a JSON payload on argv, prints JSON result. QA-only — do
not use outside the ingress-test-engine."""
import json, sys
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = "QA harness shim (see docs/superpowers/specs/2026-09-03-ingress-test-engine-design.md)"

    def add_arguments(self, parser):
        parser.add_argument("--lane", required=True)
        parser.add_argument("--payload", required=True)
        parser.add_argument("--practice-slug", required=True)

    def handle(self, *args, **opts):
        from UserAuthentication.models import Practice
        payload = json.loads(opts["payload"])
        practice = Practice.objects.get(slug=opts["practice_slug"])
        out = {}
        if opts["lane"] == "ai_email":
            from TreatmentPlan.services.email_intake import create_intake_from_email
            intake = create_intake_from_email(
                practice=practice,
                parsed_data={"first_name": payload["first_name"], "last_name": payload["last_name"],
                             "email": payload["email"], "phone": payload["phone"], "source": "qa"},
                sender_email=payload["email"],
            )
            out = {"id": intake.id}
        elif opts["lane"] == "csv_import":
            from TreatmentPlan.views.patient_views import _process_csv_row  # VERIFY symbol
            row = _process_csv_row(practice, payload)        # VERIFY signature
            out = {"id": getattr(row, "id", None)}
        elif opts["lane"] == "booking":
            from TreatmentPlan.tasks import create_booking_intake  # VERIFY symbol
            obj = create_booking_intake(practice.id, payload)      # VERIFY signature
            out = {"id": getattr(obj, "id", None)}
        elif opts["lane"] == "archive_restore":
            from TreatmentPlan.models import Intake
            src = Intake.objects.get(pk=payload["source_id"], practice=practice)
            restored = src.restore()                                # VERIFY restore API
            out = {"id": restored.id}
        elif opts["lane"] == "admin_edit":
            from TreatmentPlan.models import Intake
            obj = Intake.objects.get(pk=payload["id"], practice=practice)
            for k, v in payload["set"].items():
                setattr(obj, k, v)
            obj.save()          # fires signals — this is what we are testing
            out = {"id": obj.id}
        elif opts["lane"] == "backfill":
            out = {"ok": True, "note": "backfills are invoked directly by the runner"}
        print(json.dumps(out))
```

`VERIFY` each imported symbol against the actual backend code before finishing this task; where a symbol doesn't exist, replace the lane body with the smallest real call that exercises the same ingest function.

- [ ] **Step 2: `engine/drivers/shim.py`**

```python
import json, subprocess
from .base import BaseDriver, register, DriverError

@register
class ShimDriver(BaseDriver):
    name = "ai_email"          # subclasses override
    lane = "ai_email"

    def fire(self, record):
        cmd = [self.cfg.backend_venv_python, "manage.py", "qa_ingress_shim",
               "--lane", self.lane, "--practice-slug", self.cfg.qa_practice_slug,
               "--payload", json.dumps(record.__dict__)]
        r = subprocess.run(cmd, cwd=self.cfg.backend_dir, capture_output=True, text=True, timeout=120)
        if r.returncode != 0:
            raise DriverError(r.stderr[-500:])
        return {"status": "shim", "response": r.stdout.strip()[-500:]}

for lane in ["csv_import", "booking", "archive_restore", "admin_edit", "backfill"]:
    globals()[lane] = type(lane, (ShimDriver,), {"name": lane, "lane": lane})
    # registration happens via register() loop below
from .base import register as _reg
for _cls in [globals()[l] for l in ["csv_import", "booking", "archive_restore", "admin_edit", "backfill"]]:
    _reg(_cls)
```

- [ ] **Step 3: Update registry test with the shim names; run — PASS**
- [ ] **Step 4: Commit both repos** — engine: `git commit -am "feat: shell-shim drivers"`; Django repo (its own git): commit `qa_ingress_shim.py`.

---

### Task 10: Runner

**Files:**
- Create: `engine/runner.py`, `tests/test_runner_order.py`

- [ ] **Step 1: Implement**

```python
import time
from concurrent.futures import ThreadPoolExecutor
from .drivers import REGISTRY
from .drivers.base import BaseDriver, DriverError
from .db import snapshot, resolve_person
from .scorer import classify

class RunResult:
    def __init__(self):
        self.deliveries = []      # per-record fire outcomes
        self.classification = {}
        self.expect_failures = []
        self.snapshot = {}

def run_scenario(scenario, cfg, token, settle_seconds=6.0) -> RunResult:
    res = RunResult()
    driver_for = {}
    for rec in scenario.records:
        d = REGISTRY.get(rec.entry_point)
        if not d:
            raise DriverError(f"no driver for entry_point={rec.entry_point}")
        driver_for[rec.ref] = d(cfg, token)
    ordered = list(scenario.records)
    if scenario.order == "reverse":
        ordered.reverse()
    if scenario.order == "concurrent":
        with ThreadPoolExecutor(max_workers=len(ordered)) as ex:
            outs = list(ex.map(lambda r: _fire(driver_for[r.ref], r), ordered))
    else:
        outs = [_fire(driver_for[r.ref], r) for r in ordered]
    res.deliveries = outs
    time.sleep(settle_seconds)          # let Celery/signals settle; poll loop is a V2 nicety
    marker = f"qa+%-{cfg.run_marker}%"  # runner passes the scenario run-id prefix instead
    res.snapshot = snapshot(cfg.dev_dsn, cfg.qa_practice_slug, marker)
    actual = _actual_grouping(scenario, res.snapshot)
    res.classification = classify(scenario.truth.persons, actual)
    _check_expect(scenario, res)
    return res

def _fire(driver, record):
    try:
        return {"ref": record.ref, **driver.fire(record)}
    except DriverError as e:
        return {"ref": record.ref, "status": "error", "response": str(e)}

def _actual_grouping(scenario, snap):
    """ref -> resolved person id (or None). Matches rows by email, whitespace/case-tolerant."""
    norm = lambda s: (s or "").strip().lower()
    by_email = {}
    for row in snap["records"]:
        by_email.setdefault(norm(row["email"]), row["person_id"])
    actual = {}
    for rec in scenario.records:
        pid = by_email.get(norm(rec.email))
        actual[rec.ref] = resolve_person(snap, pid)
    return actual

def _check_expect(scenario, res):
    # exact-value assertions on top of the pair classification
    if "person_count" in scenario.expect:
        groups = {p for p in _actual_grouping(scenario, res.snapshot).values() if p}
        if len(groups) != scenario.expect["person_count"]:
            res.expect_failures.append(
                f"person_count: want {scenario.expect['person_count']}, got {len(groups)}")
```

- [ ] **Step 2: Order-mode test (pure):** extract `ordered` logic into `_order(records, mode)` and test all three modes.
- [ ] **Step 3: Live smoke:** run a 1-record `add_patient` scenario against DEV; confirm `res.classification["accuracy"] == 1.0` and the record landed in the QA practice.
- [ ] **Step 4: Commit** — `git commit -am "feat: scenario runner with order modes + settle"`

---

### Task 11: HTML report

**Files:**
- Create: `engine/report.py`

- [ ] **Step 1: Implement** — render one self-contained HTML file per run: header with run timestamp + totals (precision/recall/accuracy as big numbers), per-scenario cards with the verdict-pair table (verdict classes colored: green correct, red missed/wrong merge), and the deliveries table. Use the same visual language as `docs/ingest-web-diagram.html` (inline CSS, no JS framework needed). Keep a `reports/index.html` list linking all runs.
- [ ] **Step 2: Smoke:** generate a report from a live run, open `reports/run-*.html` in the browser at `http://localhost:8088` (the existing static server) or `python -m http.server`.
- [ ] **Step 3: Commit** — `git commit -am "feat: html scorecard reports"`

---

### Task 12: CLI (`setup`, `run`, `canary`)

**Files:**
- Create: extend `engine/cli.py`; Create: `engine/setup.py`

- [ ] **Step 1: `engine/setup.py`** — via the shim mechanism, create (idempotently) the QA practice + QA staff user + `UserPracticeRelationship`, print ids to paste into `.env`. Implement as a `qa_ingress_setup` management command in the Django repo (same file pattern as Task 9 Step 1) — the engine CLI just invokes it and parses the JSON.
- [ ] **Step 2: CLI commands**

```python
@cli.command()
def setup():
    """Create/verify QA practice + user on DEV; print .env lines."""
    ...

@cli.command()
@click.argument("scenario_files", nargs=-1)
@click.option("--report/--no-report", default=True)
def run(scenario_files, report):
    """Run YAML scenarios (or shipped scenarios/*.yaml) against DEV."""
    ...

@cli.command()
def canary():
    """Run the canary mutation check: score must DROP under a broken matcher."""
```
- [ ] **Step 3: `ingress-engine setup` against DEV — prints ids; put them in `.env`.**
- [ ] **Step 4: `ingress-engine run scenarios/archetype-*.yaml` — full suite green on current code.**
- [ ] **Step 5: Commit both repos.**

---

### Task 13: Shipped scenario fixtures

**Files:**
- Create: `scenarios/archetype-1-unique.yaml` … `archetype-8-cross-lane.yaml`, `adversarial-a1-malformed.yaml`, `adversarial-a2-idempotency.yaml`, `adversarial-a3-concurrent.yaml`

- [ ] **Step 1:** Hand-write archetype scenarios 1–8 (small: 2–5 records each) using the format from Task 2; generate them via the generator where convenient and check the YAML in.
- [ ] **Step 2:** A1 malformed: records with `email: ""`, `phone: "not-a-phone"`, `phone: "X" * 10000`; `expect` block asserts HTTP status < 500 and `no_duplicate_person`.
- [ ] **Step 3:** A2: same record twice, `order: sequential`; A3: `order: concurrent` with two lanes for one person; both `expect: {person_count: 1}`.
- [ ] **Step 4:** `ingress-engine run scenarios/*.yaml` — record the baseline score in `reports/`. On the **current** backend, archetype 3/8 may legitimately expose real bugs — file findings; do not weaken expectations to make them pass.
- [ ] **Step 5: Commit** — `git commit -am "test: shipped archetype + adversarial scenarios"`

---

### Task 14: Web UI (compose / run / scorecard)

**Files:**
- Create: `engine/server.py`, `web/index.html`

- [ ] **Step 1: `engine/server.py`** (FastAPI):

```python
from fastapi import FastAPI
from fastapi.responses import FileResponse, JSONResponse
from pydantic import BaseModel
from .generator import generate
from .models import Scenario

app = FastAPI()

class ComposeParams(BaseModel):
    persons: int = 0
    families: int = 0
    family_size: int = 3
    duplicates: dict[str, int] = {}
    entry_points: dict[str, str] = {}
    order: str = "sequential"
    seed: int = 0
    regions: list[str] = ["GB"]
    invalid_phones: int = 0

@app.get("/")
def index():
    return FileResponse("web/index.html")

@app.post("/api/compose")
def compose(p: ComposeParams):
    s = generate(persons=p.persons, families=p.families, family_size=p.family_size,
                 duplicates=p.duplicates, entry_points=p.entry_points,
                 order=p.order, seed=p.seed, regions=p.regions,
                 invalid_phones=p.invalid_phones)
    return {"yaml": s.to_yaml(), "records": [r.__dict__ for r in s.records],
            "truth": s.truth.__dict__}

@app.post("/api/run")
def run(body: dict):
    from .runner import run_scenario
    from .config import Config
    from .auth import mint_access_token
    cfg = Config.load()
    scenario = Scenario.from_yaml(body["yaml"])
    token = mint_access_token(cfg.secret_key, cfg.qa_user_id, cfg.qa_practice_id, cfg.qa_practice_slug)
    res = run_scenario(scenario, cfg, token)
    return {"classification": {f"{a}|{b}": v for (a, b), v in res.classification["verdicts"].items()},
            "scores": {k: res.classification[k] for k in ("precision", "recall", "accuracy")},
            "expect_failures": res.expect_failures,
            "deliveries": res.deliveries}
```

- [ ] **Step 2: `web/index.html`** — three tabs (Compose / Run / Scorecard), vanilla JS `fetch` to the two endpoints, Compose has number inputs (persons, families, family size, rubbish-phone count), a regions text field (comma-separated country codes for libphonenumber generation), per-archetype duplicate counters, entry-point `<select>` per generated record row, seeded generate button, records preview table with truth-group coloring; Run tab posts the YAML and shows progress; Scorecard tab renders the four verdict classes with color + precision/recall/accuracy tiles. Same styling family as `docs/ingest-web-diagram.html`.
- [ ] **Step 3:** `uvicorn engine.server:app --port 8090` — compose a mix (2 families + 4 duplicates), run, verify scorecard renders against live DEV.
- [ ] **Step 4: Commit** — `git commit -am "feat: simulator web UI"`

---

### Task 15: Engine battle-tests (self-tests + canary + fuzzing + fault injection)

**Files:**
- Create: `tests/test_scorer_conformance.py`, `tests/test_property_based.py`, `tests/test_adversarial_inputs.py`, `tests/test_fault_injection.py`, `tests/test_canary.py`, `engine/canary.py`, `engine/validation.py`
- Modify: `pyproject.toml` (add `hypothesis>=6` to dev deps)

- [ ] **Step 1: Add hypothesis to dev deps**: `"dev": ["pytest>=8", "hypothesis>=6"]`, reinstall (`pip install -e ".[dev]"`).

- [ ] **Step 2: Scorer conformance (per archetype, pure python)**

```python
# tests/test_scorer_conformance.py
import pytest
from engine.generator import generate
from engine.archetypes import ARCHETYPES
from engine.scorer import classify

ALL = ["reformatted_phone", "padded_phone", "new_email_same_person", "name_variant",
       "email_noise", "merged_person_returns"]

@pytest.mark.parametrize("arch", ALL)
def test_perfect_system_scores_full(arch):
    s = generate(persons=3, duplicates={arch: 1}, seed=1)
    ids = {k: 100 + i for i, k in enumerate(set(s.truth.persons.values()))}
    v = classify(s.truth.persons, {r: ids[k] for r, k in s.truth.persons.items()})
    assert v["accuracy"] == 1.0

@pytest.mark.parametrize("arch", ALL)
def test_twins_system_misses_every_merge(arch):
    s = generate(persons=3, duplicates={arch: 1}, seed=1)
    v = classify(s.truth.persons, {r: 1000 + i for i, r in enumerate(sorted(s.truth.persons))})
    assert v["missed_merge"] > 0 and v["wrong_merge"] == 0
```

- [ ] **Step 3: Property-based fuzzing (invariants over random draws)**

```python
# tests/test_property_based.py
from hypothesis import given, settings
from hypothesis import strategies as st
from engine.generator import generate
from engine.scorer import classify

regions = st.sampled_from(["GB", "US", "FR", "NG", "DE", "AU", "IN", "BR"])
arch_names = st.sampled_from(["reformatted_phone", "padded_phone", "new_email_same_person",
                              "name_variant", "email_noise", "merged_person_returns"])
dup_map = st.dictionaries(arch_names, st.integers(0, 3), max_size=3)

@settings(max_examples=50, deadline=None)
@given(persons=st.integers(0, 20), families=st.integers(0, 4),
       family_size=st.integers(1, 8), duplicates=dup_map,
       seed=st.integers(0, 10**6), regions=st.lists(regions, min_size=1, max_size=3),
       invalid=st.integers(0, 4))
def test_generator_invariants(persons, families, family_size, duplicates, seed, regions, invalid):
    s = generate(persons=persons, families=families, family_size=family_size,
                 duplicates=duplicates, seed=seed, regions=regions, invalid_phones=invalid)
    # 1. determinism
    s2 = generate(persons=persons, families=families, family_size=family_size,
                  duplicates=duplicates, seed=seed, regions=regions, invalid_phones=invalid)
    assert s.to_yaml() == s2.to_yaml()
    # 2. truth covers every ref, no orphans
    assert {r.ref for r in s.records} == set(s.truth.persons)
    # 3. classification is total and accuracy is a sane fraction
    actual = {ref: hash(k) % 50 for ref, k in s.truth.persons.items()}  # arbitrary grouping
    v = classify(s.truth.persons, actual)
    assert 0.0 <= v["accuracy"] <= 1.0
    assert len(v["verdicts"]) == len(s.records) * (len(s.records) - 1) // 2
    # 4. identity mapping is always perfect
    ids = {k: 100 + i for i, k in enumerate(set(s.truth.persons.values()))}
    assert classify(s.truth.persons, {r: ids[k] for r, k in s.truth.persons.items()})["accuracy"] == 1.0
```

- [ ] **Step 4: Adversarial inputs to the engine — `engine/validation.py` + tests**

```python
# engine/validation.py
"""Strict scenario validation: the engine must REJECT broken fixtures loudly,
never silently run an empty or half-parsed scenario."""
class ScenarioInvalid(ValueError):
    pass

def validate(scenario) -> None:
    refs = [r.ref for r in scenario.records]
    if not refs:
        raise ScenarioInvalid("scenario has no records")
    if len(set(refs)) != len(refs):
        raise ScenarioInvalid("duplicate refs")
    unknown = set(scenario.truth.persons) - set(refs)
    if unknown:
        raise ScenarioInvalid(f"truth refs with no record: {sorted(unknown)}")
    missing = set(refs) - set(scenario.truth.persons)
    if missing:
        raise ScenarioInvalid(f"records with no truth entry: {sorted(missing)}")
    if scenario.order not in ("sequential", "reverse", "concurrent"):
        raise ScenarioInvalid(f"bad order: {scenario.order!r}")
```

```python
# tests/test_adversarial_inputs.py
import pytest, yaml
from engine.models import Scenario
from engine.validation import validate, ScenarioInvalid
from engine.scorer import classify

def _scen(**over):
    base = dict(scenario="t", order="sequential",
                records=[{"ref": "A", "person_key": "p1", "first_name": "A", "last_name": "B",
                          "email": "qa+x-a@qa.example.com", "phone": "", "country_code": "GB",
                          "entry_point": "legacy_webhook"}],
                truth={"persons": {"A": "p1"}, "shared_channels": [], "households": []},
                expect={})
    base.update(over)
    return Scenario.from_yaml(yaml.safe_dump(base))

def test_rejects_duplicate_refs():
    from dataclasses import asdict
    dup = asdict(_scen().records[0])
    s = _scen(records=[dup, dict(dup)])
    with pytest.raises(ScenarioInvalid, match="duplicate"):
        validate(s)

def test_rejects_truth_orphan():
    s = _scen(truth={"persons": {"A": "p1", "GHOST": "p9"}, "shared_channels": [], "households": []})
    with pytest.raises(ScenarioInvalid, match="GHOST"):
        validate(s)

def test_rejects_bad_order():
    s = _scen(order="random-ish")
    with pytest.raises(ScenarioInvalid, match="order"):
        validate(s)

def test_rejects_empty_scenario():
    with pytest.raises(Exception):
        Scenario.from_yaml(yaml.safe_dump({"scenario": "empty", "records": [],
                                           "truth": {"persons": {}, "shared_channels": [], "households": []}}))

def test_scorer_empty_scenario_convention():
    v = classify({}, {})
    assert v["accuracy"] == 1.0  # documented convention: nothing to get wrong
```

- [ ] **Step 5: Fault injection (runner, fake drivers — no DB needed)**

```python
# tests/test_fault_injection.py
from engine.runner import _fire
from engine.drivers.base import BaseDriver, DriverError

class ExplodingDriver(BaseDriver):
    name = "explode"
    def fire(self, record):
        raise DriverError("connection reset")

def test_driver_error_becomes_per_record_failure():
    class R: ref = "A"
    out = _fire(ExplodingDriver(None), R())
    assert out["status"] == "error" and "connection reset" in out["response"]

def test_run_never_swallows_all_errors(monkeypatch):
    # runner must mark the run failed if EVERY delivery errored
    from engine.runner import run_result_is_fatal
    deliveries = [{"ref": "A", "status": "error", "response": "x"},
                  {"ref": "B", "status": "error", "response": "y"}]
    assert run_result_is_fatal(deliveries)
    assert not run_result_is_fatal([{"ref": "A", "status": 200, "response": ""}])
```
Add the tiny `run_result_is_fatal(deliveries)` helper to `engine/runner.py` (returns True iff all deliveries errored) and have `run_scenario` mark such runs `failed=True` in the report.

- [ ] **Step 6: Canary (`engine/canary.py`)** — run archetype-2/3 scenarios twice against DEV — once normally, once with a "no-op identity" simulation where the scorer is handed the *pre-run* grouping (i.e., pretend nothing merged). Assert the second score < first. This validates the scorer's sensitivity without mutating the backend.
- [ ] **Step 7:** `python -m pytest -q` — all green, including ~50 hypothesis draws.
- [ ] **Step 8: Commit** — `git commit -am "test: battle battery — conformance, fuzzing, adversarial inputs, fault injection, canary"`

---

### Task 16: README + wiring into the workspace

**Files:**
- Create: `ingress-test-engine/README.md`, `ingress-test-engine/Makefile`
- Modify: root `AGENTS.md` (sub-project table + run command)

- [ ] **Step 1:** README: purpose, setup (`ingress-engine setup`), run (CLI + `make ui` → uvicorn), scenario authoring guide (fixture format + archetype reference table), how to add a driver, safety rules (QA practice only, read-only DB, never PROD).
- [ ] **Step 2:** Makefile: `make test`, `make ui`, `make run`, `make canary`.
- [ ] **Step 3:** Add one row to `AGENTS.md` §1 table: `ingress-test-engine | Python/FastAPI | identity regression harness (synthetic ingress + scoring)`.
- [ ] **Step 4:** Full suite + one live UI run as final acceptance. Commit.

---

## Self-review notes

- **Spec coverage:** phone factory for any region + rubbish modes (Task 3), archetypes 1–8 (Task 4, 13), adversarial A1–A3 (Task 13), all 18 V1 drivers (Tasks 8–9), scorer with precision/recall (Task 6), order modes + concurrency (Task 10), HTML report (Task 11), web UI compose/run/scorecard (Task 14), QA-practice setup + read-only DSN (Tasks 1, 7, 12), self-tests + canary (Task 15), README/AGENTS wiring (Task 16). V2 Go drivers intentionally absent.
- **Type consistency:** `Record.person_key` ↔ `Truth.persons[ref]`; driver `name` values match `Record.entry_point` values in generator lanes list and shipped YAML; `run_scenario(scenario, cfg, token)` signature consistent between Tasks 10 and 14.
- **Known VERIFY points** are marked inline (endpoint paths, CSV/booking/restore symbol names, simplejwt claim set, table names) — each is a bounded 5-minute lookup in the named file, not open-ended.

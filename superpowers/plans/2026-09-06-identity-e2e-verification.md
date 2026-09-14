# Patient-Identity E2E Verification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Drive the real application as a real user against a restored production
database and try to re-create every patient-identity defect fixed on 2026-09-06,
with all outbound messaging intercepted so nothing reaches a real patient.

**Architecture:** A single fail-closed "outbound sandbox" intercepts Twilio, SMTP,
Postmark and WhatsApp at the transport layer and records every attempted send to a
queryable outbox. Django runs against `prod_rehearsal2`; the Vite frontend and the Go
`email-service` point at the same database. Verification is manual browser driving via
Playwright MCP plus direct webhook POSTs for the surfaces that have no UI.

**Tech Stack:** Django 5.2 + Postgres 17, Go (EmailServiceGo, Gin), React/Vite
frontend, Playwright MCP for browser driving, Twilio/SendGrid/Postmark as the
intercepted egress points.

**Spec:** `docs/PATIENT_IDENTITY_AUDIT_2026-09-06.md` — the 44 findings and their
AS BUILT blocks are the acceptance criteria this plan verifies.

## Global Constraints

- **No message may reach a real human.** `prod_rehearsal2` holds real patient names,
  emails and mobile numbers. The sandbox is a prerequisite for booting, not a
  nice-to-have. The boot guard must refuse to start rather than risk a send.
- **Never `git add` / `commit` / `push`.** The user performs all VCS operations.
- **Never hardcode secrets.** Credentials come from env vars / `.env` files.
- **All person/patient/contact handling stays practice-scoped.** Never cross-practice.
- Backend tests run with `--keepdb`, never `--noinput`.
- Always `source TreatmentPathBackend/venv/bin/activate` first.
- **Disk is at 99% (5.1 GB free).** Do not copy the 8.6 GB database, do not produce
  large build artifacts, do not enable verbose SQL logging to disk (the Go service
  floods logs — see `project_prod_recon_gaps`).
- `prod_rehearsal2` is a **scratch copy**. Writing to it, running destructive
  migration `0168`, and resetting passwords on it are all expected and safe.
- Ports: Django `8000`, Vite `8080`, Go `email-service` `8081`.

---

## Environment facts established during reconnaissance

Do not re-derive these.

| Fact | Value |
|---|---|
| Rehearsal DB | `prod_rehearsal2`, 8,613 MB, already restored — **no restore needed** |
| DB creds | `DB_USER=mannie`, host `localhost:5432`, password in `TreatmentPathBackend/.env` |
| Login user | `manifestkelvin@gmail.com` / `maniZolas1008`, user id **66**, `user_type=admin`, `current_practice_id=16` (Danbury Dental Care) |
| Practices present | 9, 13, 16, 19, 20, 21, 22, 23, 24, 26, 27, 28, 29, 30 |
| Pending migrations on rehearsal2 | dentallyIntegration 0166→0169, TreatmentPlan 0145, medicalHistory 0005, marketingBroadcast 0038 |
| Backend branch | `dedup-normalization-unification` |
| Frontend | `perfect-pixel-playground-project`, branch `mannieJuly`, Vite, `node_modules` present (778 MB) |
| Frontend API base | `VITE_BASE_URL` / `VITE_API_BASE_URL` in `.env`, read by `src/config/environment.ts:60-63` |
| Twilio client constructions | 8 sites, all lazy `from twilio.rest import Client` — **no shared choke point** |
| Twilio send call sites | `Tasks/tasks.py:294`, `TreatmentPlan/journey/dispatch.py:224`, `dentallyIntegration/recall_automation.py:654`, `dentallyIntegration/confirmation_automation.py:321`, `dentallyIntegration/views/recall_views.py:1900`, `automations/actions.py:1809` |
| Email backend | `settings.py:387` SMTP → `smtp.sendgrid.net` |
| Marketing email | `marketingBroadcast/marketing_email_client.py` (Postmark) |
| Call-agent webhooks (Go) | `POST /call-agent/post-call-webhook`, `POST /call-agent/tools/lookup_patient`, `POST /call-agent/incoming-call`, `POST /call-agent/call-status` |

---

### Task 1: Outbound sandbox — the safety gate

Nothing else in this plan may run until this task's Step 7 passes. Every later task
depends on it.

**Files:**
- Create: `TreatmentPathBackend/TreatmentPath/outbound_sandbox/__init__.py`
- Create: `TreatmentPathBackend/TreatmentPath/outbound_sandbox/apps.py`
- Create: `TreatmentPathBackend/TreatmentPath/outbound_sandbox/outbox.py`
- Create: `TreatmentPathBackend/TreatmentPath/outbound_sandbox/interceptors.py`
- Create: `TreatmentPathBackend/TreatmentPath/outbound_sandbox/email_backend.py`
- Create: `TreatmentPathBackend/TreatmentPath/outbound_sandbox/views.py`
- Create: `TreatmentPathBackend/TreatmentPath/outbound_sandbox/urls.py`
- Create: `TreatmentPathBackend/TreatmentPath/outbound_sandbox/tests.py`
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPath/settings.py` (append sandbox block)
- Modify: `TreatmentPathBackend/TreatmentPath/TreatmentPath/urls.py` (mount sandbox urls)

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `outbound_sandbox.outbox.record(channel: str, to: str, body: str, meta: dict) -> dict`
    — appends one JSON object to `OUTBOX_PATH` and returns it.
  - `outbound_sandbox.outbox.read_all() -> list[dict]`
  - `outbound_sandbox.outbox.clear() -> None`
  - `outbound_sandbox.interceptors.install() -> list[str]` — returns the names of the
    channels successfully patched; raises `RuntimeError` if any expected channel
    could not be patched.
  - HTTP `GET /api/backend/_sandbox/outbox/` → `{"count": int, "messages": [...]}`
  - HTTP `DELETE /api/backend/_sandbox/outbox/` → `{"cleared": true}`
  - Each outbox record has keys: `ts`, `channel` (`"sms"|"email"|"whatsapp"|"postmark"`),
    `to`, `body`, `meta`, `sid`.

- [ ] **Step 1: Write the failing test**

Create `TreatmentPathBackend/TreatmentPath/outbound_sandbox/tests.py`:

```python
"""The outbound sandbox must intercept EVERY egress path, not just the tidy ones.

These tests are the only thing standing between a restored production database
and a real SMS to a real patient, so they assert on the transport layer rather
than on any individual call site.
"""

from django.test import TestCase, override_settings

from outbound_sandbox import outbox
from outbound_sandbox.interceptors import install


class TwilioInterceptionTests(TestCase):
    def setUp(self):
        outbox.clear()
        install()

    def test_a_twilio_send_is_recorded_and_never_leaves_the_process(self):
        """A Client built the way production builds it must not reach the network."""
        from twilio.rest import Client

        client = Client("ACtest", "sandboxtoken")
        message = client.messages.create(
            to="+447399674001", from_="+447700000000", body="hello"
        )

        self.assertTrue(message.sid.startswith("SM"))
        records = outbox.read_all()
        self.assertEqual(len(records), 1)
        self.assertEqual(records[0]["channel"], "sms")
        self.assertEqual(records[0]["to"], "+447399674001")
        self.assertEqual(records[0]["body"], "hello")

    def test_the_returned_object_looks_like_a_real_send_to_calling_code(self):
        """Call sites read .sid and .status; a send must still 'look sent'."""
        from twilio.rest import Client

        message = Client("ACtest", "tok").messages.create(
            to="+447399674002", from_="+447700000000", body="x"
        )
        self.assertEqual(message.status, "queued")
        self.assertIsNotNone(message.sid)


class EmailInterceptionTests(TestCase):
    def setUp(self):
        outbox.clear()

    @override_settings(
        EMAIL_BACKEND="outbound_sandbox.email_backend.OutboxEmailBackend"
    )
    def test_django_mail_is_recorded(self):
        from django.core.mail import send_mail

        send_mail("subj", "body", "from@x.com", ["patient@example.com"])

        records = [r for r in outbox.read_all() if r["channel"] == "email"]
        self.assertEqual(len(records), 1)
        self.assertEqual(records[0]["to"], "patient@example.com")
        self.assertEqual(records[0]["meta"]["subject"], "subj")
```

- [ ] **Step 2: Run the tests to verify they fail**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test outbound_sandbox --keepdb -v2
```

Expected: FAIL — `ModuleNotFoundError: No module named 'outbound_sandbox'`.

- [ ] **Step 3: Write the outbox store**

`outbound_sandbox/outbox.py`:

```python
"""Append-only record of every message the app TRIED to send.

Deliberately a flat JSONL file, not a model: it must work before migrations run,
must survive a a failed request, and must be readable by curl and by tests without
touching the restored production database.
"""

import json
import os
import threading
import time
import uuid
from pathlib import Path

from django.conf import settings

_LOCK = threading.Lock()


def _path() -> Path:
    return Path(
        getattr(settings, "OUTBOX_PATH", None)
        or os.environ.get("OUTBOX_PATH")
        or "/tmp/treatmentpath-outbox.jsonl"
    )


def record(channel: str, to: str, body: str, meta: dict | None = None) -> dict:
    entry = {
        "ts": time.time(),
        "channel": channel,
        "to": to,
        "body": body,
        "meta": meta or {},
        "sid": f"SM{uuid.uuid4().hex}",
    }
    with _LOCK:
        with _path().open("a", encoding="utf-8") as handle:
            handle.write(json.dumps(entry) + "\n")
    return entry


def read_all() -> list[dict]:
    path = _path()
    if not path.exists():
        return []
    with path.open(encoding="utf-8") as handle:
        return [json.loads(line) for line in handle if line.strip()]


def clear() -> None:
    with _LOCK:
        _path().write_text("", encoding="utf-8")
```

- [ ] **Step 4: Write the interceptors**

`outbound_sandbox/interceptors.py`. The Twilio patch targets the HTTP transport, not
`Client`, because all 8 construction sites do a lazy `from twilio.rest import Client`
and hold their own reference to the class — patching the class would miss them.
Every `Client()` built without an explicit `http_client` routes through
`TwilioHttpClient.request`.

```python
"""Transport-level interception of every outbound channel.

Patch point rationale: production builds Twilio clients in 8 places with a lazy
`from twilio.rest import Client`, so patching `Client` itself would miss any site
that had already imported it. `TwilioHttpClient.request` is the single funnel every
client instance passes through, so one patch there is airtight regardless of how or
where the client was constructed.
"""

import json
import logging

from outbound_sandbox import outbox

logger = logging.getLogger(__name__)

_INSTALLED = False


def _fake_twilio_response(kwargs):
    """Return the JSON body Twilio would have returned for a successful create."""
    from twilio.http.response import Response

    data = kwargs.get("data") or {}
    to = data.get("To") or data.get("to") or ""
    body = data.get("Body") or data.get("body") or ""

    entry = outbox.record(
        "whatsapp" if str(to).startswith("whatsapp:") else "sms",
        str(to),
        str(body),
        {"from": data.get("From") or data.get("from"), "url": kwargs.get("uri")},
    )
    payload = {
        "sid": entry["sid"],
        "status": "queued",
        "to": to,
        "from": data.get("From"),
        "body": body,
        "num_segments": "1",
        "error_code": None,
        "error_message": None,
        "date_created": None,
        "date_updated": None,
        "date_sent": None,
        "account_sid": "ACsandbox",
        "uri": "/2010-04-01/Messages/%s.json" % entry["sid"],
        "subresource_uris": {},
    }
    return Response(201, json.dumps(payload))


def _install_twilio() -> bool:
    from twilio.http.http_client import TwilioHttpClient

    def sandboxed_request(self, method, uri, **kwargs):
        logger.warning("OUTBOUND SANDBOX intercepted Twilio %s %s", method, uri)
        return _fake_twilio_response({"uri": uri, **kwargs})

    TwilioHttpClient.request = sandboxed_request
    return True


def _install_requests_guard() -> bool:
    """Backstop: block direct HTTP to known messaging vendors.

    Postmark and any hand-rolled vendor call go through `requests`. Rather than
    chase each client, refuse the request at the adapter and record it.
    """
    import requests

    blocked = ("api.postmarkapp.com", "api.sendgrid.com", "api.twilio.com")
    original = requests.adapters.HTTPAdapter.send

    def guarded_send(self, request, **kwargs):
        if any(host in request.url for host in blocked):
            payload = {}
            try:
                payload = json.loads(request.body or "{}")
            except (TypeError, ValueError):
                payload = {"raw": str(request.body)[:500]}
            outbox.record(
                "postmark",
                str(payload.get("To") or payload.get("to") or ""),
                str(payload.get("HtmlBody") or payload.get("TextBody") or "")[:2000],
                {"url": request.url, "subject": payload.get("Subject")},
            )
            from requests.models import Response as RequestsResponse

            response = RequestsResponse()
            response.status_code = 200
            response._content = json.dumps(
                {"ErrorCode": 0, "Message": "OK", "MessageID": "sandbox"}
            ).encode()
            response.url = request.url
            response.request = request
            return response
        return original(self, request, **kwargs)

    requests.adapters.HTTPAdapter.send = guarded_send
    return True


def install() -> list[str]:
    global _INSTALLED
    if _INSTALLED:
        return ["already-installed"]

    installed = []
    for name, fn in (("twilio", _install_twilio), ("requests", _install_requests_guard)):
        if fn():
            installed.append(name)
        else:
            raise RuntimeError(f"outbound sandbox could not patch {name}")

    _INSTALLED = True
    logger.warning("OUTBOUND SANDBOX ACTIVE: %s", ", ".join(installed))
    return installed
```

- [ ] **Step 5: Write the email backend and the app config**

`outbound_sandbox/email_backend.py`:

```python
from django.core.mail.backends.base import BaseEmailBackend

from outbound_sandbox import outbox


class OutboxEmailBackend(BaseEmailBackend):
    """Records mail instead of sending it, one outbox entry per recipient."""

    def send_messages(self, email_messages):
        sent = 0
        for message in email_messages:
            for recipient in message.to:
                outbox.record(
                    "email",
                    recipient,
                    message.body,
                    {
                        "subject": message.subject,
                        "from": message.from_email,
                        "cc": list(message.cc or []),
                        "bcc": list(message.bcc or []),
                    },
                )
            sent += 1
        return sent
```

`outbound_sandbox/apps.py`:

```python
import os

from django.apps import AppConfig


class OutboundSandboxConfig(AppConfig):
    name = "outbound_sandbox"

    def ready(self):
        if os.environ.get("OUTBOUND_SANDBOX") == "1":
            from outbound_sandbox.interceptors import install

            install()
```

`outbound_sandbox/__init__.py`:

```python
default_app_config = "outbound_sandbox.apps.OutboundSandboxConfig"
```

- [ ] **Step 6: Write the read endpoint**

`outbound_sandbox/views.py`:

```python
from django.http import JsonResponse
from django.views.decorators.csrf import csrf_exempt

from outbound_sandbox import outbox


@csrf_exempt
def outbox_view(request):
    """Read/clear the sandbox outbox. Only routed when OUTBOUND_SANDBOX=1."""
    if request.method == "DELETE":
        outbox.clear()
        return JsonResponse({"cleared": True})

    messages = outbox.read_all()
    channel = request.GET.get("channel")
    if channel:
        messages = [m for m in messages if m["channel"] == channel]
    return JsonResponse({"count": len(messages), "messages": messages})
```

`outbound_sandbox/urls.py`:

```python
from django.urls import path

from outbound_sandbox.views import outbox_view

urlpatterns = [path("outbox/", outbox_view, name="sandbox-outbox")]
```

Append to `TreatmentPath/settings.py`:

```python
# --- Outbound sandbox -------------------------------------------------------
# Fails CLOSED: booting against a restored production database with live senders
# is the one mistake that cannot be undone, so refuse to start instead.
OUTBOUND_SANDBOX = os.environ.get("OUTBOUND_SANDBOX") == "1"
OUTBOX_PATH = os.environ.get("OUTBOX_PATH", "/tmp/treatmentpath-outbox.jsonl")

_db_name = DATABASES["default"]["NAME"]
_is_restored_prod = any(
    token in _db_name for token in ("prod_", "rehearsal", "pristine", "control")
)
if _is_restored_prod and not OUTBOUND_SANDBOX:
    raise RuntimeError(
        f"Refusing to start: DB '{_db_name}' looks like restored production data "
        "but OUTBOUND_SANDBOX is not set to 1. Real patients would receive real "
        "messages. Set OUTBOUND_SANDBOX=1."
    )

if OUTBOUND_SANDBOX:
    INSTALLED_APPS = INSTALLED_APPS + ["outbound_sandbox"]
    EMAIL_BACKEND = "outbound_sandbox.email_backend.OutboxEmailBackend"
    TWILIO_ACCOUNT_SID = "ACsandbox0000000000000000000000000"
    TWILIO_AUTH_TOKEN = "sandboxtoken"
```

Mount in `TreatmentPath/urls.py`, inside the existing `urlpatterns` list:

```python
if getattr(settings, "OUTBOUND_SANDBOX", False):
    urlpatterns += [path("api/backend/_sandbox/", include("outbound_sandbox.urls"))]
```

- [ ] **Step 7: Run the tests to verify they pass, then prove the guard fails closed**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test outbound_sandbox --keepdb -v2
```
Expected: PASS, 4 tests.

Then prove the boot guard actually refuses (this is the assertion that matters most):

```bash
DB_NAME=prod_rehearsal2 python manage.py check 2>&1 | tail -3
```
Expected: `RuntimeError: Refusing to start: DB 'prod_rehearsal2' looks like restored
production data but OUTBOUND_SANDBOX is not set to 1.`

```bash
DB_NAME=prod_rehearsal2 OUTBOUND_SANDBOX=1 python manage.py check 2>&1 | tail -3
```
Expected: `System check identified no issues`.

- [ ] **Step 8: Prove zero network egress empirically**

Config review is not proof. Capture packets while forcing a send.

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
sudo timeout 20 tcpdump -nn -c 5 'port 443 and (host api.twilio.com or host smtp.sendgrid.net)' \
  > /tmp/egress.txt 2>&1 &
OUTBOUND_SANDBOX=1 python -c "
import django, os
os.environ.setdefault('DJANGO_SETTINGS_MODULE','TreatmentPath.settings')
django.setup()
from twilio.rest import Client
m = Client('ACx','tok').messages.create(to='+447399674001', from_='+447700000000', body='egress probe')
print('returned sid:', m.sid, 'status:', m.status)
"
wait
grep -c . /tmp/egress.txt
```
Expected: the send returns a `SM…` sid, and `/tmp/egress.txt` captures **0** packets
to Twilio/SendGrid. If tcpdump needs a password that cannot be supplied
non-interactively, substitute `strace -f -e trace=connect` or skip with an explicit
written note — but do not silently skip.

---

### Task 2: Boot the sandbox environment against rehearsal2

**Files:**
- Create: `TreatmentPathBackend/.env.sandbox`
- Create: `perfect-pixel-playground-project/.env.local`
- Modify: none

**Interfaces:**
- Consumes: Task 1's `OUTBOUND_SANDBOX` guard and `_sandbox/outbox/` endpoint.
- Produces: Django on `http://127.0.0.1:8000`, Vite on `http://127.0.0.1:8080`,
  both against `prod_rehearsal2`; login `manifestkelvin@gmail.com` / `maniZolas1008`.

- [ ] **Step 1: Write the sandbox env file**

`TreatmentPathBackend/.env.sandbox` — note this file carries no secrets of its own;
it reuses the DB password already in `.env` via the shell, and pins deliberately
invalid vendor credentials so a leak fails closed.

```bash
DB_NAME=prod_rehearsal2
OUTBOUND_SANDBOX=1
OUTBOX_PATH=/tmp/treatmentpath-outbox.jsonl
DEBUG=True
TWILIO_ACCOUNT_SID=ACsandbox0000000000000000000000000
TWILIO_AUTH_TOKEN=sandboxtoken
TWILIO_PHONE_NUMBER=+447700000000
```

- [ ] **Step 2: Apply the pending migrations to rehearsal2**

This applies dentallyIntegration 0166-0169 (0168 is **destructive** — it deletes 146
unattributable AI-summary rows and 199 orphans), TreatmentPlan 0145, medicalHistory
0005, marketingBroadcast 0038. On a scratch copy this is intended, and it doubles as
the rehearsal for the real deploy.

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
set -a; . ../.env; . ../.env.sandbox; set +a
python manage.py showmigrations dentallyIntegration TreatmentPlan medicalHistory marketingBroadcast | grep -E '\[ \]'
python manage.py migrate 2>&1 | tail -20
```
Expected: the four apps' pending migrations apply cleanly; record the row counts
0168 reports.

- [ ] **Step 3: Set a known password for the login user on the scratch copy**

```bash
python manage.py shell -c "
from django.contrib.auth import get_user_model
U = get_user_model()
u = U.objects.get(email='manifestkelvin@gmail.com')
u.set_password('maniZolas1008'); u.is_active = True; u.save()
print('ok', u.id, u.current_practice_id)
"
```
Expected: `ok 66 16`.

- [ ] **Step 4: Start Django and verify the sandbox endpoint answers**

```bash
python manage.py runserver 127.0.0.1:8000
# in another shell:
curl -s http://127.0.0.1:8000/api/backend/_sandbox/outbox/
```
Expected: `{"count": 0, "messages": []}`.

- [ ] **Step 5: Point the frontend at local Django and start it**

`perfect-pixel-playground-project/.env.local`:

```bash
VITE_BASE_URL=http://127.0.0.1:8000
VITE_API_BASE_URL=http://127.0.0.1:8000/api/backend
VITE_API_URL=http://127.0.0.1:8000/api/backend
VITE_WEBHOOK_BASE_URL=http://127.0.0.1:8000/api/backend/messaging
```

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/perfect-pixel-playground-project
npm run dev -- --port 8080
```
Expected: Vite serves on 8080.

- [ ] **Step 6: Log in through the browser and confirm the practice**

Use Playwright MCP: navigate to `http://127.0.0.1:8080`, log in as
`manifestkelvin@gmail.com` / `maniZolas1008`, screenshot the landed dashboard.
Expected: authenticated, practice context = Danbury Dental Care (16).

---

### Task 3: SMS recipient-identity matrix — the core of the user's concern

For each panel: send a message, then read the outbox and assert the number it went to
resolves to **exactly one** Person, and that that Person is the one the UI displayed.
This is the shared-channel-attribution rule (`project_shared_channel_attribution`) and
covers findings #7, #10, #11, #13, #29, #37.

**Files:**
- Create: `docs/superpowers/plans/2026-09-06-identity-e2e-results.md` (running log)

**Interfaces:**
- Consumes: Task 2's running stack, Task 1's outbox endpoint.
- Produces: one results table row per panel with the outbox record and the
  Person-resolution verdict.

- [ ] **Step 1: Pick the hardest targets, not the easy ones**

Query for patients that share a phone with a relative — these are exactly the rows
that used to mis-address. Run against rehearsal2:

```sql
SELECT cc.canonical_value, count(DISTINCT pc.person_id) AS people,
       string_agg(DISTINCT p.first_name || ' ' || p.last_name, ' | ') AS names,
       cc.practice_id
FROM "TreatmentPlan_contactchannel" cc
JOIN "TreatmentPlan_personchannel" pc ON pc.channel_id = cc.id
JOIN "TreatmentPlan_person" p ON p.id = pc.person_id
WHERE cc.kind = 'phone'
GROUP BY cc.canonical_value, cc.practice_id
HAVING count(DISTINCT pc.person_id) > 1
ORDER BY people DESC
LIMIT 20;
```
Record the top shared-phone families; these are the test subjects for every panel.

- [ ] **Step 2: Send from each panel and capture the outbox**

Before each send: `curl -X DELETE http://127.0.0.1:8000/api/backend/_sandbox/outbox/`.
After each send: `curl -s http://127.0.0.1:8000/api/backend/_sandbox/outbox/ | python -m json.tool`.

Panels to exercise, each against a shared-phone family member:

| # | Panel | Route to reach it |
|---|---|---|
| 1 | Intake journey stage | Journeys → Intake → open card → Message |
| 2 | Nurture journey stage | Journeys → Nurture → open card → Message |
| 3 | Treatment Plan stage | Journeys → Treatment Plan → open card → Message |
| 4 | Active/custom stage | Journeys → custom stage → open card → Message |
| 5 | Contacts / patient workspace | Contacts → patient → Message |
| 6 | Day List | Day List → appointment row → Message/Confirm |
| 7 | Recall | Recalls → row → Send recall SMS |
| 8 | Recall Dormant tab | Recalls → Dormant → row → Send (this is #8's lane) |
| 9 | Confirmations | Day List → Confirmations → send |
| 10 | Tasks | Task with SMS action (`Tasks/tasks.py:294`) |
| 11 | Consent / Documents | Documents → request signature (`Documents/utils/notifications.py`) |
| 12 | Marketing broadcast test-send | Marketing → campaign → test send |

- [ ] **Step 3: Assert recipient identity for every captured send**

For each outbox record, resolve the destination:

```sql
SELECT p.id, p.first_name, p.last_name, cc.practice_id
FROM "TreatmentPlan_contactchannel" cc
JOIN "TreatmentPlan_personchannel" pc ON pc.channel_id = cc.id
JOIN "TreatmentPlan_person" p ON p.id = pc.person_id
WHERE cc.canonical_value = :to_number;
```

PASS = the number resolves to exactly one Person **and** that Person is the one the
UI named. FAIL = it resolves to several and the app picked one anyway, or the
message body addresses a different human than the row it was sent from.

- [ ] **Step 4: Record every result in the results doc**

Append a row per panel: panel, target patient, `to` number, resolved Person(s),
verdict, screenshot path. Include the failures verbatim — a panel that could not be
reached is recorded as NOT TESTED, never as PASS.

---

### Task 4: Journey rename + cross-stage move

Covers #12 (contact stripped on update), #27 (read-only `patient_name` PATCH
no-ops), #17/#25/#35/#40 (activity log attribution), and the archive/restore id-churn
trap from `project_archive_no_longer_deletes`.

**Files:**
- Modify: `docs/superpowers/plans/2026-09-06-identity-e2e-results.md`

**Interfaces:**
- Consumes: Task 2's stack.
- Produces: verdicts for #12, #27, #17, #25, #35, #40.

- [ ] **Step 1: Rename an intake and check the log names the right human**

Open an Intake belonging to a shared-email family. Rename it. Then open Activity
History and assert the entry names the renamed patient, not a relative — and that no
relative's timeline gained the entry (#17, #35).

- [ ] **Step 2: Attempt the read-only echo (#27)**

PATCH the intake sending back `patient_name` exactly as the API returned it.
Expected: **not** a silent no-op — either the field is accepted or the API rejects
it explicitly. Capture the request/response from the browser network panel.

- [ ] **Step 3: Try to strip all contact details (#12)**

Edit the intake, clear both email and phone, save.
Expected: rejected, because the request itself touches the contact fields. Then
confirm the narrower rule holds: PATCH an unrelated field on an already-contactless
row and expect success (this is the regression that my first fix broke).

- [ ] **Step 4: Move the intake across stages**

Intake → Nurture → Treatment Plan → Active → back. After each move assert:
the patient id is unchanged, `person_id` is still set, notes survive, and the
activity log entries stayed on the same human.

- [ ] **Step 5: Archive and restore**

Archive the record, confirm it is not deleted, restore it, confirm the id did not
churn and the notes are still attached (`project_contact_identity_rehearsal` flagged
orphaned notes from id churn).

---

### Task 5: Call-agent webhooks — the surfaces with no UI

Covers #38 (first-human-on-a-shared-phone), #39 (DOB accepted and ignored),
#41 (Intakes inserted outside the identity graph).

**Files:**
- Modify: `docs/superpowers/plans/2026-09-06-identity-e2e-results.md`

**Interfaces:**
- Consumes: the Go `email-service` running against `prod_rehearsal2` on `:8081`.
- Produces: verdicts for #38, #39, #41.

- [ ] **Step 1: Start the Go service against rehearsal2**

```bash
cd /home/mannie/Desktop/Projects/treatmentpath/EmailServiceGo
DB_NAME=prod_rehearsal2 go run ./cmd/server 2>&1 | tail -40
curl -s http://127.0.0.1:8081/call-agent/health
```
Expected: healthy. If the service demands vendor credentials it does not need for
these endpoints, record that as a blocker rather than supplying real ones.

- [ ] **Step 2: Drive `lookup_patient` with an ambiguous phone (#38, #39)**

Use a shared-phone family from Task 3 Step 1.

```bash
curl -s -X POST http://127.0.0.1:8081/call-agent/tools/lookup_patient \
  -H 'Content-Type: application/json' \
  -d '{"practice_id":16,"caller_phone":"<SHARED NUMBER>","first_name":"","last_name":""}' | python -m json.tool
```
Expected (post-fix): the sole-candidate rule returns **no** confident patient
because there is more than one — not an arbitrary first row.

Then repeat passing the DOB of the *second* family member and assert the DOB
actually filters (#39 — the audit's tests covered only the parser, not the wiring):

```bash
curl -s -X POST http://127.0.0.1:8081/call-agent/tools/lookup_patient \
  -H 'Content-Type: application/json' \
  -d '{"practice_id":16,"caller_phone":"<SHARED NUMBER>","first_name":"<NAME>","dob":"<DD/MM/YYYY>"}' | python -m json.tool
```
Expected: the correct single human, and a wrong DOB returns none.

- [ ] **Step 3: Post a call webhook and check the Intake lands in the graph (#41)**

```bash
curl -s -X POST http://127.0.0.1:8081/call-agent/post-call-webhook \
  -H 'Content-Type: application/json' \
  -d @/tmp/postcall.json | python -m json.tool
```
Build `/tmp/postcall.json` from the real payload shape in
`internal/callagent/` handlers. Then:

```sql
SELECT id, first_name, last_name, person_id, practice_id
FROM "TreatmentPlan_intake" ORDER BY id DESC LIMIT 5;
```
Expected: the new Intake has a non-NULL `person_id` and the Person resolves from the
caller's channel, in the right practice.

- [ ] **Step 4: Re-check the Billy Wright / Carly case (#38)**

Confirm intake 396's stored attribution and whether the new resolver would still
produce it. Record the verdict against the 49-row triage query already written into
`TreatmentPathBackend/to-run-inprod/2026-09-06-patient-identity-audit-followups.txt`.

---

### Task 6: Online booking, day list, recall and the remaining identity lanes

Covers #18, #19, #20, #28, #33, #34 (booking), #5 and #36 (day list), #2, #8, #42
(recall), #3 and #6 (family), #26, #30, #31 (medical history), #43 (consent),
#22, #23, #24 (Notes/Tasks/Appointments scoping).

**Files:**
- Modify: `docs/superpowers/plans/2026-09-06-identity-e2e-results.md`

**Interfaces:**
- Consumes: Task 2's stack.
- Produces: verdicts for the remaining findings.

- [ ] **Step 1: Book online as a family member sharing an email (#18, #19, #20, #33, #34)**

Find the public booking URL for practice 16, book as a relative on a shared family
email using a *different* name. Then assert in SQL:

```sql
SELECT id, first_name, last_name, email, person_id, archived_at
FROM "TreatmentPlan_patient" ORDER BY id DESC LIMIT 5;
```
Expected: a NEW patient with a non-NULL `person_id` (#33) — not a link onto the
relative's record (#18). Archive a patient first and re-book to confirm the archived
row is never matched (#34).

- [ ] **Step 2: Check the ATTENTION note for an unmatched "existing" booker (#28)**

Book declaring EXISTING with details that cannot match. Expected: the appointment
notes carry the ATTENTION line rather than silently discarding the declaration.

- [ ] **Step 3: Day list — AI summary attribution and history counts (#5, #36)**

Open the Day List for a practice with a duplicated patient name. Confirm the AI
summary shown belongs to that `dentally_patient_id` and no other, and that the
history counts (prior no-shows / cancellations) no longer include other practices.
Cross-check one patient's count in SQL against a practice-scoped query.

- [ ] **Step 4: Recall list and Dormant tab (#2, #8, #42)**

Open a recall row for a multi-word surname and confirm clicking through opens that
human's record (#2). Repeat on the Dormant tab, which passed no context map (#8).

- [ ] **Step 5: Family lookup across practices (#3, #6)**

On a patient whose email is shared across practices 13/16, open family members.
Expected: only same-practice relatives. Then use "Add family member" and confirm it
no longer welds two humans into one Person (#3).

- [ ] **Step 6: Medical history portal (#26, #30, #31)**

Issue a medical-history link, then: (a) try to read it with only the name and no DOB
— expect refusal (#26); (b) POST straight to submit without verifying — expect
refusal (#30); (c) use a verification code against a *different* treatment plan —
expect refusal (#31). Note #31 needs the frontend to send `treatment_plan_id`;
if the frontend does not yet, record it as the known outstanding item, not a pass.

- [ ] **Step 7: Cross-practice writes on Notes, Tasks, Appointments, Consent (#22, #23, #24, #43)**

For each, attempt to create/patch referencing a patient from another practice via the
browser network panel or curl with the session token. Expected: 400/404 on all four,
never a successful write.

- [ ] **Step 8: Full-stack regression re-run and final report**

```bash
source /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/venv/bin/activate
cd /home/mannie/Desktop/Projects/treatmentpath/TreatmentPathBackend/TreatmentPath
python manage.py test --keepdb 2>&1 | tail -20
```
Compare failures by name against the recorded 34-name baseline. Then write the final
report: per-finding verdict (RE-CREATED / HELD / NOT TESTED), with evidence.

---

## Cleanup

- [ ] Stop Django, Vite and the Go service.
- [ ] Delete `/tmp/treatmentpath-outbox.jsonl`, `/tmp/egress.txt`, `/tmp/postcall.json`.
- [ ] Leave `outbound_sandbox/`, `.env.sandbox` and `.env.local` in place — they are
      the deliverable that makes this repeatable — but state clearly in the report
      that `outbound_sandbox` is dev-only, is inert unless `OUTBOUND_SANDBOX=1`, and
      that the boot guard is the reason it is safe to leave in the tree.
- [ ] Do not commit. Report the file list for the user to review.

## Known risks

1. **rehearsal2 already has the identity repairs applied.** Some original damage has
   been repaired away, so a defect that cannot be re-created from existing rows is not
   proof the code is fixed. The stronger test is creating NEW data through the UI and
   seeing whether the app still *makes* the problem — Tasks 3-6 are written that way.
2. **Migration 0168 is destructive.** It runs in Task 2 Step 2 against the scratch
   copy. If rehearsal2 is later needed in its current state for another purpose, dump
   it first — but the disk has only 5.1 GB free and the DB is 8.6 GB, so a dump does
   not fit. Confirm with the user before running Task 2 Step 2 if that matters.
3. **Secrets already in the tree** (found during recon, not introduced here):
   `settings.py:265` has a hardcoded DB password fallback, and `settings.py:535-537`
   has a real-looking Twilio SID/token in comments. Out of scope for this plan;
   report them.
4. **Disk at 99%.** Do not enable SQL logging to disk; the Go service floods logs.

---

## Appendix: finding → task coverage (self-review)

Every one of the 44 findings, and where it is exercised. "Not user-reachable" means
the lane is a batch/sync job with no UI; those are verified by driving the job or by
SQL, not by the browser, and are called out so they are not silently skipped.

| Finding | Where verified |
|---|---|
| #1 name-split makes duplicate Persons | Task 5 Step 3 + Task 6 Step 1 — create a NEW person via call webhook and via booking, assert one Person not two |
| #2 recall list opens wrong person | Task 6 Step 4 |
| #3 "Add family member" merges humans | Task 6 Step 5 |
| #4 treatment plan reassignable across practices | Task 6 Step 7 (add TreatmentPlan to the four) |
| #5 AI summaries shared between same-named patients | Task 6 Step 3 |
| #6 family lookup leaks across practices | Task 6 Step 5 |
| #7 lookup computes a phone key the writer never writes | Task 3 (every panel) |
| #8 Dormant recall tab loses the contact | Task 6 Step 4 |
| #9 `Person.resolve` without dob | Task 5 Step 2 (DOB passed) + Task 6 Step 1 |
| #10 `+GB…` phones, SMS never sends | Task 3 — a send that reaches the outbox at all proves the number canonicalised |
| #11 merge suggester compares raw strings | Not user-reachable via send; drive the duplicates panel in Task 6 Step 5 |
| #12 lead stripped of all contact details | Task 4 Step 3 |
| #13 sites prefer Dentally's broken normalized phone | Task 3 panels 6-9 (day list / recall / confirmations) |
| #14 workflow patient lookup case-sensitive on email | Task 6 Step 7 — trigger a workflow with a mixed-case email |
| #15 bad/foreign practitioner assignment returns success | Task 6 Step 7 |
| #16 Dentally bridge scores across practices | Not user-reachable — run `bridge_dentally_identity --dry-run` and check the practice split |
| #17 Activity History shows another patient's notes | Task 4 Step 1 |
| #18 booking links to wrong family member | Task 6 Step 1 |
| #19 booking phone match dead | Task 6 Step 1 |
| #20 booking name split on payment | Task 6 Step 1 |
| #21 first-space splitters feeding `Person.resolve` | Covered by #1's checks |
| #22 clinical notes/letters accept any practice's patient | Task 6 Step 7 |
| #23 Tasks scopes user FKs not patient FKs | Task 6 Step 7 |
| #24 Appointments accepts any patient/clinician | Task 6 Step 7 |
| #25 `by_contact` returns oldest family member's log | Task 4 Step 1 — request `by_contact` on a shared email, expect 300 + candidates |
| #26 medical-history downgrades DOB verification | Task 6 Step 6a |
| #27 read-only `patient_name` PATCH no-ops | Task 4 Step 2 |
| #28 unread `patient_type` | Task 6 Step 2 |
| #29 hand-rolled phone normalisation in consent SMS | Task 3 panel 11 |
| #30 medical-history submit never checked verification | Task 6 Step 6b |
| #31 verification code not bound to its plan | Task 6 Step 6c |
| #32 practice check fails open | Task 6 Step 7 — act as a user with no `current_practice` |
| #33 booking creates Patient with no Person | Task 6 Step 1 |
| #34 `_match_patient` matches archived patient | Task 6 Step 1 |
| #35 `patient_activities` unvalidated `patient_id` | Task 4 Step 1 |
| #36 day-list history counts cross practices | Task 6 Step 3 |
| #37 marketing profile = arbitrary member of fused Person | Task 3 panel 12 |
| #38 call agent picks first human on shared phone | Task 5 Steps 2 and 4 |
| #39 call-agent DOB accepted and ignored | Task 5 Step 2 |
| #40 ActivityLog accepts foreign Persons | Task 4 Step 1 + Task 6 Step 7 |
| #41 Go call-agent Intakes outside identity graph | Task 5 Step 3 |
| #42 recall sync re-splits Dentally names | Not user-reachable — inspect after a recall sync run, or SQL-check the 1,131 rows |
| #43 consent crosses practices | Task 6 Step 7 |
| #44 (folded into #38) | Task 5 |

**Gaps acknowledged:** #16 and #42 are sync jobs with no user-facing trigger; they are
verified by running the job and by SQL, and the results doc must say so rather than
claiming a browser test. #11's merge suggester is reachable only through the
duplicates panel, which `project_merge_never_completes` says has a history of not
surfacing suggestions — if it does not surface, that is recorded as NOT TESTED.

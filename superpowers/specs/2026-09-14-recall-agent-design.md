# Recall Agent — Design

**Date:** 2026-09-14
**Status:** Design approved, not implemented
**Scope:** v1 in-app AI agent, recall domain only, practice-scoped, single-patient actions

---

## 1. Purpose

Give each practice a natural-language agent that performs recall work on the
user's behalf: pull up a patient, show their risk and segment and value, draft a
recall message, and send it by SMS or email — plus the single-patient recall
actions (star, delay, archive, assign).

The agent is **not** a new permission system. Every action it takes is an
ordinary authenticated request to an endpoint that already exists, made with the
requesting user's own identity, so the permission checks that already guard the
UI are the checks that guard the agent. The agent never decides whether someone
is allowed to do something.

### Out of scope for v1

- **Bulk send.** Single patient at a time only.
- Any recall domain other than the actions listed in §3.
- Memory across conversations, multi-agent orchestration, RAG.
- DeepSeek. Build against the Claude API first; the endpoint swap is a config
  change, not an architecture change.
- Frontend work beyond the streaming channel contract.

---

## 2. Findings that shaped this design

These were verified against the codebase, not assumed. They are recorded because
each one closed off an approach that looked reasonable beforehand.

### 2.1 `bulk_send` will not send free text

`dentallyIntegration/views/recall_views.py:1622` requires a `template_id`
resolving to an active `template_type='recall'` template, and renders it against
the patient context. There is no ad-hoc recall send path.

### 2.2 The messaging endpoints are not a substitute

- `messages/send-email/` → `send_plain_email` accepts free-text `content` and
  does accept `message_purpose` (`messaging/views/message_views.py:1661`), but
  **never checks `use_email`**. No consent gate.
- `messages/send-sms/` **does not exist** — it is commented out at
  `messaging/urls.py:284`. Only `bulk-send-sms` exists, which stamps no
  `message_purpose` and checks no consent.

Routing agent sends through these would drop recall messages out of recall
reporting and bypass patient opt-outs, asymmetrically across the two channels.

### 2.3 `recalls` is a single all-or-nothing feature

`UserAuthentication/views/access_control.py:720` defines `recalls` as `True` for
`practice_admin`, `practice_dentist`, `practice_staff` and `practice_member`
alike. The only role differentiation anywhere in the recall viewset is
`_can_unarchive` (`recall_views.py:790`), which bypasses the feature matrix and
checks `user_type in ("admin", "superuser")`.

Consequence: role-scoped tool filtering buys nothing inside recalls today. With
bulk send out of scope the flatness is acceptable for v1, because every action
the agent can take unattended is reversible by the same user — **except
archive**, which §5.2 handles specifically.

### 2.4 `at_risk_reasons` is not on the detail endpoint

`recall_detail` returns appointment counts, last appointment type, future
appointment and attendance segment. `at_risk_reasons` and `attendance_segment`
are `SerializerMethodField`s fed by context maps built in the **list** view
(`dentallyIntegration/serializers.py:1004`, `:1493`). Reading a patient's risk
means calling the list endpoint filtered to one id, not the detail endpoint.

### 2.5 Assign has no single-patient route

`recalls/bulk-assign/` is the only assignment endpoint. The agent calls it with
a one-element list. This is not bulk send returning by the back door — the tool
signature remains single-patient.

### 2.6 API keys are method-scoped, not path-scoped

`UserAuthentication/api_key_authentication.py:126` enforces scopes against
`API_KEY_METHOD_SCOPES` (`retrieve`/`create`/`update`/`delete`). There is no
path restriction. See §4.3 for what this does and does not buy.

### 2.7 Infrastructure state (measured 2026-09-14)

- DEV runs `django` (gunicorn), **`daphne`**, `celery-worker`, `celery-beat`,
  `nginx`, `db`, `redis` as containers on `Treatmentpath-DEV`.
- Gunicorn: 6 workers, 120s timeout.
- No Celery queues configured — everything on the default queue. 30min hard
  limit, 25min soft, `prefetch_multiplier=1`.
- Postgres is exposed on the tailnet (`100.106.198.82:5432`). **Redis is not** —
  it has no host port and is reachable only inside the DEV docker network.
- `SMTP-SERVER-DEV`: 2 vCPU, 3.7GB RAM, load ~0.04 (idle), disk 72% after the
  2026-09-14 cleanup.
- Measured latency `SMTP-SERVER-DEV` → DEV API: 39ms RTT over tailnet; 190ms for
  a cold HTTPS call including TLS; ~40–60ms warm with connection reuse.
  `api.anthropic.com` edge: 1.6ms.

---

## 3. Tool surface

Five read tools and six write operations — eight callables, since star and delay
each carry a paired undo. Two of the writes are proposal-only.

### Reads

| Tool | Backs onto | Purpose |
|---|---|---|
| `find_recall_patient(query)` | `recalls/` filtered | "pull up Sarah Bell" |
| `get_recall_patient(id)` | `recalls/` id-filtered + `recalls/<id>/detail/` | risk, segment, spend, appointments |
| `list_recalls(filters)` | `recalls/` | "who's high value and overdue" |
| `list_recall_templates(channel)` | active recall templates | the template menu |
| `preview_recall_message(...)` | the §5.1 send endpoint with `dry_run=true` | rendered preview |

### Writes

| Tool | Route | Gate |
|---|---|---|
| `star_recall` / `unstar_recall` | `recalls/<id>/star/` POST/DELETE | direct |
| `delay_recall` / `undelay_recall` | `recalls/<id>/delay/` POST/DELETE | direct |
| `assign_recall` | `recalls/bulk-assign/`, one-element list | direct |
| `archive_recall` | `recalls/<id>/archive/` POST | **confirm** |
| `send_recall_message` | recall send path (§5.1) | **confirm** |

### Hard constraints on tool inputs

- `practice` is never a model-supplied parameter. It is resolved server-side
  from `request.user.current_practice`. A hallucinated practice id has nowhere
  to land.
- `page_size` is never a model-supplied parameter. It is pinned at 25.
- The agent only ever drives **existing, proven filter paths**. It never
  composes novel filter combinations — PROD Postgres runs a 64MB `/dev/shm`
  and non-sargable date filters go parallel and 500 there. The recall list was
  deliberately rewritten to be sargable; that property must not be bypassed.
- Tool results return a **trimmed projection** (name, id, due date, segment,
  risk reasons, spend), never the raw serializer payload. See §7.

---

## 4. Execution model

### 4.1 Flow

1. The user's message hits a **thin HTTP POST**. It mints a per-run credential
   (§4.3), enqueues the run on the `agent` Celery queue, and returns a `run_id`.
   The gunicorn worker is released in milliseconds.
2. A **Celery worker** runs the agent loop, making **loopback HTTP calls** to the
   app's own endpoints carrying that credential.
3. Output streams to the browser over the **existing Channels websocket**.
4. On completion the credential is revoked. If the run dies, `expires_at`
   collects it.

### 4.2 Why loopback HTTP rather than in-process dispatch

In-process dispatch (synthesizing a request and calling the viewset directly)
skips middleware. That is disqualifying: `SubscriptionMiddleware` performs real
gating and only exempts the `internal/` prefix, so in-process calls would
silently bypass subscription checks.

Loopback runs the full stack — middleware, `DualAuthentication`,
`IsAuthenticated`, `check_user_feature_access`, practice scoping. The agent is
genuinely just another client, which is the property the whole design rests on.

### 4.3 The per-run credential

An `APIKey` row (`UserAuthentication/models.py:1128`) is minted per run:

- bound to the requesting user and practice
- `expires_at = now + 10 minutes` (set explicitly — `create_api_key` only takes
  `expires_in_days`, which is far too coarse)
- method scopes narrowed to what the recall tools need
- `ip_whitelist` pinned to the agent host when it runs off-box
- named `agent-run:<uuid>`, revoked on completion

This gives short-lived, revocable, IP-bound, method-scoped credentials, with
`last_used_at` and `total_requests` providing usage audit for free.

#### Why this needed checking: the email-send auth trap

`bulk_send`'s email path requires `HTTP_AUTHORIZATION` to start with `Bearer `
(`recall_views.py:1754`) and hands that token to `EmailServiceClient`, which
requires *"a JWT auth token with practice_id claim"* and forwards it to
EmailServiceGo — a **separate service doing its own JWT validation**
(`messaging/email_service.py:63-83`). An `APIKey` presented there would 401.

That constraint is real but it is **specific to the legacy `bulk_send` path**.
The automation path does not have it: `RecallDelivery.ensure_email()` mints its
own service token via `_mint_practice_auth_token(practice)`
(`UserAuthentication/alert_email_utils.py:78`) and needs no caller credential at
all. Note also that `bulk_send`'s Bearer extraction sits inside
`if not dry_run and channel == "email"`, so previews never reach it.

§5.1 therefore routes **all** agent sends through the automation path rather than
`bulk_send`. With that decided, the caller's credential never has to satisfy
EmailServiceGo, and `APIKey` — the stronger option — is free to be used.

One consequence to record: `_mint_practice_auth_token` selects the practice's
prime admin, so EmailServiceGo-side attribution is practice-level, not
user-level. That is how recall automation already behaves. Our own audit (§6)
records the actual requesting user, which is where user attribution lives.

**What the credential does not do:** restrict *which* endpoints the run can hit
(§2.6). Containment is structural — the agent can only invoke the tools we write,
and those call fixed recall URLs; it cannot compose arbitrary requests. The tool
layer holds that line, not the credential.

> **Open decision:** adding optional path-prefix scoping to
> `APIKeyAuthentication` is roughly 20 lines and useful well beyond this
> feature. Recommended as defence in depth, but separable from v1.

### 4.4 Model configuration

- `claude-opus-5`, via the Anthropic SDK.
- **A manual tool-use loop** — `while stop_reason == "tool_use"` over
  `client.messages.create` — **not** the Tool Runner. See §4.5: the Tool Runner
  lives in the `client.beta.messages.*` namespace and sends `anthropic-beta`
  headers, which forecloses provider portability. Its main draw is per-turn
  approval hooks, and this design deliberately puts confirmation outside the
  model entirely (§5.2), so those hooks were never needed. Audit is logging
  inside each tool wrapper.
- **Not** the Claude Agent SDK — that is Claude Code as a library, with built-in
  filesystem and bash tools and a Node subprocess per session. Wrong product for
  an API-calling job; we would spend our time removing capabilities we never
  wanted.
- Adaptive thinking. Streaming, so tokens and tool progress reach the websocket.
- Per-run caps: ~15 tool calls, ~3 minute timeout (not Celery's 25-minute soft
  limit), and a repeat-call guard to prevent retry storms.

---

### 4.5 Provider portability (DeepSeek)

DeepSeek publishes an **official** Anthropic-format endpoint at
`https://api.deepseek.com/anthropic`, usable from the standard `anthropic`
Python SDK via `base_url`. Their docs state tool use is fully compatible — tool
names, input schemas, descriptions, and `tool_choice`
(`none`/`auto`/`any`/`tool`). Only `disable_parallel_tool_use` is ignored.

That compatibility is documented against `client.messages.create` — the stable
Messages API. It says nothing about the `client.beta.messages.*` namespace. This
is the reason §4.4 specifies a manual loop rather than the Tool Runner: staying
on the stable surface keeps the swap a config change, as intended.

**Current blocker to actually testing it (verified 2026-09-14):** the DEV Django
container has only `OPENROUTER_API_KEY` — no `DEEPSEEK_API_KEY`, no
`ANTHROPIC_API_KEY`. The OpenRouter key is valid (HTTP 200), but OpenRouter
speaks **OpenAI** format, not Anthropic format, so it cannot serve the
`anthropic` SDK's Messages API shape. This matches the 2026-08-18 finding that
the direct DeepSeek keys were returning 401.

A valid direct DeepSeek key is therefore a prerequisite before the swap is
testable. Latency should be measured before committing to it — it is the thing
most likely to change, and not in our favour.

---

## 5. Writes and confirmation

### 5.1 The recall send path

Both channels go through the recall send path, never the messaging endpoints
(§2.2). This means, symmetrically for SMS and email: `use_sms`/`use_email`
honoured, `message_purpose='recall'` stamped so sends land in recall reporting,
sandbox and `dry_run` respected, person/channel attribution correct.

**One new endpoint handles both**, and it does not extend `bulk_send`. It builds
on the automation path instead:

```
RecallDelivery(practice, dry_run, sandbox)   # recall_automation.py:440
  → _consent_allows(practice, patient_id, channel)
  → send_recall_message(delivery, rec, channel, subject, body)
```

- **Free-text sends** pass the agent-drafted `subject`/`body` straight through.
- **Template sends** resolve the active recall template first and pass its
  `subject`/`content` as that same `subject`/`body`.

Routing both through one path means one audited send, one consent gate, one
purpose stamp — and, per §4.3, no caller-credential constraint from
EmailServiceGo. `send_recall_message` already creates the
`message_purpose='recall'` event and attributes the session to the patient.

The endpoint belongs in `dentallyIntegration` alongside the other recall
endpoints, not in the agent app, because it is a normal API the UI could use too.

`bulk_send` is left untouched and keeps serving the existing UI.

"Send both" is two calls and two confirmations. That is correct, not annoying —
it is two messages to a patient.

### 5.2 The model cannot perform a one-way action

Rather than asking the model to respect a confirmation step, the capability is
removed from it.

`send_recall_message` and `archive_recall` are **proposal tools**. When the model
calls one, it renders the outcome (for sends, via the §5.1 endpoint with
`dry_run=true`) and returns a preview plus a short-lived, server-signed
`confirmation_token` bound to practice + patient + channel + rendered body.
Nothing is dispatched.

The user sees the preview and clicks Confirm. The **frontend** then calls the
real endpoint with that token — the model is not in that call path at all.

Archive is gated for a specific reason: `_can_unarchive` restricts undo to
`admin`/`superuser`, so a `practice_staff` user who archives the wrong patient by
agent cannot undo it themselves. Star, delay and assign are reversible by the
same user, touch no patient, and execute directly.

The invariant: **everything the model can execute unattended, the user can undo
unattended.**

### 5.3 Prompt injection

Patient-authored content (names, note text, inbound message history) reaches the
agent's context, and a patient can write "ignore previous instructions". Retrieved
content is framed as data, never as instructions.

The structural guarantee in §5.2 does the heavy lifting: a prompt-injected or
confused model can, at absolute worst, *propose* a message that a human then
declines.

---

## 6. Audit

Every tool call logs user, practice, tool, parameters, HTTP status, and for sends
the resulting message id.

Sends land in `activityLog` alongside manual ones, and the existing
`message_purpose='recall'` stamp keeps them in recall reporting.

**An agent-sent recall should be indistinguishable from a hand-sent one in
reporting, and distinguishable in audit.**

---

## 7. Load and cost

Estimated from the measured recall list cost (39 queries / 267ms for 593 rows
post-optimization). To be confirmed by instrumenting the first runs.

A typical exchange is 4–6 model turns and 5–8 loopback calls: roughly **50–100 DB
queries and well under a second of total gunicorn time**, across 30–45s of wall
clock. Almost all of that wall clock is Anthropic latency.

Note what Celery moves and what it does not: it takes the *loop* off gunicorn,
but each tool call is still a real HTTP request landing on those same 6 workers.
They are short, so this is fine — but it is the number that scales with
concurrency, and relocating the agent to another host does not change it.

### The four risks, ranked

1. **An unbounded `list_recalls`.** "Show me all my high value patients" is one
   sentence. Mitigated by pinning `page_size` at 25 (§3).
2. **Token cost — the real bill, not CPU.** Every tool result is resent to the
   model on every subsequent turn. 20 full serializer payloads is 4–5k tokens,
   re-sent 4 times. Mitigated by the trimmed projection (§3). This is the
   highest-leverage decision in the design.
3. **Retry storms.** Mitigated by the per-run call cap and timeout (§4.4).
4. **Queue contention.** Mitigated by the dedicated `agent` queue (§8).

---

## 8. Where it lives

A new Django app, `recallAgent/`, inside `TreatmentPathBackend` — **not** a
separate service. Everything it needs is Django-side: credential minting is an
a short-lived JWT, audit is `activityLog`, streaming is the configured Channels layer,
async execution is the deployed Celery worker.

It is already decoupled where it counts: because tools call recall endpoints over
loopback HTTP, `recallAgent` imports nothing from `dentallyIntegration`. That is
the clean boundary a separate service would give, without a second deployment
target.

```
recallAgent/
  tools/          # one module per tool, loopback HTTP only
  runner.py       # Tool Runner loop, claude-opus-5
  credentials.py  # mint the short-lived per-run JWT
  audit.py        # activityLog integration
  tasks.py        # Celery task (queue: agent)
  views.py        # thin POST to start a run
  consumers.py    # websocket streaming
```

### Deployment placement

"Separate server" here means the **same image with a different command** — a
worker consuming the `agent` queue — against the same broker and database. Not a
separate codebase.

- **DEV / pilot:** `SMTP-SERVER-DEV` is sufficient. 2 vCPU and 3.7GB RAM against
  an idle load, and the measured +320ms per run is ~1% of wall clock.
- **PROD:** do **not** co-locate the agent with mail. That box delivers patient
  email; an agent run that pins CPU or fills disk would take deliverability with
  it. Use a separate small box for the agent queue.

### Deployment prerequisites

1. **Redis must be reachable from the agent host.** It currently has no host port
   and is internal to the DEV docker network, so a remote worker cannot consume
   the queue at all. Bind it to the tailnet interface as Postgres already is
   (`100.106.198.82:5432`) and set a password. **This is the blocker.**
2. `CELERY_TASK_ROUTES` — the project's first queue definition — plus an
   `agent`-queue worker on DEV and PROD.
3. Disk headroom on the agent host. (Addressed on `SMTP-SERVER-DEV`
   2026-09-14: 84% → 72%, and docker log rotation configured.)

---

## 9. To verify during implementation

Both would be quiet failures if wrong, and both are cheap to check:

1. Does a JWT minted with a shortened `set_exp()` still carry the `practice_id`
   claim that `EmailServiceClient` and EmailServiceGo require?
2. Does `SubscriptionMiddleware` treat a loopback-originated request the same as
   a browser-originated one?

---

## 10. Build order

Local first, deploy last, per the agreed sequencing.

1. Free-text recall send endpoint in `dentallyIntegration` + tests.
2. `recallAgent` app skeleton: credential mint/revoke, audit, loopback client.
3. Read tools + trimmed projections.
4. Tool Runner loop with caps.
5. Write tools; two-phase confirmation for send and archive.
6. Websocket streaming + the thin start endpoint.
7. Instrumentation: real query counts and token usage per run, replacing the §7
   estimates.
8. Deploy: Redis on tailnet, `CELERY_TASK_ROUTES`, agent worker, placement.

Tests mirror existing `dentallyIntegration` patterns and run with `--keepdb`
(never `--noinput`).

**No changes to any existing recall endpoint.** The read tools wrap what is
already there.

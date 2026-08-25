# Truthful outbound delivery status via SES event publishing

Date: 2026-08-13
Status: Approved design, not yet implemented
Scope: EmailServiceGo (`internal/email`) + one-time AWS configuration

## Problem

The inbox shows a delivery indicator under each outgoing email: one tick for
sent, two for delivered. That indicator is driven by a pipeline that cannot
produce a correct answer in production.

The pipeline is:

1. Django saves the `EmailMessages` row, then hands the send to EmailServiceGo
   with `django_message_id` in the job metadata
   (`messaging/views/message_views.py:1883`).
2. EmailServiceGo's Postfix log watcher (`internal/email/postfix/watcher.go`)
   tails the `postfix-smtp` container's mail log, parses delivery and bounce
   lines, and calls Django's `internal/email-delivery/` callback with the
   resulting status.
3. Django's `email_message_delivery_callback`
   (`messaging/views/message_views.py:3645`) writes `EmailMessages.status` and
   broadcasts `email_delivery_status` over the session WebSocket.
4. The frontend renders the tick
   (`components/inbox/MessageDeliveryStatus.tsx`).

Step 2 is the broken link. In production `EMAIL_PROVIDER=ses`, and
`cmd/server/main.go:145` hard-fails at boot if it is anything else. The
outbound worker therefore relays every message directly to
`email-smtp.eu-west-2.amazonaws.com` (`worker.go`, `smtpSvc.SendEmail`). That
traffic never traverses the local `postfix-smtp` container, so the watcher has
no outbound log lines to parse. Whatever happens to a message after SES accepts
it — delivered, hard bounce, spam complaint — is invisible to us.

The watcher is not dead code: it remains the correct mechanism for non-production
environments where `EMAIL_PROVIDER=postfix`. The defect is that production has no
equivalent.

### Not verified

Whether production `EmailMessages` rows currently reach `status='delivered'` at
all was not confirmed empirically — the prod database read was blocked by the
permission classifier. The design does not depend on the answer: if some other
path is supplying status today, SES events supersede it with a more accurate
source; if nothing is, SES events fill a vacuum.

## Goal

Outbound delivery outcomes in production come from SES itself, so the inbox's
delivery indicator reflects what actually happened to the email.

Explicit non-goals, deliberately excluded:

- **Open / read tracking.** Considered and dropped. It is unreliable by nature
  (image blocking hides real opens; Apple Mail Privacy Protection manufactures
  false ones), and it only works on HTML email. Revisit separately if wanted.
- **Click tracking.** Rewrites every link to an `awstrack.me` redirect, visible
  to patients and a deliverability risk on clinical mail.
- **Replacing the Postfix watcher.** It stays as-is for non-production.

## Design

### Overview

Create an SES configuration set with an event destination that publishes
`DELIVERY`, `BOUNCE`, `COMPLAINT`, and `DELIVERY_DELAY` to an SNS topic. SNS
POSTs those events to a new webhook on EmailServiceGo. The webhook verifies the
message, correlates it back to the originating email, updates Go's own
`EmailSendingLog`, and — when the email came from the inbox — calls Django's
**existing** delivery callback with the mapped status.

```
outbound worker ──X-SES-CONFIGURATION-SET──▶ SES ──event──▶ SNS topic
                                                              │ HTTPS POST
                                                              ▼
                                          EmailServiceGo /webhooks/ses
                                                              │
                                    ┌─────────────────────────┴───────────┐
                                    ▼                                     ▼
                        update EmailSendingLog          Django internal/email-delivery/
                                                        (existing, unchanged)
                                                                          │
                                                        EmailMessages.status + WebSocket
```

Django, the serializers, and the frontend need **no changes**. The existing
callback's contract already accepts exactly the statuses this produces, and the
frontend already renders them.

### Why this shape

The alternative of a new consolidated Django endpoint was rejected: it would
mean duplicating or migrating logic that already exists and works in
`email_message_delivery_callback` (status write, marketing bounce auto-blocking,
WebSocket broadcast), while leaving the old endpoint in place for the Postfix
path anyway.

Having SNS post directly to Django was also rejected: the correlation data and
`EmailSendingLog` both live in Go, and Django has no SES awareness. That split
would spread one concern across two services.

### Correlation

The outbound worker already stamps a custom header on every message
(`worker.go`, where `X-TreatmentPath-Django-Message-ID` is set). SES echoes the
message's headers back inside the event's `mail.headers` array, so the Django
message id returns to us with no new identifier scheme.

The trap is `mail.headersTruncated`. When SES truncates headers, that array is
incomplete and the custom header may be absent. Therefore the worker also
stamps the id into `X-SES-MESSAGE-TAGS`, which SES surfaces in the always-present
`mail.tags` object.

Resolution order, implemented as one named helper:

1. `mail.headers[]` entry named `X-TreatmentPath-Django-Message-ID`.
2. Fall back to `mail.tags["django_message_id"][0]`.
3. If neither is present, update `EmailSendingLog` only and skip the Django
   callback. This is normal, not an error — see Coverage below.

Message tag values are restricted to alphanumerics, hyphens, and underscores.
Django message ids are integers, so they satisfy this without encoding.

Go's own `EmailSendingLog` is correlated separately, by the SES `mail.messageId`
against the message id recorded at send time.

### Event to status mapping

| SES event | Condition | Status sent to Django |
|---|---|---|
| `Delivery` | — | `delivered` |
| `Bounce` | `bounceType == "Permanent"` | `bounced` |
| `Bounce` | `bounceType` is `Transient` or `Undetermined` | `deferred` |
| `Complaint` | — | `complaint` |
| `DeliveryDelay` | — | `deferred` |
| anything else | — | ignored, acknowledged with 200 |

The bounce split is a correctness requirement, not a refinement. Django's
callback auto-blocks the recipient's address via `auto_block_address` when a
`bounced` status arrives for a marketing-sourced message. A transient bounce is
a temporary condition such as a full mailbox or an unreachable MTA; mapping it
to `bounced` would permanently block a patient who did nothing wrong. Only
`Permanent` bounces block.

`deferred` is already an accepted value in the existing callback's status set,
so this needs no Django change.

### Coverage and its limits

State plainly what this does and does not fix:

- **Inbox emails** get corrected `EmailMessages.status` and a live WebSocket
  update, because they are the only sends that carry `django_message_id`. A
  grep across the Django codebase confirms `messaging/views/message_views.py`
  is the sole writer of that metadata key.
- **All other SES-relayed sends** (recall, alerts, confirmations) get an
  accurate `EmailSendingLog` in Go, but no Django-side row is updated, because
  no correlation key exists for them. Extending coverage later means adding
  `django_message_id` at those call sites; the webhook needs no change.
- **Direct-MX sends bypass SES entirely.** `shouldUseDirectMX` in the worker
  routes mail addressed to our own registered domains straight to the recipient
  MX, skipping the SES relay. Those messages generate no SES events and keep
  whatever status they already had.

### Components

All new Go code lives under `internal/email/ses/`, alongside the existing
`service.go`.

**`internal/email/ses/events.go`** — parsing and mapping, no I/O.
- SNS envelope types (`Type`, `TopicArn`, `Message`, `SubscribeURL`,
  `SigningCertURL`, `Signature`, `SignatureVersion`).
- SES event types covering `eventType`, `mail` (with `messageId`, `headers`,
  `headersTruncated`, `tags`), `bounce`, `complaint`, `deliveryDelay`.
- `ExtractDjangoMessageID(mail)` implementing the resolution order above.
- `MapEventToStatus(event) (string, bool)` implementing the mapping table.

Pure functions with no dependencies, so they are directly unit-testable.

**`internal/email/ses/sns.go`** — SNS message authenticity.
- Signature verification against the certificate named by `SigningCertURL`.
- **The `SigningCertURL` host must be validated against the expected AWS
  pattern before fetching it.** An unvalidated fetch lets an attacker point us
  at a certificate they control and forge any event. This is the single most
  security-sensitive line in the change.
- `SubscriptionConfirmation` handling: confirm only when `TopicArn` matches the
  configured topic, so an attacker cannot subscribe us to a topic of theirs.
- `UnsubscribeConfirmation` is logged and acknowledged, never acted on.

**`internal/email/ses/webhook.go`** — the HTTP handler wiring the above
together, registered as `webhooks.POST("/ses", ...)` in
`internal/email/api/router.go` beside the existing `/webhooks/postmark` route.
The `/webhooks` group has no auth middleware, which is correct: SNS cannot send
a bearer token, so authenticity comes from signature verification instead.

The two empty `TODO` stubs in that group, `HandleDeliveryStatus` and
`HandleBounce` (`internal/email/api/handlers.go:1190`), are left untouched. They
are unrelated dead placeholders and repurposing them would obscure the change.

**`internal/email/notify/django.go`** — the Django callback helper.
`notifyDjangoEmailDelivery` currently lives inside
`internal/email/postfix/watcher.go`. The SES path needs an identical call, so it
is lifted into one shared helper that both the watcher and the SES webhook use.
This is a move, not a rewrite: same payload, same `X-Workflow-Secret` header,
same timeout, same swallow-and-log error behaviour.

**`internal/email/worker/outbound/worker.go`** — where it already sets
`X-TreatmentPath-Django-Message-ID`, additionally set, only when
`w.sesEnabled` **and** the configuration set name is non-empty:
- `X-SES-CONFIGURATION-SET: <name>`
- `X-SES-MESSAGE-TAGS: django_message_id=<id>` (only when an id exists)

### Configuration

One new environment variable on the `email-service` container:

- `SES_CONFIGURATION_SET` — the configuration set name. **Unset means the
  headers are not added and behaviour is byte-for-byte identical to today.**
- `SES_SNS_TOPIC_ARN` — the expected topic ARN, used to reject subscription
  confirmations for any other topic.

This is the deploy-safety mechanism. Referencing a configuration set that does
not yet exist in AWS puts every outbound send at risk, so the code must be
deployable before the AWS side is built. With the variable unset, it is.

### AWS runbook (one-time, manual, `eu-west-2`)

Provisioned by hand and recorded here; there is no IaC for this stack. Add to
the to-run-in-prod list.

Order matters — the topic and subscription must exist and be confirmed before
`SES_CONFIGURATION_SET` is set on the container.

```bash
REGION=eu-west-2
CONFIG_SET=treatmentpath-prod
TOPIC_NAME=treatmentpath-ses-events
ENDPOINT=https://mail.pathwaymail.co/webhooks/ses

# 1. SNS topic
TOPIC_ARN=$(aws sns create-topic --name "$TOPIC_NAME" --region "$REGION" \
  --query TopicArn --output text)

# 2. Allow SES to publish to it (attach to the topic's access policy)
#    Principal: ses.amazonaws.com, Action: SNS:Publish, Resource: $TOPIC_ARN

# 3. SES configuration set
aws sesv2 create-configuration-set \
  --configuration-set-name "$CONFIG_SET" --region "$REGION"

# 4. Event destination -> SNS
aws sesv2 create-configuration-set-event-destination \
  --configuration-set-name "$CONFIG_SET" \
  --event-destination-name sns-events \
  --event-destination "Enabled=true,\
MatchingEventTypes=DELIVERY,BOUNCE,COMPLAINT,DELIVERY_DELAY,\
SnsDestination={TopicArn=$TOPIC_ARN}" \
  --region "$REGION"

# 5. Subscribe the webhook. The service must already be deployed and reachable
#    so it can auto-confirm the subscription.
aws sns subscribe --topic-arn "$TOPIC_ARN" --protocol https \
  --notification-endpoint "$ENDPOINT" --region "$REGION"

# 6. Verify the subscription is confirmed (not "PendingConfirmation")
aws sns list-subscriptions-by-topic --topic-arn "$TOPIC_ARN" --region "$REGION"
```

Then set `SES_CONFIGURATION_SET=treatmentpath-prod` and
`SES_SNS_TOPIC_ARN=$TOPIC_ARN` on the `email-service` container and restart.

No nginx change is required: `mail.pathwaymail.co` already terminates TLS and
proxies `location /` to the Go service.

### Error handling

- **Always acknowledge with 200 once the message is authentic.** SNS retries on
  non-2xx, and an event we cannot correlate will never become correlatable, so
  retrying it is pure noise. Unmatched events are logged, not errored.
- **Reject with 403 when signature verification fails**, and log it — that is a
  genuine security signal, not routine traffic.
- **A failing Django callback is logged and swallowed**, matching the watcher's
  existing behaviour. Go's `EmailSendingLog` is still updated, so the event is
  not lost from the system.
- **Events may arrive out of order.** SNS gives no ordering guarantee, so a
  late `Delivery` could overwrite a `Bounce`. Guard the Go-side status update
  so a terminal status (`bounced`, `complaint`) is not overwritten by
  `delivered` or `deferred`.
- **Duplicate events are possible** — SNS delivers at-least-once. All updates
  are idempotent writes of the same status, so redelivery is harmless.

## Testing

Go, table-driven, in `internal/email/ses/`:

- `MapEventToStatus`: one case per row of the mapping table, including both
  bounce branches and the ignored-event case.
- `ExtractDjangoMessageID`: header hit; `headersTruncated` with tags fallback;
  neither present.
- SNS signature verification: valid signature accepted; tampered payload
  rejected; **`SigningCertURL` on a non-AWS host rejected without any network
  fetch being attempted**.
- `SubscriptionConfirmation` for an unexpected `TopicArn` is refused.
- Out-of-order guard: `delivered` arriving after `bounced` does not downgrade.

Every test's expected value is derived from the AWS documented payloads quoted
in this spec, not from the implementation's own output, and each is shown able
to fail before its pass is trusted.

The shared Django-callback helper is a pure move, so the existing watcher tests
serve as the equivalence check that it still behaves identically.

## Rollout

1. Deploy the code with `SES_CONFIGURATION_SET` unset. No behaviour change:
   no headers added, no events generated, webhook idle.
2. Run the AWS runbook. Confirm the SNS subscription reaches `Confirmed`.
3. Set the two environment variables and restart `email-service`.
4. Send one inbox email to an external address and confirm a `Delivery` event
   arrives and the tick updates.
5. Send to a known-invalid address on a real domain and confirm the `Bounce`
   maps to `bounced`.

Rollback is unsetting `SES_CONFIGURATION_SET` and restarting: SES stops
emitting events for new sends immediately.

## Risks

| Risk | Mitigation |
|---|---|
| Config set referenced before it exists in AWS | Env-var gate; code ships inert |
| Forged events on a public unauthenticated endpoint | SNS signature verification with `SigningCertURL` host validation |
| Transient bounce permanently blocks a patient | Only `Permanent` maps to `bounced` |
| Out-of-order events downgrade a terminal status | Terminal-status guard on update |
| Silent regression for non-production Postfix flow | Watcher untouched; shared helper move covered by existing tests |

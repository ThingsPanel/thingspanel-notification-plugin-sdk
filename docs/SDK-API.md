# Plugin SDK and HTTP API

The normative wire contract is [`plugin-v1.openapi.yaml`](../specs/notification/plugin-v1.openapi.yaml).
It is snake_case, version `1.0`, and independent from the business v2 API.
The schema below is a quick map; the OpenAPI file is authoritative.

## Provider interface

Each executable implements `sdk.Plugin`:

```go
type Plugin interface {
    Manifest() sdk.Manifest
    Validate(context.Context, sdk.ValidateRequest) sdk.ValidateResult
    Send(context.Context, sdk.SendRequest) (sdk.SendResult, error)
    Health(context.Context) error
}
```

`Manifest` is stable for the registered plugin version. Configuration and
recipient constraints are embedded JSON Schema Draft 2020-12 documents. The
Core compiles them with its Draft 2020-12 validator and does not apply a
separate keyword allowlist. It rejects network and file `$ref` loading while
allowing references within the embedded schema. List every top-level
credential under `secret_fields`; list stable provider account identity fields
under optional `identity_fields`.

The machine-readable fields are demonstrated in
[`manifest.example.json`](manifest.example.json) and defined under
`components.schemas.Manifest` in the canonical OpenAPI file. The required
manifest fields are `api_version`, `plugin_id`, `name`, `plugin_version`,
`channels`, `content_modes`, `config_schema`, `recipient_schema`,
`secret_fields`, and `capabilities.delivery_receipts`. Optional
`identity_fields` names immutable account identity inputs. Unknown top-level
manifest fields are rejected.

Core validation and browser authoring have different boundaries:

| Layer | Current behavior |
| --- | --- |
| Core schema validation | Generic JSON Schema Draft 2020-12 compilation and validation; no separate keyword allowlist. Network/file external `$ref` loading is denied; references within the embedded schema work. |
| Current UI authoring | Recognizes `type`, `title`, `description`, `default`, `enum`, `const`, `properties`, `required`, `additionalProperties`, `items`, `minItems`, `maxItems`, `uniqueItems`, `minLength`, `maxLength`, `format`, `pattern`, `minimum`, `maximum`, `exclusiveMinimum`, `exclusiveMaximum`, and `multipleOf`. Formats are limited to `email` and `uri`; patterns must pass the UI's safe-pattern check. The editor renders string, integer, number, boolean, object, and array fields; `$ref` is not supported by the editor. |
| Reference-provider evidence | Existing adapters use `const`, `format`, string length bounds, numeric bounds, `pattern`, `minItems`, and nested object/array constraints. These are covered by local Core schema tests and offline provider tests; this does not prove live provider behavior. |

A schema accepted by Core is not guaranteed to be editable by the current UI.
Before registration, compile it on the target Core and test every field through
the target UI, including save and validation. Do not rely on remote references
or on a keyword that the target Core version has not compiled successfully.

## Endpoints and auth

All plugin endpoints use `Authorization: Bearer <plugin token>`. The SDK hashes
the configured token and uses constant-time comparison. Keep the listener
private behind the platform's trusted network path; `/healthz` also requires
the plugin token.

| Endpoint | Method | Meaning |
| --- | --- | --- |
| `/v1/manifest` | GET | Authenticated manifest; no send |
| `/v1/validate` | POST | Validate config; must not send a notification |
| `/v1/send` | POST | One target and one provider submission attempt |
| `/healthz` | GET | Basic process/provider readiness; no send |

The SDK defaults to a 1 MiB request body limit and rejects duplicate JSON keys,
explicit `null`, unknown fields, incorrect key casing, trailing JSON, excessive
nested JSON, invalid envelopes, and expired sends before calling the provider.
Every handler response is capped at 64 KiB. Manifest bytes include one final
newline; registration digest calculation must hash the exact HTTP response
bytes.

## Validate and send

`Validate` is a no-send path. It returns `valid=true` exactly when `errors` is
empty. A remote credential check may be read-only, bounded, and must never send
a test notification. Do not put config values, recipients, provider response
bodies, or credential details in `FieldError.message`.

`Send` receives a single immutable attempt with `delivery_id`, `attempt_id`,
tenant/instance/config-version context, one recipient, one content snapshot, and
`expires_at`. Validate channel, recipient, content mode, template allowlist,
config, and expiry before contacting a provider. A delivery ID or attempt ID is
not provider-side idempotency. Disable automatic retries and never retry a
write whose outcome may be ambiguous.

The request uses snake_case and includes the full immutable configuration
version for this attempt. This abbreviated fixture has no real credential or
destination:

```json
{
  "api_version": "1.0",
  "delivery_id": "fixture-delivery-001",
  "attempt_id": "fixture-attempt-001",
  "tenant_id": "fixture-tenant",
  "instance_id": "fixture-instance",
  "config_version": 1,
  "channel": "email",
  "recipient": {"kind": "email", "address": "operator@example.test"},
  "content": {"kind": "text", "title": "Fixture", "text": "local test"},
  "config": {"account_label": "fixture"},
  "expires_at": "2030-01-02T00:00:00Z"
}
```

The response is an outcome union, always HTTP 200 for a valid plugin-level send
result:

| Outcome | Meaning | Retry rule |
| --- | --- | --- |
| `accepted` | Provider confirmed submission acceptance | Not proof of delivery; receipt flag controls later reconciliation |
| `rejected` | Plugin can prove this attempt was not accepted | Retry only with `error.retryable=true`; `receipt_expected` must be false |
| `unknown` | Provider side effect may have happened but result is uncertain | `retryable` must be false; stop automatic resubmission |

When `accepted` has `receipt_expected=true`, return a non-empty
`provider_message_id` that is stable in the provider account/region namespace.
The core may later mark delivered only from a trusted receipt. HTTP 202 from the
business API is likewise acceptance, not delivery.

## Receipts

`/api/v1/notification-instances/{instance_id}/receipts` is hosted by the
notification core, not by the plugin. The plugin must authenticate the
provider's original callback/queue message, normalize one event, post it with
the dedicated instance receipt credential, and ACK/delete the provider message
only after the core confirms durable acceptance. Do not reuse the plugin
bearer token. Receipt bodies contain `event_id`, `provider_message_id`, status,
and RFC3339 occurrence time; `delivery_id` may be included only when safely
correlated. Use stable event IDs when provider callbacks repeat. Statuses are
`delivered`, `failed`, or final `unknown`; intermediate `queued`, `sent`, or
`ringing` states are not final receipts.

If durable receipt storage, authentication, or core ACK is unavailable, retain
the source event for a bounded retry or durable buffer. Do not advertise
`delivery_receipts=true` or return `receipt_expected=true` until that full
path is deployed and tested. Current Alibaba SMS worker interfaces and report
normalization alone are not proof of an end-to-end receipt path.

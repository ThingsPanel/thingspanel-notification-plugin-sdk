# Plugin security and operational rules

## Credentials and logs

- Mark every top-level credential as a manifest `secret_fields` entry. Keep
  secrets out of `identity_fields` unless the field is a public account
  identity such as a provider key ID.
- Never log plugin bearer tokens, provider access keys, webhook URLs containing
  tokens, signatures, recipient addresses, message/template values, complete
  provider response bodies, or receipt credentials.
- Errors returned by provider adapters must use fixed safe codes/messages.
  Avoid wrapping raw `net/http`, SMTP, SDK, URL, request, or response errors in
  logs or `DeliveryError` fields.
- Keep plugin bearer and receipt credentials separate. Manifest and health
  endpoints are authenticated by the plugin bearer too.

## Network boundaries

- Use fixed HTTPS service endpoints or operator-configured exact allowlists;
  tenant request content must not choose the host, proxy, or redirect target.
- Disable environment proxy inheritance and redirects for provider calls.
- Resolve and pin allowlisted hosts when the adapter supports DNS controls.
  Refuse loopback, metadata, link-local, multicast, and reserved destinations
  unless a separately controlled private-service exception is explicitly part
  of the provider design.
- Bound connection, TLS, response-header, and total request time. Bound every
  response body before decoding.
- Never rely only on an HTTP status to infer no side effect. For non-idempotent
  calls, unclear transport or final-response outcomes become `unknown` and
  must not be automatically retried.

## Browser access

CORS is disabled by default. When a browser flow is needed, configure exact
UI origins through `NOTIFICATION_PLUGIN_UI_ORIGINS`; do not reflect arbitrary
origins, enable credentials, or widen methods/headers. CORS preflight must not
call the provider. CORS is not a replacement for bearer authorization.

## Receipts and state claims

An `accepted` send means provider submission acceptance. Only a durable,
authenticated, correctly correlated provider receipt can support a delivered
state. A queue interface, report normalizer, or provider `BizId` alone is not a
receipt pipeline. Set receipt capability false until the complete provider to
plugin to core path and its ACK/retry behavior are verified.

## Real-send test gate

Production account tests require a separate explicit authorization that names
the account, exact test recipient/target, attempt count, and cost ceiling. This
documentation task and the checked-in fixtures do not authorize real sends.

# Develop and package notification plugins

Each provider is an independently versioned Go repository that implements the frozen plugin HTTP v1 contract. Start from [`../examples/minimal-plugin`](../examples/minimal-plugin/); it includes an SDK source snapshot, binds only to loopback, and always rejects sends without network I/O.

## Local development

```sh
go test ./...
go vet ./...
(cd examples/minimal-plugin && go test ./... && go vet ./... && go build ./cmd/plugin)
```

The template is intentionally a standalone module with no `require` or `replace` directives and no external dependencies. Copy it to a new provider repository, choose a real module path you control, and implement the SDK `Plugin` interface. The provider repositories for SMTP, DingTalk, Alibaba Cloud SMS, and legacy webhook each carry an SDK source snapshot so they remain independently buildable. When updating a snapshot, compare it with this repository's `sdk/` and `mock/` packages and run all local tests.

## Implementation checklist

1. Define an exact manifest for implemented channels and capabilities only.
2. Make validation side-effect free. Validate config, recipient, content, template allowlists, and expiry before contacting a provider.
3. Fix the provider host or enforce an operator-owned exact allowlist; disable redirects and proxy inheritance, bound connection/TLS/response time, and cap response bodies.
4. Disable automatic retries for non-idempotent sends. A result is `rejected` only when the plugin can prove the provider did not accept the attempt. Uncertain writes become `unknown` and non-retryable.
5. Test acceptance, explicit refusal, disconnect after write begins, redirects, oversized or malformed replies, validation no-send, secret redaction, and result-union correctness using local fixtures.
6. Add receipt handling only after authenticity validation, durable buffering, dedicated Core authentication, stable event identity, and Core ACK behavior are tested end to end.

## Repository build gates

Run tests, vet, and build with `GOPROXY=off GOSUMDB=off` where dependencies permit. Check `go.mod` and `go.sum` for unexpected modules, run the secret scan, review every tracked file, and verify there are no Core/Encore imports, credentials, private reports, or runtime configuration. Never run real provider sends as part of ordinary CI.

## Package a provider executable

From the provider repository, test and build a Linux release binary without downloading modules:

```sh
GOPROXY=off GOSUMDB=off go test ./...
GOPROXY=off GOSUMDB=off go vet ./...
mkdir -p dist
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 GOPROXY=off GOSUMDB=off \
  go build -trimpath -o dist/notification-plugin ./cmd/plugin
go version -m dist/notification-plugin
shasum -a 256 dist/notification-plugin > dist/SHA256SUMS
```

Repeat the build with the target `GOOS` and `GOARCH` for each supported deployment platform. Keep `dist/`, provider credentials, and instance configuration out of Git. Review the executable, checksum, module path, manifest capabilities, and release notes before tagging or publishing a version. Do not report `accepted` as delivery, enable receipt capability without a verified provider-to-Core ACK path, or use real-send smoke tests as routine packaging checks.

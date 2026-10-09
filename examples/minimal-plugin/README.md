# Standalone no-send plugin example

This self-contained Go module demonstrates the frozen plugin HTTP v1 shape. It includes an SDK source snapshot under `sdk/`, a tiny provider, and SDK handler contract tests. The snapshot is kept local so the example builds with no remote Go modules and no Encore/Core imports.

The executable binds to loopback only. Its `/v1/send` implementation always returns `rejected` with `example_no_external_send`; it never sends an email or makes a provider call.

```sh
GOPROXY=off GOSUMDB=off go test ./...
GOPROXY=off GOSUMDB=off go vet ./...
GOPROXY=off GOSUMDB=off go build ./cmd/plugin
NOTIFICATION_PLUGIN_TOKEN=local-fixture-token go run ./cmd/plugin
```

In another local shell, query the authenticated manifest at `http://127.0.0.1:5080/v1/manifest`. The token is supplied through the environment and is not compiled into the example. Keep real tokens private.

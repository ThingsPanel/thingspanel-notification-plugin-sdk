# ThingsPanel notification plugin SDK

Portable Go SDK and contract reference for independent notification plugins using the frozen plugin HTTP v1 API. This repository has no Encore dependency and no external Go dependencies. The `examples/minimal-plugin` module vendors the matching SDK source snapshot and always rejects sends without network I/O.

The public module path is `github.com/ThingsPanel/thingspanel-notification-plugin-sdk`. The API contract is [`specs/notification/plugin-v1.openapi.yaml`](specs/notification/plugin-v1.openapi.yaml); see [SDK behavior](docs/SDK-API.md), [development and packaging](docs/DEVELOPMENT.md), [security](docs/SECURITY.md), and the [manifest example](docs/manifest.example.json).

## Build and test

```sh
go test ./...
go vet ./...
(cd examples/minimal-plugin && go test ./... && go vet ./... && go build ./cmd/plugin)
```

No command in the example contacts a provider. Provider credentials and production targets are not included. `accepted` means provider submission acceptance, not delivery; uncertain sends are `unknown` and must not be retried automatically.

This project is licensed under Apache-2.0. Provider plugins are released independently; this SDK repository does not claim that unimplemented in-app, Aliyun voice, Aliyun email, AWS, APP, or Twilio adapters exist.

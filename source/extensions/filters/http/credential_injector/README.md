# Credential Injector (`envoy.filters.http.credential_injector`)

Injects a credential (e.g. Basic, Bearer, OAuth2 client-credentials) into outgoing HTTP request headers before the request leaves Envoy. The actual credential source is pluggable via the `NamedCredentialInjectorConfigFactory` registry; this filter only wires the selected injector to the request path and counts outcomes.

Proto: `envoy.extensions.filters.http.credential_injector.v3.CredentialInjector`. Registered both as a downstream and an upstream HTTP filter (`config.cc:71-74`).

## Files
- `credential_injector_filter.h/cc` — `FilterConfig` (thread-shared) + `CredentialInjectorFilter` (per-stream, decoder only).
- `config.h/cc` — `CredentialInjectorFilterFactory` (a `DualFactoryBase`, also aliased as `UpstreamCredentialInjectorFilterFactory`).

## Lifecycle
`CredentialInjectorFilter` extends `Http::PassThroughDecoderFilter` (`credential_injector_filter.h:61`). The factory looks up the nested `credential` typed extension via `Config::Utility::getFactory<NamedCredentialInjectorConfigFactory>` (`config.cc:22`), translates its config, and calls `createCredentialInjectorFromProto` (`config.cc:36`) to build a `CredentialInjectorSharedPtr`. A single `FilterConfig` (holding the injector, `overwrite`, `allow_request_without_credential`, stats) is shared across workers, and each request gets a fresh `CredentialInjectorFilter` added via `addStreamDecoderFilter` (`config.cc:44`).

Overridden callback:
- `decodeHeaders` (`credential_injector_filter.cc:48`): calls `FilterConfig::injectCredential`, then either continues the chain or emits a `401 Unauthorized` with rc-details `failed_to_inject_credential` (`credential_injector_filter.cc:52-55`).

No encode-side overrides; injection happens only on the request path.

## Decision / logic
All branching lives in `FilterConfig::injectCredential` (`credential_injector_filter.cc:19-43`):
- `injector_->inject(headers, overwrite_)` returns an `absl::Status`.
- `absl::IsAlreadyExists(status)` (only possible when `overwrite_ == false`): increment `already_exists_`, log trace, continue (`credential_injector_filter.cc:24-30`).
- Any other non-OK status: increment `failed_`, log debug, and return `allow_request_without_credential_` — when true, the request proceeds with no credential; when false, the filter rejects with `401` (`credential_injector_filter.cc:33-37`, `credential_injector_filter.cc:52`).
- OK status: increment `injected_` and continue (`credential_injector_filter.cc:41-42`).

The `ASSERT(!overwrite_)` at `credential_injector_filter.cc:25` documents the invariant that injectors only return `AlreadyExists` when overwrite was disabled.

## Configuration
- `credential` (`TypedExtensionConfig`) — selects and configures the credential provider implementation.
- `overwrite` — if true, the injector replaces any existing credential header; if false, the existing header is preserved and `already_exists` is incremented.
- `allow_request_without_credential` — controls the fail-open vs. fail-closed behavior when injection fails.

No `createRouteSpecificFilterConfigTyped` is provided, so there is no per-route override; the same `FilterConfig` applies stream-wide.

## Stats
Emitted under `<stats_prefix>credential_injector.`:
- `injected` — credential was added/replaced successfully (`credential_injector_filter.cc:41`).
- `failed` — injector returned a non-OK, non-`AlreadyExists` status (`credential_injector_filter.cc:35`).
- `already_exists` — credential already present and `overwrite` is false (`credential_injector_filter.cc:27`).

Defined via `ALL_CREDENTIAL_INJECTOR_STATS` (`credential_injector_filter.h:17-21`).

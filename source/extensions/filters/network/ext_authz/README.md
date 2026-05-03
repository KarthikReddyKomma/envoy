# External Authorization (L4) (`envoy.filters.network.ext_authz`)

A `Network::ReadFilter` that performs a one-shot external authorization (`CheckRequest`) on the first byte of application data for a TCP connection. While the check is in flight the filter chain is paused; on a deny/error response the connection is terminated (optionally with a TLS fatal alert), otherwise buffered bytes are released to downstream read filters.

Proto: `envoy.extensions.filters.network.ext_authz.v3.ExtAuthz`.

## Files
- `config.h/cc` — `ExtAuthzConfigFactory`, extends `Common::FactoryBase`. Builds the shared `Config`, constructs a `GrpcClientImpl` from the proto `grpc_service` (default 200 ms timeout) and registers the filter on the manager (`config.cc:23-45`). Registered under the legacy name `envoy.ext_authz` (`config.cc:50-51`).
- `ext_authz.h/cc` — `Config` (immutable per-listener state, stats + toggles) and `Filter` (the `ReadFilter`/`ConnectionCallbacks`/`ExtAuthz::RequestCallbacks`).

## Lifecycle
- `initializeReadFilterCallbacks()` stores the callbacks pointer and registers the filter as a `ConnectionCallbacks` listener so it sees close events (`ext_authz.h:129-132`).
- `onNewConnection()` returns `Continue` immediately — the check is deferred until the first `onData` so the `CheckRequest` can include TLS, peer cert and SNI info that is populated after the initial handshake (`ext_authz.cc:94-97`, comment at 86).
- `onData(data, end_stream)` checks `filter_enabled_metadata` against `streamInfo().dynamicMetadata()` and increments `disabled_` + continues if disabled (`ext_authz.cc:79-83`). If the state machine is still `NotStarted` it calls `callCheck()`. The return is `StopIteration` while waiting for the authz response and `Continue` once allowed (`ext_authz.cc:90-91`).
- `onWrite` — not implemented (read filter only).
- `onEvent(event)` — on `RemoteClose`/`LocalClose` while `status_ == Calling`, cancels the outstanding gRPC check and decrements `active_` (`ext_authz.cc:99-109`).

## State machine
Two enums track progress (`ext_authz.h:146-150`):
- `Status { NotStarted, Calling, Complete }` — position in the check round-trip.
- `FilterReturn { Stop, Continue }` — what `onData` should return while in `Calling`.

`callCheck()` (`ext_authz.cc:56-77`):
1. `fillMetadataContext` copies filter-metadata entries matching `metadata_context_namespaces` / `typed_metadata_context_namespaces` from the connection stream-info into a fresh `Metadata` proto (`ext_authz.cc:27-46`).
2. `CheckRequestUtils::createTcpCheck` populates the request with peer identity, TLS session info and destination labels (`ext_authz.cc:64-66`).
3. Records the monotonic start time, flips `status_ = Calling`, bumps `active_`/`total_`, and issues `client_->check(...)`. The synchronous/asynchronous completion is tracked via `calling_check_` so the `continueReading()` call in `onComplete` is skipped for inline completion (`ext_authz.cc:73-76`, `180-183`).

`onComplete(response)` (`ext_authz.cc:111-185`) finalizes:
- Per-status stats: `ok_`, `error_`, `denied_` (`ext_authz.cc:115-135`).
- On `OK` records `ext_authz_duration` (ms) into `response->dynamic_metadata` (`ext_authz.cc:118-128`).
- Sets `streamInfo().dynamicMetadata()` under key `envoy.filters.network.ext_authz` if the response carries any fields (`ext_authz.cc:137-141`).
- Deny **or** (Error and !`failure_mode_allow`): increments `cx_closed_`, optionally invokes `SSL_send_fatal_alert(ssl, SSL_AD_ACCESS_DENIED)` when `send_tls_alert_on_denial_` and a TLS connection is present (`ext_authz.cc:149-162`), then closes with `NoFlush` and tags response flag `UnauthorizedExternalService` + `AuthzDenied`/`AuthzError` response code details (`ext_authz.cc:164-170`).
- Otherwise sets `filter_return_ = Continue` (optionally bumping `failure_mode_allowed_`) and calls `continueReading()` unless the completion happened inline (`ext_authz.cc:171-184`).

## Configuration (highlights)
Read by `Config` ctor (`ext_authz.h:52-76`):
- `stat_prefix` → prefix for all emitted stats (`ext_authz.cc:50-54`).
- `failure_mode_allow` — fail-open on transport/gRPC errors.
- `include_peer_certificate`, `include_tls_session` — controls TLS details on the request.
- `send_tls_alert_on_denial` — TLS fatal alert before close.
- `filter_enabled_metadata` — optional `MetadataMatcher` gate consulted on every `onData`.
- `metadata_context_namespaces` / `typed_metadata_context_namespaces` — forwarded filter-metadata keys.
- `bootstrap_metadata_labels_key` — node metadata labels copied into `destination_labels` and forwarded in the check.
- `grpc_service` — authorization service; `timeout` defaults to 200 ms (`config.cc:28`).

No per-route config; filter is strictly per-listener.

## Stats
Emitted under `ext_authz.<stat_prefix>.` (`ext_authz.cc:50-54`). From `ALL_TCP_EXT_AUTHZ_STATS` (`ext_authz.h:29-37`):
- `cx_closed` — connections closed by the filter (deny or fail-closed error).
- `denied` — `CheckStatus::Denied`.
- `error` — `CheckStatus::Error`.
- `failure_mode_allowed` — errors that were allowed through due to `failure_mode_allow`.
- `ok` — `CheckStatus::OK`.
- `total` — checks initiated.
- `disabled` — checks skipped due to `filter_enabled_metadata`.
- `active` (gauge) — checks currently in flight.

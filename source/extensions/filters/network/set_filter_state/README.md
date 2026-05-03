# Set Filter State (`envoy.filters.network.set_filter_state`)

Non-terminal L4 filter that writes values into the connection-scoped `StreamInfo::FilterState` at specific lifecycle points: when the connection is first accepted, when the downstream TLS handshake completes, and/or when the first downstream data arrives. It reuses `Filters::Common::SetFilterState::Config` (shared with the HTTP filter of the same name) so value formatting, object factories, shared state, and read-only/mutable semantics are identical across protocols. Useful for stamping per-connection keys that later filters (tcp_proxy, router, dynamic forward proxy, etc.) consume via filter state.

Proto: `envoy.extensions.filters.network.set_filter_state.v3.Config`.

## Files
- `config.h` — the `SetFilterState` read filter class: three `ConfigSharedPtr` slots plus bitfield flags to gate one-shot execution.
- `config.cc` — filter method bodies and `SetFilterStateConfigFactory` registration.

The filter depends on the shared helper at `source/extensions/filters/common/set_filter_state/` (`Filters::Common::SetFilterState::Config::updateFilterState`).

## Lifecycle
- `initializeReadFilterCallbacks()` (config.h:27) stashes the callbacks and decides how the TLS-handshake hook will fire:
  - If `on_downstream_tls_handshake` is configured and the downstream connection has an `ssl()` session, set `waiting_for_downstream_tls_handshake_` and register as a `ConnectionCallbacks` listener (config.h:30-32).
  - If `on_downstream_tls_handshake` is configured but the connection is plaintext, set `apply_downstream_tls_handshake_on_new_connection_` so the hook runs during `onNewConnection` — mirroring `tcp_proxy` behaviour (config.h:33-37).
  - If `on_downstream_data` is configured, arm `waiting_for_downstream_data_` (config.h:40-42).
- `onNewConnection()` (config.cc:19) — runs `on_new_connection_->updateFilterState(...)` when configured (config.cc:20-22), then runs the TLS hook once if the non-TLS shortcut was armed (config.cc:23-25). Returns `Continue`.
- `onData()` (config.cc:29) — one-shot: the first call clears `waiting_for_downstream_data_` and runs `on_downstream_data_->updateFilterState(...)` (config.cc:30-33). Subsequent `onData` calls are no-ops. Returns `Continue`.
- `onEvent()` (config.cc:37) — `ConnectionCallbacks`. On `Connected` with `waiting_for_downstream_tls_handshake_` true it calls `onDownstreamTlsHandshake()` once (config.cc:41-45). On `LocalClose` / `RemoteClose` it clears the waiting flag so no further work is attempted after teardown (config.cc:47-51).
- No `onWrite()` — this is a `ReadFilter`.

## Decision / logic
- TLS vs plaintext branch (config.h:29-37): decides whether the handshake hook is deferred to a `Connected` event or fired immediately from `onNewConnection`.
- One-shot guards: `downstream_tls_handshake_` and `waiting_for_downstream_data_` are bitfield booleans ensuring each hook runs at most once per connection (config.h:57-60, config.cc:54-61).
- `onDownstreamTlsHandshake()` (config.cc:54) sets `downstream_tls_handshake_ = true`, clears both `waiting_for_downstream_tls_handshake_` and `apply_downstream_tls_handshake_on_new_connection_`, and invokes `on_downstream_tls_handshake_->updateFilterState` if the config is present.
- All three configs use `StreamInfo::FilterState::LifeSpan::Connection` (config.cc:79, 87, 93) so values live for the connection only.

## Configuration
- `on_new_connection` (repeated `FilterStateValue`) — applied during `onNewConnection` (config.cc:77-80).
- `on_downstream_tls_handshake` (repeated `FilterStateValue`) — applied after downstream TLS completes; or at new-connection time for plaintext sockets (config.cc:82-88).
- `on_downstream_data` (repeated `FilterStateValue`) — applied on the first `onData` callback (config.cc:90-95).

Each `FilterStateValue` entry carries the key, factory name, value (with substitution-format support), `read_only`/`shared_with_upstream`/`skip_if_empty` semantics, as defined by `Filters::Common::SetFilterState::Config`.

## Stats
None. The filter has no stats counters or gauges; it is purely a data-transfer filter.

## Factory
`SetFilterStateConfigFactory` (config.cc:66) extends `Common::FactoryBase` under the literal name `"envoy.filters.network.set_filter_state"` (config.cc:70) and registers via `REGISTER_FACTORY` as a `NamedNetworkFilterConfigFactory` (config.cc:111). `isTerminalFilterByProtoTyped` returns `false` (config.cc:104-108). The factory builds up to three `Filters::Common::SetFilterState::Config` instances (skipping any empty `on_*` list, config.cc:76-95) and returns a lambda that constructs a `SetFilterState` read filter with them (config.cc:97-101).

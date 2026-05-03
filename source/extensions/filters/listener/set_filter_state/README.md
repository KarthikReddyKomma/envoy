# Set Filter State Listener Filter (`envoy.filters.listener.set_filter_state`)

Writes a configurable set of filter-state values at connection lifespan when a new connection is accepted. The actual parsing/evaluation of the `FilterStateValue` entries is delegated to the shared `Filters::Common::SetFilterState::Config` helper so the same grammar (object factories, formatters, lifespan handling) is reused by the HTTP and network `set_filter_state` filters. Used to seed connection-scoped state such as tenant IDs, policy keys, or address objects that downstream listener filters, network filters, or the router will later consume.

Proto: `envoy.extensions.filters.listener.set_filter_state.v3.Config`.

## Files
- `config.h` — `SetFilterState` class: a trivial listener filter that holds a `Filters::Common::SetFilterState::ConfigSharedPtr` and exposes `onAccept`/`onData`/`maxReadBytes` (the latter two are inline no-ops returning `Continue` and 0, `config.h:20`-`config.h:23`).
- `config.cc` — `SetFilterState::onAccept` implementation plus `SetFilterStateConfigFactory`: builds the shared config only if `on_accept` entries are provided and installs the filter via `filter_manager.addAcceptFilter`; registers under `envoy.filters.listener.set_filter_state` (`config.cc:27`-`config.cc:52`).

## Lifecycle
- `onAccept(cb)` — when a shared `on_accept_` config is present, calls `on_accept_->updateFilterState({}, cb.streamInfo())`, which evaluates each configured entry and writes into `streamInfo().filterState()` at the connection lifespan; returns `Continue` unconditionally (`config.cc:14`-`config.cc:19`). The empty parameter list `{}` means the common helper is not given request headers, which is correct for a listener filter context where no request exists yet.
- `onData()` — returns `Continue` (unused).
- `maxReadBytes()` — returns 0 (no byte inspection).

## Decision / logic
- Registration is gated on the proto having at least one `on_accept` entry; otherwise `on_accept_config` remains null and the filter turns into a no-op (`config.cc:36`-`config.cc:39`). Lifespan is fixed to `StreamInfo::FilterState::LifeSpan::Connection` because a listener filter runs before any request exists.
- No modifications are made to `ConnectionSocket` state (SNI, ALPN, addresses); only `cb.streamInfo().filterState()` is mutated.

## Configuration
- `on_accept[]` — list of `FilterStateValue` entries evaluated at `onAccept`. Each entry specifies `object_key`, factory key, shared/read-only/mutable state type, and a value (raw bytes, JSON, or formatter expression); the shared `Filters::Common::SetFilterState::Config` resolves these at construction time (`config.cc:37`).

## Stats
None emitted directly by this filter.

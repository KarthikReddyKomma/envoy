# Set Filter State (`envoy.filters.http.set_filter_state`)

A decode-only HTTP filter that writes entries into the per-stream `StreamInfo::FilterState` at the start of a request. Values are computed from substitution formatters over the request headers, so operators can populate filter-state keys that downstream filters, clusters, or transport sockets consume (e.g. upstream host override, SNI, ALPN). The heavy lifting is delegated to the shared `Filters::Common::SetFilterState::Config` library; this folder only wires it into the HTTP filter chain and layers listener, virtual-host, and route-level configs.

Proto: `envoy.extensions.filters.http.set_filter_state.v3.Config`.

## Files
- `config.h` — declares the `SetFilterState` filter (a `PassThroughDecoderFilter`) and the `SetFilterStateConfig` factory with its three `createFilterFactory...` variants plus `createRouteSpecificFilterConfigTyped`.
- `config.cc` — implements `decodeHeaders`, the two factory variants, the per-route factory, and `REGISTER_FACTORY` (`config.cc:81`).

No separate `*_filter.cc` exists; the filter class is small and lives in `config.cc`.

## Lifecycle
Installed as a decoder filter (`config.cc:46-47`, `config.cc:76-77`). The shared `Filters::Common::SetFilterState::Config` is constructed once at factory time with `LifeSpan::FilterChain` (`config.cc:42-44`) and is stored by `shared_ptr` on every filter instance.

Overridden callbacks:
- `decodeHeaders(RequestHeaderMap&, bool)` (`config.cc:22-36`) — executes the state updates and returns `Continue`.

No other decoder or encoder callbacks are overridden; the base `PassThroughDecoderFilter` continues the rest unchanged.

## Decision / logic
Inside `decodeHeaders`:
- `config.cc:24`: applies the listener-level policy first: `config_->updateFilterState({&headers}, decoder_callbacks_->streamInfo())`.
- `config.cc:27-31`: walks all per-filter configs attached to the route using `Http::Utility::getAllPerFilterConfig<Filters::Common::SetFilterState::Config>(decoder_callbacks_)`. This picks up virtual-host and route overrides in order; each layer calls `updateFilterState` on the same stream, so later layers can add, overwrite, or shadow keys.
- `config.cc:32-34`: if the listener-level config requested `clear_route_cache` and the filter is running downstream, it calls `downstreamCallbacks()->clearRouteCache()` so that route selection is recomputed after filter-state mutations (useful when subsequent filters route on state that this filter just wrote).

The filter always returns `FilterHeadersStatus::Continue`; it never stops iteration or sends a local reply.

## Configuration
- `on_request_headers` — repeated substitution-formatted entries consumed by `Filters::Common::SetFilterState::Config` to produce typed filter-state objects (`config.cc:43`, `config.cc:59`, `config.cc:73`).
- `clear_route_cache` (bool) — listener-level only; see decision logic above (`config.cc:32`).

Per-route configuration:
- `createRouteSpecificFilterConfigTyped` (`config.cc:51-61`) builds a standalone `const Filters::Common::SetFilterState::Config` using a `GenericFactoryContextImpl`. Route- and virtual-host-level configs are applied in `decodeHeaders` via `getAllPerFilterConfig` (`config.cc:27`), which merges configs from all tiers.

Three factory entry points exist to support different call sites:
- `createFilterFactoryFromProtoTyped(FactoryContext&)` (`config.cc:38-49`).
- `createFilterFactoryFromProtoWithServerContextTyped(ServerFactoryContext&)` (`config.cc:63-79`) — used where only a `ServerFactoryContext` is available; wraps it in a `GenericFactoryContextImpl` (the TODO at `config.cc:67-69` flags a known validation-visitor bug).

## Stats
None. The filter emits no counters, gauges, or histograms.

## Factory
`SetFilterStateConfig` (`config.h:31`):
- `FactoryBase<envoy::extensions::filters::http::set_filter_state::v3::Config>` with name `"envoy.filters.http.set_filter_state"` (`config.h:34`).
- Registered via `REGISTER_FACTORY(SetFilterStateConfig, NamedHttpFilterConfigFactory)` (`config.cc:81`).

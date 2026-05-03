# On-Demand (`envoy.filters.http.on_demand`)

Decoder filter that performs on-demand xDS discovery when a route (VHDS) or cluster (OD-CDS) isn't
yet present at request time. If the route cannot be resolved, it requests a route-config update
and pauses the filter chain; once the update has been pushed to the worker, the stream is either
recreated (body fully buffered) or resumed in-place. If the route resolves but its cluster is
unknown, it kicks off an on-demand CDS fetch via a configured `OdCdsApiHandle` with a bounded
timeout, then recreates or resumes the stream based on the discovery result.

Proto: `envoy.extensions.filters.http.on_demand.v3.OnDemand` (+ `PerRouteConfig`).

## Files
- `config.h/cc` — `OnDemandFilterFactory` (downstream + server-context factories, per-route).
- `on_demand_update.h/cc` — `OnDemandFilterConfig`, `DecodeHeadersBehavior`
  (`RdsDecodeHeadersBehavior`, `RdsCdsDecodeHeadersBehavior`), `OnDemandRouteUpdate` filter,
  OdCDS-API handle allocation.

## Lifecycle
- Factory builds `OnDemandFilterConfig` once from the proto (choosing behavior based on whether
  `odcds` is set), then each worker receives
  `addStreamDecoderFilter(std::make_shared<OnDemandRouteUpdate>(config))` (`config.cc:18-20`,
  `config.cc:28-30`).
- `OnDemandFilterConfig` (`on_demand_update.h:39-55`) owns a `DecodeHeadersBehaviorPtr`:
  - `rds()` ⇒ `RdsDecodeHeadersBehavior`, which only invokes `handleMissingRoute`.
  - `cdsRds(odcds, timeout)` ⇒ `RdsCdsDecodeHeadersBehavior`, which runs route discovery first
    and then, if a route is found, on-demand CDS (`on_demand_update.cc:20-41`).
- `createDecodeHeadersBehavior` (`on_demand_update.cc:56-111`) resolves the OdCDS API handle:
  - When `xdstp_based_config_singleton_subscriptions` is on and no `source`/`resources_locator` is
    set, uses `XdstpOdCdsApiImpl::create` for singleton subscription
    (`on_demand_update.cc:63-73`).
  - When `resources_locator` is empty but source is ADS and `odcds_over_ads_fix` is on, again
    uses `XdstpOdCdsApiImpl::create` (`on_demand_update.cc:82-90`).
  - Otherwise falls back to `OdCdsApiImpl::create` (`on_demand_update.cc:92-96`) or decodes
    `resources_locator` into an xDS `ResourceLocator` (`on_demand_update.cc:98-105`).
  - Default timeout is 5000 ms (`on_demand_update.cc:108-109`).

## Decode path
- `decodeHeaders` (`on_demand_update.cc:164-171`): saves `end_stream`, resolves the effective
  config via `getConfig()` (per-route preferred), invokes
  `config->decodeHeadersBehavior().decodeHeaders(*this)`, and returns
  `filter_iteration_state_` (set to `StopIteration` whenever discovery was actually requested).
- `handleMissingRoute` (`on_demand_update.cc:147-162`): if route already resolved, sets
  `Continue` and returns it; otherwise installs a `RouteConfigUpdatedCallback`, calls
  `downstreamCallbacks()->requestRouteConfigUpdate(...)`, and returns the route pointer (still
  likely null). The flag `decode_headers_active_` is set around this to prevent
  `onRouteConfigUpdateCompletion` from calling `continueDecoding` while still inside
  `decodeHeaders`.
- `handleOnDemandCds` (`on_demand_update.cc:174-202`): if `clusterInfo()` already present, or no
  route-entry, or empty `clusterName()` ⇒ `Continue`. Otherwise installs a
  `ClusterDiscoveryCallback` and issues `odcds.requestOnDemandClusterDiscovery(cluster_name,
  callback, timeout)`; keeps the handle in `cluster_discovery_handle_`; sets state to
  `StopIteration`.
- `decodeData` (`on_demand_update.cc:212-217`): if iteration was stopped, returns
  `StopIterationAndWatermark` to hold body; otherwise `Continue`.
- `decodeTrailers` (`on_demand_update.cc:219-222`): records `downstream_end_stream_` and
  continues.
- `onDestroy` (`on_demand_update.cc:232-235`): resets callback handles so late discovery
  completions cannot reanimate a destroyed filter.
- `getConfig` (`on_demand_update.cc:204-210`): `resolveMostSpecificPerFilterConfig` wins; else the
  filter-level config.

## Completion callbacks
- `onRouteConfigUpdateCompletion(route_exists)` (`on_demand_update.cc:240-266`):
  - Sets iteration state back to `Continue`.
  - If still inside `decodeHeaders` (re-entrancy via the RDS provider), returns without resuming —
    `decodeHeaders` itself will propagate the right status.
  - Determines `can_recreate_stream`: if `on_demand_track_end_stream` runtime feature is on, uses
    `downstream_end_stream_`; else uses the legacy "no decoding buffer" test.
  - If the route was discovered and the stream is safe to recreate ⇒ `recreateStream(nullptr)`;
    otherwise ⇒ `continueDecoding()`.
- `onClusterDiscoveryCompletion(status)` (`on_demand_update.cc:268-302`):
  - Resets discovery handle.
  - If `on_demand_cluster_no_recreate_stream` runtime flag is on, unconditionally
    `continueDecoding()` so later filters can reselect the cluster.
  - Otherwise, if the cluster became `Available` and `can_recreate_stream`, call
    `recreateStream(nullptr)` and `clearRouteCache()`; else `continueDecoding()`.

## Decision / logic
- Runtime flags (`on_demand_update.cc:63`, `82`, `249`, `274`, `283`) control xDS-TP singleton
  behavior, ADS-OD-CDS path, end-stream tracking, and the cluster-no-recreate behavior.
- `downstream_end_stream_` is tracked through `decodeHeaders`, `decodeData`, `decodeTrailers` so
  that stream recreation only occurs when the body was fully read (otherwise the recreated stream
  would be missing data).

## Configuration
- `OnDemand.odcds`: optional `OnDemandCds { source, resources_locator, timeout }`. Without it,
  only on-demand RDS is enabled (`on_demand_update.cc:59-61`).
- `PerRouteConfig` has the same shape; per-route config wins at request time via
  `resolveMostSpecificPerFilterConfig` (`on_demand_update.cc:204-210`).

## Factory
- `OnDemandFilterFactory` registered as `envoy.filters.http.on_demand` (via `HttpFilterNames`
  constant) — `REGISTER_FACTORY(OnDemandFilterFactory, NamedHttpFilterConfigFactory)`
  (`config.cc:45`). Both `createFilterFactoryFromProtoTyped` and
  `createFilterFactoryFromProtoWithServerContextTyped` supported (`config.cc:12-31`).
- Per-route via `createRouteSpecificFilterConfigTyped` returning another `OnDemandFilterConfig`
  (`config.cc:33-40`).

## Stats
- None emitted by this filter itself. Discovery outcomes surface via the RDS / CDS subsystems'
  own stats (cluster-manager and route-config-provider counters).

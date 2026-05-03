# EDS Cluster (`envoy.cluster.eds`)

Endpoint Discovery Service cluster: builds its host set from a streaming xDS subscription to `envoy.config.endpoint.v3.ClusterLoadAssignment` resources. An optional second subscription layer (LEDS - Locality Endpoint Discovery Service) lets each locality pull its `LbEndpoint`s from a separate xDS collection, which is useful when endpoints are numerous and locality-sharded.

Note: a legacy, more prose-heavy README already exists in this folder as `EDS_CLUSTER.md`; this file is the developer-oriented summary that matches the format of the other cluster extension READMEs.

Proto: the cluster is driven directly by the core `envoy.config.cluster.v3.Cluster` message (no extension-specific config message). The consumed resources are `envoy.config.endpoint.v3.ClusterLoadAssignment` and `envoy.config.endpoint.v3.LbEndpoint` (for LEDS).

## Files
- `eds.h` / `eds.cc` - `EdsClusterImpl`, `BatchUpdateHelper`, `EdsClusterFactory`.
- `leds.h` / `leds.cc` - `LedsSubscription` and `LedsSubscriptionPtr`.
- `EDS_CLUSTER.md` - pre-existing long-form documentation.

## Cluster type
- `EdsClusterImpl` extends `Upstream::BaseDynamicClusterImpl` and multiply inherits:
  - `Config::SubscriptionBase<ClusterLoadAssignment>` - integrates with the xDS subscription machinery (`eds.h:33`).
  - `Config::EdsResourceRemovalCallback` - receives cache-eviction notifications when an xDS resource cache is configured (`eds.h:34`, `eds.h:76`).
- `EdsClusterFactory` extends `ClusterFactoryImplBase` ("envoy.cluster.eds", `eds.h:136`).
- `initializePhase()` is computed at construction (`eds.cc:40-46`): `Primary` when the xDS config source is path-based (file-based, no transport dependency), otherwise `Secondary` (the EDS subscription rides over another cluster).

## Initialization / host set
- Ctor (`eds.cc:28-64`):
  - Creates an `assignment_timeout_` timer that triggers `onAssignmentTimeout()` when a warming update does not arrive.
  - Gets the optional `edsResourcesCache()` from the cluster manager.
  - Builds the `subscription_` either via `xdsManager().subscribeToSingletonResource(...)` when runtime flag `envoy.reloadable_features.xdstp_based_config_singleton_subscriptions` is on, or via the legacy `subscriptionFactory().subscriptionFromConfigSource(...)`.
- `startPreInit()` calls `subscription_->start({edsServiceName()})` (`eds.cc:73`); `edsServiceName()` returns `info_->edsServiceName()` when set, else the cluster name (`eds.h:67-70`).
- `onConfigUpdate(resources, version)` path (state-of-the-world) and the delta path at `onConfigUpdate(added, removed, version)` both funnel into `update(cluster_load_assignment)`.
- `update(...)` stores the assignment (`cluster_load_assignment_` is owned because LEDS subscriptions reference its `LocalityLbEndpoints` long after the call returns), creates/updates the per-locality `LedsSubscription`s for any locality carrying `leds_cluster_locality_config`, and when all LEDS localities have received their first update (`validateAllLedsUpdated()`, `eds.h:84`) it runs `BatchUpdateHelper::batchUpdate` through `priority_set_.batchHostUpdate(...)`.
- `BatchUpdateHelper::batchUpdate` (`eds.cc:75-150+`) walks every `LocalityLbEndpoints`, calls `priority_state_manager.initializePriorityFor(...)`, and then emits `LbEndpoint`s either directly or from the matching `LedsSubscription::getEndpointsMap()`. It then calls `parent_.updateHostsPerLocality(...)` per priority using `crossPriorityHostMap()` as the existing-host index to keep stable `HostSharedPtr`s across updates.
- `updateHostsPerLocality(...)` (`eds.h:59-65`) diffs new vs. old hosts, updates `locality_weights_map_[priority]`, and calls `PrioritySet::updateHosts(...)`.
- `onAssignmentTimeout()` triggers a warming-failure path that can fall back to the EDS resource cache (see `using_cached_resource_`, `eds.h:129`).
- `onCachedResourceRemoved(name)` (from `EdsResourceRemovalCallback`) clears out the fallback when the cache entry goes away (`eds.h:76`).

## Load balancing hooks
- The factory returns a `nullptr` `ThreadAwareLoadBalancer` (`eds.cc` create path), so the top-level `Cluster::lb_policy` (ROUND_ROBIN, LEAST_REQUEST, RING_HASH, MAGLEV, etc.) is applied against the `PrioritySet` that EDS maintains.
- `reloadHealthyHostsHelper(host)` (`eds.h:79`) is an override used by health-check state transitions to rebuild healthy host views without re-issuing xDS.

## Key decision points
- Runtime flag that switches to singleton xDS subscriptions - `eds.cc:48-63`.
- Path-based config -> Primary init phase - `eds.cc:40-46`.
- LEDS readiness gate before applying an update - `eds.h:84`, `eds.cc:89` (`ASSERT(it->second->isUpdated())`).
- Overprovisioning factor and weighted priority health pulled from `load_assignment.policy()` on every update - `eds.cc:110-113`.
- Priorities absent from the new update are explicitly emptied - `eds.cc:139-149`.

## Configuration
- `eds_cluster_config.service_name` (defaults to cluster name via `edsServiceName()`).
- `eds_cluster_config.eds_config` - any `core.ConfigSource` (ADS, GRPC, path, xdstp).
- Assignment warming timeout and EDS cache behavior come from cluster-manager-level knobs.

## Stats
- Standard `ClusterImplBase` config-update stats: `update_attempt`, `update_success`, `update_no_rebuild`, `update_failure`, `update_rejected`.
- Subscription-level stats (`update_time`, version gauges) emitted by `SubscriptionBase`.
- Membership and load-balancer stats from `BaseDynamicClusterImpl`.

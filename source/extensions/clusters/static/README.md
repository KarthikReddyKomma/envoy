# Static Cluster (`envoy.cluster.static`)

The simplest cluster type: every host is listed verbatim in the config with a resolved IP address. No DNS, no xDS, no lazy creation - `startPreInit()` just publishes the pre-built priority set and the cluster is immediately usable. Health checkers, if configured, fire once the initial hosts are registered.

Proto: driven directly by `envoy::config::cluster::v3::Cluster.load_assignment` (no extension-specific config message).

## Files
- `static_cluster.h` / `static_cluster.cc` - `StaticClusterImpl`, `StaticClusterFactory`.

## Cluster type
- `StaticClusterImpl` extends `Upstream::ClusterImplBase` (`static_cluster.h:17`). It is *not* a `BaseDynamicClusterImpl` because the host set never changes after init.
- `initializePhase()` returns `Primary` (`static_cluster.h:20`); nothing to wait for.
- `StaticClusterFactory` extends `ClusterFactoryImplBase` and registers `envoy.cluster.static` (`static_cluster.h:41-44`).

## Initialization / host set
- Ctor (`static_cluster.cc:12-57`):
  - Builds a `PriorityStateManager` with a `nullptr` HostUpdateCb (no batching needed, this is pre-init).
  - Reads `overprovisioning_factor` and `weighted_priority_health` from `load_assignment.policy()` with the default `kDefaultOverProvisioningFactor` when unset (`static_cluster.cc:20-22`).
  - Iterates every `LocalityLbEndpoints` and every `LbEndpoint` (`static_cluster.cc:24-53`):
    - For each endpoint, resolves the primary address via `resolveProtoAddress(lb_endpoint.endpoint().address())`.
    - If the endpoint declares `additional_addresses`, prepends the primary and appends each additional address to an `address_list` (happy-eyeballs list) (`static_cluster.cc:27-46`). Unless runtime feature `envoy.reloadable_features.happy_eyeballs_sort_non_ip_addresses` is on, every additional address must be an IP address or the cluster fails with `throwEnvoyExceptionOrPanic(...)`.
    - Calls `priority_state_manager_->registerHostForPriority(hostname, address, address_list, locality_lb_endpoint, lb_endpoint)`.
  - Validates the whole set via `validateEndpoints(cluster_load_assignment.endpoints(), priority_state_manager_->priorityState())` (`static_cluster.cc:55-56`).
- `startPreInit()` (`static_cluster.cc:59-79`):
  - If a health checker is attached, seeds every host with `Host::HealthFlag::FAILED_ACTIVE_HC` so the HC machinery starts them as unhealthy and fires healthy transitions as checks pass.
  - Walks each priority in `priority_state_` and calls `updateClusterPrioritySet(...)` to push the hosts into the real priority set.
  - Releases the `priority_state_manager_` and calls `onPreInitComplete()`.
- Factory (`static_cluster.cc:81-100`) rejects any locality that declares `leds_cluster_locality_config` - LEDS is only supported in EDS clusters.

## Load balancing hooks
- Factory returns a `nullptr` `ThreadAwareLoadBalancer` (`static_cluster.cc:97`). The top-level `Cluster::lb_policy` applies (ROUND_ROBIN, RANDOM, RING_HASH, MAGLEV, LEAST_REQUEST, etc.). No cluster-specific host selection.

## Key decision points
- `FAILED_ACTIVE_HC` marking on first push when a health checker exists, so traffic only flows after HC passes - `static_cluster.cc:62-65`.
- LEDS rejection at factory time - `static_cluster.cc:87-93`.
- `additional_addresses` validity gated by a runtime feature - `static_cluster.cc:37-45`.
- `priority_state_manager_.reset()` after push to free the temporary builder - `static_cluster.cc:76`.

## Configuration
- `load_assignment`: the full `ClusterLoadAssignment` with resolved IPs, localities, metadata, weights, and health status.
- `load_assignment.policy.overprovisioning_factor` and `weighted_priority_health`.
- Standard `Cluster` fields: `lb_policy`, `health_checks`, `circuit_breakers`, `transport_socket`, etc.

## Stats
- Inherits all `ClusterImplBase` stats. Because membership is static, `update_attempt` / `update_success` fire at most once at init.

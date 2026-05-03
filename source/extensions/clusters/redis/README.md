# Redis Cluster (`envoy.clusters.redis`)

Cluster implementation for `Redis Cluster` (sharded Redis). The host set is produced by periodically issuing `CLUSTER SLOTS` (and optionally `INFO` for zone discovery) against a discovery node; the response describes slot-to-node mappings, which are turned into `RedisHost` objects and a slot-indexed load balancer that the `redis_proxy` network filter uses to route commands by key slot.

A legacy, long-form developer doc already lives in this folder (`REDIS_CLUSTER.md`); this README is the concise, file:line-referenced summary matching the other cluster extension READMEs.

Proto: `envoy.extensions.clusters.redis.v3.RedisClusterConfig` (see `api/envoy/extensions/clusters/redis/v3/redis_cluster.proto`).

## Files
- `redis_cluster.h` / `redis_cluster.cc` - `RedisCluster`, inner `RedisHost`, `DnsDiscoveryResolveTarget`, `RedisDiscoveryClient`, `RedisDiscoverySession`, `ZoneDiscoveryCallback`, `RedisClusterFactory`.
- `redis_cluster_lb.h` / `redis_cluster_lb.cc` - `ClusterSlot`, `ClusterSlots`, `RedisClusterLoadBalancer`, `RedisLoadBalancerContextImpl` (CRC16-based slot picker).
- `crc16.h` / `crc16.cc` - Redis CRC16 implementation used for hash-slot calculation.
- `REDIS_CLUSTER.md` - pre-existing long-form documentation.

## Cluster type
- `RedisCluster` extends `Upstream::BaseDynamicClusterImpl` (`redis_cluster.h:91`): hosts arrive asynchronously from topology discovery.
- `initializePhase()` returns `InitializePhase::Primary` (`redis_cluster.h:126`).
- `RedisClusterFactory` (`redis_cluster.h:335`) is `ConfigurableClusterFactoryBase<RedisClusterConfig>` registered as `envoy.clusters.redis`.

## Initialization / host set
- Ctor (`redis_cluster.cc:42-69`):
  - Copies refresh timers from the proto: `cluster_refresh_rate` (default 5s), `cluster_refresh_timeout` (default 3s), `redirect_refresh_interval` (default 5s), `redirect_refresh_threshold` (default 5), `failure_refresh_threshold`, `host_degraded_refresh_threshold`.
  - Builds one `DnsDiscoveryResolveTarget` per configured bootstrap endpoint (`redis_cluster.cc:52-58`). These exist only to bootstrap the very first topology fetch by DNS-resolving seed names.
  - Creates a shared `RedisDiscoverySession` that owns Redis connections to cluster members (`redis_cluster.cc:48`).
  - Registers the cluster with a process-wide `Common::Redis::ClusterRefreshManager` (`redis_cluster.cc:62-68`). The manager tracks redirects / failures and pokes the discovery timer when a threshold is crossed. Cluster is held by `weak_ptr` to survive the cluster's destruction safely.
- `startPreInit()` (`redis_cluster.cc:85-92`) calls `startResolveDns()` on every seed target; once DNS resolves, the discovery session issues `CLUSTER SLOTS`.
- `RedisDiscoverySession::onResponse(...)` validates the slots payload, resolves any hostnames returned by Redis back to IPs, and calls `onClusterSlotUpdate(slots, host_zone_map)`.
- `onClusterSlotUpdate(...)` (`redis_cluster.cc:108+`) walks the slot ranges, creates or reuses `RedisHost` entries (primary + replicas), feeds them through `updateDynamicHostList(...)` to compute the added/removed diffs, and finally calls `updateAllHosts(added, removed, priority)` which rebuilds the single priority via `PriorityStateManager`.
- It also notifies the slot-aware load-balancer factory (`lb_factory_`, a `ClusterSlotUpdateCallBackSharedPtr`) so that the `redis_proxy` filter's LB can see the new slot-to-host mapping.
- Optional zone discovery (`enable_zone_discovery`): per host, the session sends `INFO`, parses `availability_zone` via `RedisCluster::parseAvailabilityZone(...)` (`redis_cluster.h:129`), and stamps each `RedisHost`'s locality zone via `makeLocalityWithZone(...)` (`redis_cluster.cc:24-32`). Results flow back through `ZoneDiscoveryCallback`.

## Load balancing hooks
- The factory returns `nullptr` for the `ThreadAwareLoadBalancer` - the `redis_proxy` network filter installs its own `RedisClusterLoadBalancer` via the `lb_factory_` passed in at cluster construction (`redis_cluster.h:320`). This LB is slot-indexed: a `RedisLoadBalancerContext` exposes the CRC16 slot of the command key, which selects the primary/replica for that slot.

## Key decision points
- Discovery driver: DNS for bootstrap seeds, then a persistent Redis connection on one of the discovered hosts - `redis_cluster.cc:85-92` and `RedisDiscoverySession::startResolveRedis`.
- Refresh cadence is governed by timers + the shared `ClusterRefreshManager`, which can be nudged by MOVED/ASK redirect rates from the data plane - `redis_cluster.cc:62-68`.
- The destructor clears `redis_discovery_session_` and `dns_discovery_resolve_targets_` *before* other members to prevent callbacks firing mid-teardown; `is_destroying_` guards any late invocations - `redis_cluster.cc:71-83`.
- Primary vs replica hosts are distinguished via `RedisHost::primary_` (`redis_cluster.h:181`) and used by the LB to honor `ReadPolicy`.

## Configuration
- `cluster_refresh_rate`, `cluster_refresh_timeout`, `redirect_refresh_interval`, `redirect_refresh_threshold`, `failure_refresh_threshold`, `host_degraded_refresh_threshold`.
- `enable_zone_discovery` - toggles `INFO`-based zone lookup.
- Authentication is drawn from the `redis_proxy` protocol options: `auth_username_` and `auth_password_` are resolved at cluster construction (`redis_cluster.cc:49`).

## Stats
- Inherits `ClusterImplBase` stats.
- Discovery stats: `update_attempt`, `update_success`, `update_no_rebuild`, `update_failure`.
- Additional DNS stats are emitted via `updateDnsStats(...)` during bootstrap resolution.

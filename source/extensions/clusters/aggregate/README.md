# Aggregate Cluster (`envoy.clusters.aggregate`)

An aggregate cluster presents a linear priority list built from several existing clusters. It does not produce its own host set; instead it borrows the `PrioritySet` of each referenced thread-local cluster and re-keys those priorities into a single combined priority space. Failover / locality routing is then applied across the concatenation, so traffic first exhausts cluster[0] priorities, then falls through to cluster[1], etc.

Proto: `envoy.extensions.clusters.aggregate.v3.ClusterConfig` (see `api/envoy/extensions/clusters/aggregate/v3/cluster.proto`).

## Files
- `cluster.h` / `cluster.cc` - `Cluster`, `AggregateClusterLoadBalancer`, `AggregateLoadBalancerFactory`, `AggregateThreadAwareLoadBalancer`, `ClusterFactory`.
- `lb_context.h` - `AggregateLoadBalancerContext`: wraps the downstream `LoadBalancerContext` and overrides `determinePriorityLoad()` so the child cluster's LB is forced to route to the already-chosen linearized priority slot.

## Cluster type
- Base class `Upstream::ClusterImplBase` (`cluster.h:40`). No hosts of its own - `startPreInit()` immediately calls `onPreInitComplete()` (`cluster.h:64`), marking it ready.
- `initializePhase()` returns `Secondary` (`cluster.h:43-45`): it depends on the referenced clusters existing in the cluster manager, so it is warmed in the second initialization phase.

## Initialization / host set
- Constructor copies `config.clusters()` into an immutable `ClusterSet` (`cluster.cc:15-20`).
- No host resolution. Membership is computed lazily per worker by `AggregateClusterLoadBalancer` via `linearizePrioritySet()` (`cluster.cc:55-102`).
- `linearizePrioritySet()` walks the configured cluster list in order; for each non-empty `HostSet` of each child cluster it:
  - copies the hosts into `PriorityContext::priority_set_` under a fresh linearized priority index (`cluster.cc:86-95`),
  - records `(child_cluster_name, child_priority) -> linearized_priority` in `cluster_and_priority_to_linearized_priority_` (`cluster.cc:93-94`),
  - records `(child_priority, ThreadLocalCluster*)` in `priority_to_cluster_` (`cluster.cc:90-91`).

## Load balancing hooks
- `AggregateThreadAwareLoadBalancer::factory()` (`cluster.h:166`) returns a `LoadBalancerFactory` that creates one `AggregateClusterLoadBalancer` per worker (`cluster.h:151-155`).
- On construction each worker LB subscribes to `ClusterManager::addThreadLocalClusterUpdateCallbacks` and, for every referenced cluster already present, installs a member-update callback (`cluster.cc:22-41`, `cluster.cc:43-53`). Any add/update/remove of a tracked child triggers `refresh()` (`cluster.cc:104-113`, `115-135`), rebuilding the linearized `PrioritySet` and the inner `LoadBalancerImpl`.
- `chooseHost()` (`cluster.cc:150-172`): runs `determinePriorityLoad` across the linearized priorities using `hostToLinearizedPriority` as the retry priority remap, picks a linearized priority with `choosePriority`, then delegates to that child cluster's own load balancer - wrapping the context in `AggregateLoadBalancerContext` so the child's `determinePriorityLoad` is short-circuited to the already-chosen priority (`lb_context.h:38-57`).
- `peekAnotherHost`, `selectExistingConnection`, `lifetimeCallbacks` forward to the inner LB (`cluster.cc:182-206`).

## Key decision points
- Linearized priority layout - `cluster.cc:60-99` (explanatory comment + build loop).
- Exclude-on-remove dance to avoid dangling `ThreadLocalCluster*` - `cluster.cc:126-135`.
- Retry priority remap - `hostToLinearizedPriority`, `cluster.cc:137-148`.
- Load assignment back to child LB happens in `LoadBalancerImpl::chooseHost` - `cluster.cc:150-172`.

## Configuration
- `clusters`: repeated string - names of the clusters being aggregated, in priority order.

## Stats
- No stats are emitted by the aggregate cluster itself beyond the standard `ClusterImplBase` counters; the referenced child clusters publish their own stats.

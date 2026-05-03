# Composite Cluster (`envoy.clusters.composite`)

A composite cluster is an *ordered list of delegate clusters* that is walked per retry attempt: the 1st attempt uses `clusters[0]`, the 2nd uses `clusters[1]`, and so on. It holds no hosts itself; every `chooseHost()` picks the child cluster matching the current router attempt counter and then delegates host selection to that child's own load balancer. The use case is "retry this request on a completely different cluster" rather than "retry on a different host of the same cluster". It is also the transport underneath `mcp_multicluster`.

Proto: `envoy.extensions.clusters.composite.v3.ClusterConfig` (see `api/envoy/extensions/clusters/composite/v3/cluster.proto`).

## Files
- `cluster.h` / `cluster.cc` - `Cluster`, `CompositeClusterLoadBalancer`, `CompositeLoadBalancerFactory`, `CompositeThreadAwareLoadBalancer`, `ClusterFactory`.
- `lb_context.h` - `CompositeLoadBalancerContext` (wrapper context passed down to the child LB; carries the cluster index so child LBs see a consistent view).

## Cluster type
- Base class `Upstream::ClusterImplBase` (`cluster.h:25`). No host set of its own; `startPreInit()` immediately calls `onPreInitComplete()` (`cluster.h:41`).
- `initializePhase()` returns `Secondary` (`cluster.h:28`) - the cluster depends on its children being registered, so it is warmed in phase two.

## Initialization / host set
- Constructor (`cluster.cc:15-26`) copies `config.clusters()` (a `repeated Cluster`) into an immutable `ClusterSet` of names (only the `name` field is retained - `cluster.cc:22-23`).
- The composite cluster has no endpoints. Hosts are discovered on demand inside the LB by calling `ClusterManager::getThreadLocalCluster(name)` (`cluster.cc:78`).

## Load balancing hooks
- `CompositeThreadAwareLoadBalancer::factory()` (`cluster.h:93`) installs `CompositeLoadBalancerFactory`; each worker gets its own `CompositeClusterLoadBalancer` (`cluster.h:83`, `cluster.cc:28-33`).
- `CompositeClusterLoadBalancer` subscribes to `ClusterManager::addThreadLocalClusterUpdateCallbacks` (`cluster.cc:32`) but only uses the callback for logging - it does not rebuild state when a child cluster changes (`cluster.cc:86-100`).
- `chooseHost()` (`cluster.cc:102-133`):
  1. Reads `StreamInfo::attemptCount()` via `getAttemptCount()` (`cluster.cc:35-48`). Envoy's router increments this to 1 for the first try.
  2. Maps `attempt_count -> cluster_index = attempt_count - 1` via `mapAttemptToClusterIndex()` (`cluster.cc:50-67`); attempt 0 is treated as invalid, attempts beyond `clusters_->size()` return `nullopt`, which fails the selection.
  3. Resolves that index to a `ThreadLocalCluster*` with `getClusterByIndex()` (`cluster.cc:69-84`).
  4. Wraps the downstream context in `CompositeLoadBalancerContext(context, cluster_index)` (`cluster.cc:129`) and delegates to `cluster->loadBalancer().chooseHost(&composite_context)` (`cluster.cc:132`).
- `peekAnotherHost` / `selectExistingConnection` repeat the same attempt->cluster mapping and delegate (`cluster.cc:135-171`).
- `lifetimeCallbacks()` always returns `{}` (`cluster.cc:173-177`) - aggregating lifetime callbacks across heterogeneous children is not implemented.

## Key decision points
- Attempt-to-index translation (1-based router attempts, 0-based vector) - `cluster.cc:50-67`.
- Fail-closed when attempt count exceeds configured clusters - `cluster.cc:65-66`.
- Fail-closed when the named child cluster is absent from the cluster manager - `cluster.cc:79-82`.
- `onClusterAddOrUpdate` / `onClusterRemoval` do not refresh any state (`cluster.cc:86-100`) - the composite is stateless between calls.

## Configuration
- `clusters`: repeated `Cluster` messages. Only the `name` of each entry is read (`cluster.cc:22-23`). Order is the retry order.

## Stats
- None specific to this cluster. Standard `ClusterImplBase` stats are still emitted; per-attempt traffic shows up on whichever child cluster was selected.

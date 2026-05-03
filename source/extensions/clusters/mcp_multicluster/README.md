# MCP Multicluster Cluster (`envoy.clusters.mcp_multicluster`)

Thin adapter over the `composite` cluster purpose-built for the MCP (Model Context Protocol) HTTP-to-JSON bridge filter. At config time the factory rewrites the user-supplied `mcp_multicluster.v3.ClusterConfig` into an equivalent `composite.v3.ClusterConfig` (one entry per `servers[i].mcp_cluster.cluster`) and defers to the composite cluster factory to do the heavy lifting. The original proto is stashed into typed filter metadata so downstream MCP filters can still read it from the cluster.

Proto: `envoy.extensions.clusters.mcp_multicluster.v3.ClusterConfig` (see `api/envoy/extensions/clusters/mcp_multicluster/v3/cluster.proto`).

## Files
- `cluster.h` / `cluster.cc` - `ClusterFactory` only. No custom `Cluster` class, no custom LB.

## Cluster type
- There is no dedicated cluster class. The actual runtime cluster is `Extensions::Clusters::Composite::Cluster` (base `Upstream::ClusterImplBase`), constructed by the embedded `composite_cluster_factory_` member (`cluster.h:26`).
- Factory registration name: `envoy.clusters.mcp_multicluster` (`cluster.h:20`). The factory extends `Upstream::ConfigurableClusterFactoryBase<mcp_multicluster::v3::ClusterConfig>`.

## Initialization / host set
- `ClusterFactory::createClusterWithConfig` (`cluster.cc:17-29`):
  1. Clones the incoming `envoy::config::cluster::v3::Cluster` into `cluster_with_metadata` (`cluster.cc:20`).
  2. Packs the original MCP proto under `metadata.typed_filter_metadata["envoy.clusters.mcp_multicluster"]` (`cluster.cc:21`) so the MCP HTTP filter can retrieve it via `ClusterInfo::metadata()`.
  3. Builds a `composite::v3::ClusterConfig` by iterating `proto_config.servers()` and calling `add_clusters()->set_name(server.mcp_cluster().cluster())` (`cluster.cc:24-26`).
  4. Forwards to `composite_cluster_factory_.createClusterWithConfig(...)` which returns the pair `(ClusterImplBaseSharedPtr, ThreadAwareLoadBalancerPtr)` - concretely a `Composite::Cluster` plus `CompositeThreadAwareLoadBalancer`.
- Because the actual cluster is composite, there are no hosts of its own; child clusters are resolved lazily via `ClusterManager::getThreadLocalCluster(name)`. See `../composite/README.md` for the composite lifecycle.

## Load balancing hooks
- Provided entirely by composite: `CompositeThreadAwareLoadBalancer -> CompositeLoadBalancerFactory -> CompositeClusterLoadBalancer`. `chooseHost()` maps router attempt count to child cluster index. See `../composite/cluster.cc:102-133`.

## Key decision points
- MCP config survives as typed filter metadata on the composite cluster - `cluster.cc:21`.
- Ordering of `servers[i]` in the MCP proto becomes the retry ordering consumed by composite's attempt-to-index mapping - `cluster.cc:24-26`.
- No re-implementation of lifecycle: everything rides on composite. Bugs in attempt handling or child-cluster lookup live in `../composite/cluster.cc`.

## Configuration
- `servers`: repeated `McpServer`-shaped message whose `mcp_cluster.cluster` names a cluster that must already exist (or arrive via CDS) in the cluster manager.
- All other fields on the top-level `Cluster` (connect_timeout, circuit_breakers, etc.) are carried verbatim through `cluster_with_metadata`.

## Stats
- None specific to mcp_multicluster; traffic stats live on the selected child cluster picked by composite.

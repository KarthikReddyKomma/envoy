# Cluster Provided (`envoy.load_balancing_policies.cluster_provided`)

A stub load-balancing policy that tells Envoy "the cluster type owns host
selection". No Envoy-side picker is constructed here. Use this when the cluster
implementation itself supplies the load balancer (e.g. the original destination
cluster picks the host from request metadata, or a custom cluster type returns
its own LB). Any of the normal LB algorithms would be meaningless in that case.

Proto: `envoy.extensions.load_balancing_policies.cluster_provided.v3.ClusterProvided`.

## Files
- `config.h` — defines the registered factory `Factory` under the name
  `envoy.load_balancing_policies.cluster_provided` and an empty
  `ClusterProvidedLbConfig` so the generic LB config machinery has something to
  hand back. `loadConfig`/`loadLegacy` just return the empty config.
- `config.cc` — implements `Factory::create`, which returns `nullptr`, and
  registers the factory via `REGISTER_FACTORY`.

## Load balancer class
None. There is no concrete `LoadBalancer`. `Factory::create` explicitly returns
a null `ThreadAwareLoadBalancerPtr` so that the cluster manager routes through
the cluster's own host-selection path instead of an LB.

## Algorithm
Not applicable. Host selection is delegated to the cluster implementation; see
the `OriginalDstCluster` or any custom cluster that declares itself
`CLUSTER_PROVIDED`.

## Key decision points
- Factory identity: `config.h:20-25`.
- `Factory::create` returning `nullptr` (the whole point of this policy):
  `config.cc:8-15`.
- Registration: `config.cc:20`.

## Configuration
The proto has no fields. The LB config object stores nothing.

## Stats
None specific to this policy.

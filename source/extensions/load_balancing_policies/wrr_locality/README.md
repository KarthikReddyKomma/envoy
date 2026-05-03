# WRR Locality (`envoy.load_balancing_policies.wrr_locality`)

A thin wrapper that combines locality-weighted load balancing with a
configurable endpoint-picking policy. Today the only supported
endpoint-picking policy is `envoy.load_balancing_policies.client_side_weighted_round_robin`;
this wrapper forces the common LB config to include
`locality_weighted_lb_config` before delegating, so the child LB honors
xDS-provided locality weights. Use this when you want locality-weighted
distribution (e.g. cross-region weighting) on top of ORCA-driven client-side
round robin.

Proto: `envoy.extensions.load_balancing_policies.wrr_locality.v3.WrrLocality`.

## Files
- `config.h/cc` — `Factory` registered as
  `envoy.load_balancing_policies.wrr_locality`. `loadConfig` walks
  `endpoint_picking_policy.policies` to find the first registered LB factory,
  enforces that it is a `ClientSideWeightedRoundRobin::Factory`, parses the
  child proto, and packages the child factory + child config into a
  `WrrLocalityLbConfig`.
- `wrr_locality_lb.h/cc` — `WrrLocalityLbConfig` (typed config wrapper),
  `WrrLocalityLoadBalancer` (thread-aware LB that owns the child's thread-aware
  LB), and `WorkerLocalLbFactory` that builds the per-worker child LB with
  locality weighting forced on.

## Load balancer class
`WrrLocalityLoadBalancer` implements `Upstream::ThreadAwareLoadBalancer` and
delegates both `factory()` and `initialize()` through to the child
`ClientSideWeightedRoundRobinLoadBalancer` (so the ORCA weight manager timer
still starts and the EDF refresh fan-out still works).

## Algorithm
Host selection is performed by the child LB — this wrapper does not implement
`chooseHostOnce` itself. Its only contribution is forcing locality weighting
on:

- `WrrLocalityLoadBalancer` construction (`wrr_locality_lb.cc:8-19`):
  1. Downcast the `WrrLocalityLbConfig` off the incoming `LoadBalancerConfig`.
  2. Call the child factory's `create` with the child config to get a
     `ThreadAwareLoadBalancerPtr` for CSWRR.
  3. Wrap that LB's factory in a `WorkerLocalLbFactory` that intercepts
     `create(params)` on the worker thread.

- `WorkerLocalLbFactory::create` (`wrr_locality_lb.cc:21-34`):
  1. Downcast the held factory to
     `ClientSideWeightedRoundRobinLoadBalancer::WorkerLocalLbFactory`.
  2. Copy `cluster_info.lbConfig()` and call `mutable_locality_weighted_lb_config()`
     on the copy — this enables locality weighting even if the cluster config
     didn't request it.
  3. Call `createWithCommonLbConfig(modified_common_config, params)` on the
     CSWRR worker factory, which constructs a `WorkerLocalLb`
     (`RoundRobinLoadBalancer`) parametrized with the overridden common LB
     config.

## Key decision points
- Endpoint-picking policy resolution and type enforcement: `config.h:39-77`
  (reject anything other than CSWRR with `InvalidArgumentError`).
- Forcing locality weighted balancing on the common LB config:
  `wrr_locality_lb.cc:31-32`.
- Creating the per-worker LB through CSWRR's factory:
  `wrr_locality_lb.cc:33`.
- Registration: `config.cc:11`.

## Configuration
Proto fields:
- `endpoint_picking_policy.policies[]` — ordered list of LB extensions. The
  first registered one wins. Must resolve to the CSWRR policy today; its
  `typed_config` is parsed and forwarded as the child config.

No other fields. Locality weighting comes from the cluster's locality weights
and is always enabled here.

## Stats
Inherits from the child LB. Adds no stats of its own.

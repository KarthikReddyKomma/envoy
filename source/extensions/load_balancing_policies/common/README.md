# Common Load Balancing Helpers

Shared building blocks used by all the concrete load-balancing policies. There
is no registered LB policy in this directory; the files here are linked in by
the policy-specific BUILD targets (`round_robin`, `least_request`, `random`,
`ring_hash`, `maglev`, `subset`, `client_side_weighted_round_robin`,
`wrr_locality`). It provides the base classes that implement panic mode,
priority load distribution, zone-aware routing, the EDF-based weighted-RR
scheduler, the thread-aware LB skeleton used by hash-based policies, the
locality weighted-RR helper, the generic factory template, and the ORCA
weight manager.

## Files
- `factory_base.h` — `FactoryBase<ProtoType, Impl>` template that concrete
  policies inherit to wire a `ThreadAwareLoadBalancer` wrapper around a
  stateless worker factory. Also defines `ActiveOrLegacy<ActiveType, LegacyType>`,
  a helper that downcasts an LB config pointer to either the new proto-based
  config or the legacy one.
- `load_balancer_impl.h/cc` — The heart of the directory. Defines:
  - `LoadBalancerBase` — common state (stats, random, panic threshold) and
    priority load distribution (`distributeLoad`, `recalculatePerPriorityState`,
    `recalculatePerPriorityPanic`, `chooseHostSet`,
    `choosePriority`).
  - `ZoneAwareLoadBalancerBase` — adds `hostSourceToUse`, locality routing,
    `chooseHost` that calls the subclass's `chooseHostOnce` with retry handling
    (`load_balancer_impl.cc:658-676`).
  - `EdfLoadBalancerBase` — weighted RR using an `EdfScheduler<Host>` per
    `HostsSource`. Falls back to a simple index when all host weights are equal.
    Owns slow-start state and applies it via `applySlowStartFactor`.
  - `LoadBalancerConfigHelper` — free helpers that copy slow-start, locality,
    and consistent-hash options from the legacy `CommonLbConfig` into the new
    per-policy protos.
- `locality_wrr.h/cc` — `LocalityWrr` builds one EDF scheduler per
  health class (healthy / degraded) across localities so
  `ZoneAwareLoadBalancerBase` can pick a locality in proportion to its
  effective weight.
- `orca_weight_manager.h/cc` — `OrcaLoadReportHandler`, `OrcaHostLbPolicyData`
  (per-host atomic weight + timestamps), and `OrcaWeightManager` which owns a
  main-thread timer, iterates the priority set, applies blackout/expiration
  windows, and calls a `WeightsUpdatedCb` when weights change. Used by
  `client_side_weighted_round_robin`.
- `thread_aware_lb_impl.h/cc` — `ThreadAwareLoadBalancerBase` used by the
  hash-based policies (Ring Hash, Maglev). Concrete subclasses implement
  `createLoadBalancer(normalized_host_weights, min, max)` to produce a
  `HashingLoadBalancer`, which is published to workers via a shared
  `LoadBalancerFactoryImpl`. Also provides `BoundedLoadHashingLoadBalancer`,
  the bounded-load variant shared by Ring Hash and Maglev, and
  `TypedHashLbConfigBase`, which compiles hash policies out of the proto.

## Key helpers and their entry points
- EDF pick per request:
  `load_balancer_impl.cc:1145` (`EdfLoadBalancerBase::chooseHostOnce`).
- EDF refresh when hosts or weights change:
  `load_balancer_impl.cc:1035` (`EdfLoadBalancerBase::refresh`).
- Priority/panic selection for the non-hash LBs:
  `load_balancer_impl.cc:379` (`LoadBalancerBase::chooseHostSet`).
- Zone-aware outer loop (retries through `chooseHostOnce`):
  `load_balancer_impl.cc:658` (`ZoneAwareLoadBalancerBase::chooseHost`).
- `HostsSource` selection (zone aware vs. all healthy):
  `load_balancer_impl.cc:837` (`hostSourceToUse`).
- Locality WRR rebuild:
  `locality_wrr.cc` (`LocalityWrr::rebuildLocalityScheduler`,
  `LocalityWrr::effectiveLocalityWeight`).
- Thread-aware publish path:
  `thread_aware_lb_impl.cc` (`ThreadAwareLoadBalancerBase::refresh` and
  `LoadBalancerFactoryImpl::create`).
- ORCA weight update tick: `orca_weight_manager.cc`
  (`OrcaWeightManager::updateWeightsOnMainThread`,
  `OrcaWeightManager::updateWeightsOnHosts`).

## Who uses what
- `round_robin`, `least_request`, `random`, `client_side_weighted_round_robin`
  extend `EdfLoadBalancerBase` (via `RoundRobinLoadBalancer` for the weighted
  variant) and use `FactoryBase`.
- `ring_hash`, `maglev` extend `ThreadAwareLoadBalancerBase` and build
  `HashingLoadBalancer` subclasses.
- `subset` and `override_host` do not extend these bases; they wrap a child
  LB.
- `wrr_locality` reuses `ClientSideWeightedRoundRobinLoadBalancer::WorkerLocalLbFactory`
  and injects a locality-weighted common LB config.
- `client_side_weighted_round_robin` owns an `OrcaWeightManager` to drive its
  weight updates.

## Stats
No stats are registered here. `ClusterLbStats` are owned by the cluster and
passed in; the concrete policies record into them.

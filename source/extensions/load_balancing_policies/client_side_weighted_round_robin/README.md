# Client-Side Weighted Round Robin (`envoy.load_balancing_policies.client_side_weighted_round_robin`)

A round-robin load balancer that computes per-host weights on the client side
from ORCA (Open Request Cost Aggregation) load reports received on upstream
responses. Each worker runs a standard weighted round-robin (EDF) picker, but
the weights themselves are derived from real-time utilization/QPS/EPS metrics
reported by the backends, so over time traffic skews away from overloaded hosts.
Use this when the backends export ORCA load reports (e.g. xDS-managed gRPC
services) and you want Envoy to balance by observed utilization rather than by
a static configured weight.

Proto: `envoy.extensions.load_balancing_policies.client_side_weighted_round_robin.v3.ClientSideWeightedRoundRobin`.

## Files
- `config.h/cc` — `Factory` (thread-aware factory) registered under the name
  `envoy.load_balancing_policies.client_side_weighted_round_robin`. Builds
  `ClientSideWeightedRoundRobinLbConfig` from the proto on the main thread and
  constructs the thread-aware LB.
- `client_side_weighted_round_robin_lb.h/cc` — `ClientSideWeightedRoundRobinLbConfig`
  (typed config wrapper), `ClientSideWeightedRoundRobinLoadBalancer`
  (thread-aware LB), nested `WorkerLocalLb` (per-worker picker that extends
  `RoundRobinLoadBalancer`), `WorkerLocalLbFactory`, and the `ThreadLocalShim`
  used to push weight-refresh callbacks out to every worker thread.

## Load balancer class
- Thread-aware wrapper: `ClientSideWeightedRoundRobinLoadBalancer`
  implements `Upstream::ThreadAwareLoadBalancer`.
- Per-worker picker: `WorkerLocalLb` extends
  `Upstream::RoundRobinLoadBalancer`, which itself extends
  `EdfLoadBalancerBase` (see `../common/`).
- Weight computation: `Extensions::LoadBalancingPolicies::Common::OrcaWeightManager`
  lives on the main thread, owns a timer, and recomputes weights periodically.

## Algorithm
1. On each upstream response, the transport layer delivers the ORCA load report
   into `OrcaHostLbPolicyData::onOrcaLoadReport` attached to that host. The
   handler converts `(qps, eps, utilization, error_utilization_penalty)` into a
   normalized weight and stores it atomically on the host
   (`common/orca_weight_manager.h:73-103`).
2. A periodic timer on the main thread (`weight_update_period`) walks the
   priority set, applies the blackout (`blackout_period`) and expiration
   (`weight_expiration_period`) windows, falls back to the median weight when a
   host has no valid report, and writes the final weight into
   `Host::weight()`.
3. When any weight changes, `WorkerLocalLbFactory::applyWeightsToAllWorkers`
   posts to every worker's `ThreadLocalShim`. Each worker then runs
   `refresh(priority)` on its `EdfLoadBalancerBase`, which rebuilds the EDF
   scheduler for that priority.
4. Per request, `EdfLoadBalancerBase::chooseHostOnce` picks the next host via
   EDF if weights differ, or via a simple round-robin index otherwise
   (see `../common/load_balancer_impl.cc:1145`).

## Key decision points
- Factory create/load hooks: `config.h:26-42`.
- Timing defaults (blackout 10s, expiration 180s, update 1s):
  `client_side_weighted_round_robin_lb.cc:37-42`.
- Per-worker EDF refresh wired through the TLS shim:
  `client_side_weighted_round_robin_lb.cc:58-68`.
- Main-thread kick that refreshes all workers:
  `client_side_weighted_round_robin_lb.cc:84-90`.
- `OrcaWeightManager` wiring at LB construction:
  `client_side_weighted_round_robin_lb.cc:104-115`.
- `initialize()` starts the timer:
  `client_side_weighted_round_robin_lb.cc:118-120`.

## Configuration
Proto fields consumed in `ClientSideWeightedRoundRobinLbConfig`
(`client_side_weighted_round_robin_lb.cc:28-47`):
- `metric_names_for_computing_utilization` — named metrics from the ORCA report
  used to derive utilization.
- `error_utilization_penalty` — how strongly errors inflate utilization.
- `blackout_period` (default 10s) — reports are ignored until a host has been
  reporting this long.
- `weight_expiration_period` (default 180s) — stale reports revert the host to
  the default (median) weight.
- `weight_update_period` (default 1s) — how often the main-thread timer
  recomputes weights.
- `slow_start_config` — forwarded into the worker-local RR config via
  `round_robin_overrides_` so slow-start on new hosts still works.

## Stats
The policy reuses `ClusterLbStats` from the per-worker `RoundRobinLoadBalancer`
(via `EdfLoadBalancerBase`); it adds no additional stats struct of its own.

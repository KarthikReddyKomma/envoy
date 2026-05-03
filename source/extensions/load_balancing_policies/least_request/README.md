# Least Request (`envoy.load_balancing_policies.least_request`)

Least-request load balancing. In the common equal-weight case it uses "power
of N choices" (P2C by default): pick `choice_count` hosts uniformly at random
and route to the one with the fewest active requests. With a `FULL_SCAN`
selection method it instead walks every host and uses reservoir sampling to
break ties uniformly. In the weighted case it falls back to the shared EDF
scheduler with a dynamic weight of
`load_balancing_weight / (active_requests + 1)^active_request_bias`. Use this
when you want simple adaptive balancing that reacts to host saturation and
don't need session affinity.

Proto: `envoy.extensions.load_balancing_policies.least_request.v3.LeastRequest`.

## Files
- `config.h/cc` — `Factory` (`envoy.load_balancing_policies.least_request`)
  that uses the generic `Common::FactoryBase`. `TypedLeastRequestLbConfig`
  wraps the proto and supports the legacy `Cluster.LeastRequestLbConfig` via a
  converting constructor. `LeastRequestCreator::operator()` constructs the
  per-worker `LeastRequestLoadBalancer`.
- `least_request_lb.h/cc` — `LeastRequestLoadBalancer`, including the two
  unweighted pick paths (`unweightedHostPickNChoices`,
  `unweightedHostPickFullScan`) and the weighted `hostWeight` formula.

## Load balancer class
`LeastRequestLoadBalancer` extends `EdfLoadBalancerBase` (`../common/load_balancer_impl.h`).
That gives it zone-aware priority/panic handling, slow-start, and the EDF
scheduler for the weighted path; the unweighted path is the distinctive piece.

## Algorithm
`chooseHostOnce(context)` lives on the base class
(`../common/load_balancer_impl.cc:1145`) and dispatches to either the EDF
scheduler (when weights differ) or `unweightedHostPick` (when all weights are
equal).

Unweighted path in `least_request_lb.cc`:
- `FULL_SCAN` (`least_request_lb.cc:75-117`): iterate all hosts, tracking the
  candidate with the lowest `rq_active`. Ties use reservoir sampling so every
  tied host has equal probability.
- `N_CHOICES` (default, `least_request_lb.cc:119-141`): draw `choice_count`
  random indices (default 2) and return the one with the fewest active
  requests. This is the classic P2C.

Weighted path in `hostWeight` (`least_request_lb.cc:6-47`): the EDF scheduler
queries `hostWeight(host)` on each `pickAndAdd`, so the host's effective
weight is scaled every time it is picked. `active_request_bias` of 1.0 gives
direct inverse scaling; 0.0 degenerates to plain RR; other values use
`pow(rq_active+1, active_request_bias)`. Slow-start (if configured) is applied
last via `applySlowStartFactor`.

`refresh(priority)` (`least_request_lb.h:48-61`) re-reads `active_request_bias`
from runtime each time a host set updates.

`unweightedHostPeek` returns `nullptr` (`least_request_lb.cc:49-55`): pre-connect
peek would race with other threads' picks and is not meaningful for
least-request.

## Key decision points
- Factory create: `config.cc:28-41`.
- Selection method dispatch: `least_request_lb.cc:61-73`.
- P2C loop: `least_request_lb.cc:119-141`.
- Full-scan reservoir sampling of tied hosts: `least_request_lb.cc:99-113`.
- Weighted formula: `least_request_lb.cc:35-40`.
- Active-request-bias runtime refresh: `least_request_lb.h:48-60`.

## Configuration
Proto fields (see `least_request_lb.h:37-43`):
- `choice_count` — N for the "power of N choices" pick. Default 2. Ignored for
  `FULL_SCAN`.
- `active_request_bias` — runtime-backed exponent applied to active-request
  count when weighting. Default 1.0.
- `selection_method` — `N_CHOICES` (default) or `FULL_SCAN`.
- `slow_start_config` — optional slow-start window and aggression.
- `locality_lb_config` — optional locality weighting / zone-aware config,
  converted from the legacy `CommonLbConfig` when needed (`config.cc:12-23`).

## Stats
Uses `ClusterLbStats` from the base class. No `least_request`-specific stats.

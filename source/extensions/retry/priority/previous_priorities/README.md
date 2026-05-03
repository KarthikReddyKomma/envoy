# PreviousPriorities Retry Priority

A retry priority policy that excludes previously-attempted priorities from
load distribution, pushing retries toward untried priority levels. Falls back
to the original priority load when every priority has been attempted.

Proto: `envoy.extensions.retry.priority.previous_priorities.v3.PreviousPrioritiesConfig`

## Files
- `previous_priorities.h` - predicate class and factory helper declarations.
- `previous_priorities.cc` - priority-load recomputation logic.
- `config.h` / `config.cc` - factory registered as
  `envoy.retry_priorities.previous_priorities`.

## Interface
Implements `Upstream::RetryPriority`. The factory implements
`Upstream::RetryPriorityFactory`.

## Logic
State:
- `attempted_hosts_` - every host `onHostAttempted` has been called with.
- `excluded_priorities_` - bool-per-priority flags for priorities that have
  been attempted.
- `per_priority_load_` / `per_priority_health_` / `per_priority_degraded_` -
  cached recomputed loads.

`determinePriorityLoad` is called by the load balancer on each retry. The
flow:
1. If fewer than `update_frequency` retries have happened, return the original
   priority load unchanged (`previous_priorities.cc:16-17`).
2. Otherwise, every `update_frequency` retries, walk `attempted_hosts_`,
   translate each one to its priority via `priority_mapping_func`, and mark
   that priority excluded.
3. Call `adjustForAttemptedPriorities`, which uses the base LB helper
   `LoadBalancerBase::recalculatePerPriorityState` and then redistributes
   100 units of load over the non-excluded priorities proportionally to their
   healthy/degraded availability.
4. If after exclusion the total availability is zero, clear
   `attempted_hosts_` and recompute from scratch - this prevents the policy
   from starving requests when every priority has been tried.
5. If availability is still zero, return the original load (all priorities
   genuinely unavailable).

## Key decision points
- `previous_priorities.cc:16` - retry gating based on `update_frequency_`.
- `previous_priorities.cc:23-28` - marking priorities of attempted hosts as
  excluded.
- `previous_priorities.cc:53-60` - reset-on-exhaustion fallback.
- `previous_priorities.cc:81-99` - load distribution loop (noted as a
  duplicate of `distributeLoad` in the core LB code).

## Configuration
`PreviousPrioritiesConfig.update_frequency` controls how often the priority
load is recomputed; the factory also receives the cluster's `max_retries`
which pre-reserves the `attempted_hosts_` vector.

## Stats
None emitted by this extension; the router's retry counters capture usage.

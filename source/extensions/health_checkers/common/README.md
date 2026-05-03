# Common Health Checker Base

Shared skeleton used by every active health checker in Envoy (HTTP, TCP, gRPC,
Redis, Thrift). Implements the timing, state tracking, transport options,
failure thresholding, event logging, stats, and cross-thread plumbing so that
protocol-specific checkers only need to implement one session.

Proto: not a user-facing extension; driven by `envoy.config.core.v3.HealthCheck`.

## Files
- `health_checker_base_impl.h` - `HealthCheckerImplBase` and its inner
  `ActiveHealthCheckSession`.
- `health_checker_base_impl.cc` - interval/jitter math, cluster member
  update handling, callback fan-out.

## Interface
Provides `Upstream::HealthCheckerImplBase` (inherits `Upstream::HealthChecker`)
and the nested `ActiveHealthCheckSession` base class. Protocol extensions
(http, tcp, grpc, redis, thrift) derive from these and override `makeSession`,
`healthCheckerType`, `onInterval`, `onTimeout`, and `onDeferredDelete`.

## Logic
Per-host lifecycle:
1. `start()` watches `Cluster::prioritySet()` via the member-update callback
   registered in the constructor. When hosts are added, `addHosts` creates an
   `ActiveHealthCheckSession` by calling the subclass `makeSession`.
2. Each session arms an `interval_timer_` (with jitter) that fires
   `onIntervalBase` -> subclass `onInterval`. Parallel `timeout_timer_`
   enforces the per-probe deadline.
3. Subclasses report results via `handleSuccess(degraded)` /
   `handleFailure(type, retriable)` / `setUnhealthy`, which update
   `num_unhealthy_` / `num_healthy_` counters, possibly flip the host's
   `HealthFlag`, fire `HostStatusCb` callbacks, log events, and update
   stats gauges (`healthy`, `degraded`) and counters (`attempt`, `success`,
   `failure`, `network_failure`, `passive_failure`, `verify_cluster`).

Intervals are selected by `interval(state, transition)` with
`intervalWithJitter`: distinct interval values for initial, no-traffic, edge
transitions, unhealthy, and steady-state healthy checks.

## Key decision points
- `health_checker_base_impl.h:44` - `HealthCheckerImplBase` declaration.
- `health_checker_base_impl.h:62-65` - `setUnhealthy` / `start` entry points
  in `ActiveHealthCheckSession`.
- `health_checker_base_impl.cc:15-46` - constructor pulls every timing knob
  out of the proto and subscribes to cluster member updates.
- `health_checker_base_impl.cc:48-73` - transport socket options / match
  metadata initialization shared by all checkers.

## Configuration
Common fields on `envoy.config.core.v3.HealthCheck`: `timeout`, `interval`,
`initial_jitter`, `interval_jitter`, `interval_jitter_percent`,
`unhealthy_threshold`, `healthy_threshold`, `reuse_connection`,
`no_traffic_interval`, `no_traffic_healthy_interval`, `unhealthy_interval`,
`unhealthy_edge_interval`, `healthy_edge_interval`, `tls_options`,
`transport_socket_match_criteria`, `always_log_health_check_failures`,
`always_log_health_check_success`.

## Stats
Defined by `ALL_HEALTH_CHECKER_STATS` in `health_checker_base_impl.h:24-32`:
counters `attempt`, `failure`, `network_failure`, `passive_failure`,
`success`, `verify_cluster`; gauges `healthy`, `degraded`.

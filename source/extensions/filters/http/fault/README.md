# Fault Injection Filter (`envoy.filters.http.fault`)

Injects delay, abort, and response-path rate limiting for resilience testing.
Decisions are probabilistic and route-scoped.

Proto: `envoy.extensions.filters.http.fault.v3.HTTPFault`.

## Lifecycle

- `decodeHeaders()` (`fault_filter.cc:86`) — decides whether to delay or
  abort; if a delay fires, an optional abort is scheduled in the timer
  callback (`fault_filter.cc:381–405`).
- `encodeData()` (`fault_filter.cc:446`) — applies the response rate limiter
  on the way out.

## Decisions

- Feature flags are runtime-driven (`fault.http.*`):
  - `isDelayEnabled()` (`fault_filter.cc:199`)
  - `isAbortEnabled()` (`fault_filter.cc:211`)
- Each fault checks `tryIncActiveFaults()` (`fault_filter.cc:346`) against
  `max_active_faults` before proceeding — prevents resource exhaustion from
  too many in-flight delays.
- Scope filters: `upstream_cluster`, `downstream_nodes`, header matchers
  (`fault_filter.cc:104`, `415`, `426`).

## Configuration

- `delay` — fixed or header-configured delay
  (`fault.http.delay.fixed_duration_ms`).
- `abort` — HTTP or gRPC abort
  (`fault.http.abort.http_status`, `abort_grpc_status`).
- `max_active_faults` — gauge-bounded concurrency limit.
- `response_rate_limit` — kbps throttle on the response body.
- `upstream_cluster`, `downstream_nodes`, `headers` — scope.

## Stats

`aborts_injected`, `delays_injected`, `faults_overflow`,
`response_rl_injected`, and the gauge `active_faults`
(`fault_filter.h:33–38`).

## Files

- `fault_filter.{h,cc}` — filter.
- `config.{h,cc}` — factory.

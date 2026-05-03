# Adaptive Concurrency (`envoy.filters.http.adaptive_concurrency`)

HTTP filter that caps in-flight request concurrency using a feedback loop over observed request latencies. A shared `ConcurrencyController` (currently only `GradientController`) computes a moving concurrency limit from a minimum round-trip time (minRTT) probe and recent RTT samples; the filter asks the controller for a forward/block decision on `decodeHeaders` and feeds it a latency sample once encoding completes.

Proto: `envoy.extensions.filters.http.adaptive_concurrency.v3.AdaptiveConcurrency`.

## Files
- `adaptive_concurrency_filter.h/cc` — `AdaptiveConcurrencyFilter` and `AdaptiveConcurrencyFilterConfig`.
- `config.h/cc` — `AdaptiveConcurrencyFilterFactory`; wires up a singleton `GradientController` per filter-chain.
- `controller/controller.h` — `ConcurrencyController` interface (`forwardingDecision`, `recordLatencySample`, `cancelLatencySample`, `concurrencyLimit`).
- `controller/gradient_controller.{h,cc}` — the shipped controller implementation.

## Lifecycle
Extends `Http::PassThroughFilter` (`adaptive_concurrency_filter.h:54`) and overrides three methods — one on the decode side and two on the encode side.

- `decodeHeaders(headers, end_stream)` (`adaptive_concurrency_filter.cc:42`):
  - Short-circuits with `Continue` when the filter is runtime-disabled or when `streamInfo().healthCheck()` is true — health checks must not bias the latency distribution (`adaptive_concurrency_filter.cc:46-48`).
  - Calls `controller_->forwardingDecision()`. If `Block`, responds with `sendLocalReply(concurrency_limit_exceeded_status, "reached concurrency limit", ..., "reached_concurrency_limit")` and returns `StopIteration` (`adaptive_concurrency_filter.cc:50-55`).
  - On `Forward`, stamps `now = time_source.monotonicTime()` and stores a `Cleanup` RAII that, when destroyed, calls `controller_->recordLatencySample(now)` (`adaptive_concurrency_filter.cc:59-61`).
- `encodeComplete()` (`adaptive_concurrency_filter.cc:66`): drops the `deferred_sample_task_`, which fires the latency-sample callback. This is the normal success path — sample is computed as `now - start`.
- `onDestroy()` (`adaptive_concurrency_filter.cc:68`): if the filter is torn down before `encodeComplete` (e.g. reset, timeout), it `cancel()`s the cleanup (so the latency callback does *not* run) and calls `controller_->cancelLatencySample()` so the controller still decrements its outstanding-request counter.

## Decision / logic
- The Block/Forward branch is the only admission decision. It is the controller's responsibility to track outstanding requests and return `Block` when the current window is full (`controller/controller.h:17-24`).
- Latency samples are only taken for requests that went through `Forward`; the RAII in `decodeHeaders` guarantees exactly one of `recordLatencySample` / `cancelLatencySample` fires per forwarded request.
- `toErrorCode(status)` clamps the configured `concurrency_limit_exceeded_status` to >= 400, falling back to `503 ServiceUnavailable` otherwise (`adaptive_concurrency_filter.cc:20-26`).

## Configuration
- `enabled` — `Runtime::FeatureFlag`, consulted every request via `filterEnabled()` (`adaptive_concurrency_filter.h:35`).
- `concurrency_limit_exceeded_status` — status returned when blocked; normalized by `toErrorCode` (`adaptive_concurrency_filter.cc:36`).
- `concurrency_controller_config` (oneof) — only `gradient_controller_config` is accepted; asserted in the factory (`config.cc:24-25`). The gradient controller itself is configured with `min_rtt_calc_params`, `sample_aggregate_percentile`, `concurrency_limit_params`, etc.
- No per-route configuration.

## Stats
- Filter: emitted under `<stats_prefix>adaptive_concurrency.` (`config.cc:20`).
- Gradient controller: emitted under `<stats_prefix>adaptive_concurrency.gradient_controller.` (`config.cc:30`, `gradient_controller.h:30-37`):
  - counter `rq_blocked`
  - gauges `burst_queue_size`, `concurrency_limit`, `gradient`, `min_rtt_calculation_active`, `min_rtt_msecs`, `sample_rtt_msecs`.

## Factory
`AdaptiveConcurrencyFilterFactory` extends `FactoryBase` with name `envoy.filters.http.adaptive_concurrency` (`config.h:20`). It implements both `createFilterFactoryFromProtoTyped` and `createFilterFactoryFromProtoWithServerContextTyped`, delegating to a single helper that (`config.cc:15-41`):

1. Builds `GradientControllerConfig` and a shared `GradientController` bound to the server main-thread dispatcher, runtime, random generator, and time source.
2. Builds the shared `AdaptiveConcurrencyFilterConfig`.
3. Returns a callback that calls `callbacks.addStreamFilter(...)` with a new `AdaptiveConcurrencyFilter` holding both shared pointers, so all streams share the same controller (`config.cc:37-40`).

Registered via `REGISTER_FACTORY(AdaptiveConcurrencyFilterFactory, NamedHttpFilterConfigFactory)` (`config.cc:46`).

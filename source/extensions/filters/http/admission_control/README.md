# Admission Control (`envoy.filters.http.admission_control`)

Client-side admission control filter. Every encoded response is classified as success/failure by a pluggable `ResponseEvaluator`; these outcomes are accumulated in a thread-local sliding window. On the decode path, the filter computes a rejection probability from the current success rate and probabilistically sheds load with `503` to protect the upstream from a death spiral.

Proto: `envoy.extensions.filters.http.admission_control.v3.AdmissionControl`.

## Files
- `admission_control.h/cc` — `AdmissionControlFilter`, `AdmissionControlFilterConfig`, stats.
- `thread_local_controller.{h,cc}` — `ThreadLocalControllerImpl` maintains a bucketed ring over a sampling window; exposes `recordSuccess`, `recordFailure`, `requestCounts()`, `averageRps()`.
- `evaluators/` — `ResponseEvaluator` interface and `SuccessCriteriaEvaluator` (built from proto `success_criteria`).
- `config.h/cc` — downstream and upstream factories.

## Lifecycle
`AdmissionControlFilter` extends `Http::PassThroughFilter` (`admission_control.h:89`). Overrides one decode and two encode callbacks.

- `decodeHeaders(headers, end_stream)` (`admission_control.cc:80`):
  - If runtime-disabled or `streamInfo().healthCheck()`, sets `record_request_=false` and `Continue` — health checks neither sample nor reject (`admission_control.cc:81-85`).
  - If the controller's `averageRps()` is below `rps_threshold`, skips admission (`Continue`) — not enough traffic to make meaningful decisions (`admission_control.cc:87-91`).
  - Calls `shouldRejectRequest()`. If true, bumps `rq_rejected`, sets `record_request_=false` so the forged 503 is not counted as an outcome, and `sendLocalReply(503, "", ..., "denied_by_admission_control")` (`admission_control.cc:93-104`).
- `encodeHeaders(headers, end_stream)` (`admission_control.cc:109`):
  - No-op when `record_request_` is false (`admission_control.cc:114-116`).
  - For gRPC, tries `getGrpcStatus(headers)`; if absent, flags `expect_grpc_status_in_trailer_=true` and waits for trailers (`admission_control.cc:119-126`).
  - Otherwise calls `responseEvaluator().isGrpcSuccess(...)` / `isHttpSuccess(getResponseStatus(headers))` and invokes `recordSuccess()`/`recordFailure()` which bump `rq_success`/`rq_failure` and feed the controller (`admission_control.cc:128-140`, `admission_control.h:110-118`).
- `encodeTrailers(trailers)` (`admission_control.cc:145`): when we deferred classification to trailers, reads the gRPC status there and records success/failure accordingly.

## Decision / logic
The rejection probability in `shouldRejectRequest()` (`admission_control.cc:161-179`):

```
probability = (total - successes / sr_threshold) / (total + 1)
if aggression != 1.0: probability = probability ^ (1 / aggression)
probability = min(probability, max_rejection_probability)
reject if (accuracy * max(probability, 0)) > (random() % accuracy)
```

- `sr_threshold` is a percentage normalized to `[0, 1]` by dividing by 100 and clamped to 1.0 (`admission_control.cc:61-64`).
- `aggression` is clamped to `>= 1.0` (`admission_control.cc:57-59`). Higher aggression rejects more eagerly for the same success-rate shortfall.
- `max_rejection_probability` caps the computed probability (default 80%).
- The accuracy is 1e4, giving four significant figures of resolution when doing the probabilistic comparison with `random()` (`admission_control.cc:176-178`).

## Configuration
- `enabled` — runtime feature flag checked on every decode (`admission_control.cc:81`).
- `sampling_window` — backing ring window; default 30s (`config.cc:18`, `config.cc:49-54`). Validated `>= 1.0%` for sr_threshold (`config.cc:41-43`).
- `sr_threshold` — success-rate threshold percentage (default 95%).
- `aggression` — rejection curve exponent (default 1.0).
- `rps_threshold` — minimum RPS before the filter rejects anything (default 0).
- `max_rejection_probability` — cap (default 80%).
- `evaluation_criteria` (oneof): only `success_criteria` is implemented and is required — `EVALUATION_CRITERIA_NOT_SET` is rejected with `InvalidArgument` (`config.cc:65-66`).
- No per-route config.

## Stats
Prefix `<stats_prefix>admission_control.` (`config.cc:45`):

- `rq_rejected` — probabilistic rejection at `decodeHeaders`.
- `rq_success` — classified success (contributes to sliding window).
- `rq_failure` — classified failure.

## Factory
`AdmissionControlFilterFactory` is a dual (downstream + upstream) factory; both are registered:

- `REGISTER_FACTORY(AdmissionControlFilterFactory, NamedHttpFilterConfigFactory)` (`config.cc:82`).
- `REGISTER_FACTORY(UpstreamAdmissionControlFilterFactory, UpstreamHttpFilterConfigFactory)` (`config.cc:84`).

`createFilterFactory` (`config.cc:37-77`):
1. Allocates a `ThreadLocal::TypedSlot<ThreadLocalControllerImpl>`; each worker's controller keeps its own sliding window over `sampling_window`.
2. Builds a `SuccessCriteriaEvaluator` from proto.
3. Constructs `AdmissionControlFilterConfig` (runtime, random, scope, TLS slot, evaluator).
4. Returns a callback that calls `addStreamFilter(std::make_shared<AdmissionControlFilter>(filter_config, prefix))`.

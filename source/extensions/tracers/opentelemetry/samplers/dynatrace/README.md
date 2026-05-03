# Dynatrace sampler (`envoy.tracers.opentelemetry.samplers.dynatrace`)

Adaptive sampler that mimics Dynatrace OneAgent's ATM (Adaptive Traffic
Management) behavior for OTel spans. Incoming requests are bucketed by
HTTP method + path; each bucket has its own sampling exponent (log2 of the
multiplicity) which is recomputed periodically based on observed traffic.
A Dynatrace-specific tag is written into the W3C `tracestate` header so
downstream Dynatrace components respect the upstream decision.

Proto: `envoy.extensions.tracers.opentelemetry.samplers.v3.DynatraceSamplerConfig`.

## Files
- `config.h/cc` - `DynatraceSamplerFactory` registered in the
  `envoy.tracers.opentelemetry.samplers` category. Builds
  `SamplerConfigProviderImpl` and the `DynatraceSampler`.
- `dynatrace_sampler.h/cc` - the sampler itself. Owns
  `sampling_controller_` and a main-thread `Timer` that calls
  `update()` every `SAMPLING_UPDATE_TIMER_DURATION` (1 minute).
- `sampler_config.h/cc` - typed wrapper around the remote Dynatrace
  sampling config payload.
- `sampler_config_provider.h/cc` - `SamplerConfigProviderImpl` is an
  `Http::AsyncClient::Callbacks` that periodically pulls fresh sampler
  config from the Dynatrace API.
- `sampling_controller.h/cc` - request-rate bookkeeping. Uses a
  `StreamSummary` (Space-Saving / HeavyHitter) over sampling keys, derives
  per-bucket `SamplingState` exponents, and exposes
  `getSamplingKey`/`offer`/`update`.
- `stream_summary.h` - templated Space-Saving algorithm implementation.
- `dynatrace_tag.h` - encode/decode of the Dynatrace tracestate tag
  (version + flags + sampling exponent + path info + optional trace
  capture reason).
- `tenant_id.h` - `calculateTenantId()` used to build the tracestate key
  `<tenantId>-<clusterIdHex>@dt`.
- `trace_capture_reason.h` - enum and string conversion for the "why was
  this sampled" extension in the Dynatrace tag.

## Tracer role
Not a tracer. Plugged into `Tracer::startSpan` via `callSampler`.

## Flow
- For every span:
  - Build sampling key from `trace_context->method()` +
    `trace_context->path()` (query stripped).
  - `sampling_controller_.offer(key)` records the request for the current
    period.
  - Parse the incoming `tracestate` via
    `opentelemetry::trace::TraceState::FromHeader`.
  - If a Dynatrace tag for `<tenantId>-<clusterIdHex>@dt` is present and
    valid: honor its `ignored` flag, copy the tracestate through as-is,
    add Dynatrace span attributes.
  - Else (root span): hash the trace id (`MurmurHash::murmurHash2`),
    consult the per-bucket `SamplingState::shouldSample`, create a new
    `DynatraceTag` with the sampling exponent and trace capture reason
    `Atm`, write it into `tracestate`, add attributes.
- Periodic timer (1 min) calls `SamplingController::update()` to
  recompute exponents from the observed `StreamSummary`.

## Key decision points
- `dynatrace_sampler.cc:24` - `SAMPLING_UPDATE_TIMER_DURATION = 1 minute`.
- `dynatrace_sampler.cc:58-59` - tracestate key is
  `<calculateTenantId(tenant)>-<hex(cluster_id)>@dt`.
- `dynatrace_sampler.cc:27-49` - span attributes written:
  `supportability.atm_sampling_ratio`, `sampling.threshold` (only when
  multiplicity > 1, derived as `2^56 - 2^56/multiplicity`), and
  `trace.capture.reasons` when the tag's TCR extension is valid.
- `dynatrace_sampler.cc:94-103` - an existing Dynatrace tag fully wins; no
  re-sampling is performed.

## Configuration
- `tenant`, `cluster_id` - identify the Dynatrace tenant; used as the
  tracestate key.
- `http_service` / remote config fields - feed `SamplerConfigProviderImpl`
  to periodically fetch the sampling configuration from the Dynatrace API.
- `root_spans_per_minute` (on the remote config) - target traffic budget
  used by `SamplingController::update`.

## Stats / errors
No dedicated stat scope; HTTP async client failures are logged and retried
inside `SamplerConfigProviderImpl`. Missing / malformed incoming Dynatrace
tags silently fall back to hash-based sampling.

# OpenTelemetry samplers

Extension category: `envoy.tracers.opentelemetry.samplers`.

A sampler decides, per span, whether it should be recorded and whether it
should be exported. The OpenTelemetry driver resolves a single sampler
from the `OpenTelemetryConfig.sampler` field and invokes its
`shouldSample(...)` for every `Tracer::startSpan` call via `callSampler`
(`tracer.cc:37-57`). The returned `SamplingResult` controls the span's
`sampled` bit, its additional attributes, and its outgoing `tracestate`.

Proto: the category takes a `TypedExtensionConfig` pointing at any sampler
config under `envoy.extensions.tracers.opentelemetry.samplers.v3.*`.

## Files
- `sampler.h` - the core types:
  - `Decision { Drop, RecordOnly, RecordAndSample }`.
  - `SamplingResult` with `decision`, optional `attributes`
    (`OtelAttributes`), and a `tracestate` string; helper
    `isRecording()` / `isSampled()`.
  - `Sampler` base with
    `shouldSample(stream_info, parent_context, trace_id, name, spankind,
    trace_context, links)` and `getDescription()`.
  - `SamplerFactory` typed factory (category
    `envoy.tracers.opentelemetry.samplers`).
- `always_on/`, `cel/`, `dynatrace/`, `parent_based/`,
  `trace_id_ratio_based/` - concrete samplers, each with its own README.

## Tracer role
Not a tracer. Pure sampling-decision component. The OpenTelemetry driver
does not call `shouldSample` for spans created from an already-sampled
parent context by some samplers (see `parent_based/`).

## Flow
- `Tracer::startSpan` -> `callSampler` passes the current
  `StreamInfo::StreamInfo`, the parent `SpanContext` (if any), the
  generated trace id, the operation name, the OTel span kind, and the
  incoming `Tracing::TraceContext` ref.
- The sampler returns a `SamplingResult`.
- The driver writes `setSampled(result.isSampled())`, applies the returned
  `attributes` via `setAttribute`, and overwrites `tracestate` when
  non-empty.

## Key decision points
- `sampler.h:44-48` - `isRecording()` includes `RecordOnly`; `isSampled()`
  is strict.
- `tracer.cc:47` - only the `sampled` flag is written back, not the
  `RecordOnly` signal; `RecordOnly` effectively behaves like `Drop` at
  export time because `finishSpan` drops unsampled spans.

## Configuration
One sampler per OpenTelemetry tracer. The typed config's `name` selects
the registered factory.

## Stats / errors
No shared stats. Each sampler may emit or log on its own.
Unknown factory name throws `EnvoyException` in the driver
(`opentelemetry_tracer_impl.cc:44`).

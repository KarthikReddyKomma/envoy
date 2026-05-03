# Trace-id-ratio-based sampler (`envoy.tracers.opentelemetry.samplers.trace_id_ratio_based`)

Deterministic probabilistic sampler: a configurable fraction of trace ids
are sampled, selected by a deterministic function of the trace id. Two
services that see the same trace id will make the same decision, which
keeps traces coherent across the mesh even if only some hops run this
sampler.

Proto: `envoy.extensions.tracers.opentelemetry.samplers.v3.TraceIdRatioBasedSamplerConfig`
(uses `envoy.type.v3.FractionalPercent` for `sampling_percentage`).

## Files
- `config.h/cc` - `TraceIdRatioBasedSamplerFactory` registered in the
  `envoy.tracers.opentelemetry.samplers` category.
- `trace_id_ratio_based_sampler.h/cc` - sampler implementation. Converts
  the first 8 bytes of the hex trace id to `uint64_t` and runs it through
  `ProtobufPercentHelper::evaluateFractionalPercent(sampling_percentage_,
  value)` to decide.

## Tracer role
Not a tracer. Plugged into `Tracer::startSpan` via `callSampler`.

## Flow
- `traceIdToUint64(trace_id)` parses the first 16 hex chars (8 bytes) into
  a `uint64_t`. If the trace id is shorter than 16, the function logs at
  `warn` and returns 0.
- If `sampling_percentage.numerator == 0`, immediately `Drop`.
- Otherwise `evaluateFractionalPercent` decides `RecordAndSample` vs
  `Drop`.
- `tracestate` is forwarded from the parent context when one exists.

## Key decision points
- `trace_id_ratio_based_sampler.cc:28-43` - short trace ids fall back to
  the uint value 0, which under any nonzero probability may still sample
  if the numerator allows it; callers should ensure valid 32-hex trace
  ids.
- `trace_id_ratio_based_sampler.cc:51-56` - description string is built as
  `TraceIdRatioBasedSampler{<num>/<denom_int>}`; note denominator is
  converted to an int via
  `ProtobufPercentHelper::fractionalPercentDenominatorToInt`.
- `trace_id_ratio_based_sampler.cc:69` - `numerator == 0` short-circuits
  to `Drop` without consulting the trace id.

## Configuration
- `sampling_percentage` - `envoy.type.v3.FractionalPercent` giving the
  fraction of trace ids to sample.

## Stats / errors
No stats. Short trace ids log at `warn`.

# Always-on sampler (`envoy.tracers.opentelemetry.samplers.always_on`)

Records and samples every span. Equivalent to the OTel spec "AlwaysOnSampler".

Proto: `envoy.extensions.tracers.opentelemetry.samplers.v3.AlwaysOnSamplerConfig`.

## Files
- `config.h/cc` - `AlwaysOnSamplerFactory` registered in the
  `envoy.tracers.opentelemetry.samplers` category. Validates the typed
  config and returns a shared `AlwaysOnSampler`.
- `always_on_sampler.h/cc` - `AlwaysOnSampler::shouldSample` always returns
  `Decision::RecordAndSample`. If a parent context is supplied, its
  `tracestate` is echoed back into the result so downstream services see
  the same tracestate.

## Tracer role
Not a tracer. Plugged into `Tracer::startSpan` via `callSampler`.

## Flow
- For every `shouldSample` invocation: return
  `{decision: RecordAndSample, tracestate: parent_context.tracestate()}`.
- No attributes are added.

## Key decision points
- `always_on_sampler.cc:22-24` - tracestate is only forwarded if the
  parent context is present.
- `always_on_sampler.cc:29` - `getDescription()` returns the literal
  `"AlwaysOnSampler"`.

## Configuration
None; the config message is empty.

## Stats / errors
No stats. No failure modes.

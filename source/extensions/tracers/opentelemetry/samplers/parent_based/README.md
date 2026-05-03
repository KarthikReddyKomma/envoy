# Parent-based sampler (`envoy.tracers.opentelemetry.samplers.parent_based`)

Defers to the parent span context when one exists: if the parent is
sampled, this span is sampled; if it is not, this span is dropped. When
there is no parent context, the decision is delegated to a wrapped inner
sampler configured by the user. This mirrors the OTel spec
`ParentBasedSampler`.

Proto: `envoy.extensions.tracers.opentelemetry.samplers.v3.ParentBasedSamplerConfig`.

## Files
- `config.h/cc` - `ParentBasedSamplerFactory` registered in the
  `envoy.tracers.opentelemetry.samplers` category. Looks up the
  `wrapped_sampler` (a nested `TypedExtensionConfig`) through
  `SamplerFactory` and wraps it.
- `parent_based_sampler.h/cc` - `ParentBasedSampler::shouldSample` branches
  on parent presence.

## Tracer role
Not a tracer. Plugged into `Tracer::startSpan` via `callSampler`.

## Flow
- Parent missing or has empty `traceId`: forward the call directly to the
  wrapped sampler's `shouldSample`.
- Parent present: return
  `{RecordAndSample if parent.sampled() else Drop,
    tracestate: parent.tracestate()}`.

## Key decision points
- `parent_based_sampler.cc:20-23` - the branch condition uses both
  "context absent" and "traceId empty" so that a malformed parent is
  treated as no parent.
- `parent_based_sampler.cc:26-32` - no attributes are ever added on this
  path; only the wrapped path can contribute attributes.

## Configuration
- `wrapped_sampler` - required `TypedExtensionConfig` selecting the inner
  sampler used for root spans.

## Stats / errors
No stats. Wrapped-sampler factory lookup failures surface as
`EnvoyException` during config load.

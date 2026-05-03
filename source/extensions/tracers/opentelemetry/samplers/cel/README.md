# CEL sampler (`envoy.tracers.opentelemetry.samplers.cel`)

Samples based on a CEL (Common Expression Language) expression evaluated
against the current request. If the expression returns `true`, the span
is recorded and sampled; any other result (including evaluation errors
and non-boolean values) drops the span. Lets operators write arbitrary
boolean rules over headers, stream info, and local node info.

Proto: `envoy.extensions.tracers.opentelemetry.samplers.v3.CELSamplerConfig`
(holds an `xds.type.v3.CelExpression`).

## Files
- `config.h/cc` - `CELSamplerFactory` registered in the
  `envoy.tracers.opentelemetry.samplers` category. Resolves the shared
  CEL `BuilderInstance` from the server factory context and constructs a
  `CELSampler`.
- `cel_sampler.h/cc` - `CELSampler::shouldSample` compiles the expression
  once at construction (throwing `EnvoyException` on compile failure),
  then on every call runs it against the incoming request headers (when
  the `TraceContext` carries them), stream info, and local info.

## Tracer role
Not a tracer. Plugged into `Tracer::startSpan` via `callSampler`.

## Flow
- Construction: `Expr::CompiledExpression::Create(builder, expr)` compiles
  the expression; failure throws.
- Per span: pull request headers from
  `trace_context.requestHeaders().ptr()` if available, evaluate against
  `local_info_`, stream_info, and those headers.
- If the evaluation has no value, errors, or is not a bool, return
  `Decision::Drop`.
- If the bool evaluates true, return
  `{RecordAndSample, tracestate: parent.tracestate()}`.

## Key decision points
- `cel_sampler.cc:17-25` - compile-time failure is fatal to config load.
- `cel_sampler.cc:34-42` - only request headers are supplied; response
  headers/trailers are always nullptr, so the expression cannot inspect
  responses.
- `cel_sampler.cc:44-53` - any non-truthy / non-bool / errored evaluation
  becomes a `Drop`.

## Configuration
- `expression` - `xds.type.v3.CelExpression` evaluated per span.

## Stats / errors
No stats. Compile failure: `EnvoyException`. Runtime evaluation failures
are silent drops.

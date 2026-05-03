# AWS X-Ray tracer (`envoy.tracers.xray`)

Tracer that forwards segments to a local AWS X-Ray daemon over UDP. The
daemon (not Envoy) talks to the X-Ray service. Propagation uses the
`x-amzn-trace-id` header with the `Root=...;Parent=...;Sampled=...`
syntax. Sampling is driven by a local JSON manifest of X-Ray sampling
rules (host / method / URL wildcards, `fixed_target` + `rate`).

Proto: `envoy.config.trace.v3.XRayConfig`.

## Files
- `config.h/cc` - `XRayTracerFactory` (subclass of
  `Common::FactoryBase`). Reads the sampling-rules JSON via
  `Config::DataSource::read`, validates that `daemon_endpoint` is a UDP
  `SocketAddress` with a numeric port, and builds an
  `XRayConfiguration` passed to the `Driver`.
- `xray_configuration.h` - POD holding daemon endpoint, segment name,
  sampling-rules JSON, origin, and an aws metadata map.
- `xray_tracer_impl.h/cc` - `Driver` with a thread-local `TlsTracer`.
  Parses the `x-amzn-trace-id` header (`parseXRayHeader`), reconciles the
  upstream sampling decision with the local manifest, and returns either
  a real `Span` or a non-sampled span (never `nullptr`, to force the
  upstream to observe the decision via `injectContext`).
- `tracer.h/cc` - `Tracer` + `Span`. `Tracer::startSpan` builds a new
  `Span` with an X-Ray-format trace id (`1-<epoch hex 8>-<uuid 24>`);
  `Span::finishSpan` JSON-serializes via `daemon.proto` and hands the
  bytes to the `DaemonBroker`.
- `daemon_broker.h/cc` - `DaemonBrokerImpl` wraps an `IoHandle` and
  prefixes each payload with the required X-Ray daemon header before
  sending over UDP.
- `daemon.proto` - segment document message sent to the daemon.
- `sampling_strategy.h` - abstract `SamplingStrategy::shouldTrace`.
- `localized_sampling.h/cc` - `LocalizedSamplingRule`,
  `LocalizedSamplingManifest`, and `LocalizedSamplingStrategy`. Loads the
  JSON manifest, matches host / HTTP method / URL path wildcards, then
  defers to a `Reservoir` + rate check.
- `reservoir.h` - mutex-guarded per-second token bucket used by
  `LocalizedSamplingRule::reservoir`.
- `util.h/cc` - small helpers (wildcard matching, etc).

## Tracer role
- `Tracing::Driver::startSpan(...)` - `Driver::startSpan`
  (`xray_tracer_impl.cc:64`) parses the X-Ray header, resolves the
  sampling decision, then calls either `Tracer::startSpan` or
  `Tracer::createNonSampledSpan`.
- `Span::injectContext` - writes `x-amzn-trace-id:
  Root=<trace_id>;Parent=<span_id>;Sampled=<0|1>`. Even non-sampled spans
  inject so the upstream honors the decision.
- `Span::finishSpan` - serializes the segment via `daemon.proto` and
  sends it to the daemon through `DaemonBroker::send`.

## Flow
- Request arrives -> parse `x-amzn-trace-id` (if any). Explicit
  `Sampled=1` / `Sampled=0` short-circuit the sampler. Missing /
  `Unknown` -> fall through to the sampling strategy.
- Local sampling strategy builds a `SamplingRequest{host, method, path}`
  and asks `LocalizedSamplingStrategy::shouldTrace`. If a custom rule
  matches it is used; otherwise the default rule applies. Rules consume a
  token from `Reservoir` then flip a random coin against `rate`.
- If sampled -> `Tracer::startSpan`. If not sampled -> a non-sampled
  `Span` that still propagates the header (`xray_tracer_impl.cc:119`).
- Finished spans: each call to `Span::finishSpan` serializes and sends
  immediately to the daemon via UDP - no batching or retry.

## Key decision points
- `xray_tracer_impl.cc:16` - `DefaultDaemonEndpoint = "127.0.0.1:2000"`
  used when the config omits the endpoint.
- `config.cc:35-42` - `daemon_endpoint` must be a numeric-port UDP
  `SocketAddress`; anything else throws `EnvoyException`.
- `tracer.cc:37-48` - trace id format is `1-<epoch hex8>-<uuid24>`.
- `xray_tracer_impl.cc:118-120` - non-sampled spans still propagate the
  decision so upstreams do not re-sample.
- `localized_sampling.h:31-44` - sampling rule requires host *and* HTTP
  method *and* URL path to match.

## Configuration
- `daemon_endpoint` - UDP address of the local X-Ray daemon (defaults to
  `127.0.0.1:2000`).
- `segment_name` - name used for the root segment.
- `sampling_rule_manifest` - JSON sampling-rules manifest as a
  `DataSource`.
- `segment_fields.origin`, `segment_fields.aws.fields` - custom values
  placed on the segment.

## Stats / errors
No dedicated stats scope. `DataSource::read` failure on the sampling
manifest logs at `error` and the tracer falls back to the built-in default
rule (`LocalizedSamplingManifest::createDefault`). UDP send failures are
swallowed by the `DaemonBroker`.

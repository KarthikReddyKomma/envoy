# OpenTelemetry tracer (`envoy.tracers.opentelemetry`)

Native OTLP tracer. Produces `opentelemetry.proto.trace.v1.Span` messages
and exports them either over OTLP/gRPC (`ExportTraceServiceRequest`) or
OTLP/HTTP. Propagation is strictly W3C Trace Context
(`traceparent` + `tracestate`). Sampling and resource attributes are
pluggable: both are resolved through factory extension points wired through
this directory's subfolders.

Proto: `envoy.config.trace.v3.OpenTelemetryConfig`.

## Files
- `config.h/cc` - `OpenTelemetryTracerFactory` registered with
  `REGISTER_FACTORY(..., TracerFactory)`; builds a `Driver`.
- `opentelemetry_tracer_impl.h/cc` - `Driver` owns a thread-local
  `TlsTracer` holding a `Tracer`. `Driver::startSpan` extracts the W3C
  context (if any) and invokes `Tracer::startSpan` with the appropriate
  `OTelSpanKind` derived from `Tracing::Config` (spawnUpstreamSpan +
  traffic direction, `opentelemetry_tracer_impl.cc:51`).
- `tracer.h/cc` - `Tracer` buffers `Span` protos and flushes them to the
  exporter in batches. Runs the configured sampler on every `startSpan`
  through `callSampler` (`tracer.cc:37`).
- `span_context.h` - simple immutable W3C-style context
  (version/trace id/span id/sampled/tracestate).
- `span_context_extractor.h/cc` - parses `traceparent` and joins all
  `tracestate` header values.
- `trace_exporter.h` - `OpenTelemetryTraceExporter` interface (`log(const
  ExportTraceServiceRequest&)`).
- `grpc_trace_exporter.h/cc` - OTLP/gRPC implementation over
  `Grpc::AsyncClient<ExportTraceServiceRequest, ExportTraceServiceResponse>`;
  calls `trace.v1.TraceService.Export`.
- `http_trace_exporter.h/cc` - OTLP/HTTP implementation that POSTs the
  serialized `ExportTraceServiceRequest` via `Http::AsyncClient` using a
  configured `HttpService`.
- `otlp_utils.h/cc` - shared constants, `OTelSpanKind` /
  `OTelAttribute` / `OtelAttributes` types, User-Agent helper, and
  `populateAnyValue` which converts `OTelAttribute` variants into
  `opentelemetry.proto.common.v1.AnyValue`.
- `resource_detectors/` - pluggable `ResourceDetector` implementations and
  the `ResourceProvider` that merges their output with the static
  `service.name` and `telemetry.sdk.*` attributes (see its README).
- `samplers/` - pluggable `Sampler` implementations invoked from
  `Tracer::startSpan` (see its README).

## Tracer role
- `Tracing::Driver::startSpan(...)` - `Driver::startSpan` extracts the
  parent context via `SpanContextExtractor`, builds the kind, calls
  `Tracer::startSpan` (root or child overload) which constructs a `Span`
  and runs the sampler.
- `Span::injectContext` - writes `traceparent =
  00-<trace_id>-<span_id>-<flags>` and echoes `tracestate`
  (`tracer.cc:94`).
- `Span::finishSpan` - if sampled, sets `end_time_unix_nano` and calls
  `Tracer::sendSpan`, which appends to the in-memory `span_buffer_` and
  flushes when the buffer reaches `max_cache_size` or when the flush timer
  fires.
- `Span::spawnChild` - builds a `SpanContext` from the current span and
  calls `Tracer::startSpan` with kind `SPAN_KIND_CLIENT`.
- `setTag`, `log`, `setAttribute` translate to span attributes through
  `OtlpUtils::populateAnyValue` (exact key rules in `tracer.cc:107-120`).

## Flow
- Span creation: `Driver::startSpan` -> `Tracer::startSpan` -> `Span`
  constructor -> optional `Sampler::shouldSample` -> attributes / sampled
  flag / tracestate written back onto the span.
- Context propagation: W3C `traceparent` + `tracestate` only. Other
  propagators are not wired in.
- Batching: `Tracer` pushes `finishSpan` spans into `span_buffer_`;
  `flushSpans()` builds an `ExportTraceServiceRequest`, attaches the
  `Resource` produced by resource detectors, and calls
  `exporter_->log(...)`.
- Export: gRPC (`OpenTelemetryGrpcTraceExporter`) or HTTP
  (`OpenTelemetryHttpTraceExporter`). The sampling
  (gRPC) service method is `trace.v1.TraceService.Export`.

## Key decision points
- `opentelemetry_tracer_impl.cc:85` - rejects configs that set both
  `grpc_service` and `http_service`.
- `opentelemetry_tracer_impl.cc:121` - `max_cache_size` default is
  `DEFAULT_MAX_CACHE_SIZE = 1024`.
- `opentelemetry_tracer_impl.cc:51-63` - `getSpanKind` rules for
  server/client based on traffic direction and spawnUpstreamSpan.
- `tracer.cc:23` - only W3C version `00` is produced on inject.
- `tracer.cc:83-89` - unsampled spans are dropped at `finishSpan`.

## Configuration
- `grpc_service` or `http_service` (mutually exclusive) - chooses exporter.
- `service_name` - mapped to `service.name` resource attribute; defaults
  to `kDefaultServiceName` when empty.
- `resource_detectors` - list of `TypedExtensionConfig`, each resolved
  through the `envoy.tracers.opentelemetry.resource_detectors` registry.
- `sampler` - single `TypedExtensionConfig` resolved through the
  `envoy.tracers.opentelemetry.samplers` registry.
- `max_cache_size` - span buffer high-water mark before a flush.

## Stats / errors
- Counters under `tracing.opentelemetry.*`:
  `spans_sent`, `timer_flushed`, `spans_dropped` (see
  `OPENTELEMETRY_TRACER_STATS`, `tracer.h:26-29`).
- Context extraction failure returns a `Tracing::NullSpan`
  (`opentelemetry_tracer_impl.cc:153`).
- Sampler/resource-detector factory not found throws `EnvoyException`
  (`opentelemetry_tracer_impl.cc:44`, `resource_provider.cc:98`).

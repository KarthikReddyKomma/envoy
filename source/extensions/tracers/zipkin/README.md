# Zipkin tracer (`envoy.tracers.zipkin`)

Tracer that reports spans to a Zipkin-compatible collector over HTTP. It
supports both legacy `collector_cluster` / `collector_endpoint` config and
a full `HttpService` config with arbitrary request headers. Propagation
defaults to B3 (multi-header) and can fall back to W3C
(`traceparent`/`tracestate`) when `trace_context_option` is
`USE_B3_WITH_W3C_PROPAGATION`. Payloads are emitted as Zipkin v1 JSON, v2
JSON, or proto-encoded `ListOfSpans` depending on
`CollectorEndpointVersion`.

Proto: `envoy.config.trace.v3.ZipkinConfig`.

## Files
- `config.h/cc` - `ZipkinTracerFactory` registered with
  `REGISTER_FACTORY(..., TracerFactory)`; creates the `Driver`.
- `zipkin_tracer_impl.h/cc` - top-level driver:
  - `CollectorInfo` (cluster, endpoint, hostname, endpoint version,
    shared-span-context flag, optional `HttpServiceHeadersApplicator`).
  - `Driver` allocates a TLS `Tracer` and exposes `startSpan`. Accepts
    either `collector_service` (preferred) or legacy
    `collector_cluster`/`collector_endpoint`.
  - `ReporterImpl` - implements `Zipkin::Reporter` and
    `Http::AsyncClient::Callbacks`. Buffers spans, flushes on buffer full
    or timer, POSTs to the collector using `Http::AsyncClient`.
  - `ZIPKIN_TRACER_STATS` macro defines per-tracer counters.
- `tracer.h/cc` - `Tracer::startSpan` creates root / child / shared-context
  `Span` objects, generates 64-bit or 128-bit trace ids (optionally with a
  32-bit timestamp prefix when `timestamp_trace_ids=true`), and invokes
  the `Reporter`.
- `tracer_interface.h` - abstract `TracerInterface` / `Reporter` so tests
  can inject a fake reporter.
- `span_context.h`, `span_context_extractor.h/cc` - B3 extraction. The
  extractor also supports `USE_B3_WITH_W3C_PROPAGATION` which reads the
  OpenTelemetry W3C context extractor and converts it to Zipkin format.
- `zipkin_core_types.h/cc` - Zipkin `Span`, `Endpoint`, `Annotation`,
  `BinaryAnnotation` value types + their JSON serialization (v1 and v2).
- `zipkin_core_constants.h`, `zipkin_json_field_names.h` - wire constants.
- `span_buffer.h/cc` - bounded `SpanBuffer` and the three serializers:
  `JsonV1Serializer`, `JsonV2Serializer`, `ProtobufSerializer`.
- `util.h/cc` - id generation / formatting helpers.

## Tracer role
- `Tracing::Driver::startSpan(...)` - `Driver::startSpan`
  (`zipkin_tracer_impl.cc` `Driver` section) extracts B3 (or W3C fallback)
  context, asks the `Tracer` for a span via one of
  `Tracer::startSpan(config, name, timestamp[, previous_context])`. The
  `operation_name` arg is deliberately ignored per the Zipkin model
  (`zipkin_tracer_impl.h:97-100`).
- `Span::injectContext` - writes B3 headers (`x-b3-traceid`,
  `x-b3-spanid`, `x-b3-parentspanid`, `x-b3-sampled`, `x-b3-flags` or the
  single `b3` header) and optionally also W3C `traceparent`/`tracestate`
  in `USE_B3_WITH_W3C_PROPAGATION` mode.
- `Span::finishSpan` - fills in duration/annotations and calls
  `Tracer::reportSpan` -> `ReporterImpl::reportSpan`, which adds the span
  to the buffer and flushes when full.

## Flow
- Span creation: B3 extraction -> root / child / shared-context span
  decision in `span_context_extractor.cc`.
- Context propagation: B3 by default; optional W3C fallback
  (`Driver::w3cFallbackEnabled`) enables both reading and injecting W3C.
- Batching: `SpanBuffer` up to
  `runtime.tracing.zipkin.min_flush_spans` (default 5). Timer
  `runtime.tracing.zipkin.flush_interval_ms` (default 5000) also triggers
  flush (see `zipkin_tracer_impl.h:124-132`).
- Export: `Http::AsyncClient` POST to
  `cluster=collector_->cluster_`, path `collector_->endpoint_`, host
  `collector_->hostname_`. The `CollectorEndpointVersion` picks the
  serializer and Content-Type.

## Key decision points
- `zipkin_tracer_impl.cc:55-93` - `collector_service` preferred; if
  absent, both `collector_cluster` and `collector_endpoint` are required
  or `EnvoyException` is thrown.
- `zipkin_tracer_impl.cc:96-99` - cluster existence validated up-front via
  `Config::Utility::checkCluster`.
- `tracer.h:83` - `generateTraceId` optionally prefixes 32 bits of epoch
  seconds for trace id sortability.
- `zipkin_core_constants.h` - `DEFAULT_SHARED_SPAN_CONTEXT` controls
  whether client and server share a span id for one request.
- `span_context_extractor.cc` - W3C fallback only kicks in when
  `w3c_fallback_enabled_` is true and no B3 headers were found.

## Configuration
Legacy path:
- `collector_cluster`, `collector_endpoint`, optional
  `collector_hostname`.

HttpService path (preferred):
- `collector_service.http_uri.cluster`, `http_uri.uri` (hostname + path
  parsed via `parseUri`).
- `collector_service.request_headers_to_add` - custom headers applied by
  `HttpServiceHeadersApplicator`.

Common:
- `collector_endpoint_version` - `HTTP_JSON`, `HTTP_JSON_V1`, `HTTP_PROTO`.
- `trace_id_128bit` - 128-bit vs 64-bit trace ids.
- `shared_span_context` - use same span id for both ends of a hop.
- `split_spans_for_request`, `timestamp_trace_ids` - Envoy-specific
  tweaks.
- `trace_context_option` - `USE_B3` or `USE_B3_WITH_W3C_PROPAGATION`.

## Stats / errors
Counters under `tracing.zipkin.*` (see `ZIPKIN_TRACER_STATS`,
`zipkin_tracer_impl.h:26-32`): `spans_sent`, `timer_flushed`,
`reports_skipped_no_cluster`, `reports_sent`, `reports_dropped`,
`reports_failed`. Collector cluster missing at runtime surfaces as
`reports_skipped_no_cluster` (tracked via
`Upstream::ClusterUpdateTracker`). Async HTTP failures increment
`reports_failed`; non-2xx responses increment `reports_dropped`.

# SkyWalking tracer (`envoy.tracers.skywalking`)

Tracer that integrates with Apache SkyWalking through the
[cpp2sky](https://github.com/SkyAPM/cpp2sky) client library. Envoy reports
trace segments to a SkyWalking OAP (backend) over gRPC using the
SkyWalking `TraceSegmentReportService`. Propagation uses the single
SkyWalking header (`sw8`) handed off via
`cpp2sky::createSpanContext`.

Proto: `envoy.config.trace.v3.SkyWalkingConfig`.

## Files
- `config.h/cc` - `SkyWalkingTracerFactory` registered with
  `REGISTER_FACTORY(..., TracerFactory)`. Builds the `Driver`.
- `skywalking_tracer_impl.h/cc` - `Driver` with a thread-local slot. For
  each request it either creates a fresh `TracingContext` or builds one
  from the incoming `sw8` propagation header via
  `cpp2sky::createSpanContext`. Also resolves defaults for `service_name`
  and `instance_name` (falls back to local info or `"EnvoyProxy"`).
- `tracer.h/cc` - `Tracer` wraps a `TraceSegmentReporter`. `Tracer::
  startSpan` creates a cpp2sky entry span; `Span` adapts
  `cpp2sky::TracingSpan` to the Envoy `Tracing::Span` interface.
- `trace_segment_reporter.h/cc` - `Grpc::AsyncStreamCallbacks` that holds
  an open `TraceSegmentReportService.collect` gRPC stream, buffers up to
  `delayed_buffer_size` `SegmentObject` messages when the stream is down,
  and reconnects with a `JitteredExponentialBackOffStrategy`.
- `skywalking_stats.h` - `SKYWALKING_TRACER_STATS` macro defining
  `cache_flushed`, `segments_dropped`, `segments_flushed`, `segments_sent`.

## Tracer role
- `Tracing::Driver::startSpan(...)` - `Driver::startSpan`
  (`skywalking_tracer_impl.cc:49`) builds a `TracingContext` and calls
  `Tracer::startSpan(path, protocol, tracing_context)`. When there is no
  `sw8` header and the incoming decision is `traced=false` it returns
  `Tracing::NullSpan`.
- `Span::injectContext` - writes the SkyWalking `sw8` header via
  `cpp2sky::TracingContext::createSW8HeaderValue`.
- `Span::finishSpan` - ends the cpp2sky span; when the last span in the
  segment finishes the segment is handed to
  `TraceSegmentReporter::report`.

## Flow
- Span creation: `Driver::startSpan` extracts/creates `TracingContext` ->
  `Tracer::startSpan` -> `TracingContext::createEntrySpan` (root) or
  `createExitSpan` (child).
- Context propagation: SkyWalking `sw8` only (see
  `skywalkingPropagationHeaderKey()`).
- Batching: cpp2sky aggregates spans into a `SegmentObject`; when the
  segment is complete the tracer pushes it into the reporter's gRPC
  stream.
- Export: gRPC bidirectional stream
  `skywalking.v3.TraceSegmentReportService.collect`. If the stream is
  down, messages queue into `delayed_segments_cache_` up to
  `delayed_buffer_size`; older ones are dropped and `segments_dropped` is
  incremented. `setRetryTimer` drives `establishNewStream` via backoff.

## Key decision points
- `skywalking_tracer_impl.cc:17` -
  `DEFAULT_DELAYED_SEGMENTS_CACHE_SIZE = 1024`.
- `skywalking_tracer_impl.cc:22` - default service/instance name
  `"EnvoyProxy"` when nothing else is available.
- `skywalking_tracer_impl.cc:60-64` - if there is no propagation header
  and Envoy's own decision says not to sample, a `NullSpan` is returned -
  the upstream hop is not informed of the decision.
- `skywalking_tracer_impl.cc:72-86` - cpp2sky may throw non-`TracerException`
  exceptions, so all exceptions are swallowed here to avoid crashing
  Envoy; on parse failure the span falls back to root-span creation when
  `decision.traced`.
- `skywalking_tracer_impl.cc:92-107` - service/instance name resolution
  order: proto -> local info -> `"EnvoyProxy"`.

## Configuration
- `grpc_service` - upstream for the OAP backend (`collect` RPC).
- `client_config.service_name`, `instance_name`, `backend_token`,
  `max_cache_size` - SkyWalking client settings.

## Stats / errors
- Counters under `tracing.skywalking.*`: `cache_flushed`,
  `segments_dropped`, `segments_flushed`, `segments_sent` (see
  `skywalking_stats.h:10`).
- gRPC stream close triggers `handleFailure` -> `setRetryTimer`; during
  outage `delayed_segments_cache_` queues up segments up to
  `delayed_buffer_size`. Overflow drops oldest and increments
  `segments_dropped`.
- `sw8` parse failures log at `warn` and fall back to a fresh span (or a
  `NullSpan` when not sampled).

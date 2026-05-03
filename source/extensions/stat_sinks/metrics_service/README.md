# Metrics service sink

gRPC-based stats sink. Converts each `Stats::MetricSnapshot` into
Prometheus `MetricFamily` messages, batches them, and streams them to an
upstream gRPC service implementing
`envoy.service.metrics.v3.MetricsService.StreamMetrics`. The first
message on every new stream carries the local `Node` identifier so the
receiver can tag metrics by source. Registered as
`envoy.stat_sinks.metrics_service`.

Proto: `envoy.config.metrics.v3.MetricsServiceConfig`.

## Files
- `config.h` / `config.cc` — `MetricsServiceSinkFactory`, wires the gRPC
  async client and instantiates `MetricsServiceSink`.
- `grpc_metrics_service_impl.h` / `.cc` — `GrpcMetricsStreamer`
  interface, `GrpcMetricsStreamerImpl` (the gRPC client wrapper),
  `MetricsFlusher` (snapshot -> Prometheus proto), templated
  `MetricsServiceSink`.
- `grpc_metrics_proto_descriptors.h` / `.cc` — proto descriptor
  validation at startup (catches missing generated descriptors).
- `BUILD` — extension registration.

## Interface
- `Stats::Sink::flush(MetricSnapshot&)` —
  `grpc_metrics_service_impl.h:152`. Delegates to
  `MetricsFlusher::flush()` to produce a `RepeatedPtrField` of
  Prometheus `MetricFamily`, then hands the buffer to
  `GrpcMetricsStreamer::send()`.
- `Stats::Sink::onHistogramComplete()` — no-op
  (`grpc_metrics_service_impl.h:155`). Histograms are read out of
  `snapshot.histograms()` at flush time.
- `GrpcMetricsStreamer::send(MetricsPtr&&)` — pure virtual
  (`grpc_metrics_service_impl.h:43`). Production impl at
  `grpc_metrics_service_impl.cc:31` lazily opens the bidi gRPC stream
  and writes one or more `StreamMetricsMessage` frames.

## Flow
1. Server stats flush timer fires → `MetricsServiceSink::flush(snapshot)`.
2. `MetricsFlusher::flush(snapshot)` (`grpc_metrics_service_impl.cc:87`)
   preallocates a `RepeatedPtrField`, iterates counters/gauges/histograms,
   skips unused metrics via the `predicate_` (`metric.used()` by default,
   `grpc_metrics_service_impl.h:100`), and fills
   `io::prometheus::client::MetricFamily` entries with a timestamp from
   `snapshot.snapshotTime()`.
3. `GrpcMetricsStreamerImpl::send()` (`grpc_metrics_service_impl.cc:31`):
   - If `stream_ == nullptr`, calls `client_->start(service_method_, ...)`
     and sets `send_identifier = true` so the first frame carries the node
     proto.
   - If `batch_size_ == 0` or the vector fits, sends the whole thing in
     one `StreamMetricsMessage` (`cc:47-49`).
   - Otherwise splits into contiguous ranges of `batch_size_` metrics and
     calls `sendBatch()` repeatedly; only the first batch gets the
     identifier (`cc:56-61`, `:77-80`).
4. `sendBatch()` (`cc:64`) copies entries into
   `message.mutable_envoy_metrics()`, optionally attaches
   `identifier.node`, and writes via `stream_->sendMessage(message, false)`.
5. On `onRemoteClose` the stream pointer is dropped
   (`grpc_metrics_service_impl.h:77-81`); the next flush reopens it and
   re-sends the identifier.

## Key decision points
- Prefix / tag handling is driven by `emit_tags_as_labels`
  (`config.cc:42`). When true, `MetricsFlusher::emit_labels_` causes each
  tag to be emitted as a Prometheus label; otherwise they are baked into
  the metric name. The Envoy stat prefix is preserved as-is in the
  emitted `MetricFamily.name`.
- `report_counters_as_deltas` (`config.cc:41`) toggles between cumulative
  counter values and per-flush deltas.
- `histogram_emit_mode` (`config.cc:42`) selects SUMMARY,
  HISTOGRAM, or both; flags stored in `emit_summary_` / `emit_histogram_`
  (`grpc_metrics_service_impl.h:102-105`). `flushHistogram()` and
  `flushSummary()` both emit into the same `MetricFamily` when
  `SUMMARY_AND_HISTOGRAM` is chosen.
- `validateProtoDescriptors()` is called up front in the factory
  (`config.cc:21`) to fail fast if the metrics-service descriptor pool is
  not linked in.
- `Config::Utility::checkTransportVersion` (`config.cc:27`) enforces v3
  transport. Missing or mismatched versions return `absl::Status` and
  abort sink creation.
- Streams are lazily (re)created on every flush if closed — errors on
  stream open are logged and deferred to the next flush
  (`grpc_metrics_service_impl.cc:38-42`). There is no retry/backoff; the
  stats flush interval acts as the natural retry cadence.
- The service method descriptor
  `envoy.service.metrics.v3.MetricsService.StreamMetrics` is resolved
  once in the constructor (`grpc_metrics_service_impl.cc:27`).

## Configuration

`envoy.config.metrics.v3.MetricsServiceConfig` fields:

- `grpc_service` (required) — `envoy.config.core.v3.GrpcService` pointing
  at the receiver cluster / Google gRPC endpoint.
- `transport_api_version` — must be `V3` (enforced at
  `config.cc:27`).
- `report_counters_as_deltas` (`BoolValue`) — emit deltas instead of
  cumulative totals (default `false`, `config.cc:41`).
- `emit_tags_as_labels` (bool) — project Envoy tags into Prometheus
  labels.
- `histogram_emit_mode` — `SUMMARY_AND_HISTOGRAM` (default), `SUMMARY`,
  or `HISTOGRAM`.
- `batch_size` (uint32) — metrics per `StreamMetricsMessage`. `0` means
  no batching (one message per flush).

## Stats / errors
- No sink-local Envoy counters — observability for this sink is provided
  by the `grpc_service`'s own upstream cluster stats.
- `onRemoteClose` logs a debug line with the gRPC status
  (`grpc_metrics_service_impl.h:77-81`) and nulls the stream; the next
  flush retries.
- Failure to create the async client surfaces as `absl::Status` from
  `createStatsSink` (`config.cc:32`).

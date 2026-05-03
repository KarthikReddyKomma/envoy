# OpenTelemetry metrics sink

OTLP metrics sink. Converts each Envoy `MetricSnapshot` into an
`opentelemetry.proto.collector.metrics.v1.ExportMetricsServiceRequest`
and ships it out over either gRPC
(`OpenTelemetryGrpcMetricsExporterImpl`) or HTTP/JSON
(`OpenTelemetryHttpMetricsExporter`). Supports cumulative or delta
temporality, optional per-stat matching / aggregation rewrites, resource
detection, and prefix tagging. Registered as
`envoy.stat_sinks.open_telemetry`.

Proto: `envoy.extensions.stat_sinks.open_telemetry.v3.SinkConfig`.

## Files
- `config.h` / `config.cc` — `OpenTelemetrySinkFactory`, oneof dispatch
  between gRPC and HTTP exporters.
- `open_telemetry_impl.h` / `.cc` — `OtlpOptions`, `OtlpMetricsFlusher`
  (snapshot -> OTLP request), `MetricAggregator`,
  `OpenTelemetryGrpcMetricsExporterImpl`, `OpenTelemetrySink`.
- `open_telemetry_http_impl.h` / `.cc` — HTTP/1.1 OTLP exporter using
  the cluster manager's HTTP async client.
- `open_telemetry_proto_descriptors.h` / `.cc` — startup descriptor
  validation.
- `stat_match_action.h` / `.cc` — match-tree actions used to drop or
  rewrite individual stats before export.
- `BUILD` — extension registration.

## Interface
- `Stats::Sink::flush(MetricSnapshot&)` —
  `open_telemetry_impl.h:315`. Timestamps the snapshot in ns, calls
  `OtlpMetricsFlusher::flush(snapshot, last_flush_ns, proxy_start_ns)`,
  pushes the resulting `ExportMetricsServiceRequest` into the exporter,
  and advances `last_flush_time_ns_`.
- `Stats::Sink::onHistogramComplete()` — no-op
  (`open_telemetry_impl.h:324`).
- `OtlpMetricsExporter::send(MetricsExportRequestPtr&&)` — pure virtual
  (`open_telemetry_impl.h:258`). Implemented by the gRPC
  (`OpenTelemetryGrpcMetricsExporterImpl`) and HTTP
  (`OpenTelemetryHttpMetricsExporter`) subclasses.
- `OtlpMetricsFlusher::flush()` — pure virtual
  (`open_telemetry_impl.h:185`). Production impl
  `OtlpMetricsFlusherImpl` (`open_telemetry_impl.h:195`) walks
  counters/gauges/histograms, converts each to an OTLP `Metric`, and
  packs them into a `ResourceMetrics` with the resource-detected
  attributes.

## Flow
1. Factory (`config.cc:16`) validates descriptors, builds
   `OtlpOptions` using the OpenTelemetry tracer's `ResourceProviderImpl`
   to get resource attributes (`config.cc:23-29`).
2. Oneof switch (`config.cc:33`):
   - `kGrpcService` → obtain a `RawAsyncClient`, construct
     `OpenTelemetryGrpcMetricsExporterImpl`.
   - `kHttpService` → construct `OpenTelemetryHttpMetricsExporter` using
     `server.clusterManager()` directly.
   - unspecified → `absl::InvalidArgumentError` (`config.cc:64`).
3. `OpenTelemetrySink` is initialised with
   `last_flush_time_ns_ = proxy_start_time_ns_ = current system time`
   (`open_telemetry_impl.h:311-312`).
4. On each flush timer tick:
   - `OtlpMetricsFlusherImpl::flush()` iterates snapshot.
     For each metric, `getMetricConfig()` runs the match tree
     (`stat_match_action.*`) to decide `drop_stat` or a per-metric
     `ConversionAction`.
   - Metric names are prefixed with `stat_prefix_` (set to
     `"<prefix>."` when non-empty, `open_telemetry_impl.cc:239`). If
     `use_tag_extracted_name` is true, tag-extracted names are used and
     tags become OTLP KeyValue attributes when
     `emit_tags_as_attributes` is true.
   - Temporality is chosen per counter/histogram via
     `report_counters_as_deltas` / `report_histograms_as_deltas`; the
     flusher sets `start_time_unix_nano` to either
     `last_flush_time_ns` (delta) or `proxy_start_time_ns` (cumulative)
     in `setCommonDataPoint()` (`open_telemetry_impl.h:113-131`).
   - If `enable_metric_aggregation` is true, points flow through
     `MetricAggregator` which groups data points by
     `(metric_name, attributes)` and sums duplicates
     (`open_telemetry_impl.h:43-142`); otherwise entries land directly in
     `non_aggregated_metrics_`.
   - Resource-level attributes from the detected resource are attached
     once per `ResourceMetrics`.
5. Exporter sends:
   - gRPC: `OpenTelemetryGrpcMetricsExporterImpl` uses method
     `opentelemetry.proto.collector.metrics.v1.MetricsService/Export`
     via `Grpc::AsyncClient::send`; `onSuccess`/`onFailure` just log.
   - HTTP: serializes the request to bytes and posts to the configured
     HTTP service cluster.

## Key decision points
- Temporality defaults: `AGGREGATION_TEMPORALITY_CUMULATIVE` unless the
  corresponding `report_*_as_deltas` flag is set. The start-time choice
  in `setCommonDataPoint()` (`open_telemetry_impl.h:120-130`) is what
  makes deltas measure the flush interval and cumulatives measure from
  proxy start.
- `use_tag_extracted_name` defaults to `true`
  (`open_telemetry_impl.cc:238`). When false the full tagged stat name is
  used verbatim and attributes are skipped.
- `emit_tags_as_attributes` defaults to `true`
  (`open_telemetry_impl.cc:236`).
- `stat_prefix_` is stored with the trailing `"."` pre-joined
  (`open_telemetry_impl.cc:239`) so the hot path can just concatenate.
- Matcher-driven drops/renames: the proto's `stat_config` matcher is
  compiled into a `MatchTree<StatMatchingData>`; actions live in
  `stat_match_action.*`. A `drop_stat` action skips the metric entirely.
- Aggregation is opt-in via `enable_metric_aggregation` — without it,
  each snapshot entry becomes its own OTLP `Metric` and no merging
  happens across the same name+attribute set.
- HTTP path builds its own message serialization inside
  `open_telemetry_http_impl.cc`; gRPC path reuses the async client
  framework which handles retries, queueing, and flow control.
- `validateProtoDescriptors()` at the top of `createStatsSink`
  (`config.cc:18`) fails fast if the OTLP proto descriptors aren't
  linked.

## Configuration

`envoy.extensions.stat_sinks.open_telemetry.v3.SinkConfig` fields:

- `grpc_service` or `http_service` (oneof, required) — OTLP endpoint.
- `report_counters_as_deltas` / `report_histograms_as_deltas` (bool) —
  switch to delta temporality for that metric type.
- `emit_tags_as_attributes` (`BoolValue`, default `true`) — project
  Envoy tags into OTLP `KeyValue` attributes.
- `use_tag_extracted_name` (`BoolValue`, default `true`) — use the
  tag-extracted stat name in `Metric.name`.
- `prefix` (string) — optional metric name prefix; joined with `"."`.
- `resource_detectors` (repeated) — OpenTelemetry resource detector
  configs; merged into the emitted `Resource`.
- `stat_config` — match tree controlling per-stat drop / conversion.
- `enable_metric_aggregation` — enable the `MetricAggregator`
  attribute-based merging.

## Stats / errors
- No sink-local Envoy counters. Observability for export failures is
  provided by the underlying gRPC/HTTP async client's cluster stats.
- `onFailure` (gRPC) and HTTP error paths log at debug level.
- Sink creation failures (bad cluster, descriptor validation, unknown
  protocol oneof) surface as `absl::Status` from `createStatsSink`.

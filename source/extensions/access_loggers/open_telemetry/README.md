# OpenTelemetry Access Logger

Access-log sink that emits OTLP `LogRecord`s to an OpenTelemetry collector. Records are formatted via the OTel-specific substitution formatter (body = `AnyValue`, attributes = `KeyValueList`), batched into a single `ExportLogsServiceRequest` (one `ResourceLogs` -> one `ScopeLogs` -> many `LogRecord`s), and pushed as an OTLP batch on a flush interval or when the buffered size crosses a threshold. Two transports are supported and selected at config time: OTLP/gRPC (`opentelemetry.proto.collector.logs.v1.LogsService.Export`, unary) and OTLP/HTTP (POST of binary-serialized `ExportLogsServiceRequest` through a configured Envoy cluster).

Proto: `envoy.extensions.access_loggers.open_telemetry.v3.OpenTelemetryAccessLogConfig` (see `api/envoy/extensions/access_loggers/open_telemetry/v3/logs_service.proto`). Factory name: `envoy.access_loggers.open_telemetry` (legacy alias `envoy.open_telemetry_access_log`) - config.cc:97.

## Files
- `access_log_impl.h` / `.cc` - `OpenTelemetry::AccessLog` (the gRPC variant of the `AccessLog::Instance`). Builds a `LogRecord` per stream and hands it to the per-thread logger.
- `http_access_log_impl.h` / `.cc` - `HttpAccessLog` / `HttpAccessLoggerImpl` (OTLP/HTTP variant). Buffers and serializes the full `ExportLogsServiceRequest` to a binary body and POSTs through `cluster_manager`.
- `grpc_access_log_impl.h` / `.cc` - `GrpcAccessLoggerImpl` (batched OTLP log-record pusher over unary gRPC) plus `GrpcAccessLoggerCacheImpl` singleton. Uses `Common::UnaryGrpcAccessLogClient` with per-request `OTelLogRequestCallbacks` to track `partial_success.rejected_log_records`.
- `config.h` / `.cc` - `AccessLogFactory` registration; singletons for both caches; transport selection (exactly one of `grpc_service`, `http_service`, or `common_config.grpc_service`).
- `otlp_log_utils.h` / `.cc` - helpers: `initOtlpMessageRoot`, `packBody`/`unpackBody`, `populateTraceContext`, `addFilterStateToAttributes`, `addCustomTagsToAttributes`, `getBufferFlushInterval`/`getBufferSizeBytes`, `getGrpcService`, `OtlpAccessLogStatsPrefix` constant.
- `substitution_formatter.h` / `.cc` - `OpenTelemetryFormatter` specialized for `KeyValueList`/`AnyValue` output.
- `access_log_proto_descriptors.h` / `.cc` - verifies the `LogsService.Export` descriptor is in the generated pool at config time.

## Interface
- `AccessLog::Instance::log(...)` -> `Common::ImplBase::log` -> `OpenTelemetry::AccessLog::emitLog` (access_log_impl.cc:60) or `HttpAccessLog::emitLog`.
- gRPC logger transport: `GrpcAccessLoggerImpl::addEntry(LogRecord&&)` (grpc_access_log_impl.cc:62); `initMessage()` is a no-op - the `ResourceLogs`/`ScopeLogs` root is built once in the constructor (grpc_access_log_impl.cc:44).
- HTTP logger: `HttpAccessLoggerImpl::log(LogRecord&&)` with its own in-process `flush()` and periodic timer.

## Flow
1. Config load (`config.cc:43`): validate descriptors, ensure exactly one transport is configured, parse substitution formatters, then build either:
   - `HttpAccessLog` if `http_service` is set, using the HTTP cache singleton; or
   - `OpenTelemetry::AccessLog` otherwise, using the gRPC cache singleton.
2. `OpenTelemetry::AccessLog` constructor (access_log_impl.cc:36): transport-version check, TLS slot init that lazily asks the cache for a `GrpcAccessLoggerImpl` keyed on the config. Body is stored as an `AnyValue` packed into a `KeyValueList` by `packBody` so it can share the `KeyValueList`-producing formatter; attributes are stored directly.
3. Per record (access_log_impl.cc:60): set `time_unix_nano` from `stream_info.startTime()`, run the body and attributes formatters, unpack the body back to `AnyValue`, call `populateTraceContext(log_entry, trace_id, span_id)` if an active span is present, then `addFilterStateToAttributes` and `addCustomTagsToAttributes`, and finally `logger_->log(std::move(log_entry))`.
4. Batching (gRPC): `GrpcAccessLoggerImpl::addEntry` appends into `root_->mutable_log_records()` and increments `batched_log_entries_` (grpc_access_log_impl.cc:62). The shared `Common::GrpcAccessLogger` base issues a flush when `approximate_message_size_bytes_ >= buffer_size_bytes` (default 16384) or when the `buffer_flush_interval` timer fires (default 1000 ms).
5. Push (gRPC): flush is a single unary `LogsService/Export` call with the current `ExportLogsServiceRequest`. After send, `OTelLogRequestCallbacks` increments `logs_written_` for accepted records and `logs_dropped_` for `partial_success.rejected_log_records` (grpc_access_log_impl.h:59-68). `clearMessage()` clears only `log_records` - `resource.attributes`, `scope`, and `log_name` persist across batches.
6. Push (HTTP): `HttpAccessLoggerImpl::flush()` serializes the `ExportLogsServiceRequest` to a byte string, looks up the configured cluster, and sends an async HTTP request with the binary body. On serialization failure or missing cluster, the batch is dropped and `root_->clear_log_records()` is called (http_access_log_impl.cc:65-80).
7. Reconnect: gRPC uses `GrpcCommon::optionalRetryPolicy(common_config)` (grpc_access_log_impl.cc:41) via `UnaryGrpcAccessLogClient`. On stream/unary failure the request callback runs `onFailure` which bumps `logs_dropped_` for the whole batch. The next `addEntry` triggers a fresh unary call - there is no persistent stream.

## Key decision points
- `config.cc:62-72` - enforces exactly one transport; zero or multiple throws `EnvoyException`.
- `grpc_access_log_impl.cc:39` - hard-coded method `opentelemetry.proto.collector.logs.v1.LogsService.Export`.
- `grpc_access_log_impl.h:35` - TODO to stop caching OTel loggers by `HTTP`/`TCP` type (both use `LogRecord`); today it reuses the generic `GrpcAccessLogger` cache.
- `grpc_access_log_impl.cc:44` - OTel message root is built once at logger creation (`initOtlpMessageRoot`) and only `log_records` is cleared on flush.
- `grpc_access_log_impl.h:59` - partial-success handling: if `rejected_log_records > sending_log_entries` (unexpected) the whole batch is counted as dropped.
- `access_log_impl.cc:54` - config `body` is stored as `KeyValueList` only if set; otherwise no body formatter is built.
- `access_log_impl.cc:76` - trace/span IDs sourced from `log_context.activeSpan()` when available.
- `http_access_log_impl.cc:54` - HTTP variant flushes synchronously when the accumulated byte budget is crossed; the timer handles idle flushing.
- `http_access_log_impl.cc:74-79` - HTTP flush drops the batch if the configured cluster is missing.

## Configuration
```yaml
access_log:
- name: envoy.access_loggers.open_telemetry
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.open_telemetry.v3.OpenTelemetryAccessLogConfig
    common_config:
      log_name: my_otlp_log
      transport_api_version: V3
      grpc_service:
        envoy_grpc: { cluster_name: otel_collector }
      buffer_flush_interval: 1s       # default 1s
      buffer_size_bytes: 16384         # default 16 KiB
    resource_attributes:
      values:
      - key: service.name
        value: { string_value: "envoy" }
    body:
      string_value: "%REQ(:METHOD)% %REQ(:PATH)%"
    attributes:
      values:
      - key: http.status_code
        value: { string_value: "%RESPONSE_CODE%" }
    stat_prefix: main
```
Alternative transports: `http_service: { http_uri: { cluster: otel_http, uri: "http://otel/v1/logs" } }` or a top-level `grpc_service`.

## Stats / errors
- Stats prefix: `<OtlpAccessLogStatsPrefix><config.stat_prefix>` (grpc_access_log_impl.cc:43, http_access_log_impl.cc:37). `OtlpAccessLogStatsPrefix` is defined in `otlp_log_utils.h`.
- gRPC counters come from `ALL_GRPC_ACCESS_LOGGER_STATS` (`common/grpc_access_logger.h`): `logs_written`, `logs_dropped`, plus client-level stream stats.
- HTTP counters use `ALL_OTLP_ACCESS_LOG_STATS` (see `http_access_log_impl.h`).
- Config-time errors: descriptor missing (config.cc:47), transport misconfiguration (config.cc:62-72), unsupported transport version (access_log_impl.cc:46).
- Runtime errors: partial collector rejection increments `logs_dropped`; HTTP serialization failure or missing cluster logs a warning and drops the batch.

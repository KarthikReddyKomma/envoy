# gRPC Access Loggers (HTTP + TCP)

Access-log sinks that stream `envoy.service.accesslog.v3.StreamAccessLogsMessage` records over a bidi gRPC stream (`AccessLogService.StreamAccessLogs`). Two factories live here:

- `envoy.access_loggers.http_grpc` - streams `HTTPAccessLogEntry` records (HCM / HTTP filters).
- `envoy.access_loggers.tcp_grpc` - streams `TCPAccessLogEntry` records (TCP proxy, network filters).

Both share one transport (`GrpcAccessLoggerImpl`) which packs entries into a single `StreamAccessLogsMessage`, buffers by count/bytes, flushes on a timer, and reconnects on stream failure with a retry policy. One logger per `(grpc_service, log_name)` is kept in a process-wide singleton cache so multiple listener/filter instances share one stream.

Protos:
- `envoy.extensions.access_loggers.grpc.v3.HttpGrpcAccessLogConfig`
- `envoy.extensions.access_loggers.grpc.v3.TcpGrpcAccessLogConfig`
- `envoy.extensions.access_loggers.grpc.v3.CommonGrpcAccessLogConfig`
- Wire protos: `envoy.data.accesslog.v3.HTTPAccessLogEntry` / `TCPAccessLogEntry`, `envoy.service.accesslog.v3.StreamAccessLogsMessage`.

See `api/envoy/extensions/access_loggers/grpc/v3/als.proto` and `api/envoy/service/accesslog/v3/`.

## Files
- `grpc_access_log_impl.h` / `.cc` - `GrpcAccessLoggerImpl` (the streaming gRPC client wrapper) and `GrpcAccessLoggerCacheImpl` (singleton cache, one logger per config).
- `http_grpc_access_log_impl.h` / `.cc` - `HttpGrpcAccessLog` access-log `Instance`; formats `HTTPAccessLogEntry` (request headers/trailers etc.) via `GrpcCommon::Utility` then hands to the logger.
- `tcp_grpc_access_log_impl.h` / `.cc` - `TcpGrpcAccessLog` access-log `Instance`; formats `TCPAccessLogEntry` (connection properties only).
- `http_config.h` / `.cc` - `HttpGrpcAccessLogFactory` (`envoy.access_loggers.http_grpc`, legacy alias `envoy.http_grpc_access_log`).
- `tcp_config.h` / `.cc` - `TcpGrpcAccessLogFactory` (`envoy.access_loggers.tcp_grpc`).
- `config_utils.h` / `.cc` - singleton accessor `getGrpcAccessLoggerCacheSingleton(...)`.
- `grpc_access_log_utils.h` / `.cc` - `GrpcCommon::Utility::extractCommonAccessLogProperties(...)` and helpers that fill `AccessLogCommon` from `StreamInfo`.
- `grpc_access_log_proto_descriptors.h` / `.cc` - validates that the linked protobuf descriptor pool knows `AccessLogService.StreamAccessLogs` (guards against missing dependencies).

Batching/flush/reconnect logic is inherited from `source/extensions/access_loggers/common/grpc_access_logger.h` (`Common::GrpcAccessLogger<...>`). The underlying streaming client comes from `source/extensions/access_loggers/common/grpc_access_logger_clients.h` (`StreamingGrpcAccessLogClient`).

## Interface
- `AccessLog::Instance::log(...)` -> `Common::ImplBase::log` -> `HttpGrpcAccessLog::emitLog` / `TcpGrpcAccessLog::emitLog` (http_grpc_access_log_impl.h:49, tcp_grpc_access_log_impl.h:48).
- Logger transport API (`Common::GrpcAccessLogger`): `log(entry)` -> `addEntry` -> maybe `flush()`.
- Cache: `GrpcAccessLoggerCacheImpl::createLogger` (grpc_access_log_impl.cc:57) is the factory for new streams.

## Flow
1. Config load: each factory validates the proto, fetches the singleton cache via `getGrpcAccessLoggerCacheSingleton(context)`, and constructs the access-log `Instance` (http_config.cc:21, tcp_config.cc similarly).
2. Per worker thread, the `Instance` allocates a `ThreadLocal::Slot` that lazily asks the cache for a `GrpcAccessLoggerImpl` keyed by `grpc_service + log_name`. The cache either returns an existing logger or calls `createLogger` (grpc_access_log_impl.cc:57), which asks `AsyncClientManager::factoryForGrpcService(grpc_service, scope, skip_cluster_check=true)` for a raw async client, builds a `StreamingGrpcAccessLogClient` pointed at `envoy.service.accesslog.v3.AccessLogService.StreamAccessLogs` (grpc_access_log_impl.cc:28), and wraps it in `GrpcAccessLoggerImpl`.
3. Per record, `emitLog` populates an `HTTPAccessLogEntry` / `TCPAccessLogEntry` using `GrpcCommon::Utility` (common properties from `StreamInfo`, plus any request/response/trailer headers listed in the config) and calls `logger_->log(std::move(entry))`.
4. `GrpcAccessLoggerImpl::addEntry` pushes the entry into `message_.mutable_http_logs()->mutable_log_entry()` (or `tcp_logs`) (grpc_access_log_impl.cc:33-39). The base template tracks `approximate_message_size_bytes_` (sum of `entry.ByteSizeLong()`).
5. Flush triggers (see `common/grpc_access_logger.h:110-180`):
   - Buffered bytes >= `buffer_size_bytes` (default 16384) force an immediate send.
   - A periodic timer fires every `buffer_flush_interval` (default 1000 ms) and sends if non-empty.
6. On first send, `initMessage` fills the one-time `identifier` (node info + `log_name`) before the message ships (grpc_access_log_impl.cc:45-49). Subsequent messages on the same stream omit the identifier.
7. Reconnect: if the gRPC stream breaks, `StreamingGrpcAccessLogClient` reopens it on the next send, honoring `CommonGrpcAccessLogConfig.grpc_stream_retry_policy` (see `GrpcCommon::optionalRetryPolicy(config)` at grpc_access_log_impl.cc:30). Pending entries in the current `message_` survive because `isEmpty()` keeps them until a successful send.

## Key decision points
- `grpc_access_log_impl.cc:28` - hard-coded service method `envoy.service.accesslog.v3.AccessLogService.StreamAccessLogs`. The descriptor is verified by `grpc_access_log_proto_descriptors.cc` at config time.
- `grpc_access_log_impl.cc:41` - `isEmpty()` checks both `http_logs` and `tcp_logs` presence; used to decide whether to flush on timer.
- `grpc_access_log_impl.cc:45` - identifier (node + log_name) is populated on each new message start in the base template's `flush()` path.
- `grpc_access_log_impl.cc:61` - `skip_cluster_check=true` avoids throwing in worker threads; main thread should validate cluster availability.
- `common/grpc_access_logger.h:116` - `buffer_size_bytes` default 16384; `common/grpc_access_logger.h:110-111` - `buffer_flush_interval` default 1000 ms.
- `http_config.cc:46` - legacy factory alias `envoy.http_grpc_access_log` is kept for back-compat.
- `config_utils.h` - the cache is a pinned singleton so it outlives individual listeners.

## Configuration
```yaml
access_log:
- name: envoy.access_loggers.http_grpc
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.grpc.v3.HttpGrpcAccessLogConfig
    common_config:
      log_name: my_http_log          # appears in the stream identifier
      transport_api_version: V3
      grpc_service:
        envoy_grpc: { cluster_name: als_cluster }
      buffer_flush_interval: 1s      # default 1s
      buffer_size_bytes: 16384        # default 16 KiB
      grpc_stream_retry_policy:
        retry_back_off:
          base_interval: 0.5s
          max_interval: 5s
        num_retries: { value: 5 }
    additional_request_headers_to_log: ["x-user-id"]
    additional_response_headers_to_log: ["x-upstream-id"]
    additional_response_trailers_to_log: []
```
The TCP variant uses `TcpGrpcAccessLogConfig` with the same `common_config`.

## Stats / errors
- Stats prefix: `access_logs.grpc_access_log.` (grpc_access_log_impl.cc:12).
- Standard counters from `Common::GrpcAccessLogger`: `logs_written`, `logs_dropped`, `stream_closed_after_retry_limit_exceeded`, etc. (see `common/grpc_access_logger.h`).
- Config-time errors:
  - Missing `AccessLogService.StreamAccessLogs` descriptor -> `validateProtoDescriptors()` throws (http_config.cc:25).
  - Async client factory failure -> `THROW_IF_NOT_OK_REF` at grpc_access_log_impl.cc:66.
- Runtime errors: stream breakage is handled by the streaming client and retry policy; after the retry budget is exhausted the underlying message is dropped.

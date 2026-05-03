# Common

Shared helpers that are re-used by most access logger extensions. This directory does not register a factory; the code is pulled in as a library by sibling extensions (`file`, `grpc`, `open_telemetry`, `fluentd`, `stream`, ...).

No proto is owned by this folder. Consumers use `envoy.extensions.access_loggers.grpc.v3.CommonGrpcAccessLogConfig` and sibling protos.

## Files
- `access_log_base.h/cc` — `Common::ImplBase`, a base `AccessLog::Instance` that encapsulates filter evaluation and defers writing to a subclass `emitLog()`.
- `file_access_log_impl.h/cc` — `AccessLoggers::File::FileAccessLog`, writes a formatted record through `AccessLog::AccessLogManager` (used by the `file` and `stream` loggers).
- `stream_access_log_common_impl.h` — `createStreamAccessLogInstance<T, destination_type>()` template used by the `stream` stdout/stderr factories and by the file factory through its config message.
- `grpc_access_logger.h` — `Common::GrpcAccessLogger` and `Common::GrpcAccessLoggerCache` class templates that buffer HTTP/TCP log entries, batch flush them through a gRPC client, own the flush timer, and keep per-thread caches keyed by config hash + `GrpcAccessLoggerType`.
- `grpc_access_logger_clients.h` — `GrpcAccessLogClient` abstract client plus `UnaryGrpcAccessLogClient` (sends each batch via `client_->send`) and `StreamingGrpcAccessLogClient` (maintains a persistent `AsyncStream` with high-watermark backpressure).
- `grpc_access_logger_utils.h/cc` — `GrpcCommon::optionalRetryPolicy()` extracts a `RetryPolicy` from `CommonGrpcAccessLogConfig` for unary clients.

## Sink / logger role
- Implements the `AccessLog::Instance::log()` entry point. Concrete subclasses implement `emitLog()`; the base handles filter gating.
- `GrpcAccessLogger::log()` does not itself implement `AccessLog::Instance` — it is the per-config batching engine called by the gRPC logger extensions.

## Flow
1. `ImplBase::log()` runs `filter_->evaluate()` and returns if the filter rejects the entry; otherwise delegates to `emitLog()`.
2. `FileAccessLog::emitLog()` formats the record with `Formatter::FormatterPtr::format()` and calls `log_file_->write()` on the shared `AccessLogFile` acquired through `AccessLogManager::createAccessLog()`.
3. `GrpcAccessLogger::log(entry)` calls `canLogMore()` (HTTP only, accounts for backpressure in streaming mode), appends the entry through `addEntry()` (virtual, owned by the concrete logger), and flushes when `approximate_message_size_bytes_ >= max_buffer_size_bytes_`.
4. `flush()` calls `initMessage()` (if disconnected), `client_->log(message_)`, then `clearMessage()`. A periodic `flush_timer_` re-arms itself with `buffer_flush_interval_msec_`.
5. `UnaryGrpcAccessLogClient::log()` always returns `true` (fire and forget send). `StreamingGrpcAccessLogClient::log()` returns `false` when the async stream is above its write-buffer high watermark, causing the caller to drop the entry and bump `logs_dropped_`.

## Key decision points
- `access_log_base.cc:14` — filter short-circuit before `emitLog()`.
- `grpc_access_logger.h:116` — `flush_timer_` enabled at construction with `buffer_flush_interval` (default 1000 ms).
- `grpc_access_logger.h:130` — size-triggered flush once `approximate_message_size_bytes_ >= max_buffer_size_bytes_` (default 16384 B).
- `grpc_access_logger.h:179` — `canLogMore()` drops entries when the buffer is saturated after a flush attempt.
- `grpc_access_logger_clients.h:131` — streaming client refuses to send when `isAboveWriteBufferHighWatermark()`.
- `grpc_access_logger.h:234` — cache key is `{MessageUtil::hash(config), logger_type}`, so HTTP and TCP loggers for the same config are separate instances.

## Configuration
Owned by sibling extensions. This library reads from `CommonGrpcAccessLogConfig`:
- `buffer_flush_interval` (default 1 s).
- `buffer_size_bytes` (default 16384 B; set to 0 to disable the size trigger).
- `grpc_stream_retry_policy` (used only for unary clients via `optionalRetryPolicy()`).

## Stats / errors
- `logs_written` / `logs_dropped` counters (streaming clients only), scoped under the prefix passed into the `GrpcAccessLogger` constructor; unary clients do not populate them.
- `FileAccessLog` constructor throws via `THROW_IF_NOT_OK_REF` if `AccessLogManager::createAccessLog()` fails (typically bad path / permission).

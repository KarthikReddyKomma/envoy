# Fluentd Access Logger

Access-log sink that speaks the Fluentd Forward Protocol v1 over TCP. Each log record is formatted as JSON, packed into MessagePack, batched in memory, and periodically flushed as a single Fluentd "Forward" event (`[tag, [[time, record], [time, record], ...]]`) to a Fluentd aggregator reachable through a named Envoy cluster. Connection failures are retried with a jittered exponential backoff; the logger and its backing TCP connection are shared across access-log instances that target the same cluster/tag via a process-wide singleton cache.

Proto: `envoy.extensions.access_loggers.fluentd.v3.FluentdAccessLogConfig` (see `api/envoy/extensions/access_loggers/fluentd/v3/fluentd.proto`). Factory name: `envoy.access_loggers.fluentd` (config.cc:86).

## Files
- `fluentd_access_log_impl.h` / `fluentd_access_log_impl.cc` - `FluentdAccessLog` (the `AccessLog::Instance`), `FluentdAccessLoggerImpl` (one-per-cluster TCP writer), and `FluentdAccessLoggerCacheImpl` (singleton cache).
- `config.h` / `config.cc` - `FluentdAccessLogFactory` registered as `AccessLog::AccessLogInstanceFactory`; also owns the `fluentd_access_logger_cache` singleton.
- `substitution_formatter.h` / `substitution_formatter.cc` - `FluentdFormatterImpl` wrapping the standard JSON formatter and converting its output to a MessagePack payload.

Shared framing / batching lives in `source/extensions/common/fluentd/fluentd_base.{h,cc}` (`FluentdBase`, `FluentdCacheBase`, `MessagePackPacker`, `Entry`).

## Interface
- `AccessLog::Instance::log(...)` -> `Common::ImplBase::log` -> `FluentdAccessLog::emitLog(const Formatter::Context&, const StreamInfo::StreamInfo&)` (fluentd_access_log_impl.cc:69).
- Serialization: `FluentdAccessLoggerImpl::packMessage(MessagePackPacker&)` (fluentd_access_log_impl.cc:27).

## Flow
1. Config load (`config.cc:34`): validate the config, check that `proto_config.cluster()` is an active static cluster, and validate backoff bounds. Build a JSON formatter from `proto_config.record()`, wrap in `FluentdFormatterImpl`, and hand everything to the `FluentdAccessLog` constructor.
2. `FluentdAccessLog` allocates a `ThreadLocal::Slot` and, on each worker, creates a `JitteredExponentialBackOffStrategy` and calls `FluentdAccessLoggerCacheImpl::getOrCreate(config, random, backoff)` (fluentd_access_log_impl.cc:60). The cache dedupes on `(cluster, tag, stat_prefix, ...)` so multiple access-log configs share one TCP connection.
3. Per record: `emitLog` runs the formatter to produce a MessagePack byte vector, captures `stream_info.timeSource().systemTime()` truncated to seconds, and calls `logger_->log(Entry{time, bytes})` (fluentd_access_log_impl.cc:75).
4. `FluentdBase::log` appends to `entries_`. When accumulated bytes exceed `buffer_size_bytes` or the flush timer fires (`buffer_flush_interval`), `FluentdBase` serializes via `packMessage` (fluentd_access_log_impl.cc:27) and writes the resulting MessagePack buffer to the upstream TCP client.
5. Framing: `packMessage` emits a Fluentd Forward-mode event - a top-level 2-element array of `[tag, entries]`, where each entry is `[time, record_bytes]` packed as a MessagePack bin (fluentd_access_log_impl.cc:30-39).
6. Reconnect: on TCP errors the `FluentdBase` state machine closes the client and reopens it, respecting `max_connect_attempts` (fluentd_access_log_impl.cc:20-22) and the jittered exponential backoff. Exhausting reconnect attempts trips a stat and drops further records.

## Key decision points
- `config.cc:42` - reject config if `proto_config.cluster()` is not an active static cluster.
- `config.cc:48` - reject backoff configs where `max_interval < base_interval`.
- `fluentd_access_log_impl.cc:24` - buffer flush interval defaults to `DefaultBufferFlushIntervalMs = 1000` ms; buffer size defaults to `DefaultMaxBufferSize = 16384` bytes (see `source/extensions/common/fluentd/fluentd_base.h:25-26`).
- `fluentd_access_log_impl.cc:49-50` - backoff defaults: `base = 500` ms, `max = base * 10` (`fluentd_base.h:23-24`).
- `fluentd_access_log_impl.cc:58` - JSON-then-msgpack approach is acknowledged as a TODO - a native msgpack formatter would skip the intermediate JSON.
- `config.cc:21` - `FluentdAccessLoggerCacheImpl` is a pinned singleton; access-log instances across listeners/workers share it.

## Configuration
```yaml
access_log:
- name: envoy.access_loggers.fluentd
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.fluentd.v3.FluentdAccessLogConfig
    cluster: fluentd_cluster         # required; must be a static cluster
    tag: envoy.access                 # Fluentd tag
    stat_prefix: my_fluentd           # stats namespace suffix
    buffer_size_bytes: 16384          # default 16 KiB
    buffer_flush_interval: 1s         # default 1s
    retry_options:
      max_connect_attempts: 5
      backoff_options:
        base_interval: 0.5s
        max_interval: 5s
    record:
      path: "%REQ(:PATH)%"
      status: "%RESPONSE_CODE%"
```

## Stats / errors
Scope is `access_logs.fluentd.<stat_prefix>.` (see `FluentdAccessLoggerCacheImpl` constructor at fluentd_access_log_impl.cc:42 passing `"access_logs.fluentd"`). Counters / gauges are owned by `FluentdBase` in `source/extensions/common/fluentd/fluentd_base.{h,cc}` and typically include `entries_buffered`, `entries_lost`, `events_sent`, `connections_closed`, `reconnect_attempts`.

Errors:
- Missing upstream cluster -> config-time `EnvoyException` (config.cc:45).
- Invalid backoff -> config-time `EnvoyException` (config.cc:53).
- Upstream disconnects -> backoff + reconnect; after `max_connect_attempts` the logger stops and increments the drop counter.

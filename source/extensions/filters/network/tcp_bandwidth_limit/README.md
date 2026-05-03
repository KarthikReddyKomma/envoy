# TCP Bandwidth Limit (`envoy.filters.network.tcp_bandwidth_limit`)

Bidirectional L4 rate limiter for TCP connections. Independent per-direction token buckets (shared across all filter instances that share the same config) cap the byte rate on reads (`onData`) and writes (`onWrite`). When a direction runs out of tokens, excess bytes are buffered in a watermark buffer and drained on a dispatcher timer that fires every `fill_interval`.

Proto: `envoy.extensions.filters.network.tcp_bandwidth_limit.v3.TcpBandwidthLimit`.

## Files
- `config.h/cc` — `TcpBandwidthLimitConfigFactory`, registered as `envoy.filters.network.tcp_bandwidth_limit` (`config.h:15`). Builds `FilterConfig` and adds the filter to the filter manager (`config.cc:17`).
- `tcp_bandwidth_limit.h/cc` — `FilterConfig` (`tcp_bandwidth_limit.h:49`) holding the per-direction `SharedTokenBucketImpl` instances and the stats struct; `TcpBandwidthLimitFilter` (`tcp_bandwidth_limit.h:85`) implementing `Network::Filter`.

## Lifecycle
- `onNewConnection()` — no-op; returns `Continue` (`tcp_bandwidth_limit.h:92`).
- `initializeReadFilterCallbacks(cb)` / `initializeWriteFilterCallbacks(cb)` — store the callbacks and set watermark thresholds on the buffered-data buffers from `connection().bufferLimit()` (`tcp_bandwidth_limit.h:93`, `:100`). Note that `read_buffer_` watermarks are set during write init and vice-versa because each buffer guards flow from the opposite direction.
- `onData(data, end_stream)` — read path. Returns `Continue` when disabled or no read limit. Otherwise consumes tokens and may buffer (`tcp_bandwidth_limit.cc:51`).
- `onWrite(data, end_stream)` — write path, mirror of `onData` (`tcp_bandwidth_limit.cc:96`).
- Destructor disables any pending timers (`tcp_bandwidth_limit.cc:40`).

## Decision / logic
Read path (`tcp_bandwidth_limit.cc:51`):
1. Disabled or no `read_token_bucket_`? Pass through unmodified (`:52`).
2. If there is already buffered data, move new bytes into `read_buffer_`, preserve byte order, return `StopIteration` (`:59`).
3. Call `SharedTokenBucketImpl::consume(data_size, /*allow_partial=*/true)` (`:68`). If fully consumed, inject through and return `Continue` (`:92`).
4. If partially consumed, inject the `consumed` prefix via `injectReadDataToFilterChain` (`:76`), move the rest into `read_buffer_`, arm `read_timer_` at `fill_interval` and return `StopIteration` (`:84`).

Timer-driven drain in `onReadTokenTimer` / `processBufferedReadData` (`tcp_bandwidth_limit.cc:141`, `:169`) repeatedly drains `read_buffer_` until empty; `end_stream` is forwarded only when the buffer is fully drained (`:180`).

Watermark callbacks: `onReadBufferHighWatermark` disables reads on the downstream connection; `onReadBufferLowWatermark` re-enables them (`tcp_bandwidth_limit.cc:161`). Write side delegates to `WriteFilterCallbacks::onAboveWriteBufferHighWatermark`/`onBelowWriteBufferLowWatermark` to propagate backpressure (`:165`).

`updateReadRate`/`updateWriteRate` (`:205`, `:218`) update `*_total_bytes_` and recompute `*_rate_bps_` gauges every `RateUpdateIntervalMs` (1000 ms).

## Configuration
- `read_limit_kbps`, `write_limit_kbps` — kBps caps; when absent, that direction is not rate-limited (`tcp_bandwidth_limit.cc:28`).
- `fill_interval` — token bucket refill period; default 50 ms (`tcp_bandwidth_limit.cc:23`).
- `runtime_enabled` — `Runtime::FeatureFlag`, gated per-connection (`tcp_bandwidth_limit.cc:24`).
- `stat_prefix` — prepended to `.tcp_bandwidth_limit.` in stat names (`tcp_bandwidth_limit.cc:32`).

Token buckets are `SharedTokenBucketImpl` (`tcp_bandwidth_limit.h:62`) so all connections sharing the same `FilterConfig` contend for the same pool.

## Stats
Prefix: `<stat_prefix>.tcp_bandwidth_limit.` (see `ALL_TCP_BANDWIDTH_LIMIT_STATS`, `tcp_bandwidth_limit.h:27`):
- Counters: `read_enabled`, `write_enabled`, `read_throttled`, `write_throttled`, `read_total_bytes`, `write_total_bytes`.
- Gauges: `read_bytes_buffered`, `write_bytes_buffered` (Accumulate); `read_rate_bps`, `write_rate_bps` (NeverImport).

## Factory
`TcpBandwidthLimitConfigFactory::createFilterFactoryFromProtoTyped` constructs the shared `FilterConfig` and returns a lambda that instantiates one `TcpBandwidthLimitFilter` per connection via `filter_manager.addFilter` (a read + write filter). Registered through `REGISTER_FACTORY` at `config.cc:20`.

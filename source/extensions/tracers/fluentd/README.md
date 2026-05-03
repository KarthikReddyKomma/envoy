# Fluentd tracer (`envoy.tracers.fluentd`)

Tracer that emits spans as Fluentd log records over a MessagePack Forward
protocol TCP connection. Each finished span becomes a Fluentd entry with
`operation`, `trace_id`, `span_id`, `start_time`, `end_time`, plus any tags
set on the span. Context propagation piggybacks on the W3C
`traceparent`/`tracestate` headers so spans emitted by this tracer can be
stitched together by an external collector.

Proto: `envoy.extensions.tracers.fluentd.v3.FluentdConfig`.

## Files
- `config.h/cc` - `FluentdTracerFactory` registered with
  `REGISTER_FACTORY(..., TracerFactory)`. Creates/reuses a singleton
  `FluentdTracerCacheImpl` and returns the `Driver`.
- `fluentd_tracer_impl.h/cc` - contains:
  - `Driver` (thread-local slot holding the per-thread tracer).
  - `FluentdTracerImpl` (inherits `FluentdBase` from
    `source/extensions/common/fluentd`) - owns the TCP buffering,
    backoff/retry, and `packMessage` for the Forward protocol.
  - `FluentdTracerCacheImpl` - singleton keyed by config so multiple
    listeners/filters share the same TCP connection.
  - `Span` - holds the W3C `SpanContext`, accumulates tags, and on
    `finishSpan` constructs a Fluentd `Entry` and hands it to the tracer.
  - `SpanContextExtractor` - parses the incoming `traceparent` header.

## Tracer role
- `Tracing::Driver::startSpan(...)` - `Driver::startSpan` extracts the
  parent `SpanContext` from `traceparent` when present, otherwise creates a
  fresh one whose sampled bit follows `tracing_decision.traced`
  (`fluentd_tracer_impl.cc:141`).
- `Span::injectContext` - writes `traceparent` as
  `00-<trace_id>-<span_id>-<flags>` and forwards `tracestate` if any
  (`fluentd_tracer_impl.cc:231`).
- `Span::finishSpan` - packs the span as a map record keyed by `operation`,
  `trace_id`, `span_id`, `start_time`, `end_time`, plus tag entries, and
  calls `tracer_->log(std::move(entry))` (`fluentd_tracer_impl.cc:211`).

## Flow
- Span creation - `Driver::startSpan` checks the thread-local tracer and
  delegates to `FluentdTracerImpl::startSpan` (root or child).
- Context propagation - W3C `traceparent` + `tracestate` only; `setTag`
  becomes a key in the map record.
- Batching - `FluentdBase` buffers entries up to
  `buffer_size_bytes` (default from `DefaultMaxBufferSize`) or
  `buffer_flush_interval` (default `DefaultBufferFlushIntervalMs`) and then
  calls `packMessage`.
- Export - TCP to the upstream cluster named by `FluentdConfig.cluster`;
  MessagePack array of `[tag, entries, option]` where `option` includes
  `fluent_signal=2` and `TimeFormat=DateTime`
  (`fluentd_tracer_impl.cc:297`).

## Key decision points
- `fluentd_tracer_impl.cc:20-24` - exact `traceparent` format constants
  (55 bytes, 4 hyphen-separated fields).
- `fluentd_tracer_impl.cc:53-82` - strict size/hex/all-zeros validation on
  `traceparent`; any failure returns a `NullSpan`.
- `fluentd_tracer_impl.cc:122-128` - retry backoff honors
  `retry_policy.retry_back_off.base_interval/max_interval`, else defaults
  to `DefaultBaseBackoffIntervalMs` * `DefaultMaxBackoffIntervalFactor`.
- `fluentd_tracer_impl.cc:180` - option map `{fluent_signal:2,
  TimeFormat:DateTime}` hardcoded.

## Configuration
- `cluster` - upstream TCP cluster reaching the Fluentd endpoint.
- `tag` - Fluentd tag applied to every record.
- `stat_prefix` - prefix for the stats scope.
- `buffer_flush_interval`, `buffer_size_bytes` - buffering thresholds.
- `retry_policy.num_retries`, `retry_policy.retry_back_off` - transport
  retries.

## Stats / errors
- All counters emitted by `FluentdBase` under `tracing.fluentd.*` (via
  `FluentdTracerCacheImpl("tracing.fluentd")`, `fluentd_tracer_impl.h:111`):
  records dropped, records sent, connection failures, backoff events, etc.
- `traceparent` parse failures log at `trace` level and return
  `Tracing::NullSpan` (`fluentd_tracer_impl.cc:160`).
- `getBaggage`/`setBaggage` are no-ops.

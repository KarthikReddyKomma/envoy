# Stats Access Logger

An unusual access-log sink: instead of emitting records to a file or wire, it turns each log event into one or more **stat updates** (counter increments, histogram observations, or gauge operations) inside a dedicated evictable scope. Metric names and tag values can be templated with the access-log substitution language so the logger effectively materializes per-request metrics (e.g. "count 5xx per upstream cluster", "histogram of response sizes per virtual host"). Gauge operations support paired ADD/SUBTRACT across different `AccessLogType`s (e.g. `DownstreamStart` +1, `DownstreamEnd` -1) to track inflight work; paired state lives in a `StreamInfo::FilterState` object so the matching SUBTRACT fires even on abnormal teardown.

Proto: `envoy.extensions.access_loggers.stats.v3.Config` (see `api/envoy/extensions/access_loggers/stats/v3/stats.proto`). Factory name: `envoy.access_loggers.stats` (config.cc:29).

## Files
- `stats.h` / `stats.cc` - `StatsAccessLog` (the `AccessLog::Instance`), internal types `NameAndTags`, `DynamicTag`, `Histogram`, `Counter`, `Gauge`; `GaugeKey` (lock-free lookup key with borrowed-then-owned tags); `AccessLogState` (per-stream `FilterState::Object` that owns inflight gauge deltas).
- `config.h` / `config.cc` - `AccessLogFactory` registered as `AccessLog::AccessLogInstanceFactory`.

## Interface
- `AccessLog::Instance::log(...)` -> `Common::ImplBase::log` -> `StatsAccessLog::emitLog` -> `StatsAccessLog::emitLogConst` (stats.cc:286). `emitLog` delegates to the const version so all mutation is channelled through the scope / `FilterState`, keeping worker concurrency safe.

## Flow
1. Config load (stats.cc:143): create an **evictable** scope (`config.stat_prefix()`) off `context.statsScope()`, then eagerly parse each histogram / counter / gauge entry. `NameAndTags` interns the fixed stat name plus any literal tag names into the logger's `StatNamePool`; dynamic tag values become `DynamicTag{value_formatter, matcher-tree rules}` so tag values can be produced from substitution commands at record time.
2. On each log event, `emitLogConst` walks `histograms_`, `counters_`, then `gauges_`:
   - For every entry, call `NameAndTags::tags(context, stream_info, scope)` which runs each `DynamicTag`'s formatter to resolve the value, passes it through the `Matcher::MatchTree` transform rules (for normalization / bucketing), and reports a `dropped` flag if any tag resolved to a value that's meant to suppress the record.
   - If `dropped` is true the record is skipped for that stat (stats.cc:292, 310, 342).
3. Histogram path (stats.cc:289): run the value formatter (converting to `PercentScale` if `Unit::Percent`), look up `scope_->histogramFromStatNameWithTags(name, tags, unit)`, and `recordValue(value)`.
4. Counter path (stats.cc:307): run the value formatter if present else use `value_fixed_`, look up `counterFromStatNameWithTags`, and `add(value)`.
5. Gauge path (stats.cc:335): only act when the current `context.accessLogType()` matches one of the configured `(AccessLogType, OperationType)` pairs. `SET` uses `ImportMode::NeverImport` and calls `scope.gaugeFromStatNameWithTags(...).set(v)`; `PAIRED_ADD`/`PAIRED_SUBTRACT` delegate to `AccessLogState` on the stream's `FilterState` so the ADD is remembered and paired with a guaranteed SUBTRACT.
6. Paired inflight gauges (`AccessLogState` - stats.cc:64-110):
   - `addInflightGauge` looks up the `GaugeKey`, either sums into an existing entry or inserts a new one (taking ownership of the dynamic-tag storage to keep `StatName`s alive).
   - `removeInflightGauge` subtracts; when the running value hits 0 the entry is erased, letting the scope evict the gauge.
   - The `AccessLogState` destructor applies a final SUBTRACT for anything still inflight, so a half-completed stream never leaks a `+1`.

## Key decision points
- `stats.cc:143` - scope is created with `evictable=true` so stat entries with dynamic tags don't pin forever.
- `stats.h:32-78` - `GaugeKey` intentionally supports a borrowed-tags view for zero-alloc lookups; `makeOwned()` is only called before insertion.
- `stats.h:164-168` - `addInflightGauge` ignores zero values; this is how tag-dropping flows (tags_dropped -> value=0) become no-ops.
- `stats.cc:286-288` - all mutation goes through a const helper; the class is designed to be called concurrently from worker threads.
- `stats.cc:336-338` - gauge operations are keyed to `AccessLogType`, so the same config can define `+1` on `DownstreamStart` and `-1` on `DownstreamEnd` without a per-type config block.
- `stats.cc:280` - histograms declared `Unit::Percent` are multiplied by `Histogram::PercentScale` before recording.
- `stats.h:75` - dynamic tag `StatName`s are kept alive via `StatNameDynamicStorage` that is moved into the `InflightGauge`, because the symbol-table entries are refcounted.

## Configuration
```yaml
access_log:
- name: envoy.access_loggers.stats
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.stats.v3.Config
    stat_prefix: request_stats
    counters:
    - stat:
        name: requests_total
        tags:
        - { name: status, value: "%RESPONSE_CODE%" }
        - { name: cluster, value: "%UPSTREAM_CLUSTER%" }
      value: { fixed_value: 1 }
    histograms:
    - stat: { name: request_duration_ms }
      unit: MILLISECONDS
      value: "%DURATION%"
    gauges:
    - stat: { name: active_requests }
      operations:
      - { access_log_type: DownstreamStart, operation_type: PAIRED_ADD }
      - { access_log_type: DownstreamEnd,   operation_type: PAIRED_SUBTRACT }
      value: { fixed_value: 1 }
```

## Stats / errors
- All user-visible metrics live under `<stat_prefix>.*` in the logger's evictable scope.
- Matcher/tag rules may produce a `dropped` signal (stats.cc:292); dropped tags skip that stat for the record - no separate counter is emitted for drops.
- Serialization/formatting errors from value formatters return `absl::nullopt` and skip that stat (stats.cc:297, 317, 349).
- Inflight gauge safety: if `AccessLogState` outlives a stream with pending ADDs, the destructor decrements them back to 0 (stats.cc:64).

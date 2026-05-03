# Hystrix sink

Emits Hystrix-compatible Server-Sent Events (SSE) to any number of
connected dashboards. Registers an admin endpoint
`/hystrix_event_stream` that speaks `text/event-stream`; on each server
stats flush, the sink walks the active cluster map, updates a per-cluster
rolling window, and pushes a pair of `HystrixCommand` + `HystrixThreadPool`
JSON frames to every connected dashboard. Registered as
`envoy.stat_sinks.hystrix`.

Proto: `envoy.config.metrics.v3.HystrixSink`.

## Files
- `config.h` / `config.cc` — `HystrixSinkFactory`, proto -> sink wiring.
- `hystrix.h` / `hystrix.cc` — `HystrixSink`, `ClusterStatsCache`, rolling
  window helpers, SSE admin handler, JSON formatters.
- `BUILD` — extension registration.

## Interface
- `Stats::Sink::flush(MetricSnapshot&)` — `hystrix.cc:343`. Iterates
  cluster manager clusters and pushes an SSE frame per cluster per
  connected dashboard.
- `Stats::Sink::onHistogramComplete()` — no-op (`hystrix.h:54`). Latency
  percentiles are read out of `snapshot.histograms()` inside `flush()`
  instead.
- `handlerHystrixEventStream()` — admin request handler
  (`hystrix.cc:295`). Sets `Content-Type: text/event-stream`,
  `Cache-Control: no-cache`, `Connection: close`, forces
  `Access-Control-Allow-Origin: *`, disables chunked encoding for HTTP/1,
  and registers the caller's
  `Http::StreamDecoderFilterCallbacks` into `callbacks_list_`.
- `registerConnection()` / `unregisterConnection()` — manage the active
  dashboard list (`hystrix.cc:430`, `:434`). When the last dashboard
  disconnects, `resetRollingWindow()` clears the per-cluster cache.

## Flow
1. The stats flush timer (configured by `Stats::StatsConfig::flushInterval`)
   invokes `flush()`.
2. If no dashboard is connected, `flush()` returns early
   (`hystrix.cc:344`); otherwise `incCounter()` advances the ring index.
3. For each histogram tagged `cluster.upstream_rq_time` the sink builds a
   `QuantileLatencyMap` keyed on the hystrix quantile list `{0, .25, .5,
   .75, .9, .95, .99, .995, 1}` (`hystrix.h:23`, `hystrix.cc:353-378`).
4. For each active cluster, `updateRollingWindowMap()`
   (`hystrix.cc:91`) pushes the latest counter reads into the rolling
   windows for success / errors / timeouts / rejected / total.
5. `addClusterStatsToStream()` (`hystrix.cc:250`) emits one
   `HystrixCommand` JSON blob (`addHystrixCommand`, `hystrix.cc:156`) and
   one `HystrixThreadPool` blob (`addHystrixThreadPool`, `hystrix.cc:224`).
   Each blob is written as `data: { ... }\n\n`.
6. The combined buffer is written to every registered callback
   (`hystrix.cc:402-406`), followed by a `":\n\n"` SSE keep-alive ping
   (`hystrix.cc:410-414`).
7. Clusters that have disappeared since the last flush are pruned from
   `cluster_stats_cache_map_` (`hystrix.cc:417-427`).

## Key decision points
- Rolling window sizing: `window_size_ = num_buckets + 1`
  (`hystrix.cc:276-277`); default `num_buckets = 10`
  (`DEFAULT_NUM_BUCKETS`, `hystrix.h:155`). `getRollingValue()`
  (`hystrix.cc:77`) returns `current - oldest`, treating counter resets as
  zero to avoid negative deltas.
- `error_rate = 100 * (errors + timeouts + rejected) / total`
  (`hystrix.cc:176`); `total` is synthesised from the components rather
  than read from `upstream_rq_total` to avoid race-driven >100% values
  (note at `hystrix.cc:121-123`).
- Error counting deducts `upstream_rq_timeout` to avoid double-counting
  504/408 as both errors and timeouts (`hystrix.cc:105-111`).
- `rejected` maps to `upstream_rq_pending_overflow`
  (`hystrix.cc:118`, `:191`) — presented as
  `rollingCountSemaphoreRejected` in the JSON payload.
- Circuit-breaker fields are stubbed: `isCircuitBreakerOpen=false`,
  `rollingCountShortCircuited=0`, `circuitBreakerForceClosed=true`
  (`hystrix.cc:169`, `:196`, `:208`). Envoy's circuit-breaking semantics
  don't map cleanly to Hystrix short-circuiting, so only the pending
  overflow signal is forwarded.
- Thread pool frame is entirely synthetic: only `queueSize`
  (from `pendingRequests().max()`, `hystrix.cc:395`) and
  `reportingHosts` (from `membership_total` gauge, `hystrix.cc:397`) are
  real; the rest are zeros (`hystrix.cc:230-245`).
- If `server.admin()` is not configured, the sink skips handler
  registration entirely (`hystrix.cc:286-288`) — it becomes a no-op
  because `flush()` has no callbacks to push to.
- `addHistogramToStream()` nests the quantile map as
  `"latencyExecute": { "0": v, "25": v, ... }` with quantiles scaled to
  the `0-100` range (`hystrix.cc:55-66`).

## Configuration

Proto fields on `envoy.config.metrics.v3.HystrixSink`:

- `num_buckets` (optional) — number of rolling-window buckets. `0` or
  missing falls back to `DEFAULT_NUM_BUCKETS = 10`
  (`hystrix.cc:276`).

The rolling window duration is `num_buckets * stats_flush_interval`; it
is reported to the dashboard as
`propertyValue_metricsRollingStatisticalWindowInMilliseconds` using
`server.statsConfig().flushInterval()` (`hystrix.cc:399`).

## Stats / errors
- No sink-local Envoy counters.
- The sink silently drops frames when there are no connected dashboards
  (`hystrix.cc:344`).
- Dashboard disconnects are handled via the admin stream's on-destroy
  callback, which calls `unregisterConnection()` (`hystrix.cc:321-330`).
- When the last dashboard leaves, `resetRollingWindow()`
  (`hystrix.cc:129`) clears the cluster cache so a fresh reader starts
  from a clean slate.

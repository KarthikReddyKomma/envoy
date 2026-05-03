# Common statsd writer

Shared statsd emitter library. Exposes two `Stats::Sink` implementations —
`UdpStatsdSink` and `TcpStatsdSink` — plus a pluggable `TagFormat` struct.
Both the vanilla `statsd` sink and the DogStatsD/Graphite flavored sinks build
on top of this. No factory is registered here; the concrete sinks live under
`source/extensions/stat_sinks/{statsd,dog_statsd,graphite_statsd}` and link
`:statsd_lib`.

Proto: none of its own. The three consumers use
`envoy.config.metrics.v3.StatsdSink`,
`envoy.extensions.stat_sinks.dog_statsd.v3.DogStatsdSink`, and
`envoy.extensions.stat_sinks.graphite_statsd.v3.GraphiteStatsdSink`.

## Files
- `statsd.h` / `statsd.cc` — `UdpStatsdSink` (lines 33-101 / 46-186) and
  `TcpStatsdSink` (lines 106-169 / 188-404).
- `tag_formats.h` / `tag_formats.cc` — `TagFormat` struct and the two
  built-in presets: `getDefaultTagFormat()` (DogStatsD `|#k:v,k:v`) and
  `getGraphiteTagFormat()` (`;k=v;k=v`).
- `BUILD` — exports `:statsd_lib`.

## Interface
- `Stats::Sink::flush(Stats::MetricSnapshot&)` — implemented for UDP at
  `statsd.cc:58` and for TCP at `statsd.cc:219`.
- `Stats::Sink::onHistogramComplete(const Histogram&, uint64_t)` — UDP at
  `statsd.cc:117`, TCP at `statsd.cc:245`. Both branch on
  `Histogram::Unit::Percent` (percents emitted as `|h`, everything else as
  `|ms` after `std::chrono::milliseconds` conversion).
- Internal `Writer` interface at `statsd.h:38-42` — a
  `ThreadLocal::ThreadLocalObject` with `write(string)` and
  `writeBuffer(Buffer::Instance&)`. `WriterImpl` at `statsd.h:72` owns the UDP
  `IoHandle` and calls `Network::Utility::writeToSocket` (`statsd.cc:39`, `:43`).

## Flow

UDP path:

1. The server's stats flush timer fires and calls `flush(snapshot)`
   (`statsd.cc:58`).
2. `buildMessage()` (`statsd.cc:138`) serialises each counter/gauge/host
   counter/host gauge as `prefix.name[:value|type][tags]`. Tag position is
   driven by `TagFormat::tag_position`.
3. `writeBuffer()` (`statsd.cc:90`) packs datagrams up to `buffer_size_`
   bytes, separating entries with `\n`. Oversized single metrics bypass the
   buffer (`statsd.cc:92-94`). At end of flush, `flushBuffer()`
   (`statsd.cc:109`) drains to the socket via the TLS `Writer`.
4. Histograms are emitted one-per-datagram directly from
   `onHistogramComplete` (`statsd.cc:135`).

TCP path:

1. `flush()` (`statsd.cc:219`) calls `TlsSink::beginFlush(true)` which
   reserves a 16 KiB slice (`FLUSH_SLICE_SIZE_BYTES`, `statsd.h:160`).
2. Each counter/gauge goes through `commonFlush()` (`statsd.cc:280`), which
   writes `prefix.name:value|type\n` using `memcpy` + `StringUtil::itoa` for
   hot-path perf.
3. When the reserved slice is about to overflow, the code does
   `endFlush(false)` + `beginFlush(false)` to get a fresh slice
   (`statsd.cc:286-288`).
4. `endFlush(true)` commits and calls `write(buffer_)` (`statsd.cc:354`),
   which lazily obtains a `tcpConn` from the configured upstream cluster and
   then writes the buffer.

## Key decision points
- UDP buffering is opt-in via `buffer_size_` (`statsd.h:99`); a value of 0
  means "emit one datagram per metric". See `statsd.cc:92-107`.
- `use_tag_` toggles between `tagExtractedName()` and `name()` (`statsd.cc:167`).
  When tags are off, the tag string is empty (`statsd.cc:176`).
- TCP overflow guard: if
  `cluster_traffic_stats.upstream_cx_tx_bytes_buffered_` exceeds
  `MAX_BUFFERED_STATS_BYTES` (16 MiB, `statsd.h:157`) the connection is
  force-closed and the buffer dropped (`statsd.cc:366-373`); `cx_overflow_stat_`
  is incremented.
- Connection is lazily created on first write and torn down on
  `LocalClose`/`RemoteClose` via `deferredDelete` (`statsd.cc:330-335`).
- `TcpStatsdSink::create()` (`statsd.cc:208`) returns `absl::StatusOr` — it
  validates `local_info` (node/cluster set) and the configured upstream
  cluster before the object is exposed.
- Histograms on TCP never interleave with counter/gauge flushes; the
  `current_slice_mem_ == nullptr` asserts at `statsd.cc:342` / `:349`
  guarantee that.
- Text readouts are intentionally unsupported today (see `TODO(efimki)` at
  `statsd.cc:87` and `:241`).

## Configuration

This library does not read a proto directly. Callers pass:

- `prefix` (default `"envoy"`, `statsd.h:28`).
- `use_tag` — boolean, decides whether to emit tags and whether to use the
  tag-extracted stat name.
- `buffer_size` — optional UDP datagram batching threshold.
- `tag_format` — `TagFormat` struct controlling start/assign/separator tokens
  and tag position. Two presets in `tag_formats.cc` (default DogStatsD-style,
  plus Graphite-style).
- TCP only: `cluster_name` resolved through `ClusterManager` + a
  `Stats::Scope` for the overflow counter.

## Stats / errors
- `statsd.cx_overflow` — counter, incremented when the TCP upstream TX
  buffered bytes cross 16 MiB; constructed at `statsd.cc:194`, incremented at
  `statsd.cc:370`.
- UDP write errors are silently swallowed by `Network::Utility::writeToSocket`
  — there is no per-failure stat.
- `TcpStatsdSink::create()` surfaces cluster/local-info misconfiguration as
  `absl::Status`.

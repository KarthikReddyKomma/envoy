# Statsd sink

Vanilla (Etsy-style) statsd sink. Wraps the shared writer library under
`common/statsd/` and picks between UDP or TCP transport based on which
field of the proto `oneof` is set. Tags are not emitted (`use_tag = false`
on UDP; TCP has no tag support by design). Registered as
`envoy.stat_sinks.statsd`.

Proto: `envoy.config.metrics.v3.StatsdSink`.

## Files
- `config.h` — `StatsdSinkFactory` declaration, `StatsdName` constant
  (`config.h:13`).
- `config.cc` — factory implementation, oneof dispatch to
  `UdpStatsdSink` or `TcpStatsdSink::create()`.
- `BUILD` — extension registration; depends on
  `//source/extensions/stat_sinks/common/statsd:statsd_lib`.

## Interface

All `Stats::Sink` overrides come from the shared writer:

- `UdpStatsdSink::flush(MetricSnapshot&)` —
  `source/extensions/stat_sinks/common/statsd/statsd.cc:58`.
- `TcpStatsdSink::flush(MetricSnapshot&)` — `statsd.cc:219`.
- `UdpStatsdSink::onHistogramComplete()` — `statsd.cc:117`.
- `TcpStatsdSink::onHistogramComplete()` — `statsd.cc:245`.

Factory:

- `StatsdSinkFactory::createStatsSink()` (`config.cc:17`) returns
  `absl::StatusOr<Stats::SinkPtr>`.
- `createEmptyConfigProto()` returns `StatsdSink` proto
  (`config.cc:44`).
- `name()` returns `envoy.stat_sinks.statsd`.

## Flow
1. Factory validates proto (`config.cc:21-23`).
2. Switch on `statsd_specifier_case` (`config.cc:24`):
   - `kAddress` → `Network::Address::resolveProtoAddress()`
     (`config.cc:26`), then construct `UdpStatsdSink` with
     `use_tag = false` and the default tag format (`config.cc:30`). UDP
     path has no batching buffer size configured here.
   - `kTcpClusterName` → call `TcpStatsdSink::create()`
     (`config.cc:35`). This validates `local_info` and the upstream
     cluster, returns `absl::StatusOr`.
   - `STATSD_SPECIFIER_NOT_SET` → falls through to `PANIC` at
     `config.cc:41`, aborting startup.
3. The returned sink is installed on the stats flush timer by the
   server; flushes emit `prefix.name:value|c|g|ms|h` datagrams/lines via
   the shared writer.

## Key decision points
- `use_tag = false` is hard-coded for UDP (`config.cc:31`) because
  vanilla statsd has no inline tag syntax. For tagged statsd, use the
  `dog_statsd` or `graphite_statsd` sinks.
- The UDP path does not plumb `max_bytes_per_datagram` — each metric is
  written as its own datagram. (This proto predates the DogStatsD
  proto's batching field.)
- `TcpStatsdSink::create()` returns a `StatusOr` so that a missing
  `local_info.node` or a misconfigured `tcp_cluster_name` fails sink
  creation cleanly.
- `LEGACY_REGISTER_FACTORY` registers the deprecated alias
  `envoy.statsd` (`config.cc:53`) alongside `envoy.stat_sinks.statsd`.

## Configuration

`envoy.config.metrics.v3.StatsdSink` fields:

- `statsd_specifier` (oneof, required):
  - `address` — UDP endpoint of the statsd server.
  - `tcp_cluster_name` — name of an Envoy cluster whose upstream hosts
    speak line-oriented TCP statsd.
- `prefix` (optional) — metric name prefix. Empty falls back to
  `"envoy"` (`common/statsd/statsd.cc:51`).

## Stats / errors
- TCP path exposes `statsd.cx_overflow` via the shared writer
  (`common/statsd/statsd.cc:370`). UDP path has no sink-local counters.
- Cluster/local-info validation errors from `TcpStatsdSink::create()`
  surface as `absl::Status` and abort server startup.
- UDP write errors are swallowed by `Network::Utility::writeToSocket`.

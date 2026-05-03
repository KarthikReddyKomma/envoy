# DogStatsD sink

DogStatsD-flavoured UDP statsd sink. A thin factory on top of
`Common::Statsd::UdpStatsdSink` that forces `use_tag = true` so tags are
emitted inline using the default `|#k:v,k:v` tag format. Registered under
the name `envoy.stat_sinks.dog_statsd` (`config.h:13`).

Proto: `envoy.config.metrics.v3.DogStatsdSink`. (The DogStatsD sink
reuses the long-standing `envoy.config.metrics.v3` message rather than
living under `envoy.extensions.stat_sinks.dog_statsd.v3`.)

## Files
- `config.h` — declares `DogStatsdSinkFactory` and the extension name
  constant.
- `config.cc` — parses the proto, resolves the UDP address, constructs
  the shared `Common::Statsd::UdpStatsdSink`, and registers the factory.
- `BUILD` — registers the extension, links
  `//source/extensions/stat_sinks/common/statsd:statsd_lib`.

## Interface

This directory does not override `Stats::Sink` itself; the
`Sink::flush(MetricSnapshot&)` and `Sink::onHistogramComplete()` overrides
are inherited verbatim from `Common::Statsd::UdpStatsdSink` (see
`source/extensions/stat_sinks/common/statsd/statsd.cc:58` and `:117`).

Exposed factory interface:

- `DogStatsdSinkFactory::createStatsSink(config, server)` (`config.cc:20`)
  returns `absl::StatusOr<Stats::SinkPtr>`.
- `createEmptyConfigProto()` (`config.cc:37`) returns an empty
  `DogStatsdSink` proto.
- `name()` returns `envoy.stat_sinks.dog_statsd`.

## Flow
- Server config parsing calls the factory.
- `Network::Address::resolveProtoAddress(sink_config.address())`
  (`config.cc:25`) resolves the DogStatsD agent UDP endpoint; errors are
  propagated via `RETURN_IF_NOT_OK_REF` (`config.cc:26`).
- Optional `max_bytes_per_datagram` is unpacked (`config.cc:30-32`) and
  passed as the UDP write buffer size.
- A `UdpStatsdSink` is constructed with `use_tag = true` (`config.cc:34`),
  the configured prefix, and the default `TagFormat` (DogStatsD-style from
  `common/statsd/tag_formats.cc:11`).
- At runtime the stats flush timer drives
  `UdpStatsdSink::flush()` which batches metric lines up to
  `max_bytes_per_datagram` and emits them to the socket.

## Key decision points
- `use_tag = true` is hard-coded at `config.cc:34` — DogStatsD without
  tags wouldn't be DogStatsD.
- No explicit `tag_format` is passed, so the sink falls back to the
  default (`|#k:v,k:v`) defined at `common/statsd/tag_formats.cc:11`.
- UDP only — there is no TCP variant for DogStatsD.
- Uses `LEGACY_REGISTER_FACTORY` with the deprecated alias
  `envoy.dog_statsd` at `config.cc:46` so older configs still resolve.

## Configuration

Fields on `envoy.config.metrics.v3.DogStatsdSink`:

- `address` (required) — UDP endpoint of the DogStatsD agent.
- `prefix` (optional) — metric name prefix; defaults to `"envoy"` via
  `Common::Statsd::getDefaultPrefix()` when empty
  (`common/statsd/statsd.cc:51`).
- `max_bytes_per_datagram` (optional `UInt64Value`) — max datagram size
  for UDP packing. Omit or zero means one metric per datagram.

## Stats / errors
- No sink-local stats. The shared UDP writer does not increment
  per-error counters; socket-level write failures surface only through
  the generic listener/IO paths.
- Address resolution failures surface as `absl::Status` from
  `createStatsSink`, aborting server startup.

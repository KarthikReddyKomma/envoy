# Graphite statsd sink

Graphite-tagged UDP statsd sink. Like DogStatsD it wraps the shared
`Common::Statsd::UdpStatsdSink`, but swaps in the Graphite `TagFormat`
so tags are emitted inline as `metric;k=v;k=v:value|type` rather than the
DogStatsD `|#k:v` trailing block. Registered as
`envoy.stat_sinks.graphite_statsd`.

Proto: `envoy.extensions.stat_sinks.graphite_statsd.v3.GraphiteStatsdSink`.

## Files
- `config.h` — factory declaration.
- `config.cc` — parses proto, resolves address, constructs the shared
  `UdpStatsdSink` with `getGraphiteTagFormat()`, registers the factory.
- `BUILD` — extension registration, depends on
  `//source/extensions/stat_sinks/common/statsd:statsd_lib`.

## Interface

`Stats::Sink::flush(MetricSnapshot&)` and `Stats::Sink::onHistogramComplete()`
are inherited from `Common::Statsd::UdpStatsdSink`
(`source/extensions/stat_sinks/common/statsd/statsd.cc:58`, `:117`).

Factory methods:

- `GraphiteStatsdSinkFactory::createStatsSink()` (`config.cc:19`).
- `createEmptyConfigProto()` returns the v3 `GraphiteStatsdSink` proto
  (`config.cc:47`).
- `name()` returns `envoy.stat_sinks.graphite_statsd` (`config.cc:51`).

## Flow
- Factory validates the proto and switches on `statsd_specifier_case`
  (`config.cc:25`). Only `kAddress` is currently supported; anything
  else falls through to `PANIC("unexpected statsd specifier enum")`
  (`config.cc:44`).
- The UDP address is resolved via
  `Network::Address::resolveProtoAddress` (`config.cc:28`).
- Optional `max_bytes_per_datagram` is forwarded as the UDP batch size
  (`config.cc:33-35`).
- A `UdpStatsdSink` is built with `use_tag = true` and
  `Common::Statsd::getGraphiteTagFormat()` (`config.cc:36-38`).
- At flush time the base class serialises each metric using the Graphite
  tag format (`start=";"`, `assign="="`, `separator=";"`,
  `TagPosition::TagAfterName`; see
  `common/statsd/tag_formats.cc:20`). Tags are appended directly after the
  metric name, before the `:value|type` suffix
  (`common/statsd/statsd.cc:153-162`).

## Key decision points
- `statsd_specifier` is a oneof, but only the `address` arm is wired up
  (`config.cc:27`). The `STATSD_SPECIFIER_NOT_SET` arm breaks into
  `PANIC` at `config.cc:44` — startup will abort if the proto is missing
  the address.
- `use_tag = true` is forced (`config.cc:37`); there is no way to disable
  tag emission for Graphite because Graphite names would otherwise carry
  the full pre-extraction stat name.
- The switch to `TagPosition::TagAfterName` is what makes this
  Graphite-compatible; see the branch at
  `common/statsd/statsd.cc:153`.
- `LEGACY_REGISTER_FACTORY` also registers the short name
  `envoy.graphite_statsd` (`config.cc:56`) for config back-compat.

## Configuration

`GraphiteStatsdSink` proto fields:

- `statsd_specifier.address` — UDP endpoint (required).
- `prefix` — optional metric name prefix; empty string falls back to
  `"envoy"` (`common/statsd/statsd.cc:51`).
- `max_bytes_per_datagram` — optional `UInt64Value` batching limit.

## Stats / errors
- No sink-local counters.
- `absl::Status` surfaces protobuf validation and address resolution
  failures during server bootstrap.
- UDP write failures are not individually reported.

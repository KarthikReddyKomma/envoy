# Common (stat sinks aggregator)

This directory is an aggregator folder. It does not define a stats sink of its
own; there is no `Stats::Sink` implementation and no `config.{h,cc}` registered
factory at this level. It exists purely to group library code that is shared
between multiple concrete stat sink extensions.

Today its only child is `statsd/`, which contains the UDP + TCP statsd writer
library (`UdpStatsdSink`, `TcpStatsdSink`) used by the `statsd`,
`dog_statsd`, and `graphite_statsd` sinks. Those three sinks each have their
own factory in a sibling directory but link against
`//source/extensions/stat_sinks/common/statsd:statsd_lib` to get the actual
flush/serialisation implementation.

## Files

This directory intentionally has no source files of its own; it only holds
subdirectories. See the child directory for the actual implementation.

- `statsd/` — shared UDP/TCP statsd serialisation and socket writer. See
  `statsd/README.md`.

## Interface

No `Stats::Sink` is declared here. The `Stats::Sink::flush(MetricSnapshot&)` /
`Stats::Sink::onHistogramComplete(Histogram&, uint64_t)` overrides that the
statsd-family sinks expose are declared one level down in
`common/statsd/statsd.h`.

## Flow

No flow happens in this directory. The runtime data path is:

1. A concrete factory in `dog_statsd/`, `graphite_statsd/`, or `statsd/`
   parses its own proto config.
2. That factory constructs a `Common::Statsd::UdpStatsdSink` or
   `Common::Statsd::TcpStatsdSink` from `common/statsd/statsd.cc`.
3. The `StatsSinkFactory::createStatsSink()` returns that instance to the
   server, which then calls `flush()` on the configured stats flush timer.

## Key decision points

- The decision to keep only `statsd/` here (rather than pulling in
  dogstatsd or graphite specifics) is enforced by the narrow type surface of
  `common/statsd/statsd.h` — it exposes only `UdpStatsdSink`/`TcpStatsdSink`
  plus the `TagFormat` hook (`common/statsd/tag_formats.h:13`) so callers can
  plug in DogStatsD- or Graphite-style tagging without forking the sink.
- There is no BUILD file at this level; callers depend directly on
  `//source/extensions/stat_sinks/common/statsd:statsd_lib`.

## Configuration

None. This directory does not register an extension and has no proto.
Configuration lives with each concrete sink:

- `envoy.extensions.stat_sinks.statsd.v3.StatsdSink`
- `envoy.extensions.stat_sinks.dog_statsd.v3.DogStatsdSink`
- `envoy.extensions.stat_sinks.graphite_statsd.v3.GraphiteStatsdSink`

## Stats / errors

None emitted from this directory. The shared TCP sink emits the
`statsd.cx_overflow` counter, but that is defined and incremented in
`common/statsd/statsd.cc:194` / `:370` and documented in
`common/statsd/README.md`.

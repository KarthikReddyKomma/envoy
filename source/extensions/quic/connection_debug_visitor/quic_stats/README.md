# QUIC Stats Debug Visitor

QUIC connection debug visitor that periodically samples
`quic::QuicConnectionStats` and exports them as Envoy counters/histograms.
Useful for observing retransmissions, path degradation, MTU, and estimated
bandwidth.

Proto: `envoy.extensions.quic.connection_debug_visitor.quic_stats.v3.Config`.

## Files
- `quic_stats.h/cc` - `QuicStatsVisitor` (per-connection sampler),
  `QuicStatsVisitorProd` (wires `session_` to QUICHE's `GetStats`),
  `Config` (listener-scope owner of the stat struct and update period),
  and `QuicStatsFactoryFactory` (registered factory).

## Interface
- Visitor base: `quic::QuicConnectionDebugVisitor`.
- Factory base:
  `Envoy::Quic::EnvoyQuicConnectionDebugVisitorFactoryFactoryInterface`.
- Extension name: `envoy.quic.connection_debug_visitor.quic_stats`.

## Logic
- `Config` reads `update_period` (optional, ms) and builds the `QuicStats`
  struct from `ALL_QUIC_STATS` under the `quic_stats` stat prefix on the
  listener scope.
- `QuicStatsVisitor` installs a dispatcher timer when `update_period_`
  is set; on tick it calls `recordStats()` and re-arms.
- `recordStats` uses `update_counter` lambdas to emit only the diff
  since the last sample (QUICHE's `QuicConnectionStats` are monotonic).
  Histograms are recorded for RTT (`cx_rtt_us`), estimated bandwidth,
  retransmission percentage, and egress/ingress MTU.
- `OnConnectionClosed` disables the timer and flushes a final
  `recordStats()` so the last window is captured even if the close
  happens between ticks.

## Key decision points
- `quic_stats.cc:61` - percentage-retransmitted is computed from the diff
  pair `(packets_sent, packets_retransmitted)` before the counters are
  updated, so the histogram reflects the interval rather than lifetime
  totals.
- `quic_stats.cc:69` - static_assert guards against overflow in the
  `Histogram::PercentScale` multiplication up to 10 trillion packets; the
  comment explicitly flags the unhandled worst case.
- `update_counter` asserts `diff >= 0` since QUICHE counters are
  monotonic; a regression would indicate a session bug.

## Configuration
- `update_period` (`google.protobuf.Duration`, optional) - if unset, stats
  are recorded only at connection close.

## Stats / errors
Counters under `quic_stats.` prefix:
`cx_tx_packets_total`, `cx_tx_packets_retransmitted_total`,
`cx_tx_amplification_throttling_total`, `cx_rx_packets_total`,
`cx_path_degrading_total`, `cx_forward_progress_after_path_degrading_total`.
Histograms: `cx_rtt_us` (microseconds), `cx_tx_estimated_bandwidth`
(bytes-per-second), `cx_tx_percent_retransmitted_packets` (percent),
`cx_tx_mtu` / `cx_rx_mtu` (bytes).

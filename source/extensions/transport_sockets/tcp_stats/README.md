# TCP Stats Transport Socket (`envoy.transport_sockets.tcp_stats`)

Wrapper transport that periodically queries the Linux kernel's `TCP_INFO` socket option on the underlying connection and emits per-connection TCP-level stats (retransmissions, RTT, bytes sent/received, unacked segments, etc.). Linux-only — building on other platforms registers the factory but `createTransportSocketFactory` throws.

Proto: `envoy.extensions.transport_sockets.tcp_stats.v3.Config` (contains `update_period` plus the wrapped `transport_socket`).

## Files
- `config.h/cc` — registers upstream and downstream config factories. Common base class `TcpStatsSocketFactory` owns the shared `Config` (stats struct + update period). `UpstreamTcpStatsSocketFactory` and `DownstreamTcpStatsSocketFactory` inherit from the appropriate passthrough factory and from `TcpStatsSocketFactory`. Non-Linux builds throw `EnvoyException` at factory creation (`config.cc:23`).
- `tcp_stats.h/cc` — defines `Config`, `TcpStats` (the counter/gauge/histogram struct), and `TcpStatsSocket` (the `PassthroughSocket` subclass that runs the periodic timer and maintains last-value deltas). The whole file is guarded by `#if defined(__linux__)`.

## Transport socket role
`TcpStatsSocket` extends `PassthroughSocket`. Overrides:
- `setTransportSocketCallbacks` (`tcp_stats.cc:41`) — stores callbacks to access `ioHandle()` and the dispatcher.
- `onConnected` (`tcp_stats.cc:46`) — if `update_period_` is set, arms a timer that repeatedly calls `recordStats()` and re-enables itself; then forwards to the inner socket.
- `closeSocket` (`tcp_stats.cc:58`) — records final stats, zeroes the two accumulated gauges, cancels the timer, then forwards.

## Lifecycle
- Connect path: inner socket's handshake (if any) runs first; timer arms on `onConnected`.
- Data path: pure passthrough; stats are recorded only on timer ticks.
- Close path: one final `recordStats()` to capture anything since the last tick, then gauges `cx_tx_unsent_bytes` / `cx_tx_unacked_segments` are decremented back to zero.

## Key decision points
- `tcp_stats.cc:77-89` — `querySocketInfo` calls `getsockopt(IPPROTO_TCP, TCP_INFO, ...)` on the `IoHandle`. A non-zero return is logged at `debug` and no stats are recorded that tick.
- `tcp_stats.cc:1-8` — a comment explains the `DO_NOT_INCLUDE_NETINET_TCP_H` hack to avoid conflicting `struct tcp_info` definitions between `netinet/tcp.h` and `linux/tcp.h`.
- `tcp_stats.cc:97-111` — `update_counter` asserts monotonic growth (`diff >= 0`) and adds the delta; `update_gauge` allows any delta since OS gauges can decrease.
- `tcp_stats.cc:119-134` — retransmission-percentage histogram is computed *before* `update_counter` calls for `data_segments` and `retransmitted_segments` so it sees the previous `last_*` values. Uses integer math scaled by `Stats::Histogram::PercentScale` to avoid floating point.
- `config.cc:18-25` — the `Config` shared state is built once per factory; `#if defined(__linux__)` otherwise the constructor throws.

## Configuration
- `update_period` — optional `Duration`. When unset, stats are recorded *only* on close. When set, a timer fires every period and records incremental updates.
- `transport_socket` — required inner transport (typically `raw_buffer` or `tls`).

## Stats
Prefixed `tcp_stats.`:
- Counters: `cx_tx_segments`, `cx_rx_segments`, `cx_tx_data_segments`, `cx_rx_data_segments`, `cx_tx_retransmitted_segments`, `cx_rx_bytes_received`, `cx_tx_bytes_sent`.
- Gauges (accumulated across all active connections): `cx_tx_unsent_bytes`, `cx_tx_unacked_segments`.
- Histograms: `cx_tx_percent_retransmitted_segments` (Percent), `cx_rtt_us` (Microseconds), `cx_rtt_variance_us` (Microseconds).

## Errors
- Non-Linux platforms: `EnvoyException("envoy.transport_sockets.tcp_stats is not supported on this platform.")` at factory-creation time.
- `getsockopt` failures are logged at `debug` and simply skip that tick.

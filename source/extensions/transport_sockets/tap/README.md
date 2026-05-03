# Tap Transport Socket (`envoy.transport_sockets.tap`)

Wrapper transport that mirrors all reads and writes through the Envoy tap subsystem for out-of-band capture (pcap-like tracing). Available as both downstream and upstream variants, each wrapping another transport (typically TLS or raw_buffer). Captures are emitted as `envoy.data.tap.v3.TraceWrapper` records (buffered or streamed).

Proto: `envoy.extensions.transport_sockets.tap.v3.Tap` (contains a `common_config`, inner `transport_socket`, and a `socket_tap_config`).

## Files
- `config.h/cc` — registers `UpstreamTapSocketConfigFactory` and `DownstreamTapSocketConfigFactory`. Both wrap the inner transport and bind a `SocketTapConfigFactoryImpl` that produces `SocketTapConfigImpl` from the `envoy.config.tap.v3.TapConfig`.
- `tap.h/cc` — defines `TapSocket` (the `PassthroughSocket` subclass that calls into `PerSocketTapper`), `TapSocketFactory` (upstream), and `DownstreamTapSocketFactory`. Factories inherit from `Common::Tap::ExtensionConfigBase` so they get dynamic tap config updates via the admin interface.
- `tap_config.h` — abstract interfaces: `PerSocketTapper` (per-connection tapper callbacks: `closeSocket`, `onRead`, `onWrite`) and `SocketTapConfig` (creates tappers, exposes a `TimeSource`).
- `tap_config_impl.h/cc` — `PerSocketTapperImpl` (builds streamed or buffered traces, applies match statuses, handles aging/threshold-based submission) and `SocketTapConfigImpl` (inherits `Common::Tap::TapConfigBaseImpl`).

## Transport socket role
`TapSocket` extends `PassthroughSocket`. Overrides four methods:
- `setTransportSocketCallbacks` (`tap.cc:31`) — creates the `PerSocketTapper` tied to the `Network::Connection`.
- `doRead` (`tap.cc:47`) — forwards to the inner socket and, if bytes were read, calls `tapper_->onRead(buffer, bytes_processed)`.
- `doWrite` (`tap.cc:56`) — copies the pending buffer (TODO: avoid copy), forwards, then calls `tapper_->onWrite(copy, bytes_processed, end_stream)`.
- `closeSocket` (`tap.cc:39`) — informs the tapper before closing.

Factories extend both `ExtensionConfigBase` (for admin-driven config updates) and `PassthroughFactory` / `DownstreamPassthroughFactory`. `currentConfigHelper<SocketTapConfig>()` fetches the latest `SocketTapConfig` from the thread-local slot on every `createTransportSocket` call (`tap.cc:80`, `tap.cc:96`).

## Lifecycle
- Connect path: passthrough. `PerSocketTapper` is created eagerly in `setTransportSocketCallbacks`.
- Data path: each successful read or write is mirrored into `PerSocketTapperImpl`. The tapper evaluates match predicates (from `TapConfigBaseImpl`) and decides whether to buffer, stream, or skip the event.
- Close path: a `closeSocket` event is delivered to the tapper first so final buffered segments can be emitted, then the inner socket closes.

## Key decision points
- `tap.cc:20-29` — `generateStats` builds the `transport.tap.<prefix>.` counter pair (`streamed_submit`, `buffered_submit`).
- `tap.cc:33-37` — the tapper is only created if the `SocketTapConfig` is non-null (i.e. a tap config has been pushed via admin or static config).
- `tap_config_impl.h:31-47` — streamed traces use `SocketStreamedTraceSegment` with `trace_id` set to the connection id; buffered traces use `SocketBufferedTrace`. Sequence numbers are attached so the receiver can reorder.
- `tap_config_impl.h:59-63` — default thresholds: `DefaultMinBufferedBytes = 9` (minimum before flushing a streamed segment) and `DefaultBufferedAgedDuration = 15` s (flush even if below size threshold).

## Configuration
- `common_config` — `envoy.config.tap.v3.CommonExtensionConfig`: either static `tap_config`, an admin handle, or a TAP_DS config source.
- `socket_tap_config` — transport-socket-specific knobs:
  - `stats_prefix` — extra prefix for the `transport.tap.` counters.
  - Streaming / buffering thresholds controlled by `tap_config_impl.cc` logic.
- `transport_socket` — required inner transport that actually carries the bytes.

## Stats
Prefixed `transport.tap.[<stats_prefix>.]`:
- `streamed_submit` (counter) — number of streamed trace segments submitted.
- `buffered_submit` (counter) — number of buffered traces submitted.

## Errors
No explicit error handling — if the tap sink is unavailable, events are silently dropped (see `Common::Tap` sink implementations). Invalid configuration is rejected at registration time by `ExtensionConfigBase`.

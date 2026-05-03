# Echo Network Filter (`envoy.echo`)

Minimal terminal L4 filter that reflects every received byte back to the peer. Primarily used as a demo, in integration tests, and as a reference example for authoring a `Network::ReadFilter`. Statically holds no per-connection state beyond the read-filter callbacks pointer.

Proto: `envoy.extensions.filters.network.echo.v3.Echo` (empty — no fields).

## Files
- `config.cc` — `EchoConfigFactory` (config.cc:18), a `Common::FactoryBase<Echo>` that registers the filter under the legacy name `envoy.echo` and marks it terminal.
- `echo.h` / `echo.cc` — `EchoFilter`, the `Network::ReadFilter` that writes `onData`'s buffer back on the same connection.

## Factory
`EchoConfigFactory::createFilterFactoryFromProtoTyped` (config.cc:24) ignores the (empty) proto and returns a lambda that adds a new `EchoFilter` (via `addReadFilter`) to each connection (config.cc:27-29). `isTerminalFilterByProtoTyped` returns `true` (config.cc:32-35) — nothing can follow echo in the filter chain. Registration uses `LEGACY_REGISTER_FACTORY(EchoConfigFactory, NamedNetworkFilterConfigFactory, "envoy.echo")` (config.cc:41), so the legacy wire name is `envoy.echo` (newer proto name resolves through the same factory).

## Filter lifecycle
`EchoFilter` (echo.h:15) is only a `Network::ReadFilter`; no write filter, no connection callbacks, no shared config.

- `initializeReadFilterCallbacks(callbacks)` (echo.h:20): stores the `ReadFilterCallbacks*` pointer; does not register for connection events.
- `onNewConnection()` (echo.h:19): inline `return Network::FilterStatus::Continue` — no initialization work.
- `onData(data, end_stream)` (echo.cc:13):
  1. Logs `"echo: got N bytes"` at trace (echo.cc:14).
  2. Calls `read_callbacks_->connection().write(data, end_stream)` — reflecting both the bytes and the stream-closed bit directly to the connection's write path (echo.cc:15). `write` drains the incoming buffer.
  3. Asserts the buffer was fully consumed (`ASSERT(0 == data.length())`, echo.cc:16).
  4. Returns `StopIteration` (echo.cc:17) so no later read filter sees the bytes. Because the factory marks echo terminal, no later filter should exist anyway.

## Decision points
- No branches. The filter copies every byte regardless of content or size. `end_stream` is forwarded so a half-close from the peer triggers the corresponding close on the write side.

## Configuration
The `Echo` proto has no fields; there is nothing to configure. Any parse of `{}` yields a valid filter instance.

## Stats
The filter emits no counters or gauges. Byte counts surface only through the owning listener's downstream connection stats and `tcp_proxy`-style accounting elsewhere in the listener.

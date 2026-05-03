# Connection Limit Network Filter (`envoy.filters.network.connection_limit`)

Per-filter-chain cap on the number of concurrent L4 connections. Counts are shared across all workers through a single `Config` object. New connections beyond `max_connections` are rejected, optionally after a configurable delay to slow down DoS attempts.

Proto: `envoy.extensions.filters.network.connection_limit.v3.ConnectionLimit`.

## Files
- `config.h` / `config.cc` — `ConnectionLimitConfigFactory`, a `Common::FactoryBase` subclass that instantiates one `Config` per filter-chain declaration and adds a `Filter` read-filter to each connection (config.cc:13-21).
- `connection_limit.h` / `connection_limit.cc` — the filter config struct and the per-connection `Filter` class.

## Shared `Config` state
- `Config` (connection_limit.h:38) holds:
  - `enabled_` — `Runtime::FeatureFlag` populated from `proto.runtime_enabled()` (connection_limit.cc:15).
  - `max_connections_` — `UInt32Value` via `PROTOBUF_GET_WRAPPED_REQUIRED` (connection_limit.cc:17).
  - `connections_` — `std::atomic<uint64_t>` active counter shared by all worker threads (connection_limit.h:56).
  - `delay_` — optional post-rejection lingering interval (connection_limit.cc:18).
  - `stats_` — generated via `generateStats` under `connection_limit.<stat_prefix>.` (connection_limit.cc:20).
- `incrementConnectionWithinLimit()` (connection_limit.cc:26) uses a CAS loop: loads `connections_` and tries `compare_exchange_weak` up to `max_connections_`. Returns `false` if the cap is already reached. `ThreadSynchronizer::syncPoint("increment_pre_cas")` is a test-only hook (connection_limit.cc:30).
- `incrementConnection()` / `decrementConnection()` (connection_limit.cc:40) are unconditional; used when we want the rejected connection still tracked until `onEvent` releases it.

## Filter lifecycle
`Filter` is a `Network::ReadFilter` + `Network::ConnectionCallbacks` (connection_limit.h:68).

- `initializeReadFilterCallbacks` (connection_limit.h:78): stashes callbacks, registers itself for connection events so `onEvent` fires on close.
- `onNewConnection()` (connection_limit.cc:61):
  1. If `config_->enabled()` is false (runtime kill-switch), returns `Continue` without counting (connection_limit.cc:62-65).
  2. Bumps `active_connections` gauge.
  3. Calls `incrementConnectionWithinLimit()`. On success, `Continue` — the connection proceeds down the chain.
  4. On failure: increments `limited_connections` counter, sets `is_rejected_=true`, and unconditionally bumps `connections_` via `incrementConnection()` so the later `onEvent(Close)` path decrements without underflow (connection_limit.cc:77, comment on :76).
  5. If `delay_` is set and positive, creates a dispatcher `Timer` that closes the connection with `NoFlush` and the reason string `over_connection_limit` after the delay (connection_limit.cc:82-86). Otherwise closes immediately (connection_limit.cc:88).
  6. Returns `StopIteration` so no downstream filters see the connection.
- `onData(data, end_stream)` (connection_limit.cc:54): if `is_rejected_` returns `StopIteration` to hold any data arriving during the delay window; otherwise `Continue`.
- `onEvent(event)` (connection_limit.cc:97): on `RemoteClose` or `LocalClose`, cancels any pending delay timer via `resetTimerState()`, decrements the shared counter, and decrements the `active_connections` gauge. Other events (e.g., `Connected`) are ignored.
- `resetTimerState()` (connection_limit.cc:47): disables and destroys `delay_timer_` if set.

## Decision points
- Runtime disable: connection_limit.cc:62 (returns `Continue`, never counted).
- Over-limit branch: connection_limit.cc:69 (CAS loop failed).
- Delayed-close vs immediate close: connection_limit.cc:81.
- StopIteration during delay: connection_limit.cc:55.

## Configuration fields
- `stat_prefix` — appended to the `connection_limit.` root, used by `generateStats` (connection_limit.cc:21).
- `max_connections` (UInt32Value, required) — shared cap.
- `delay` (Duration, optional) — time to hold a rejected connection open before closing; zero/unset closes immediately.
- `runtime_enabled` (RuntimeFeatureFlag, optional) — runtime kill switch.

## Stats
Emitted under `connection_limit.<stat_prefix>.*` (see `ALL_CONNECTION_LIMIT_STATS` in connection_limit.h:24):
- `limited_connections` (counter) — incremented on each rejected connection (connection_limit.cc:70).
- `active_connections` (gauge, Accumulate) — incremented on each accepted or delayed-rejected connection (connection_limit.cc:67), decremented on close (connection_limit.cc:102).

## Factory
`ConnectionLimitConfigFactory` (config.h:17) derives from `Common::FactoryBase<ConnectionLimit>` with name `NetworkFilterNames::get().ConnectionLimit` (config.h:21). Not terminal. Registered via `REGISTER_FACTORY(ConnectionLimitConfigFactory, NamedNetworkFilterConfigFactory)` (config.cc:26). Each `createFilterFactoryFromProtoTyped` call builds one `Config` (shared by all connections that traverse this listener's filter chain) and returns a `FilterFactoryCb` that pushes a fresh `Filter` onto the chain per connection.

# Rate Limit (`envoy.filters.network.ratelimit`)

A TCP-level (L4) rate limit filter. On every new connection the filter makes a single call to an external gRPC rate limit service using a fixed `domain` and a list of `descriptors` (optionally expanded via substitution formatters). While the call is outstanding, read-side iteration is paused so no data is forwarded to downstream filters. On `OK` the connection continues; on `OverLimit` (subject to the `ratelimit.tcp_filter_enforcing` runtime gate) the connection is closed; on `Error` behavior is governed by `failure_mode_deny`.

Proto: `envoy.extensions.filters.network.ratelimit.v3.RateLimit`.

## Files
- `config.h` / `config.cc` — `RateLimitConfigFactory`, factory that builds the shared `Config` plus a `RateLimit::Client` and installs a `Filter` as a read filter (`config.cc:36-41`).
- `ratelimit.h` / `ratelimit.cc` — stats struct `InstanceStats` (`ratelimit.h:29-36`), global `Config` (descriptor list plus compiled substitution formatters), and `Filter` implementing `Network::ReadFilter`, `Network::ConnectionCallbacks`, and `RateLimit::RequestCallbacks`.

## Lifecycle
- `onNewConnection()` (`ratelimit.cc:75-95`): if the runtime key `ratelimit.tcp_filter_enabled` is disabled (default 100%), the filter short-circuits to `Status::Complete`. Otherwise transitions `NotStarted -> Calling`, bumps `active`/`total`, expands descriptors against `streamInfo()` via `Config::applySubstitutionFormatter` (`ratelimit.cc:40-62`), and invokes `client_->limit(...)`. The `calling_limit_` guard prevents re-entrant `continueReading()` if the client completes inline. Returns `StopIteration` while still `Calling`, else `Continue`.
- `onData()` (`ratelimit.cc:70-73`): returns `StopIteration` iff `status_ == Calling`, otherwise `Continue`. No data is inspected.
- `onEvent()` (`ratelimit.cc:97-107`): on `RemoteClose`/`LocalClose` while still `Calling`, cancels the pending gRPC request and decrements the `active` gauge.
- `complete()` (`ratelimit.cc:109-155`): `RateLimit::RequestCallbacks` entry. Stores any `dynamic_metadata` on the connection's `StreamInfo` under key `envoy.filters.network.ratelimit`, transitions to `Complete`, decrements `active`, and dispatches on `LimitStatus`.

## Decision / logic
- Status machine is `NotStarted -> Calling -> Complete` (`ratelimit.h:114`).
- `OK` branch (`ratelimit.cc:122-124`): increments `ok` and, if not in the inline-call window, calls `continueReading()` (`ratelimit.cc:150-154`).
- `OverLimit` + `ratelimit.tcp_filter_enforcing` enabled (`ratelimit.cc:133-137`): increments `over_limit` and `cx_closed`, closes with reason `ratelimit_close_over_limit`.
- `Error` branch (`ratelimit.cc:138-148`): increments `error`; if `failureModeAllow()` (i.e., `!failure_mode_deny`) increments `failure_mode_allowed` and resumes reading, else closes with `ratelimit_error_failure_mode_connection_close`.
- Inline completion guard (`calling_limit_`) ensures `continueReading()` is not called when `limit()` completed synchronously from inside `onNewConnection` — the subsequent `return` there naturally produces `Continue`.

## Configuration
- `stat_prefix` — required, used as `ratelimit.<prefix>.` (`ratelimit.cc:64-68`).
- `domain` — rate limit service domain; required.
- `descriptors` — repeated; each entry's `value` is compiled into a `FormatterImpl` so `%DOWNSTREAM_REMOTE_ADDRESS%`, headers, dynamic metadata, etc. can be substituted per connection (`ratelimit.cc:28-38`).
- `timeout` — gRPC timeout; default 20 ms (`config.cc:30-31`).
- `failure_mode_deny` — if true, `Error` closes the connection; default allow.
- `rate_limit_service.grpc_service` — upstream gRPC service descriptor; wrapped into a `GrpcServiceConfigWithHashKey` (`config.cc:34-35`).

## Stats
Prefix `ratelimit.<stat_prefix>.`:
- `total` (counter) — every `onNewConnection` that calls the service.
- `active` (gauge) — number of in-flight rate limit calls.
- `ok`, `error`, `over_limit` (counters) — per terminal `LimitStatus`.
- `cx_closed` (counter) — incremented on forced close for `over_limit` or for `error` when deny.
- `failure_mode_allowed` (counter) — error path allowed through.

## Factory
`RateLimitConfigFactory` (name `envoy.filters.network.ratelimit`) is legacy-registered as a `NamedNetworkFilterConfigFactory` alias `envoy.ratelimit` (`config.cc:47-48`). `createFilterFactoryFromProtoTyped` asserts required fields and builds the `Filters::Common::RateLimit::rateLimitClient` via the common helper.

# Redis Health Checker

Active Redis health probe. Sends either a `PING` command (expecting `PONG`)
or an `EXISTS <key>` command (expecting integer reply 0, i.e. missing key)
over a real Redis client built on top of the same `redis_proxy` connection
pool infrastructure used by the filter.

Proto: `envoy.extensions.health_checkers.redis.v3.Redis`

## Files
- `redis.h` - `RedisHealthChecker`, its per-host session, and an inner
  `RedisConfig` that adapts `Config::Metadata` for the Redis client pool.
- `redis.cc` - session lifecycle, request dispatch, response handling.
- `utility.h` - helper for parsing/validating `RedisHealthChecker` config.
- `config.h` / `config.cc` - factory registered as
  `envoy.health_checkers.redis` via the custom factory mechanism.

## Interface
- `HealthCheckerFactory` (see `config.h`) implements
  `Server::Configuration::CustomHealthCheckerFactory`.
- `RedisHealthChecker` extends `Upstream::HealthCheckerImplBase`.
- `RedisActiveHealthCheckSession` extends `ActiveHealthCheckSession` and
  implements `NetworkFilters::Common::Redis::Client::ClientCallbacks` and
  `Network::ConnectionCallbacks`.

## Logic
- Construction (`redis.cc:37-41`): if `redis_config.key()` is set, the
  probe type is `Exists`; otherwise `Ping`.
- `onInterval` (`redis.cc:82`): if no client yet, build one with
  `client_factory_.create(host_, dispatcher_, redis_config_, ...)` using
  auth username/password pulled from the cluster's
  `RedisProxy::ProtocolOptions` and optional AWS IAM config/authenticator.
  Send the prebuilt `PING` or `EXISTS` RESP array via `client_->makeRequest`.
- `onResponse` (`redis.cc:103`):
  - `Ping`: success if response is `SimpleString("PONG")`; else
    `handleFailure(ACTIVE)`.
  - `Exists`: success if response is `Integer(0)` (key missing); a non-zero
    integer increments `exists_failure_` and `handleFailure(ACTIVE)`.
  - Closes the client if `reuse_connection_` is off.
- `onFailure` -> `handleFailure(NETWORK)`.
- `onRedirection` -> treated as success (a live server routed us away).
- `onTimeout` -> cancel the in-flight request and close the client.
- `onEvent` defers deletion of the client on remote/local close.

## Key decision points
- `redis.cc:107-115` - `EXISTS` response classification.
- `redis.cc:117-124` - `PING` response classification.
- `redis.cc:137-142` - redirection-as-success policy.
- `redis.h:70-103` - `RedisConfig` forces short timeouts, disables outlier
  events, and targets primaries only.

## Configuration
`Redis` fields: `key` (optional - switches to `EXISTS` probe). Auth and
TLS settings are inherited from the cluster's RedisProxy protocol options.

## Stats
`exists_failure` counter under `health_check.redis.*` (`redis.h:25-32`,
`redis.cc:57-60`). Inherits all common health-checker stats.

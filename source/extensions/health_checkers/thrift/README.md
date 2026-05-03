# Thrift Health Checker

Active Thrift health probe. Sends a single Thrift method call (user-
configured `method_name`) over a chosen transport/protocol and passes the
host if the Thrift response reports success.

Proto: `envoy.extensions.health_checkers.thrift.v3.Thrift`

## Files
- `thrift.h` - `ThriftHealthChecker` and its per-host session.
- `thrift.cc` - session lifecycle and client callback handling.
- `client.h` - `Client`, `ClientCallback`, and `ClientFactory` abstractions
  used to send/receive a Thrift message without a full proxy filter.
- `client_impl.h` / `client_impl.cc` - production client: builds a
  Thrift transport + protocol, a `DecoderEventHandler` bridge, and owns a
  real network connection.
- `utility.h` - helpers for protocol/transport enum conversion and request
  serialization.
- `config.h` / `config.cc` - factory registered as
  `envoy.health_checkers.thrift`.

## Interface
- `ThriftHealthCheckerFactory` implements
  `Server::Configuration::CustomHealthCheckerFactory`.
- `ThriftHealthChecker` extends `Upstream::HealthCheckerImplBase`.
- `ThriftActiveHealthCheckSession` extends `ActiveHealthCheckSession` and
  implements `ClientCallback`, which inherits from
  `Network::ConnectionCallbacks`.

## Logic
- Construction (`thrift.cc:21-38`): reject `Auto` transport/protocol and
  the `Twitter` protocol; these are only valid on the proxy side where
  autodetection is possible.
- `onInterval` (`thrift.cc:60`): if no client, call `client_factory_.create`
  with the configured transport/protocol/method and a fixed sequence id
  (0), then `client->start()` which dials the connection via
  `createConnection` (which delegates to
  `host_->createHealthCheckConnection`). Then `client_->sendRequest()`
  frames and writes the request.
- `onResponseResult(is_success)` (`thrift.cc:81`): `handleSuccess` or
  `handleFailure(ACTIVE, retriable=false)`. Closes the client if
  `reuse_connection_` is off.
- `onTimeout`: mark `expect_close_` and close the client.
- `onEvent` (`thrift.cc:101`): unexpected close (`!expect_close_`) is a
  `handleFailure(NETWORK)`; deferred-deletes the client in either case.

## Key decision points
- `thrift.cc:33-38` - rejection of Auto/Twitter transport/protocol combos.
- `thrift.cc:60-72` - per-interval request path, including fixed seq id.
- `thrift.cc:101-115` - close-event classification.

## Configuration
`Thrift` fields: `method_name` (required), `transport` (Framed/Unframed/
Header), `protocol` (Binary/Compact/etc, excluding Auto and Twitter).

## Stats
Inherits all common health-checker stats; no Thrift-specific stats.

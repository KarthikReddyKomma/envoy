# TCP Health Checker

Active TCP health probe. Opens a client connection to the host, optionally
writes a send-payload (and/or a PROXY protocol header), and looks for an
exact byte-pattern match in the response. If no `receive` bytes are
configured, a successful connect is a pass.

Proto: `envoy.config.core.v3.HealthCheck.TcpHealthCheck` (inline on the core
`HealthCheck` message).

## Files
- `health_checker_impl.h` - `TcpHealthCheckerImpl`, `TcpActiveHealthCheckSession`,
  `TcpSessionCallbacks` (connection callbacks + read filter combo).
- `health_checker_impl.cc` - factory registration and session event handling.

## Interface
- `TcpHealthCheckerFactory` implements
  `Server::Configuration::CustomHealthCheckerFactory`, registered as
  `envoy.health_checkers.tcp`.
- `TcpHealthCheckerImpl` extends `Upstream::HealthCheckerImplBase`.
- `TcpActiveHealthCheckSession` extends `ActiveHealthCheckSession` and
  receives callbacks via the helper `TcpSessionCallbacks`, which implements
  `Network::ConnectionCallbacks` and `Network::ReadFilterBaseImpl`.

## Logic
- `onInterval` (`health_checker_impl.cc:132`): if no client connection yet,
  call `host_->createHealthCheckConnection` to allocate one, install
  `TcpSessionCallbacks`, `connect()`, and disable Nagle (`noDelay(true)`).
  Assemble the write buffer from the optional proxy-protocol header
  (V1 or V2 local) plus `send_bytes_` and send it on the connection.
- `onData` (`health_checker_impl.cc:80`): pattern-match the incoming buffer
  against `receive_bytes_`. On a successful `PayloadMatcher::match`, call
  `handleSuccess(false)` and close the connection unless `reuse_connection_`
  is enabled.
- `onEvent` (`health_checker_impl.cc:97`): any unexpected remote/local
  close becomes `handleFailure(NETWORK)`. When `receive_bytes_` is empty and
  the connection event is `Connected`, treat the successful connect itself
  as a pass and close.
- `onTimeout`: mark `expect_close_` and abort the connection.

## Key decision points
- `health_checker_impl.cc:85-94` - payload-match success path and optional
  close.
- `health_checker_impl.cc:108-128` - connect-only success path.
- `health_checker_impl.cc:151-171` - proxy-protocol header generation.

## Configuration
`TcpHealthCheck` fields: `send` (hex/text payload), `receive` (repeated
pattern segments), optional `proxy_protocol_config` (V1 or V2).

## Stats
Inherits the common health-checker stats. No TCP-specific stats are emitted
here.

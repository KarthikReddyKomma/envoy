# HTTP Health Checker

Active HTTP/1, HTTP/2, or HTTP/3 health probe. Sends a configured HTTP
request to each host and classifies the response as Success, Degraded,
Failed, or Retriable based on the response status range, optional body
match, and optional service-name matcher.

Proto: `envoy.config.core.v3.HealthCheck.HttpHealthCheck` (inline on the
core `HealthCheck` message).

## Files
- `health_checker_impl.h` - `HttpHealthCheckerImpl`, its `HttpStatusChecker`,
  the `HttpActiveHealthCheckSession`, and the prod/factory classes.
- `health_checker_impl.cc` - factory registration, session lifecycle, HTTP
  request/response handling.

## Interface
- `HttpHealthCheckerFactory` implements
  `Server::Configuration::CustomHealthCheckerFactory` and registers under
  `envoy.health_checkers.http`.
- `HttpHealthCheckerImpl` extends `Upstream::HealthCheckerImplBase`.
- `HttpActiveHealthCheckSession` extends `ActiveHealthCheckSession` and
  implements `Http::ResponseDecoderImplBase` and `Http::StreamCallbacks`.

## Logic
- `onInterval` (`health_checker_impl.cc:271`): if no codec client yet,
  `host_->createHealthCheckConnection` + `createCodecClient` (built via
  `CodecClientProd` in the prod subclass). Register connection and HTTP
  callbacks, then `client_->newStream(*this)` and `encodeHeaders` with a
  synthesized request (method, host, path, user-agent). If configured, also
  sends a payload as the request body.
- `decodeHeaders` captures the response headers; end of stream triggers
  `onResponseComplete`.
- `decodeData` accumulates body up to `response_buffer_size_` when a body
  match (`receive_bytes_`) is expected.
- `onResponseComplete` (`health_checker_impl.cc:419`) classifies via
  `healthCheckResult`:
  - body matcher fails -> `Failed` (sets EXCLUDED_VIA_IMMEDIATE_HC_FAIL if
    header present).
  - status outside expected ranges -> `Retriable` if within retriable
    ranges, else `Failed`.
  - `x-envoy-degraded` header -> `Degraded`.
  - optional `verify_cluster` matcher checks
    `x-envoy-upstream-healthchecked-cluster`.
- `onResetStream` maps stream resets to network failure unless we expected
  the reset (`expect_reset_`).
- `onGoAway` (`health_checker_impl.cc:340`): if `NoError` and a request is
  in flight, flip `reuse_connection_` off and let the probe finish; if a
  request is in flight on error GOAWAY, record a network failure and close.

Connection reuse: a single codec client is reused across probes unless
`reuse_connection_` is off, the response requests close, or the connection
drops.

## Key decision points
- `health_checker_impl.cc:271-322` - per-interval request build and send.
- `health_checker_impl.cc:364-417` - status/body classification.
- `health_checker_impl.cc:340-362` - HTTP/2+ GOAWAY handling.

## Configuration
`HttpHealthCheck` knobs include `host`, `path`, `method`, `send`, `receive`,
`response_buffer_size`, `expected_statuses`, `retriable_statuses`,
`service_name_matcher`, `codec_client_type` (HTTP1/2/3),
`request_headers_to_add`, `request_headers_to_remove`.

## Stats
Inherits the common health checker stats (see
`source/extensions/health_checkers/common/README.md`); also sets per-host
metadata flags like `EXCLUDED_VIA_IMMEDIATE_HC_FAIL` when
`x-envoy-immediate-health-check-fail` is present on a failing response.

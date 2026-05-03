# gRPC Health Checker

Active probe using the standard `grpc.health.v1.Health/Check` RPC. Opens an
HTTP/2 codec client to the host, sends a unary `Check` request, and pass/
fails the host based on the returned `grpc-status` and `status` field of the
`HealthCheckResponse` payload.

Proto: `envoy.config.core.v3.HealthCheck.GrpcHealthCheck` (inline on the core
`HealthCheck` message).

## Files
- `health_checker_impl.h` - `GrpcHealthCheckerImpl`, the session, and the
  prod subclass that creates a real codec client.
- `health_checker_impl.cc` - factory registration and the full session
  state machine for gRPC framing, trailers, and GOAWAY handling.

## Interface
- `GrpcHealthCheckerFactory` implements
  `Server::Configuration::CustomHealthCheckerFactory`, registered as
  `envoy.health_checkers.grpc`.
- `GrpcHealthCheckerImpl` extends `Upstream::HealthCheckerImplBase`.
- `GrpcActiveHealthCheckSession` extends `ActiveHealthCheckSession` and
  implements `Http::ResponseDecoderImplBase` plus `Http::StreamCallbacks`.

## Logic
- `onInterval` (`health_checker_impl.cc:186`): if no codec client yet,
  `host_->createHealthCheckConnection` and `createCodecClient` to build a
  `Http::CodecClient`. Register connection + HTTP callbacks. Allocate a new
  stream, encode HTTP/2 gRPC request headers for
  `POST /grpc.health.v1.Health/Check` (with `authority_value_`, TE, content
  type, optional user metadata), frame an empty/populated
  `HealthCheckRequest` proto into a gRPC wire frame, and encode it as the
  body.
- `decodeHeaders` -> if the HTTP response status is not 200, short-circuit
  into `onRpcComplete` mapping `httpToGrpcStatus`. If headers arrive with
  `end_stream=true` (trailers-only), extract `grpc-status` immediately.
- `decodeData` -> accumulate one gRPC frame using `Grpc::Decoder`, parse it
  into a `grpc::health::v1::HealthCheckResponse`; streaming messages are
  treated as protocol violations.
- `decodeTrailers` (`health_checker_impl.cc:164`) -> pull `grpc-status` and
  call `onRpcComplete` with the final status.
- `onRpcComplete` decides:
  - `grpc_status == OK` and `health_check_response.status == SERVING` ->
    `handleSuccess`.
  - anything else -> `handleFailure(ACTIVE)`.
- `onGoAway`: if `NoError` and a request is in flight, set
  `received_no_error_goaway_` so the connection is torn down after the
  probe completes. Otherwise treat as a network failure.
- `onEvent` / `onResetStream` flush the client on close and record network
  failures when the reset was unexpected.

## Key decision points
- `health_checker_impl.cc:96-128` - HTTP status / gRPC protocol validation
  in `decodeHeaders`.
- `health_checker_impl.cc:130-162` - body frame decode and response parse.
- `health_checker_impl.cc:186-215+` - per-interval request construction and
  send.

## Configuration
`GrpcHealthCheck` knobs: `service_name` (becomes the
`HealthCheckRequest.service`), `authority` (override of the HTTP/2
`:authority` header), `initial_metadata` (repeated `HeaderValueOption`s
applied via the `Router::HeaderParser`).

## Stats
Inherits the common health-checker stats.

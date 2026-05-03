# External Processor Filter (`envoy.filters.http.ext_proc`)

Hooks a full-duplex gRPC stream into the HTTP filter chain so an out-of-process
service can inspect and mutate headers, body, and trailers in either direction
in real time.

Proto: `envoy.extensions.filters.http.ext_proc.v3.ExternalProcessor`.

## Stream model

The filter opens a single bidirectional gRPC stream to the processor on demand
(`openStream()`). For the lifetime of the request it exchanges
`ProcessingRequest` / `ProcessingResponse` messages. The stream can be reused
across requests on the same connection.

## `ProcessingMode`

What is sent is controlled per-direction:

| Mode                         | Request / Response headers / body / trailers |
|------------------------------|----------------------------------------------|
| `SKIP`                       | Don't send.                                   |
| `SEND`                       | Send headers / trailers (for body, see below).|
| `NONE`                       | Body: don't send.                             |
| `STREAMED`                   | Body: send chunks as they arrive.             |
| `BUFFERED`                   | Body: accumulate full body, then send.        |
| `BUFFERED_PARTIAL`           | Body: buffer up to a cap, then stream.        |
| `FULL_DUPLEX_STREAMED`       | Body: stream request and response concurrently without buffering. |

If `allow_mode_override` is set, the processor can change the mode in the
first response.

## Response semantics

`onReceiveMessage()` (`ext_proc.cc:1514`) dispatches on the response case:

- `request_headers` / `response_headers` → `handleHeadersResponse`.
- `request_body` / `response_body` → `handleBodyResponse`.
- `request_trailers` / `response_trailers` → `handleTrailersResponse`.
- `immediate_response` → short-circuit with the supplied status and body.
- `override_message_timeout` → extend the per-message timer.

`CommonResponse` may carry `HeaderMutation`, `BodyMutation`, and
`clear_route_cache`. Header mutations pass through `mutation_checker`
(`ext_proc.h:318`) against configured `HeaderMutationRules`; invalid
mutations are rejected and counted (`rejected_header_mutations`).

## Timeouts and failure modes

- `message_timeout` — per message (default 5 s, max `max_message_timeout_ms`).
- `failure_mode_allow` — on timeout / stream error / gRPC error either
  continue (`failure_mode_allowed`) or fail closed with 503.
- `graceful_grpc_close` — controls whether the stream is drained cleanly.
- `send_body_without_waiting_for_header_response` — pipeline body chunks
  without waiting for the headers response.

## Route behaviour

`clear_route_cache` flips the route, letting the processor redirect requests.
`route_cache_action` decides whether re-computation is allowed. Metrics:
`clear_route_cache_ignored`, `clear_route_cache_disabled`.

## Entry points

- `decodeHeaders()` (`ext_proc.cc:704`) — start request-side processing.
- `encodeHeaders()` — start response-side processing.
- `onReceiveMessage()` (`ext_proc.cc:1514`) — apply processor mutations,
  drive decoding/encoding state machines.

## Stats

`streams_started`, `stream_msgs_sent`, `stream_msgs_received`,
`failure_mode_allowed`, `message_timeouts`, `rejected_header_mutations`,
`override_message_timeout_received`, `spurious_msgs_received`, and route
cache stats (`ext_proc.h:38–55`).

## Files

- `ext_proc.{h,cc}` — filter, processing state machine.
- `processor_state.{h,cc}` — decoder / encoder state.
- `client.{h,cc}` — gRPC client wrapper.
- `mutation_utils.{h,cc}` — header/body mutation application.
- `config.{h,cc}` — factory.

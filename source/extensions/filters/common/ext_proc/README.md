# External Processing Client Library (shared filter infrastructure)

Header-only, templated gRPC streaming client used by the HTTP `ext_proc` filter and the network `ext_proc` filter to talk to an external processing server over a bidirectional gRPC stream. The library parameterises the request/response proto types, so the same code powers HTTP's `ProcessingRequest`/`ProcessingResponse` pair and the network filter's `ProcessingRequest`/`ProcessingResponse` pair without duplication.

## Files
- `client_base.h` - Abstract `StreamBase`, `RequestCallbacks<ResponseType>`, and `ClientBase<RequestType, ResponseType>` - minimal contract with no gRPC dependency.
- `grpc_client.h` - Bidirectional-stream contract: `ProcessorCallbacks<ResponseType>`, `ProcessorStream<RequestType, ResponseType>`, `ProcessorClient<RequestType, ResponseType>`.
- `grpc_client_impl.h` - Template implementations `ProcessorStreamImpl` and `ProcessorClientImpl`. Entirely in the header because it is instantiated per request/response pair in the consuming BUILDs.

## Public interface
- `ProcessorCallbacks<Resp>` (`grpc_client.h:17`) extends `RequestCallbacks<Resp>` with:
  - `onReceiveMessage(std::unique_ptr<Resp>&&)` - one callback per server-sent message.
  - `onGrpcError(GrpcStatus, const std::string&)` - terminal, invoked on non-OK `onRemoteClose`.
  - `onGrpcClose()` - terminal, invoked on OK remote close.
  - `logStreamInfo()` - called just before the terminal callback so the filter can snapshot stream info for access logs.
- `ProcessorStream<Req, Resp>` (`grpc_client.h:30`):
  - `send(Req&&, bool end_stream)` - write a message.
  - `close()` - idempotent half-close of the client side; returns whether it actually closed.
  - `halfCloseAndDeleteOnRemoteClose()` - closes and schedules deletion once the server half-closes or a timer fires.
  - `notifyFilterDestroy()` - must be called by the filter before it destructs so the stream drops its `OptRef` to the callbacks and clears the parent stream info (`grpc_client_impl.h:42`).
  - `streamInfo()` - underlying Envoy async-stream info, including parent stream linkage.
- `ProcessorClient<Req, Resp>::start(callbacks, config_with_hash_key, StreamOptions&, sidestream_watermark_callbacks)` (`grpc_client.h:52`) - opens a new bidi stream; returns `nullptr` on failure.
- `ProcessorClientImpl<Req, Resp>(AsyncClientManager&, Stats::Scope&, service_method)` (`grpc_client_impl.h:216`) - concrete factory; `service_method` is a static `absl::string_view` like `envoy.service.ext_proc.v3.ExternalProcessor.Process`.

## Implementation logic
- `ProcessorStreamImpl::create(...)` constructs the object, optionally wires sidestream watermark callbacks when `envoy.reloadable_features.grpc_side_stream_flow_control` is enabled, and returns `nullptr` if `startStream` fails (`grpc_client_impl.h:96`, `:104`).
- `startStream` looks up the method descriptor from the generated pool and opens `client_.start(...)`; a null stream means the upstream cluster is unreachable (`grpc_client_impl.h:117`).
- `send()` just forwards to `stream_.sendMessage` (`grpc_client_impl.h:128`).
- `close()` guards with `stream_closed_`, unregisters watermark callbacks, calls `closeStream()` and `resetStream()`, and returns whether it actually transitioned (`grpc_client_impl.h:132`).
- `halfCloseAndDeleteOnRemoteClose()` is the preferred teardown: it closes the client side and calls `waitForRemoteCloseAndDelete()` so the stream lives until the server half-closes or the timer fires, letting in-flight responses land (`grpc_client_impl.h:148`).
- `notifyFilterDestroy()` resets the `OptRef<ProcessorCallbacks>` so later inbound events drop without touching a freed filter; if the stream is still open it clears the parent stream info and removes watermark callbacks to avoid dangling references (`grpc_client_impl.h:42`).
- `onReceiveMessage` forwards only if callbacks are still live (`grpc_client_impl.h:166`). `onRemoteClose` sets `stream_closed_`, calls `logStreamInfo()`, then dispatches to either `onGrpcClose()` or `onGrpcError()` (`grpc_client_impl.h:188`).
- `ProcessorClientImpl::start` calls `AsyncClientManager::getOrCreateRawAsyncClientWithHashKey(config_with_hash_key, scope_, /*skip_cluster_check=*/true)`, periodically logs on failure, and delegates to `ProcessorStreamImpl::create` (`grpc_client_impl.h:221`).

## Consumers
- `source/extensions/filters/http/ext_proc` - instantiates `ProcessorClientImpl<envoy::service::ext_proc::v3::ProcessingRequest, ProcessingResponse>` (`source/extensions/filters/http/ext_proc/client_impl.h`).
- `source/extensions/filters/network/ext_proc` - instantiates the same template with the network-service protos (`source/extensions/filters/network/ext_proc/client_impl.h`).

## Stats / errors / failure modes
- No stats are emitted here; both consumer filters count stream starts, timeouts, error closes, and message counts in their own scopes.
- Stream start failures return `nullptr` and are surfaced via `ENVOY_LOG_PERIODIC_MISC(error, ...)` (`grpc_client_impl.h:228`) throttled to every 10 seconds - the filter is responsible for reacting (typically by setting an `ext_proc_error` response-code-detail and falling through to `failure_mode_allow`).
- Remote close with non-OK status always lands in `onGrpcError`; the filter must decide whether to reset downstream, continue, or retry.
- Use-after-free is prevented through `notifyFilterDestroy()` - if the filter forgets to call it, the stream's `OptRef` still references the destroyed filter and the next `onReceiveMessage` will crash.

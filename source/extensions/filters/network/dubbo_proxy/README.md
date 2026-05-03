# Dubbo Proxy (`envoy.filters.network.dubbo_proxy`)

A full L7 proxy for the Apache Dubbo RPC protocol, implemented as a single network read filter (`ConnectionManager`). It owns a pluggable protocol codec (framing) and serializer (payload, Hessian2 today), runs each decoded RPC through a bidirectional decoder/encoder filter chain (terminated by the built-in router), performs service/method-based routing with RDS support (DRDS), and tracks in-flight requests for correct reset semantics when the downstream connection dies.

Proto: `envoy.extensions.filters.network.dubbo_proxy.v3.DubboProxy`.

## Files
- `config.h/cc` — `DubboProxyFilterConfigFactory` (factory), `ConfigImpl` (per-listener state: stats, protocol/serialization types, route provider, filter-factory list). `config.cc:23-45` is the factory callback; `config.cc:94-178` builds `ConfigImpl`, resolves DRDS vs static route config, and registers either the filters from `dubbo_filters` or the default `envoy.filters.dubbo.router`.
- `conn_manager.h/cc` — `ConnectionManager`, the `Network::ReadFilter`. Owns the per-connection `RequestDecoder`, the `request_buffer_`, and the list of `ActiveMessage`s.
- `decoder.h/cc` — generic decoder framework: `DecoderStateMachine` (protocol state machine over `OnDecodeStreamHeader` → `OnDecodeStreamData` → `Done`), `DecoderBase`, and `Decoder<T>` / `RequestDecoder` / `ResponseDecoder` specializations.
- `decoder_event_handler.h` — `StreamHandler`, `RequestDecoderCallbacks`, `ResponseDecoderCallbacks`; the contract between decoder and conn-manager/router.
- `active_message.h/cc` — per-RPC state: `ActiveMessage` (decoder-side), `ActiveResponseDecoder` (upstream-reply decoder), and `ActiveMessageDecoderFilter` / `ActiveMessageEncoderFilter` wrappers that expose `Decoder/EncoderFilterCallbacks` to individual filters and drive filter-chain iteration.
- `protocol.h`, `dubbo_protocol_impl.h/cc` — `Protocol` interface (`decodeHeader` / `decodeData` / `encode`) and the concrete `DubboProtocolImpl` for the binary Dubbo framing (`dubbo://`).
- `serializer.h`, `dubbo_hessian2_serializer_impl.h/cc`, `hessian_utils.h/cc` — `Serializer` interface and the Hessian2 implementation for RPC invocation bodies.
- `message.h`, `message_impl.h/cc`, `metadata.h` — context/metadata objects that travel through the filter chain (request id, message type, invocation info, response status, the original bytes).
- `heartbeat_response.h/cc` — builder for heartbeat replies.
- `app_exception.h/cc` — `DirectResponse` that encodes a Dubbo exception payload for local replies.
- `stats.h` — `DubboFilterStats` counter/gauge/histogram struct (shared with connection manager, active messages and router).
- `filters/` — `NamedDubboFilterConfigFactory`, `FactoryBase`, and the filter/callbacks interfaces (`DecoderFilter`, `EncoderFilter`, `CodecFilter`, `FilterChainFactoryCallbacks`).
- `router/` — `Config` / `Route` / `RouteEntry` interfaces, `RouteMatcher` (interface/method/parameter matchers over `MultipleRouteConfiguration`), `router_impl.h/cc` (the terminal codec filter that talks to the TCP conn pool), and `rds_impl.h` (DRDS provider plumbing built on common `Rds`).

## Lifecycle (ConnectionManager — the single `Network::ReadFilter`)
- `initializeReadFilterCallbacks(callbacks)` stores the callbacks, registers as a `ConnectionCallbacks` listener, enables half-close, and sets the per-connection buffer limit to `UINT32_MAX` (`conn_manager.cc:47-52`).
- `onNewConnection()` returns `Continue` (`conn_manager.cc:43-45`). No protocol sniffing is required up-front — the decoder waits for data.
- `onData(data, end_stream)` (`conn_manager.cc:27-41`):
  1. `request_buffer_.move(data)` — accumulate across reads, since Dubbo frames arbitrary-length bodies.
  2. `dispatch()` runs the decoder against `request_buffer_`.
  3. If `end_stream`, calls `resetAllMessages(false)` (remote close with active requests) and closes the connection with `FlushWrite`.
  4. Always returns `StopIteration` — the filter consumes all raw bytes itself.
- `onWrite` is not implemented; responses are written directly via `connection().write(...)` from `ActiveResponseDecoder::onStreamDecoded` and `sendLocalReply`.
- `ConnectionCallbacks::onEvent(event)` (`conn_manager.cc:54-56`) invokes `resetAllMessages(event == LocalClose)`, which iterates `active_message_list_` and calls `onReset()` on each, incrementing either `cx_destroy_local_with_active_rq_` or `cx_destroy_remote_with_active_rq_` per active request (`conn_manager.cc:162-174`).
- `onAboveWriteBufferHighWatermark` / `onBelowWriteBufferLowWatermark` toggle `readDisable` as flow-control backpressure to the downstream (`conn_manager.cc:58-66`).

## Decode / dispatch
`dispatch()` (`conn_manager.cc:95-114`) loops `decoder_->onData(request_buffer_, underflow)` until the decoder reports buffer underflow. Any `EnvoyException` thrown from the protocol or serializer increments `request_decoding_error_`, closes the connection `NoFlush`, and resets all active messages.

`DecoderBase::onData` drives a `DecoderStateMachine` (`decoder.cc:71-85`) with two states (`decoder.cc:60-69`):
- `OnDecodeStreamHeader` calls `protocol_.decodeHeader(buffer, metadata)`. On partial data it returns `WaitForData`. For heartbeat messages (`HeartbeatRequest`/`HeartbeatResponse`), it waits for the full body then invokes `delegate_.onHeartbeat(metadata)` and transitions straight to `Done` (`decoder.cc:22-33`). Otherwise it asks the delegate (`ConnectionManager::newStream`) for an `ActiveStream`, moves the header bytes into `context->originMessage()`, and transitions to `OnDecodeStreamData`.
- `OnDecodeStreamData` calls `protocol_.decodeData(buffer, context, metadata)`. On underflow it waits; on success it moves the body into `originMessage()`, invokes `active_stream_->onStreamDecoded()` (which is `StreamHandler::onStreamDecoded` on the `ActiveMessage`), clears the stream, and reports `Done` (`decoder.cc:42-58`).

`ConnectionManager::newStream()` (`conn_manager.cc:68-75`) allocates a new `ActiveMessage`, builds its filter chain (`createFilterChain` → `ConfigImpl::createFilterChain` → each registered `DubboFilters::FilterFactoryCb`, ending in the router), links it into `active_message_list_`, and returns it as the `StreamHandler`.

`ConnectionManager::onHeartbeat(metadata)` (`conn_manager.cc:77-93`) sets the response status to `Ok`, type to `HeartbeatResponse`, and encodes via `HeartbeatResponse::encode(metadata, *protocol_, buffer)` straight to the downstream connection. Skipped if the connection is not `Open`. Increments `request_event_`.

## Active message and filter chain
`ActiveMessage::onStreamDecoded(metadata, ctx)` (`active_message.cc:238-266`) binds the decoded message to a closure `filter_action_` and calls `applyDecoderFilters(nullptr, CanStartFromCurrent)` (`active_message.cc:308-324`). The iteration walks `decoder_filters_` in order; `FilterStatus::StopIteration` pauses iteration (the filter must call `continueDecoding()`), `AbortIteration` tears down the message via `parent_.deferredMessage(*this)`, `Continue` proceeds. Request stats (`request_`, `request_twoway_`, `request_oneway_`) are bumped in `finalizeRequest()` after the chain finishes (`active_message.cc:268-287`). Oneway requests and requests that already produced a local reply are immediately removed from the active list.

Encoder filters run in reverse over `encoder_filters_` when upstream responses flow back (`applyEncoderFilters`). `ActiveResponseDecoder` (`active_message.h:29-60`, `active_message.cc:13-71`) embeds its own `ResponseDecoder` bound to a private `ProtocolPtr`, and on `onStreamDecoded` applies the encoder chain, writes the original upstream bytes to the downstream via `response_connection_.write(ctx->originMessage(), false)`, and updates `response_*` stats.

`sendLocalReply(metadata, response, end_stream)` on the conn-manager (`conn_manager.cc:116-152`) lets filters/router respond without going upstream. It encodes via `response.encode(metadata, *protocol_, buffer)` into a fresh buffer, writes it, optionally closes on `end_stream`, and increments one of `local_response_success_` / `local_response_error_` / `local_response_business_exception_` based on the returned `DirectResponse::ResponseType`.

## Routing
`ConfigImpl` holds a `Rds::RouteConfigProviderSharedPtr` chosen in the ctor (`config.cc:103-130`):
- `drds` (dynamic, single route config via xDS) — only `AGGREGATED_GRPC` / `AGGREGATED_DELTA_GRPC` API types are permitted; built through `createRdsRouteConfigProvider`.
- `multiple_route_config` (static list scoped by protocol/service).
- bare `route_config` is wrapped into a synthetic `MultipleRouteConfiguration` for uniform handling.
- `drds` + `route_config` and `multiple_route_config` + `route_config` combinations throw (`config.cc:104-105, 118-119`).

`ConfigImpl::route(metadata, random_value)` (`config.cc:151-155`) pulls the current config via the provider and calls `Router::Config::route(...)`, whose `RouteMatcher` resolves interface/method/header matchers against `metadata.invocationInfo`.

`Router::Router::onMessageDecoded` (`router/router_impl.cc:26-80+`) is the terminal `CodecFilter`:
- `callbacks_->route()` is resolved once and cached on the `ActiveMessage` (`active_message.cc:293-306`).
- No route → `AppException(ServiceNotFound)` local reply, `AbortIteration`.
- `cluster_manager_.getThreadLocalCluster(clusterName)` missing / in maintenance / no conn pool → `ServerError` local reply.
- Otherwise opens a `Tcp::ConnectionPool` connection and streams the original request bytes upstream (`UpstreamRequest::onPoolReady`), installs itself as the `UpstreamCallbacks`, and feeds the reply back through an `ActiveResponseDecoder` it drives via `DecoderFilterCallbacks::upstreamData(...)` → `startUpstreamResponse()`.

Weighted-cluster / subset load balancing uses `stream_id_` (from the active message) and `MetadataMatchCriteria` carried through the route entry (`router/router_impl.h:40-43`).

## Protocol / serialization binding
`ConfigImpl::createProtocol()` (`config.cc:157-159`) resolves `NamedProtocolConfigFactory::getFactory(protocol_type_).createProtocol(serialization_type_)`. The current mapping in `config.cc:53-91` is `ProtocolType::Dubbo` + `SerializationType::Hessian2`; new protocols/serializers can be added by registering `NamedProtocolConfigFactory` / `NamedSerializerConfigFactory` implementations. The `ConnectionManager` caches one `ProtocolPtr` per connection and derives the `RequestDecoder` from it (`conn_manager.cc:20-25`). Each `ActiveResponseDecoder` creates its own `Protocol` copy so upstream and downstream decoders are independent.

## Filter chain factory
`ConfigImpl::registerFilter` (`config.cc:161-178`) looks up a `DubboFilters::NamedDubboFilterConfigFactory` by name, translates the opaque config, and appends the returned `FilterFactoryCb` to `filter_factories_`. `createFilterChain` (`config.cc:145-149`) invokes every factory against `DubboFilters::FilterChainFactoryCallbacks` — on the `ActiveMessage`, that resolves to `addDecoderFilter` / `addEncoderFilter` / `addFilter` (CodecFilter = both). The default chain is a single `envoy.filters.dubbo.router` (`config.cc:132-142`).

## Stats
Emitted under `dubbo.<stat_prefix>.` (`config.cc:97-98`). Defined in `stats.h:16-36`:
- Decoding: `request_decoding_success`, `request_decoding_error`, `response_decoding_success`, `response_decoding_error`.
- Request accounting: `request`, `request_twoway`, `request_oneway`, `request_event` (heartbeat), `request_active` (gauge), `request_time_ms` (histogram).
- Response accounting: `response`, `response_success`, `response_error`, `response_business_exception`, `response_error_caused_connection_close`.
- Local reply path: `local_response_success`, `local_response_error`, `local_response_business_exception`.
- Connection teardown: `cx_destroy_local_with_active_rq`, `cx_destroy_remote_with_active_rq`.

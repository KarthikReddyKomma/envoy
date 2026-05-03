# Thrift Proxy (`envoy.filters.network.thrift_proxy`)

L7 proxy for Apache Thrift RPC. The connection manager decodes a framed Thrift stream from the downstream, invokes a pluggable per-request filter chain that terminates in a router, forwards the request to an upstream cluster via a TCP connection pool, decodes the response, and converts the protocol/transport if the request and upstream differ. Supports binary/compact/auto protocols, framed/unframed/header/auto transports, a route table (static or via TRDS), payload passthrough, shadow writes, and oneway/half-close semantics.

Proto: `envoy.extensions.filters.network.thrift_proxy.v3.ThriftProxy` (+ `ThriftProtocolOptions` cluster extension).

## Files

### Core
- `config.h/cc` — `ThriftProxyFilterConfigFactory` (registered as `envoy.filters.network.thrift_proxy`, `config.h:49`), `ConfigImpl` (`config.h:64`) implementing `Config`, `Router::Config`, and `ThriftFilters::FilterChainFactory`, and `ProtocolOptionsConfigImpl` (`config.h:26`) for the cluster-level transport/protocol override.
- `conn_manager.h/cc` — `ConnectionManager` (`conn_manager.h:54`), the `Network::ReadFilter`. Owns the downstream `Decoder`, the list of in-flight `ActiveRpc`s, and the per-request filter chain.
- `decoder.h/cc` / `decoder_events.h` / `passthrough_decoder_event_handler.h` — stateful two-pass decoder driving `DecoderEventHandler` callbacks for transport/message/field events.
- `stats.h` — `ThriftFilterStats` with `cx_destroy_*`, `request*`, `response*`, and `request_time_ms` histogram.

### Transports and protocols
- `transport.h`, `framed_transport_impl.*`, `unframed_transport_impl.*`, `header_transport_impl.*`, `auto_transport_impl.*` — `Transport` implementations; `AutoTransportImpl` sniffs the first frame and delegates.
- `protocol.h`, `binary_protocol_impl.*`, `compact_protocol_impl.*`, `twitter_protocol_impl.*`, `auto_protocol_impl.*` — `Protocol` implementations; auto-protocol peeks the magic bytes.
- `buffer_helper.*`, `filter_utils.*`, `thrift.h`, `thrift_object*.h/cc`, `metadata.h`, `conn_state.h`, `protocol_converter.h` — shared helpers and the protocol-to-protocol converter that handles cross-protocol proxying.
- `app_exception_impl.*` — the `AppException` used by filters and the router to send Thrift-level error replies.
- `tracing.h`, `protocol_options_config.h` — Thrift tracing tags and cluster protocol options.

### Filters
`filters/` implements the per-request filter chain:
- `filters/filter.h` — `DecoderFilter`/`EncoderFilter` (`:202`, `:235`), `FilterCallbacks`/`DecoderFilterCallbacks`/`EncoderFilterCallbacks` (`:96`, `:141`), `FilterChainFactory` (`:359`).
- `filters/filter_config.h`, `filters/factory_base.h`, `filters/pass_through_filter.h` — base classes for implementing filters.
- `filters/header_to_metadata/`, `filters/payload_to_metadata/`, `filters/ratelimit/` — shipped filters.

### Router
`router/` is the terminal decoder filter:
- `router/config.h/cc` — router filter factory (`envoy.filters.thrift.router`).
- `router/router.h`, `router/router_impl.h/cc` — `Router` (`router_impl.h:227`) implementing `ThriftFilters::DecoderFilter`, `Tcp::ConnectionPool::UpstreamCallbacks`, `Upstream::LoadBalancerContextBase`, and `RequestOwner`.
- `router/upstream_request.h/cc` — `UpstreamRequest` owns the upstream TCP conn-pool handle, encodes the request via `ProtocolConverter`, and dispatches responses to the decoder filter callbacks.
- `router/shadow_writer_impl.h/cc` — asynchronous shadow cluster writes.
- `router/rds.h`, `router/rds_impl.h` — static vs. TRDS route config providers (registered singleton `thrift_route_config_provider_manager`, `config.cc:44`).
- `router/router_ratelimit_impl.h/cc` — route-level rate-limit actions.

### Docs
`docs/` contains pre-existing design notes.

## Lifecycle (ConnectionManager as `Network::ReadFilter`)
- `onNewConnection()` — returns `Continue` (`conn_manager.h:65`).
- `initializeReadFilterCallbacks(cb)` — stores callbacks and registers itself as a `ConnectionCallbacks` for close/watermark events (`conn_manager.cc:189`).
- `onData(data, end_stream)` (`conn_manager.cc:27`): `request_buffer_.move(data)` then `dispatch()`. If `end_stream`, either close immediately, or wait for an in-flight oneway RPC to finish by setting `half_closed_ = true`. Always returns `StopIteration`.
- `onWrite` is not overridden; responses are written by the router/filter chain directly onto the downstream connection.
- `onEvent(event)` (`conn_manager.cc:196`): on `LocalClose` / `RemoteClose`, `resetAllRpcs(local_reset=LocalClose)`; bumps `cx_destroy_local_with_active_rq` / `cx_destroy_remote_with_active_rq` when `rpcs_` is non-empty.

## Dispatch loop (`ConnectionManager::dispatch`, `conn_manager.cc:65`)
```
while (!underflow) {
  FilterStatus status = decoder_->onData(request_buffer_, underflow);
  if (status == StopIteration) { stopped_ = true; break; }
}
```
Exits early if `stopped_` or `requests_overflow_` (`max_requests_per_connection` exceeded). Catches:
- `AppException` → reply to the originating RPC with `sendLocalReply` (`:88`).
- `EnvoyException` (transport/protocol mismatch) → close the connection if no RPC context exists, otherwise reply with the active RPC's transport/protocol, bumping `request_decoding_error_` and resetting all in-flight RPCs (`:97`, `:110`).

`continueDecoding()` (`conn_manager.cc:151`) resumes the loop after a stalled decoder filter finishes; if we had half-closed waiting for a oneway, it then closes the connection.

## Per-RPC state machine (`ActiveRpc`)
Each decoded request creates an `ActiveRpc` bound to fresh `Transport`/`Protocol` instances from `Config::createTransport`/`createProtocol` (`config.cc:134`, `:138`). `ActiveRpcDecoderFilter` and `ActiveRpcEncoderFilter` (`conn_manager.cc:458`, `:472`) host the per-request filter chain built by `ConfigImpl::createFilterChain` from `filter_factories_` (`config.cc:128`).

Key transitions:
- `prepareFilterAction(event, data)` (`conn_manager.cc:560`) — feeds each `DecoderEvent` through the decoder filter chain.
- `finalizeRequest()` (`conn_manager.cc:748`) — records request metadata, emits `request`/`request_call`/`request_oneway`/`request_invalid_type`/`request_passthrough`.
- `startUpstreamResponse(transport, protocol)` (`conn_manager.cc:1077`) — creates `ResponseDecoder` bound to the upstream transport/protocol; future upstream bytes flow through `ResponseDecoder::onData` (`conn_manager.cc:237`).
- `sendLocalReply(response, end_stream)` (`conn_manager.cc:1046`) — encodes a `DirectResponse` on the downstream transport/protocol.
- `onReset()` / `onError()` / `onLocalReply()` — cleanup paths that deferredDelete the RPC (`conn_manager.cc:1002`, `:1007`, `:1038`).

`half_closed_`, `stopped_`, and `requests_overflow_` flags in `ConnectionManager` sequence graceful shutdown when a oneway or filter pauses the pipeline.

## Router (`router/router_impl.h:227`)
On `messageBegin`, `Router`:
1. Picks a `Route` from `ConfigImpl::route(metadata, random)` (`config.h:77`) which delegates to the active `Router::Config` (static or TRDS-backed).
2. Creates an `UpstreamRequest` bound to `Upstream::ClusterManager::getThreadLocalCluster(...)->tcpConnPool(...)`.
3. Encodes the request through a `ProtocolConverter` when the cluster's `ProtocolOptionsConfigImpl` overrides the transport/protocol (`config.cc:36`, `:40`).
4. Dispatches shadow requests via `ShadowWriter` for any shadow policies on the route.

As `Tcp::ConnectionPool::UpstreamCallbacks`, the router consumes upstream bytes in `onUpstreamData` (`router_impl.h:315`) and feeds them into the `ResponseDecoder`; `onEvent` handles upstream close/RST by surfacing an exception reply. `close_downstream_on_error_` controls whether a decode error tears down the downstream connection.

## Decision / logic (selected branches)
- Transport/protocol detection: `AutoTransportImpl::decodeFrameStart` / `AutoProtocolImpl::readMessageBegin` decide based on first bytes; once detected, the concrete impl is stored and used for all subsequent frames.
- Filter chain: `ConfigImpl::ConfigImpl` (`config.cc:90`) installs a default `envoy.filters.thrift.router` filter if the config's `thrift_filters` list is empty.
- Route config source: `ConfigImpl` rejects the combination of `trds` + inline `route_config`, and requires `AGGREGATED_GRPC` / `AGGREGATED_DELTA_GRPC` for TRDS (`config.cc:104`).
- Max requests per connection: when `max_requests_per_connection` is exceeded, `requests_overflow_` is set; after the last in-flight RPC finishes, `doDeferredRpcDestroy` closes the connection (`conn_manager.cc:170`).
- Half-close on oneway: `onData` defers the downstream close until the oneway completes (`conn_manager.cc:41`).
- Payload passthrough: `ConfigImpl::payloadPassthrough()` (`config.h:89`) — when true, the decoder streams raw payload bytes instead of parsed fields, increasing throughput.

## Configuration (top-level)
- `stat_prefix` — prefix becomes `thrift.<stat_prefix>.` (`config.cc:82`).
- `transport`, `protocol` — default transport/protocol for the downstream; `AUTO_*` lets the auto impl sniff (`config.cc:84`).
- `thrift_filters` — ordered per-request filter list; router is appended by default.
- `route_config` or `trds` — static vs. dynamic route discovery.
- `payload_passthrough`, `header_keys_preserve_case`, `max_requests_per_connection` (`config.h:89`-`:92`).
- `access_log` — per-RPC access logs (`config.cc:123`).
- Cluster-level `ThriftProtocolOptions` — upstream transport/protocol override resolved by `ProtocolOptionsConfigImpl::transport`/`protocol` (`config.cc:36`).

## Stats
Prefix `thrift.<stat_prefix>.` (see `ALL_THRIFT_FILTER_STATS`, `stats.h:16`):
- Counters: `cx_destroy_local_with_active_rq`, `cx_destroy_remote_with_active_rq`, `downstream_cx_max_requests`, `downstream_response_drain_close`, `request`, `request_call`, `request_decoding_error`, `request_invalid_type`, `request_oneway`, `request_passthrough`, `request_internal_error`, `response`, `response_decoding_error`, `response_error`, `response_exception`, `response_invalid_type`, `response_passthrough`, `response_reply`, `response_success`.
- Gauge: `request_active`.
- Histogram: `request_time_ms`.
The router subpackage adds its own `router.*` stats.

## Factory
`ThriftProxyFilterConfigFactory::createFilterFactoryFromProtoTyped` (`config.cc:46`) resolves a process-wide `Router::RouteConfigProviderManagerImpl` via singleton, constructs `ConfigImpl`, and returns a lambda that calls `filter_manager.addReadFilter(std::make_shared<ConnectionManager>(filter_config, random, time_source, drain_decision))`. Cluster `ThriftProtocolOptions` are produced by `createProtocolOptionsTyped` (`config.h:56`). Registered via `REGISTER_FACTORY` at `config.cc:75`.

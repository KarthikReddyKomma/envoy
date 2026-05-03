# Generic Proxy (`envoy.filters.network.generic_proxy`)

A codec-agnostic L7 proxy framework. It looks structurally like the HTTP connection manager but is parameterized by a pluggable codec (the "generic codec" abstraction defined under `interface/`) and a pluggable L7 filter chain whose filters speak in `RequestHeaderFrame` / `RequestCommonFrame` / `ResponseHeaderFrame` / `ResponseCommonFrame` frames rather than HTTP-specific types. The filter is a `Network::ReadFilter` that owns a server codec, a list of `ActiveStream` objects (one per inflight request), and a frame-handler map indexed by codec-provided stream id so that inbound common (continuation) frames can be routed to their streams. Route matching is performed through a generic `xds.type.matcher.v3.Matcher` whose terminal action is a `RouteMatchAction` producing a `RouteEntryImpl` (`cluster_name`, `timeout`, `retry_policy`, per-filter config).

Proto: `envoy.extensions.filters.network.generic_proxy.v3.GenericProxy` (routes: `.v3.RouteConfiguration`; route action: `action.v3.RouteAction`).

## Files
- `config.h` / `config.cc` — `Factory` (the `NamedNetworkFilterConfigFactory`). Resolves codec + proxy factories (`factoriesFromProto`, `config.cc:17-27`), builds the route config provider (static or RDS via `routeConfigProviderFromProto`, `config.cc:29-50`), and the filter chain (`filtersFactoryFromProto`, `config.cc:52-90`) which enforces that exactly one terminal L7 filter is present and that it is last.
- `proxy.h` / `proxy.cc` — `Filter` (the network read filter), `FilterConfigImpl` (runtime-resolved config / stats / access logs / tracer), and `ActiveStream` (per-request state machine). Also contains `ActiveDecoderFilter`, `ActiveEncoderFilter`, and the `FilterChainFactoryCallbacksHelper` used to populate the filter chain.
- `route.h` / `route_impl.h` / `route_impl.cc` — `RouteEntryImpl`, `VirtualHostImpl`, `RouteMatcherImpl`, `NullRouteMatcherImpl`, and `RouteMatchActionFactory`. Virtual-host lookup supports exact, suffix-wildcard, and prefix-wildcard hosts; within a vhost, route selection is done via the generic matcher tree.
- `rds.h` / `rds_impl.h` — `RouteConfigProviderManagerImpl` built from `Rds::Common::RouteConfigProviderManagerImpl<GenericRds, RouteConfiguration, 1, RouteMatcherImpl, NullRouteMatcherImpl>`.
- `match.h` / `match.cc` / `match_input.h` — matcher data inputs (`ServiceMatchDataInput`, `HostMatchDataInput`, `PathMatchDataInput`, `MethodMatchDataInput`, `PropertyMatchDataInput`, `RequestMatchDataInput`) and a composite `RequestMatchInputMatcher` that bundles host/path/method/property substring matchers (`match.cc:20-85`).
- `stats.h` / `stats.cc` — `GenericFilterStats`, `GenericFilterStatsHelper`, and the `CodeOrFlags` singleton that preallocates stat names for all codes 0..999 and all response flags (`stats.cc:12-28`).
- `access_log.h` / `access_log.cc` — `FormatterContextExtension` (request/response pointer bundle), `StringValueFormatterProvider`, `GenericStatusCodeFormatterProvider`, and `createGenericProxyCommandParser`.
- `tracing.h` / `tracing.cc` — tracing helpers (`TraceContextBridge` and custom-tag plumbing used by `ActiveStream::modifySpan`).
- `filter_callbacks.h`, `codec_callbacks.h`, `route.h`, `proxy_config.h` — public interfaces: filter callback types, codec callback types, `RouteEntry` / `RouteMatcher` / `FilterConfig` interfaces, and `EncodingContext`.
- `interface/` — codec / filter / route / stream abstract interfaces shared with codec extensions.
- `codecs/`, `router/` — concrete codec implementations and the terminal router filter that sends frames upstream.

## Lifecycle (network filter level)
- `Filter::onNewConnection()` (`proxy.h:370-373`) — calls `server_codec_->onConnected()` and returns `Continue` so data keeps flowing.
- `Filter::onData()` (`proxy.cc:665-673`) — if the downstream connection already closed, returns `StopIteration`. Otherwise calls `server_codec_->decode(data, end_stream)` (half-close not supported — `end_stream` is expected false) and always returns `StopIteration` because the codec consumes the buffer.
- `Filter::onEvent()` (`proxy.h:388-398`) — on any non-Connected event, sets `downstream_connection_closed_` and calls `resetDownstreamAllStreams` with either `LocalConnectionTermination` or `ConnectionTermination` (distinguished by `Network::ConnectionEvent::LocalClose`).
- `Filter::initializeReadFilterCallbacks()` (`proxy.h:374-377`) — stores callbacks and registers the filter as a `ConnectionCallbacks` listener on the downstream connection.

## Codec -> stream fan-out
- `ServerCodecCallbacks::onDecodingSuccess(RequestHeaderFramePtr, StartTime)` (`proxy.cc:675-694`) — null check, duplicate-stream-id guard against `frame_handlers_`, then `newDownstreamRequest`.
- `ServerCodecCallbacks::onDecodingSuccess(RequestCommonFramePtr)` (`proxy.cc:696-714`) — looks up the stream by id in `frame_handlers_` and forwards to `ActiveStream::onRequestCommonFrame`. Unknown stream id -> `onDecodingFailure("unknown stream id")`.
- `onDecodingFailure` (`proxy.cc:716-721`) — increments `downstream_rq_decoding_error`, resets every active stream as `ProtocolError`, and closes the connection `FlushWrite`.
- `newDownstreamRequest` (`proxy.cc:745-758`) — constructs `ActiveStream`, appends it to `active_streams_`, invokes `initializeFilterChain`, and starts decoding via `continueDecoding`. Returning false from `initializeFilterChain` (no terminal decoder filter) resets the stream as `LocalConnectionTermination`.
- `registerFrameHandler` / `unregisterFrameHandler` (`proxy.cc:737-743`) maintain the stream-id -> `ActiveStream*` index used to dispatch continuation frames.
- `mayBeDrainClose` (`proxy.cc:780-785`) — when a drain decision exists and no active streams remain, calls `onDrainCloseAndNoActiveStreams` which defaults to `closeDownstreamConnection()` (`proxy.cc:788`). Derived proxies can override.

## ActiveStream state machine
- Constructor (`proxy.cc:47-85`) — seeds `StreamInfo` with the connection info provider, optionally overrides the start time from the codec, registers the frame handler if the request is multi-frame, increments `downstream_rq_total` / `downstream_rq_active` via `stats_helper_.onRequest()`, and (if tracing is enabled) starts a span using the sampling decision from `tracingDecision` (`proxy.cc:22-31`).
- `initializeFilterChain` (`proxy.cc:584-603`) — runs the configured `FilterChainFactory`, reverses `encoder_filters_` so the first encoder filter runs last, and verifies at least one decoder filter exists.
- `continueDecoding` (`proxy.cc:429-477`) — resolves the route once (`parent_.config_->routeEntry(MatchInput{request, stream_info, MatchAction::RouteAction})`), caches it, and if a thread-local cluster exists sets `UpstreamClusterInfo`. Then drives `processRequestHeaderFrame` and, while the filter chain has not stopped, processes any buffered `request_common_frames_`.
- `processRequestHeaderFrame` (`proxy.cc:216-253`) — iterates `decoder_filters_` from `decoder_filter_iter_header_`, respecting `HeaderFilterStatus::StopIteration`. Sets `stop_decoder_filter_chain_` when the iterator has not reached the terminal filter yet.
- `processRequestCommonFrame` (`proxy.cc:255-306`) — same iteration pattern with `CommonFilterStatus`. On completion, moves the frame into `request_stream_frames_handler_->onRequestCommonFrame(...)` (the hook a router filter registers via `setRequestFramesHandler`).
- `onRequestCommonFrame` (`proxy.cc:479-501`) — when `end_stream`, records `onLastDownstreamRxByteReceived` and unregisters the frame handler. If the decoder chain is currently stopped, queues the frame; otherwise processes it immediately.
- `onResponseHeaderFrame` / `onResponseCommonFrame` (`proxy.cc:503-540`) — invoked by the router via `DecoderFilterCallback`. Duplicate response headers cause `ProtocolError` reset. Sets `ResponseCodeDetails` to `"via_upstream"` and `ResponseCode` from the frame, then drives `continueEncoding`.
- `continueEncoding` (`proxy.cc:546-582`) — mirrors `continueDecoding` but over `encoder_filters_`, calling `processResponseHeaderFrame` / `processResponseCommonFrame`. When the chain completes, the header/common frame is handed to `sendFrameToDownstream`.
- `sendFrameToDownstream` (`proxy.cc:134-159`) — invokes `server_codec_->encode(frame, *this)`; failure resets as `ProtocolError`. Records `onFirstDownstreamTxByteSent` on the header frame and `onLastDownstreamTxByteSent` when `end_stream`, then `completeStream()`.
- `sendLocalReply` (`proxy.cc:161-214`) — enforces "at most one response header frame". If a response was already sent, resets as `ProtocolError`. Otherwise clears queued response frames, asks the server codec for a canonical response (`server_codec_->respond(status, data, request)`), applies the caller's `ResponseUpdateFunction`, and either sends directly (if a prior response existed but was buffered) or continues encoding so the encoder filter chain still runs on the local reply.
- `completeStream` (`proxy.cc:612-663`) — idempotent via `stream_reset_or_complete_`. On reset reason set, increments `downstream_rq_reset` and sets the matching `CoreResponseFlag` (via `flagFromDownstreamReasonReason`, `proxy.cc:33-43`). Defers deletion from `active_streams_`, unregisters the frame handler, calls `StreamInfo::onRequestComplete`, emits stats via `stats_helper_.onRequestComplete`, finalizes the active span, runs every access log with a `FormatterContextExtension` carrying request + response frame pointers, calls `onDestroy` on every decoder filter (and every non-dual encoder filter), and finally calls `parent_.mayBeDrainClose()`.

## Decision / logic
- Route resolution is lazy — happens in `ActiveStream::continueDecoding` (`proxy.cc:434-444`). `FilterConfigImpl::routeEntry` (`proxy.h:79-82`) casts the RDS config down to `RouteMatcher` and calls `routeEntry(MatchInput)`.
- `RouteMatcherImpl::findVirtualHost` (`route_impl.cc:190-222`) — fast path: empty host / no vhosts returns the default ("*") vhost. Exact map hit wins; otherwise longest-suffix wildcard then longest-prefix wildcard via `findWildcardVirtualHost` (`route_impl.cc:168-188`).
- `VirtualHostImpl::routeEntry` (`route_impl.cc:90-105`) — evaluates the compiled matcher tree; a match must yield a `RouteEntryImpl` action (enforced by `typeUrl` assert, `route_impl.cc:97-99`). No-match / insufficient-data are logged at debug and return `nullptr`.
- Duplicate host / wildcard handling and `routes`/catch-all conflict errors are in `RouteMatcherImpl` ctor (`route_impl.cc:107-166`).
- `RouteEntryImpl` ctor (`route_impl.cc:47-62`) reads `cluster`, `metadata`, `timeout` (default 15000 ms via `DEFAULT_ROUTE_TIMEOUT_MS` at `route_impl.h:60`), `retry_policy.num_retries` (default 1), and translates each `per_filter_config` entry via `createRouteSpecificFilterConfig` (`route_impl.cc:20-45`). Factory lookup is by type URL first; if `envoy.reloadable_features.no_extension_lookup_by_name` is disabled, falls back to lookup by name.
- Tracing decision: `tracingDecision` (`proxy.cc:22-31`) runs random-sampling through the runtime snapshot keyed by `tracing.random_sampling`.
- The `Filter` ctor creates the server codec eagerly (`proxy.h:364-366`) and wires itself as the `ServerCodecCallbacks`; `onDecodingFailure` is the single failure path for codec-level errors.
- Factory (`config.cc:92-157`) pins two singletons: `RouteConfigProviderManagerImpl` and the `CodeOrFlags` stat name pool (`config.cc:96-108`); `CodeOrFlags` is pinned via the third `true` argument so it persists beyond the factory lambda. A custom `ProxyFactory` returned by the codec factory may override `Filter` creation (`config.cc:150-153`).

## Configuration
- `stat_prefix` (required) — stats emitted under `generic_proxy.<stat_prefix>.` (`config.cc:137`).
- `codec_config` (required) — `TypedExtensionConfig` resolving to a `CodecFactoryConfig`; yields a `CodecFactory` and optional `ProxyFactory` (`config.cc:17-27`).
- `filters` (required) — ordered L7 filter list. Each entry is validated against the codec via `NamedFilterConfigFactory::validateCodec` (`config.cc:66-70`); exactly one filter may be terminal and it must be last (`config.cc:60-64`, `86-88`).
- Route: exactly one of `generic_rds` (must use `AGGREGATED_GRPC` / `AGGREGATED_DELTA_GRPC` if `api_config_source` is used, `config.cc:34-41`) or inline `route_config` (`config.cc:46-49`). `RouteConfiguration` supports `virtual_hosts` plus a deprecated top-level `routes` that becomes the default vhost only when no `*` vhost is declared (`route_impl.cc:154-165`).
- `tracing` — reuses the HTTP tracing proto; `provider` overrides are looked up via the singleton `TracerManager` and custom-tag contexts are fed the generic-proxy command parser (`config.cc:117-125`).
- `access_log` — each entry installed with the generic-proxy command parser so formatters can read request/response frames via `FormatterContextExtension` (`config.cc:128-135`).
- `RouteEntry` proto: `name`, `cluster`, `timeout` (default 15s), `retry_policy.num_retries` (default 1), `metadata`, `per_filter_config`.

## Stats
All emitted under `generic_proxy.<stat_prefix>.`:
- `downstream_rq_total` (counter) — incremented on every new request (`stats.cc:50-53`).
- `downstream_rq_active` (gauge, accumulate) — incremented per request, decremented on completion (`stats.cc:50-56`).
- `downstream_rq_error` (counter) — incremented when the completed response status is non-ok (`stats.cc:61-63`).
- `downstream_rq_reset` (counter) — incremented on `ActiveStream::completeStream` with a reset reason (`proxy.cc:619`, `stats.cc:44`).
- `downstream_rq_local` (counter) — incremented when a local reply was generated (`stats.cc:58-60`).
- `downstream_rq_decoding_error` (counter) — incremented by `onDecodingFailure` (`stats.cc:46-48`, `proxy.cc:718`).
- `downstream_rq_time` (histogram, ms) — request-complete duration (`stats.cc:66-70`).
- `downstream_rq_tx_time` (histogram, us) — from request start to last upstream tx byte sent (`stats.cc:73-78`).
- Dynamic per-code counters `downstream_rq_code.<code>` for every observed status 0..999 (preallocated stat names from `CodeOrFlags`, `stats.cc:17-19`; counter lookup in `onRequestComplete`, `stats.cc:83-92`).
- Dynamic per-flag counters `downstream_rq_flag.<short_string>` for each `StreamInfo::ResponseFlag` observed (`stats.cc:94-105`).
`GenericFilterStatsHelper` caches the last code and last flag counters to avoid symbol-table joins on the hot path (`stats.h:105-106`, `stats.cc:83-105`).

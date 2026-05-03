# gRPC Stats (`envoy.filters.http.grpc_stats`)

Passive observability filter that counts gRPC (and Connect) requests/responses, inspects wire frames to count protobuf messages per direction, charges per-cluster (and optionally per-service/per-method) stats through `Grpc::Context`, and optionally exposes the per-stream counts as filter state for access logs.

Proto: `envoy.extensions.filters.http.grpc_stats.v3.FilterConfig`.

## Files
- `grpc_stats_filter.h/cc` — factory `GrpcStatsFilterConfigFactory`, filter state type `GrpcStatsObject`, inner anonymous-namespace `Config`, `GrpcServiceMethodToRequestNamesMap` (symbol-table-backed allowlist), and `GrpcStatsFilter` (`PassThroughFilter`). The filter implementation lives entirely in the `.cc` inside the anonymous namespace.
- `response_frame_counter.h/cc` — `ResponseFrameCounter` extends `Grpc::FrameInspector` to also track Connect end-of-stream status.

## Lifecycle
- `decodeHeaders` (grpc_stats_filter.cc:123): classifies the request via `Grpc::Common::isGrpcRequestHeaders`, `isConnectRequestHeaders`, `isConnectStreamingRequestHeaders`. If any match, grabs `cluster_` from `decoder_callbacks_->clusterInfoSharedPtr()`. When `stats_for_all_methods_` is true, resolves service/method dynamically (with or without dot replacement via `replace_dots_in_grpc_service_name_`) and gates `do_stat_tracking_` on that result. Otherwise resolves a `RequestNames` view and sets `do_stat_tracking_ = true` whenever the path parses as a gRPC service/method; if an `allowlist_` is configured the per-stream `request_names_` is populated only for allowed entries (other entries still count but without the service.method prefix).
- `decodeData` (grpc_stats_filter.cc:173): for gRPC, inspects new bytes with `request_counter_.inspect(data)` and, for each new full frame, calls `maybeWriteFilterState()` and `context_.chargeRequestMessageStat(...)`. Connect streaming requests charge per frame as well. Connect unary requests count as exactly one message on `end_stream`.
- `encodeHeaders` (grpc_stats_filter.cc:196): captures `grpc_response_` and Connect flags. For Connect unary charges success based on `:status == "200"`. For plain responses (not Connect streaming) charges via `headers.GrpcStatus()`. On `end_stream` may charge upstream latency.
- `encodeData` (grpc_stats_filter.cc:215): mirrors `decodeData` but uses `response_counter_` and `chargeResponseMessageStat`. Connect streaming final frame additionally charges `chargeStat` using `response_counter_.connectSuccess()` plus upstream latency.
- `encodeTrailers` (grpc_stats_filter.cc:244): gRPC path only — charges `trailers.GrpcStatus()` and upstream latency.
- Filter state (`GrpcStatsObject`, grpc_stats_filter.h:16) is attached to the stream on first write via `setData("envoy.filters.http.grpc_stats", ..., Mutable, FilterChain)` and kept in sync through `maybeWriteFilterState` (grpc_stats_filter.cc:255).

## Decision / logic
- Tri-mode cluster/method binding (grpc_stats_filter.cc:130-167):
  - `stats_for_all_methods` true — dynamic symbolization of service/method for every request (memory-unbounded).
  - allowlist present — pre-symbolized names from `individual_method_stats_allowlist`; unlisted methods still tracked without service.method in the name.
  - neither — counters attributed to cluster only.
- `maybeChargeUpstreamStat` (grpc_stats_filter.cc:276) requires `enable_upstream_stats_` and both `lastUpstreamTxByteSent` / `lastUpstreamRxByteReceived` timings; otherwise skipped.
- Filter state writes are skipped entirely when `emit_filter_state_` is false.

## Configuration
- `stats_for_all_methods` — BoolValue default false.
- `individual_method_stats_allowlist` — explicit `GrpcMethodList`.
- `emit_filter_state` — expose `GrpcStatsObject` as filter state / for access logs.
- `enable_upstream_stats` — charge upstream latency per request.
- `replace_dots_in_grpc_service_name` — replaces `.` with `_` in dynamically resolved service names.
- No per-route config.

## Stats
Counters and gauges are emitted through `Grpc::Context`, not owned by this filter struct. Per-cluster (and optionally per-service/per-method) names:
- `chargeStat` — success/failure counters keyed on gRPC status.
- `chargeRequestMessageStat` — request message count.
- `chargeResponseMessageStat` — response message count.
- `chargeUpstreamStat` — upstream latency histogram (when `enable_upstream_stats` is set).

Filter state:
- `envoy.filters.http.grpc_stats` (`GrpcStatsObject`) — `request_message_count`, `response_message_count`; serializes to proto `FilterObject` or to `"<req>,<resp>"` string.

## Factory
- `REGISTER_FACTORY(GrpcStatsFilterConfigFactory, NamedHttpFilterConfigFactory)` (grpc_stats_filter.cc:324). Name `envoy.filters.http.grpc_stats`.

# HTTP Connection Manager (`envoy.filters.network.http_connection_manager`)

The HCM is Envoy's terminal L4 network filter for HTTP/1.1, HTTP/2, and HTTP/3 traffic. This directory contains only the factory / configuration wiring: it parses the `HttpConnectionManager` proto into an `HttpConnectionManagerConfig` that implements both `Http::ConnectionManagerConfig` (runtime knobs consumed by the core `Http::ConnectionManagerImpl`) and `Http::FilterChainFactory` (source of the HTTP filter chain for each stream). The actual per-connection state machine, codec dispatch, stream lifecycle, and router invocation live in `source/common/http/conn_manager_impl.{h,cc}`; this extension constructs and installs that read filter.

Proto: `envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager` (and the Envoy Mobile variant `EnvoyMobileHttpConnectionManager`).

## Files
- `config.h` / `config.cc` — factory (`HttpConnectionManagerFilterConfigFactory`, `MobileHttpConnectionManagerFilterConfigFactory`), per-listener `HttpConnectionManagerConfig`, API-listener factory (`HttpConnectionManagerFactory`), and shared singleton helpers (`Utility::createSingletons`, `Utility::createConfig`). Also defines `InternalAddressConfig` (CIDR / pipe predicate used for XFF / internal routing decisions).
- `forward_client_cert_details.h` / `forward_client_cert_details.cc` — matcher-driven client-cert-forwarding overrides: `ForwardClientCertAction` (matcher action that returns a `Http::ForwardClientCertActionConfig`), `ForwardClientCertActionFactory` (matcher action factory), `createForwardClientCertMatcher()`, and the proto-to-enum converters (`convertForwardClientCertDetailsType`, `convertForwardClientCertFormat`, `convertSetCurrentClientCertDetails`).

## Lifecycle
HCM does not implement `onNewConnection` / `onData` directly in this directory; it returns an `Http::ConnectionManagerImpl` (implemented in `source/common/http/conn_manager_impl.*`) as the read filter. The factory step is:

1. `createFilterFactoryFromProtoTyped` (`config.cc:214-216`) delegates to `createFilterFactoryFromProtoAndHopByHop` (`config.cc:218-240`).
2. `Utility::createSingletons` (`config.cc:182-203`) gets-or-creates four process-wide singletons: the TLS-cached date provider, `RouteConfigProviderManagerImpl` (for static + RDS), the optional scoped-routes config provider manager (if an `envoy.srds_factory.default` is registered), and the HTTP filter's `DownstreamFilterConfigProviderManager`. `Tracing::TracerManagerImpl::singleton` provides the tracer manager.
3. `Utility::createConfig` (`config.cc:205-212`) instantiates `HttpConnectionManagerConfig` through its ctor (`config.cc:257-542`).
4. The returned factory lambda (`config.cc:230-239`) per-connection constructs `Http::ConnectionManagerImpl` wired to the listener's `OverloadManager` (or `nullOverloadManager()` when the listener opts out via `listenerInfo().shouldBypassOverloadManager()`), and adds it as a read filter. `ConnectionManagerImpl::onNewConnection()` / `onData()` then drive codec creation and stream dispatch.
5. The Mobile variant (`config.cc:242-244`) calls the same path with `clear_hop_by_hop_headers=false` so hop-by-hop headers are preserved on Envoy Mobile.
6. `HttpConnectionManagerFactory::createHttpConnectionManagerFactoryFromProto` (`config.cc:636-672`) produces an `ApiListener` variant: same config construction, but the returned lambda manually wires `initializeReadFilterCallbacks` and force-calls `createCodec` with a dummy buffer because `onData` is never invoked in the API-listener environment.

## Decision / logic
- Codec selection. `HttpConnectionManagerConfig::createCodec` (`config.cc:544-565`) dispatches on the configured `CodecType`:
  - `HTTP1` -> `Http::Http1::ServerConnectionImpl`.
  - `HTTP2` -> `Http::Http2::ServerConnectionImpl`.
  - `HTTP3` -> via `QuicHttpServerConnectionFactory` registry (requires `ENVOY_ENABLE_QUIC`).
  - `AUTO` -> `Http::ConnectionManagerUtility::autoCreateCodec` which inspects the initial bytes to pick HTTP/1 vs HTTP/2.
  Codec-type validation at config time: HTTP/3 is rejected on non-QUIC listeners and non-HTTP/3 codecs are rejected on QUIC listeners (`config.cc:503-515`). `CodecType` enum: `config.h:196`.
- Filter chain construction. The HTTP filter chain is built by `Http::FilterChainHelper::processFilters` with the proto's `http_filters` list (`config.cc:518-520`); per-upgrade filter overlays are parsed from `upgrade_configs` (`config.cc:522-541`) and stored in `upgrade_filter_factories_` keyed case-insensitively (`config.cc:56-72`).
  - Per-stream creation happens in `HttpConnectionManagerConfig::createFilterChain` (`config.cc:567-570`) which calls `FilterChainUtility::createFilterChainForFactories`.
  - `createUpgradeFilterChain` (`config.cc:572-599`) first checks the per-route upgrade map (explicit false short-circuits to `return false`, `config.cc:578-580`), then the HCM-level upgrade map. If upgrade-type filters are configured, they replace the default chain (`config.cc:592-595`).
- Route configuration. `route_specifier_case()` switch (`config.cc:399-420`) chooses RDS (`createRdsRouteConfigProvider`, requires `config_source` or an xdstp-scheme `route_config_name`, validated by `validateRds` at `config.cc:165-172`), static inline `route_config`, or scoped routes (SRDS, requires the `envoy.srds_factory.default` factory to be compiled in).
- Path-escaping behavior. `getPathWithEscapedSlashesAction` (`config.cc:98-109`) gates the configured action on the runtime fractional percent `http_connection_manager.path_with_escaped_slashes_action_enabled`; when `IMPLEMENTATION_SPECIFIC_DEFAULT`, it falls through to `getPathWithEscapedSlashesActionRuntimeOverride` (`config.cc:82-96`) which maps integer runtime keys 0/1/2/3/4 onto `KEEP_UNCHANGED` / `REJECT_REQUEST` / `UNESCAPE_AND_REDIRECT` / `UNESCAPE_AND_FORWARD`.
- Header validator. `createHeaderValidatorFactory` (`config.cc:111-161`) short-circuits to `nullptr` when `envoy.reloadable_features.enable_universal_header_validator` is off (legacy path). When UHV is on and no `typed_header_validation_config` is set, it synthesises a default `HeaderValidatorConfig` preserving legacy defaults (`config.cc:120-134`). Without `ENVOY_ENABLE_UHV`, setting `typed_header_validation_config` errors (`config.cc:149-158`).
- Original-IP detection. If `original_ip_detection_extensions` is empty, a default `XffConfig` with the configured `xff_num_trusted_hops` is synthesised (`config.cc:343-349`). Mixing extensions with `use_remote_address` or non-zero `xff_num_trusted_hops` fails (`config.cc:350-360`).
- Internal address classification. `InternalAddressConfig::isInternalAddress` (`config.h:84-95`) returns `unix_sockets_` for pipe addresses; otherwise, if explicit CIDR ranges exist, tests membership; otherwise falls back to `Network::Utility::isInternalAddress` (RFC1918 / RFC4193).
- Tracing. `getPerFilterTracerConfig` (`config.cc:607-619`) prefers the per-HCM `tracing.provider` and falls back to the bootstrap `defaultTracingConfig().http`. When configured, the config builds a `TracingConnectionManagerConfig` with the listener's traffic direction (`config.cc:435-438`).
- Access logs. Each `access_log` entry is materialized via `AccessLog::AccessLogFactory::fromProto` (`config.cc:440-443`). `access_log_options` is mutually exclusive with the deprecated top-level `flush_access_log_on_new_request` / `access_log_flush_interval` (`config.cc:445-468`).
- Forward client cert. At config time: `forward_client_cert_details` -> `Http::ForwardClientCertType` (`forward_client_cert_details.cc:8-23`); `set_current_client_cert_details` -> vector of `Http::ClientCertDetailsType` (`forward_client_cert_details.cc:36-54`); optional `forward_client_cert_matcher` compiles via `createForwardClientCertMatcher` (`forward_client_cert_details.cc:65-70`) so matcher actions can override the default at per-request time (`ForwardClientCertActionFactory::createAction`, `forward_client_cert_details.cc:56-61`).
- Scheme / server header. Setting both `scheme_to_overwrite` and `match_upstream` logs a warn and uses the explicit overwrite (`config.cc:474-478`). Empty `server_name` falls back to `Http::DefaultServerString::get()` (`config.cc:483-487`).

## Configuration
Major knobs (see `HttpConnectionManagerConfig` accessors in `config.h:119-193` and ctor at `config.cc:257-542`):
- `codec_type`, `stat_prefix`, `http_filters`, `route_specifier` (`rds` / `route_config` / `scoped_routes`), `access_log`, `tracing`.
- Timeouts: `stream_idle_timeout` (default 5 min, `config.h:275`), `request_timeout` (default 0 / disabled), `request_headers_timeout` (default 0), `drain_timeout` (default 5s), `delayed_close_timeout` (default 1s), `common_http_protocol_options.{idle_timeout,max_connection_duration,max_stream_duration,max_headers_count,max_requests_per_connection}`. Zero-valued `idle_timeout` becomes "disabled"; unset becomes 1 hour (`config.cc:305-309`).
- Headers: `max_request_headers_kb` (proto or runtime override `http.max_request_headers_size_kb`), `server_header_transformation`, `server_name`, `via`, `scheme_header_transformation`, `headers_with_underscores_action`, `append_local_overload`, `append_x_forwarded_port`.
- Addressing: `use_remote_address`, `xff_num_trusted_hops`, `skip_xff_append`, `internal_address_config`, `original_ip_detection_extensions`, `strip_any_host_port`/`strip_matching_host_port` (mutually exclusive, `config.cc:316-327`), `strip_trailing_host_dot`.
- Protocol: `http_protocol_options`, `http2_protocol_options` (HTTP/2 `max_header_field_size_kb` must not exceed `max_request_headers_kb`, `config.cc:300-303`), `http3_protocol_options` (QUIC only), `stream_error_on_invalid_http_message`, `proxy_100_continue`.
- Path: `normalize_path` (ctor defaults from runtime `http_connection_manager.normalize_path`; build-gated default at `config.cc:273-281`), `merge_slashes`, `path_with_escaped_slashes_action`.
- Upgrade: `upgrade_configs` — per-type filter list + `enabled` flag, cannot be duplicated (case-insensitive, `config.cc:522-541`).
- Client cert: `forward_client_cert_details`, `set_current_client_cert_details`, `forward_client_cert_matcher`.
- Response headers: setting `common_http_protocol_options.max_response_headers_kb` on the HCM is rejected (`config.cc:311-314`).
- `request_id_extension` defaults to `UuidRequestIdConfig` when `typed_config` is unset (`config.cc:331-339`).
- Local reply config compiled via `LocalReply::Factory::create` (`config.cc:292-294`).

## Stats
`HttpConnectionManagerConfig` allocates three stat bundles at construction (prefix `http.<stat_prefix>.`):
- `Http::ConnectionManagerStats` via `ConnectionManagerImpl::generateStats` (`config.cc:260`).
- `Http::ConnectionManagerTracingStats` via `ConnectionManagerImpl::generateTracingStats` (`config.cc:260`).
- `Http::ConnectionManagerListenerStats` via `ConnectionManagerImpl::generateListenerStats`, scoped to `listenerScope()` (`config.cc:271`).

Codec-level stats (`Http::Http1/2/3::CodecStats`) are lazily created through `AtomicPtr` members (`config.h:216-218`) and retrieved via `CodecStats::atomicGet(...)` in `createCodec` (`config.cc:547-562`) and in UHV `getHeaderValidatorStats` (`config.cc:622-633`). The exact counter / gauge / histogram names are defined in `source/common/http/conn_manager_impl.*` and codec-specific stat files; this directory only plumbs prefixes and scopes.

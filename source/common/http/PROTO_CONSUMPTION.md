# How HTTP Proto Gets Consumed in Envoy

This document explains how the `HttpConnectionManager` protobuf is deserialized and translated into runtime objects that handle HTTP traffic. The proto entry point is `envoy::extensions::filters::network::http_connection_manager::v3::HttpConnectionManager`.

---

## 1. The Proto Shape

```
HttpConnectionManager (HCM)                       (http_connection_manager.proto)
  ├── codec_type                                   AUTO | HTTP1 | HTTP2 | HTTP3
  ├── stat_prefix
  ├── route_specifier (oneof)
  │     ├── rds            → RdsRouteConfig { route_config_name, config_source }
  │     ├── route_config   → RouteConfiguration (static, inline)
  │     └── scoped_routes  → ScopedRoutes
  ├── http_filters[]                               TypedExtensionConfig per filter
  ├── http_protocol_options                        Http1ProtocolOptions
  ├── http2_protocol_options                       Http2ProtocolOptions
  ├── http3_protocol_options                       Http3ProtocolOptions
  ├── common_http_protocol_options
  │     ├── idle_timeout, max_connection_duration
  │     ├── max_headers_count, max_requests_per_connection
  │     └── headers_with_underscores_action
  ├── tracing                                      Tracing config
  ├── access_log[]                                 AccessLog entries
  ├── access_log_options
  ├── request_id_extension                         TypedExtensionConfig (default: UUID)
  ├── original_ip_detection_extensions[]           TypedExtensionConfig (default: XFF)
  ├── early_header_mutation_extensions[]
  ├── upgrade_configs[]                            WebSocket, CONNECT tunneling
  ├── forward_client_cert_details
  ├── set_current_client_cert_details
  ├── proxy_status_config
  ├── local_reply_config
  ├── server_header_transformation
  ├── scheme_header_transformation
  └── <scalar timeouts>: stream_idle_timeout, request_timeout, drain_timeout, ...
```

---

## 2. Delivery Path: FilterChain → HCM Factory

The HCM is a **network filter** — it lives inside a listener's filter chain. When a listener's filter chain proto is processed, the network filter factory for HCM is invoked:

```cpp
// In FilterChainManagerImpl, for each filters[].typed_config:
// type_url = "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"

auto* factory = Registry::FactoryRegistry<
    Server::Configuration::NamedNetworkFilterConfigFactory>::getFactoryByType(type_url);

ProtobufTypes::MessagePtr message = factory->createEmptyConfigProto();
// → creates empty HttpConnectionManager proto

MessageUtil::unpackTo(filter_proto.typed_config(), *message);
// → fills the HCM proto from the Any field

Network::FilterFactoryCb cb = factory->createFilterFactoryFromProto(*message, context);
// → calls HttpConnectionManagerFilterConfigFactory::createFilterFactoryFromProtoTyped()
```

The factory then creates a `HttpConnectionManagerConfig` — the runtime representation of the entire HCM proto.

---

## 3. `HttpConnectionManagerConfig` Constructor: Proto Field-by-Field

The constructor signature:

```cpp
HttpConnectionManagerConfig::HttpConnectionManagerConfig(
    const envoy::extensions::filters::network::http_connection_manager::v3::HttpConnectionManager& config,
    Server::Configuration::FactoryContext& context,
    Http::DateProvider& date_provider,
    Router::RouteConfigProviderManager& route_config_provider_manager,
    Config::ConfigProviderManager* scoped_routes_config_provider_manager,
    Tracing::TracerManager& tracer_manager,
    FilterConfigProviderManager& filter_config_provider_manager,
    absl::Status& creation_status)
```

### 3.1 Scalar Fields

```cpp
stats_prefix_(fmt::format("http.{}.", config.stat_prefix()))
xff_num_trusted_hops_(config.xff_num_trusted_hops())
skip_xff_append_(config.skip_xff_append())
via_(config.via())
proxy_100_continue_(config.proxy_100_continue())
preserve_external_request_id_(config.preserve_external_request_id())
always_set_request_id_in_response_(config.always_set_request_id_in_response())
merge_slashes_(config.merge_slashes())
strip_trailing_host_dot_(config.strip_trailing_host_dot())
append_local_overload_(config.append_local_overload())
append_x_forwarded_port_(config.append_x_forwarded_port())
server_transformation_(config.server_header_transformation())
```

### 3.2 Boolean Wrapper Fields (`PROTOBUF_GET_WRAPPED_OR_DEFAULT`)

```cpp
use_remote_address_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, use_remote_address, false))

generate_request_id_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, generate_request_id, true))

stream_error_on_invalid_http_messaging_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, stream_error_on_invalid_http_message, false))

normalize_path_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, normalize_path,
        runtime.snapshot().featureEnabled("http_connection_manager.normalize_path", 100)))

add_proxy_protocol_connection_state_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, add_proxy_protocol_connection_state, true))

max_requests_per_connection_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config.common_http_protocol_options(),
                                    max_requests_per_connection, 0))

max_request_headers_kb_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, max_request_headers_kb,
        runtime.snapshot().getInteger(MaxRequestHeadersSizeOverrideKey,
                                       DEFAULT_MAX_REQUEST_HEADERS_KB)))
```

### 3.3 Duration Fields

```cpp
idle_timeout_(
    PROTOBUF_GET_OPTIONAL_MS(config.common_http_protocol_options(), idle_timeout))
// → absl::optional<std::chrono::milliseconds>

max_connection_duration_(
    PROTOBUF_GET_OPTIONAL_MS(config.common_http_protocol_options(), max_connection_duration))

max_stream_duration_(
    PROTOBUF_GET_OPTIONAL_MS(config.common_http_protocol_options(), max_stream_duration))

stream_idle_timeout_(
    PROTOBUF_GET_MS_OR_DEFAULT(config, stream_idle_timeout, StreamIdleTimeoutMs))

stream_flush_timeout_(
    PROTOBUF_GET_MS_OR_DEFAULT(config, stream_flush_timeout, stream_idle_timeout_.count()))

request_timeout_(
    PROTOBUF_GET_MS_OR_DEFAULT(config, request_timeout, RequestTimeoutMs))

drain_timeout_(
    PROTOBUF_GET_MS_OR_DEFAULT(config, drain_timeout, 5000))

delayed_close_timeout_(
    PROTOBUF_GET_MS_OR_DEFAULT(config, delayed_close_timeout, 1000))
```

### 3.4 Codec Type Selection

```cpp
switch (config.codec_type()) {
case HttpConnectionManager::AUTO:   codec_type_ = CodecType::AUTO;  break;
case HttpConnectionManager::HTTP1:  codec_type_ = CodecType::HTTP1; break;
case HttpConnectionManager::HTTP2:  codec_type_ = CodecType::HTTP2; break;
case HttpConnectionManager::HTTP3:  codec_type_ = CodecType::HTTP3; break;
}
```

At connection time, `createCodec()` uses `codec_type_` to instantiate the right codec:

```cpp
Http::ServerConnectionPtr HttpConnectionManagerConfig::createCodec(...) {
    switch (codec_type_) {
    case CodecType::HTTP1:
        return make_unique<Http::Http1::ServerConnectionImpl>(
            connection, stats, callbacks, http1_settings_, ...);
    case CodecType::HTTP2:
        return make_unique<Http::Http2::ServerConnectionImpl>(
            connection, callbacks, stats, http2_options_, ...);
    case CodecType::AUTO:
        return Http::ConnectionManagerUtility::autoCreateCodec(...);
    }
}
```

### 3.5 HTTP/1 Settings

```cpp
http1_settings_(Http::Http1::parseHttp1Settings(
    config.http_protocol_options(),
    context.serverFactoryContext(),
    context.messageValidationVisitor(),
    config.stream_error_on_invalid_http_message(),
    xff_num_trusted_hops_ == 0 && use_remote_address_))
```

`parseHttp1Settings` reads `http_protocol_options` fields like `allow_absolute_url`, `accept_http_10`, `default_host_for_http_10`, `header_key_format`, etc.

### 3.6 HTTP/2 Settings

```cpp
auto options_or_error = Http2::Utility::initializeAndValidateOptions(
    config.http2_protocol_options(),
    config.has_stream_error_on_invalid_http_message(),
    config.stream_error_on_invalid_http_message());
SET_AND_RETURN_IF_NOT_OK(options_or_error.status(), creation_status);
http2_options_ = options_or_error.value();
```

Reads fields like `initial_stream_window_size`, `initial_connection_window_size`, `max_concurrent_streams`, `hpack_table_size`, `allow_metadata`, etc.

---

## 4. Route Configuration: The `route_specifier` Oneof

This is one of the most important sections. The HCM must know how to route requests.

```cpp
switch (config.route_specifier_case()) {

case RouteSpecifierCase::kRds:
    // Dynamic routing — subscribe to RDS xDS stream
    route_config_provider_ = route_config_provider_manager.createRdsRouteConfigProvider(
        config.rds(),           // RdsRouteConfig { route_config_name, config_source }
        context_.serverFactoryContext(),
        stats_prefix_,
        context_.initManager());
    break;

case RouteSpecifierCase::kRouteConfig:
    // Static routing — full RouteConfiguration embedded in the proto
    route_config_provider_ = route_config_provider_manager.createStaticRouteConfigProvider(
        config.route_config(),  // envoy::config::route::v3::RouteConfiguration
        context_.serverFactoryContext(),
        context_.messageValidationVisitor());
    break;

case RouteSpecifierCase::kScopedRoutes:
    // Scoped routing — e.g. per-host routing tables
    scoped_routes_config_provider_ = srds_factory->createConfigProvider(
        config, context_.serverFactoryContext(), context_.initManager(),
        stats_prefix_, *scoped_routes_config_provider_manager_);
    scope_key_builder_ = srds_factory->createScopeKeyBuilder(config);
    break;
}
```

For `kRds`, the `config.rds().route_config_name` is the name of the route table to subscribe to, and `config.rds().config_source` is the xDS server config (cluster name, etc.).

For `kRouteConfig`, the entire `RouteConfiguration` proto (virtual hosts, routes, clusters, retry policies) is processed inline.

---

## 5. HTTP Filters: The `http_filters[]` List

This processes the ordered chain of HTTP filters:

```cpp
Http::FilterChainHelper<
    Server::Configuration::FactoryContext,
    Server::Configuration::NamedHttpFilterConfigFactory>
    helper(filter_config_provider_manager_, context_, ...);

helper.processFilters(config.http_filters(), "http", "http", filter_factories_);
```

For each entry in `config.http_filters()`:

```
http_filters[i]
  ├── name       (e.g. "envoy.filters.http.router")
  ├── typed_config  (google.protobuf.Any)
  │     └── e.g. envoy.extensions.filters.http.router.v3.Router
  └── disabled   (optional: disable by default for dynamic override)
```

The factory lookup:

```cpp
auto* factory = Envoy::Config::Utility::getFactory<
    Server::Configuration::NamedHttpFilterConfigFactory>(filter_proto);
// Factory identified by typed_config.type_url()

ProtobufTypes::MessagePtr message = factory->createEmptyConfigProto();
MessageUtil::unpackTo(filter_proto.typed_config(), *message);
// Unpack the Any into the typed config

Http::FilterFactoryCb cb = factory->createFilterFactoryFromProto(*message, stats_prefix_, context_);
filter_factories_.push_back(cb);
```

At request time, `createFilterChain()` calls all the factory callbacks to instantiate the actual filter objects.

### 5.1 Upgrade Filters (WebSocket / CONNECT)

```cpp
for (const auto& upgrade_config : config.upgrade_configs()) {
    const std::string& name = upgrade_config.upgrade_type();  // "websocket", "CONNECT"
    bool enabled = upgrade_config.has_enabled() ? upgrade_config.enabled().value() : true;

    if (!upgrade_config.filters().empty()) {
        // Process upgrade-specific filter chain (same pattern as http_filters)
        helper.processFilters(upgrade_config.filters(), name, "http upgrade", *factories);
    }
    upgrade_filter_factories_.emplace(name, FilterConfig{std::move(factories), enabled});
}
```

---

## 6. Request ID Extension

```cpp
envoy::extensions::filters::network::http_connection_manager::v3::RequestIDExtension
    final_rid_config = config.request_id_extension();

if (!final_rid_config.has_typed_config()) {
    // Default: inject UUID extension
    final_rid_config.mutable_typed_config()->PackFrom(
        envoy::extensions::request_id::uuid::v3::UuidRequestIdConfig());
}

auto extension_or_error = Http::RequestIDExtensionFactory::fromProto(final_rid_config, context_);
request_id_extension_ = extension_or_error.value();
```

---

## 7. Original IP Detection Extensions

```cpp
auto ip_detection_extensions = config.original_ip_detection_extensions();

if (ip_detection_extensions.empty()) {
    // Default: XFF (X-Forwarded-For)
    envoy::extensions::http::original_ip_detection::xff::v3::XffConfig xff_config;
    xff_config.set_xff_num_trusted_hops(xff_num_trusted_hops_);
    // Pack into extension config and add
}

for (const auto& extension_config : ip_detection_extensions) {
    auto* factory = Config::Utility::getFactory<Http::OriginalIPDetectionFactory>(extension_config);
    auto extension = factory->createExtension(extension_config.typed_config(), context_);
    original_ip_detection_extensions_.push_back(*extension);
}
```

---

## 8. Tracing Configuration

```cpp
if (config.has_tracing()) {
    tracer_ = tracer_manager.getOrCreateTracer(getPerFilterTracerConfig(config));
    // getPerFilterTracerConfig reads config.tracing().provider().typed_config()
    // to find the tracer implementation (zipkin, jaeger, opentelemetry, etc.)

    tracing_config_ = make_unique<Http::TracingConnectionManagerConfig>(
        context.listenerInfo().direction(),   // INBOUND | OUTBOUND
        config.tracing());
    // Reads: operation_name, request_headers_for_tags, verbose,
    //        max_path_tag_length, custom_tags[], overall_sampling, ...
}
```

---

## 9. Access Log

```cpp
for (const auto& access_log : config.access_log()) {
    AccessLog::InstanceSharedPtr log =
        AccessLog::AccessLogFactory::fromProto(access_log, context_);
    access_logs_.push_back(log);
}
```

Same `fromProto` / `typed_config` pattern used in the listener layer. Each entry has a `typed_config` pointing to file, gRPC, or custom access log implementations.

```cpp
// access_log_options handling
if (config.has_access_log_options()) {
    flush_access_log_on_new_request_ =
        config.access_log_options().flush_access_log_on_new_request();
    if (config.access_log_options().has_access_log_flush_interval()) {
        access_log_flush_interval_ = std::chrono::milliseconds(
            DurationUtil::durationToMilliseconds(
                config.access_log_options().access_log_flush_interval()));
    }
}
```

---

## 10. Proxy Status Config

```cpp
proxy_status_config_(
    config.has_proxy_status_config()
        ? make_unique<HttpConnectionManagerProto::ProxyStatusConfig>(config.proxy_status_config())
        : nullptr)
// Copies the ProxyStatusConfig sub-message into a unique_ptr
```

---

## 11. Forward Client Cert

```cpp
forward_client_cert_ =
    convertForwardClientCertDetailsType(config.forward_client_cert_details());
// Converts enum: SANITIZE | FORWARD_ONLY | APPEND_FORWARD | SANITIZE_SET | ALWAYS_FORWARD_ONLY

set_current_client_cert_details_ =
    convertSetCurrentClientCertDetails(config.set_current_client_cert_details());
// Reads: subject, cert, chain, dns, uri boolean fields
```

---

## 12. Local Reply Configuration

```cpp
auto local_reply_or_error = LocalReply::Factory::create(config.local_reply_config(), context);
SET_AND_RETURN_IF_NOT_OK(local_reply_or_error.status(), creation_status);
local_reply_ = std::move(*local_reply_or_error);
```

`local_reply_config` contains mappers that rewrite error responses (status code, body, headers) based on match conditions.

---

## 13. Header Validator Factory

```cpp
header_validator_factory_(createHeaderValidatorFactory(config, context, creation_status))
```

Reads `config.http_protocol_options()` and the UHV (Universal Header Validator) feature flags to produce a factory that validates request/response headers per protocol.

---

## 14. The `conn_manager_impl.cc` Runtime Phase

After proto is consumed at startup, `ConnectionManagerImpl` runs at request time without touching proto. The only proto-adjacent code at runtime is reading enum values already stored:

```cpp
// From conn_manager_impl.cc, at request time:
if (connection_manager_.direction_ ==
        envoy::config::core::v3::TrafficDirection::INBOUND) {
    // inbound request handling
}
```

All other decisions use the C++ structs populated during `HttpConnectionManagerConfig` construction.

---

## 15. End-to-End Flow Diagram

```
FilterChain proto (filters[].typed_config = HCM Any)
      │
      │  MessageUtil::unpackTo() → HttpConnectionManager proto
      ▼
HttpConnectionManagerFilterConfigFactory::createFilterFactoryFromProtoTyped()
      │
      ▼
Utility::createConfig() → HttpConnectionManagerConfig constructor
      │
      ├── codec_type        → CodecType::AUTO | HTTP1 | HTTP2 | HTTP3
      ├── stat_prefix       → stats_ scope
      ├── PROTOBUF_GET_*    → all scalar / duration / wrapper fields
      ├── http1_settings    ← Http1::parseHttp1Settings(http_protocol_options)
      ├── http2_options     ← Http2::Utility::initializeAndValidateOptions(http2_protocol_options)
      ├── route_specifier   → kRds    → RdsRouteConfigProvider (subscribes to xDS)
      │                     → kRoute → StaticRouteConfigProvider (inline)
      │                     → kScoped → ScopedRoutes (per-host tables)
      ├── http_filters[]    → FilterChainHelper::processFilters()
      │       └── typed_config → factory registry → FilterFactoryCb list
      ├── upgrade_configs[] → WebSocket/CONNECT filter chains
      ├── request_id_extension → UUID or custom
      ├── original_ip_detection → XFF or custom extensions
      ├── tracing           → TracerManager::getOrCreateTracer()
      ├── access_log[]      → AccessLogFactory::fromProto()
      ├── local_reply_config → LocalReply::Factory::create()
      └── proxy_status_config → copied as unique_ptr sub-message
      │
      ▼ (at connection accept time)
ConnectionManagerImpl created with shared_ptr<HttpConnectionManagerConfig>
      │
      ├── createCodec()       → Http1/Http2/Http3 ServerConnectionImpl
      ├── createFilterChain() → instantiate filter_factories_ callbacks
      └── route()             → route_config_provider_->config()->route(headers)
```

---

## 16. Key Macros and Patterns

| Pattern | Description |
|---|---|
| `PROTOBUF_GET_WRAPPED_OR_DEFAULT(msg, field, default)` | Read `BoolValue`/`UInt32Value` wrapper with fallback |
| `PROTOBUF_GET_MS_OR_DEFAULT(msg, field, default_ms)` | Read `Duration` in milliseconds with fallback |
| `PROTOBUF_GET_OPTIONAL_MS(msg, field)` | Returns `absl::optional<chrono::milliseconds>` |
| `MessageUtil::unpackTo(any, typed_msg)` | Deserialize `google.protobuf.Any` → typed message |
| `Config::Utility::getFactory<T>(typed_ext_config)` | Factory registry lookup by `type_url` |
| `factory->createEmptyConfigProto()` | Create empty typed proto for unpacking |
| `factory->createFilterFactoryFromProto(msg, ctx)` | Convert proto → `FilterFactoryCb` |
| `SET_AND_RETURN_IF_NOT_OK(status, out)` | Propagate `absl::Status` errors |
| `DurationUtil::durationToMilliseconds(duration_proto)` | Convert `google.protobuf.Duration` → ms |
| `PackFrom(msg)` → `Any` | Pack a typed proto into `google.protobuf.Any` |

# How Listener Proto Gets Consumed in Envoy

This document explains the full lifecycle of a `Listener` protobuf message — from the moment it arrives over xDS (or static bootstrap) to when the listener is actively accepting connections. Every field access, every deserialization macro, and every factory call is covered.

---

## 1. The Proto Shape

The root message is:

```
envoy::config::listener::v3::Listener          (listener.proto)
  ├── address                                   (core.proto Address)
  ├── additional_addresses[]
  ├── filter_chains[]                           (listener_components.proto FilterChain)
  │     ├── filter_chain_match
  │     ├── filters[]                           (NetworkFilter)
  │     └── transport_socket
  ├── default_filter_chain
  ├── listener_filters[]
  ├── filter_chain_matcher                      (xDS matcher API)
  ├── udp_listener_config
  ├── access_log[]
  ├── socket_options[]
  ├── connection_balance_config
  └── <scalar fields>: name, stat_prefix, bind_to_port, drain_type, ...
```

---

## 2. Delivery Path: LDS → ListenerManager

### 2.1 LDS Subscription (`lds_api.cc`)

`LdsApiImpl` subscribes to `envoy::config::listener::v3::Listener` resources from the xDS server. When resources arrive, `onConfigUpdate()` is called:

```cpp
// lds_api.cc
LdsApiImpl::LdsApiImpl(...)
    : Envoy::Config::SubscriptionBase<envoy::config::listener::v3::Listener>(validation_visitor, "name")
```

Inside `onConfigUpdate`, each decoded resource is cast to the typed proto and forwarded to the `ListenerManager`:

```cpp
absl::Status LdsApiImpl::onConfigUpdate(
    const std::vector<Config::DecodedResourceRef>& added_resources, ...) {

  for (const auto& resource : added_resources) {
    const auto& listener = dynamic_cast<const envoy::config::listener::v3::Listener&>(
        resource.get().resource());
    // Pause RDS/SDS while applying listener changes
    listener_manager_.addOrUpdateListener(listener, resource.get().version(), true);
  }
}
```

Key point: **the raw `Listener` proto is passed by `const&` all the way down**. No copying until `ListenerImpl` stores it.

---

## 3. ListenerManagerImpl: Add/Update Routing

`ListenerManagerImpl::addOrUpdateListener()` checks whether this is a new listener or an update:

- If new → calls `ListenerImpl::create(config, ...)`
- If update and only filter chains changed → calls `listener->newListenerWithFilterChain(config, ...)`
- If update with structural change → full `ListenerImpl::create(config, ...)`

The decision logic uses `ListenerMessageUtil::filterChainOnlyChange()` which compares the serialized proto bytes of everything except filter chains.

---

## 4. ListenerImpl Construction: Proto Field-by-Field

`ListenerImpl` is the central class that consumes the `Listener` proto. Its constructor accepts:

```cpp
ListenerImpl::ListenerImpl(
    const envoy::config::listener::v3::Listener& config,
    const std::string& version_info,
    ListenerManagerImpl& parent,
    const std::string& name,
    bool added_via_api,
    bool workers_started,
    uint64_t hash,
    absl::Status& creation_status)
```

### 4.1 Socket Type

```cpp
socket_type_(config.has_internal_listener()
    ? Network::Socket::Type::Stream
    : Network::Utility::protobufAddressSocketType(config.address()))
```

If the listener has an `internal_listener` field, it is always TCP stream. Otherwise the address proto is inspected to infer UDP or TCP.

### 4.2 Boolean and Numeric Fields via `PROTOBUF_GET_WRAPPED_OR_DEFAULT`

Many listener fields are Google `BoolValue` / `UInt32Value` wrappers (nullable scalars). The macro handles the `has_X()` check and fallback:

```cpp
bind_to_port_(shouldBindToPort(config))
// expands to:
//   PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, bind_to_port, true)

per_connection_buffer_limit_bytes_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, per_connection_buffer_limit_bytes, 1024 * 1024))

tcp_backlog_size_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, tcp_backlog_size, ENVOY_TCP_BACKLOG_SIZE))

max_connections_to_accept_per_socket_event_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, max_connections_to_accept_per_socket_event,
                                    Network::DefaultMaxConnectionsToAcceptPerSocketEvent))

hand_off_restored_destination_connections_(
    PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, use_original_dst, false))

ignore_global_conn_limit_(config.ignore_global_conn_limit())
bypass_overload_manager_(config.bypass_overload_manager())
continue_on_listener_filters_timeout_(config.continue_on_listener_filters_timeout())
```

### 4.3 Duration Fields via `PROTOBUF_GET_MS_OR_DEFAULT`

Duration fields come as `google.protobuf.Duration` and are read in milliseconds:

```cpp
listener_filters_timeout_(
    PROTOBUF_GET_MS_OR_DEFAULT(config, listener_filters_timeout, 15000))

per_connection_buffer_high_watermark_timeout_(
    std::chrono::milliseconds(
        PROTOBUF_GET_MS_OR_DEFAULT(config, per_connection_buffer_high_watermark_timeout, 0)))
```

### 4.4 Drain Type

```cpp
parent_.factory_->createDrainManager(config.drain_type())
// drain_type is an enum: DEFAULT, MODIFY_ONLY
```

### 4.5 Address Parsing

The `address` field is converted from proto to a runtime `Network::Address::Instance`:

```cpp
auto address_or_error = Network::Address::resolveProtoAddress(config.address());
// returns absl::StatusOr<Network::Address::InstanceConstSharedPtr>
```

Additional addresses are iterated:

```cpp
for (auto i = 0; i < config.additional_addresses_size(); i++) {
    auto address_or_error = Network::Address::resolveProtoAddress(
        config.additional_addresses(i).address());
    // per-address socket options and TCP keepalive extracted similarly
}
```

### 4.6 TCP Keepalive

```cpp
if (config.has_tcp_keepalive()) {
    Network::parseTcpKeepaliveConfig(config.tcp_keepalive())
    // returns Network::Socket::OptionsSharedPtr
}
```

### 4.7 Socket Options

Raw kernel socket options from proto are converted to `Network::SocketOption` objects:

```cpp
Network::SocketOptionFactory::buildLiteralOptions(config.socket_options())
// config.socket_options() → repeated envoy.config.core.v3.SocketOption
// each has: level, name, int_value / buf_value, state (PREBIND/BOUND/LISTENING)
```

---

## 5. Build Methods: Proto → Runtime Sub-objects

The constructor delegates to a series of `build*` methods, each responsible for one proto section:

### 5.1 `buildAccessLog(config)`

```cpp
void ListenerImpl::buildAccessLog(const envoy::config::listener::v3::Listener& config) {
    for (const auto& access_log : config.access_log()) {
        AccessLog::InstanceSharedPtr log =
            AccessLog::AccessLogFactory::fromProto(access_log, *listener_factory_context_);
        access_logs_.push_back(log);
    }
}
```

Each `AccessLog.AccessLog` proto element is deserialized via the typed extension factory (`fromProto`), which reads `typed_config.type_url()` and dispatches to the right implementation.

### 5.2 `buildInternalListener(config)`

```cpp
if (config.has_internal_listener()) {
    // Validates: no address, no reuse_port, no mptcp, no socket options
    internal_listener_config_ =
        std::make_unique<InternalListenerConfigImpl>(*internal_listener_registry);
}
```

### 5.3 `buildUdpListenerFactory(config, concurrency)`

```cpp
udp_listener_config_ = std::make_shared<UdpListenerConfigImpl>(config.udp_listener_config());
// config.udp_listener_config() → envoy.config.listener.v3.UdpListenerConfig

if (config.udp_listener_config().has_quic_options()) {
    udp_listener_config_->listener_factory_ =
        std::make_unique<Quic::ActiveQuicListenerFactory>(
            config.udp_listener_config().quic_options(), concurrency, ...);
}
```

### 5.4 `createListenerFilterFactories(config)`

```cpp
if (!config.listener_filters().empty()) {
    switch (socket_type_) {
    case Network::Socket::Type::Stream:
        listener_filter_factories_ =
            parent_.factory_->createListenerFilterFactoryList(
                config.listener_filters(), *listener_factory_context_);
        break;
    case Network::Socket::Type::Datagram:
        // UDP or QUIC path
    }
}
```

Each `listener_filters[i]` is a `ListenerFilter` proto with a `typed_config`. The factory registry looks up the implementation by `type_url` and creates a `ListenerFilterFactoryCb`.

### 5.5 `validateFilterChains(config)`

Validates that:
- TCP listeners have at least one filter chain
- Connectionless UDP listeners have zero filter chains
- Connection-oriented UDP listeners each have a transport socket

### 5.6 `buildFilterChains(config)`

This is the most involved step:

```cpp
absl::Status ListenerImpl::buildFilterChains(const envoy::config::listener::v3::Listener& config) {
    ListenerFilterChainFactoryBuilder builder(*this, *transport_factory_context_);
    return filter_chain_manager_->addFilterChains(
        config.has_filter_chain_matcher() ? &config.filter_chain_matcher() : nullptr,
        config.filter_chains(),            // repeated FilterChain
        config.has_default_filter_chain() ? &config.default_filter_chain() : nullptr,
        builder,
        *filter_chain_manager_);
}
```

`FilterChainManagerImpl::addFilterChains()` iterates every `FilterChain` proto and:
1. Builds a `FilterChainMatch` → runtime matcher for SNI, source IP, destination port, etc.
2. Creates transport socket factory from `filter_chain.transport_socket.typed_config`
3. Creates network filter factory list from `filter_chain.filters[].typed_config`

The `filter_chain_matcher` field (if present) is a full xDS matcher tree that is built into a `Matcher::MatchTree<Network::MatchingData>`.

### 5.7 `buildConnectionBalancer(config, address)`

```cpp
if (config.has_connection_balance_config()) {
    switch (config.connection_balance_config().balance_type_case()) {
    case kExactBalance:
        connection_balancers_.emplace(address, make_shared<Network::ExactConnectionBalancerImpl>());
        break;
    case kExtendBalance:
        // Reads typed_config.type_url() → factory registry lookup
        auto factory = Registry::FactoryRegistry<Network::ConnectionBalanceFactory>
                           ::getFactoryByType(type_url);
        connection_balancers_.emplace(address,
            factory->createConnectionBalancerFromProto(extend_balance, *listener_factory_context_));
        break;
    }
}
```

### 5.8 `buildSocketOptions(config)`

```cpp
if (config.has_tcp_fast_open_queue_length()) {
    addListenSocketOptions(listen_socket_options_list_[i],
        Network::SocketOptionFactory::buildTcpFastOpenOptions(
            config.tcp_fast_open_queue_length().value()));
}
```

### 5.9 `buildOriginalDstListenerFilter(config)`

```cpp
if (PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, use_original_dst, false)) {
    // Prepend the OriginalDst listener filter automatically
}
```

This is a deprecated shorthand — the preferred approach is to add the listener filter explicitly.

---

## 6. The `TypedExtensionConfig` Pattern

Anywhere you see `typed_config` in the proto, Envoy uses this pattern:

```cpp
// Proto field: google.protobuf.Any typed_config
const std::string type_url = filter_proto.typed_config().type_url();
// e.g. "type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy"

auto* factory = Registry::FactoryRegistry<Server::Configuration::NamedNetworkFilterConfigFactory>
                    ::getFactoryByType(type_url);

ProtobufTypes::MessagePtr message = factory->createEmptyConfigProto();
MessageUtil::unpackTo(filter_proto.typed_config(), *message);  // deserialize Any → typed message

Network::FilterFactoryCb cb = factory->createFilterFactoryFromProto(*message, *context_);
```

This pattern is used for:
- Network filters (`filter_chains[].filters[].typed_config`)
- Listener filters (`listener_filters[].typed_config`)
- Transport sockets (`transport_socket.typed_config`)
- Connection balancers (`extend_balance.typed_config`)
- Access log implementations (`access_log[].typed_config`)

---

## 7. Proto Storage and Comparison

`ListenerImpl` stores a **copy** of the proto for hot-reload diffing:

```cpp
config_maybe_partial_filter_chains_(config)  // stored as member variable
```

Two utility functions perform proto-level comparison:

```cpp
// Compares socket options (transparent, freebind, tcp_fast_open) using field accessors
bool ListenerMessageUtil::socketOptionsEqual(
    const envoy::config::listener::v3::Listener& lhs,
    const envoy::config::listener::v3::Listener& rhs);

// Returns true if the two listeners differ only in filter chains/matcher
// Uses MessageUtil::equals() on a scratch copy with filter chains cleared
bool ListenerMessageUtil::filterChainOnlyChange(
    const envoy::config::listener::v3::Listener& lhs,
    const envoy::config::listener::v3::Listener& rhs);
```

When only filter chains changed, Envoy does an **in-place update** without restarting the listen socket — a major operational win.

---

## 8. End-to-End Flow Diagram

```
xDS Server (LDS)
      │
      │  envoy::config::listener::v3::Listener (protobuf bytes)
      ▼
LdsApiImpl::onConfigUpdate()
      │  dynamic_cast<const Listener&>(resource)
      ▼
ListenerManagerImpl::addOrUpdateListener(config)
      │
      ├── filterChainOnlyChange? ──yes──► newListenerWithFilterChain(config)
      │
      └── no ──► ListenerImpl::create(config)
                      │
                      ├── resolveProtoAddress(config.address())
                      ├── PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, bind_to_port, true)
                      ├── PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, per_connection_buffer_limit_bytes, ...)
                      ├── PROTOBUF_GET_MS_OR_DEFAULT(config, listener_filters_timeout, 15000)
                      ├── buildAccessLog(config)              → AccessLogFactory::fromProto()
                      ├── buildInternalListener(config)
                      ├── buildUdpListenerFactory(config)     → quic_options / udp_packet_writer
                      ├── createListenerFilterFactories(config) → typed_config → factory registry
                      ├── validateFilterChains(config)
                      ├── buildFilterChains(config)           → FilterChainManagerImpl
                      │       ├── filter_chain_matcher        → Matcher::MatchTree
                      │       ├── filter_chains[].transport_socket.typed_config → TLS/raw factory
                      │       └── filter_chains[].filters[].typed_config → NetworkFilter factories
                      ├── buildConnectionBalancer(config)     → typed_config → factory registry
                      ├── buildSocketOptions(config)          → SO_FASTOPEN
                      └── buildListenSocketOptions(config)    → SO_REUSEPORT, SO_TRANSPARENT, ...
```

---

## 9. Key Macros and Utilities

| Macro / Utility | Purpose |
|---|---|
| `PROTOBUF_GET_WRAPPED_OR_DEFAULT(msg, field, default)` | Reads a `BoolValue`/`UInt32Value` wrapper field with fallback |
| `PROTOBUF_GET_MS_OR_DEFAULT(msg, field, default_ms)` | Reads a `Duration` proto field, returns milliseconds |
| `PROTOBUF_GET_OPTIONAL_MS(msg, field)` | Returns `absl::optional<std::chrono::milliseconds>` |
| `MessageUtil::unpackTo(any, message)` | Deserializes `google.protobuf.Any` into a typed message |
| `MessageUtil::equals(a, b)` | Compares two proto messages for structural equality |
| `Network::Address::resolveProtoAddress(addr_proto)` | Converts `core.v3.Address` → `Network::Address::Instance` |
| `Config::Utility::getFactory<T>(typed_config)` | Registry lookup by `type_url` from `TypedExtensionConfig` |
| `SET_AND_RETURN_IF_NOT_OK(status, out)` | Propagates `absl::Status` errors without exceptions |
| `RETURN_IF_NOT_OK(status)` | Returns early on error |

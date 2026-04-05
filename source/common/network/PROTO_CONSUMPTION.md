# How Network Proto Gets Consumed in Envoy

- **Purpose**: Explains how `envoy::config::core::v3` network proto messages (addresses, socket options, TCP keepalive, UDP configs) flow through `source/common/network/`
- **Layer role**: Building blocks for listener and upstream layers

---

## 1. The Network Proto Shape

**Core proto types from `envoy/config/core/v3/`:**

- `core.v3.Address` - Socket/Pipe/Internal addresses with protocol, port, namespace
- `core.v3.SocketOption` - Raw socket options (level, name, value, state)
- `core.v3.TcpKeepalive` - TCP keepalive params (probes, time, interval)
- `core.v3.UdpSocketConfig` - UDP settings (max datagram size, GRO preference)

---

## 2. Address Proto → Runtime Address

### 2.1 `Utility::protobufAddressToAddressNoThrow()` (`network/utility.cc`)

**Primary conversion path - dispatches on `address_case`:**
- `kSocketAddress` → `parseInternetAddressNoThrow(address, port, ipv4_compat, namespace_filepath)` → IPv4/IPv6 instance
- `kPipe` → `PipeInstance::create(path, mode)` → Unix domain socket
- `kEnvoyInternalAddress` → `EnvoyInternalInstance(server_listener_name, endpoint_id)` → Internal listener address

### 2.2 `Utility::resolveProtoAddress()` - Throwing Variant

**Usage**: Listener startup where failure is fatal
- Returns `absl::StatusOr<Network::Address::InstanceConstSharedPtr>`
- Called via `Network::Address::resolveProtoAddress(config.address())`
- Caller checks status and unwraps result

### 2.3 `Utility::protobufAddressSocketType()`

**Extracts socket type from proto:**
- `kSocketAddress` → reads `protocol` enum → TCP=Stream, UDP=Datagram
- `kPipe` / `kEnvoyInternalAddress` → always Stream

### 2.4 Runtime Address → Proto (Admin API)

**Reverse serialization via `Utility::addressToProtobufAddress()`:**
- `Address::Type::Pipe` → `mutable_pipe()->set_path()`
- `Address::Type::Ip` → `mutable_socket_address()->set_address/port()`
- `EnvoyInternal` → `mutable_envoy_internal_address()->set_server_listener_name/endpoint_id()`

---

## 3. SocketOption Proto → Kernel `setsockopt()`

### 3.1 Proto Structure

**Fields**: `level` (SOL_SOCKET, IPPROTO_TCP), `name` (SO_KEEPALIVE, TCP_NODELAY), `value` (int_value | buf_value), `state` (PREBIND | BOUND | LISTENING)

### 3.2 `SocketOptionFactory::buildLiteralOptions()` (`socket_option_factory.cc`)

**Conversion path:**
- Iterates proto repeated field `SocketOption[]`
- Creates `SocketOptionImpl` for each entry based on value_case:
  - `kIntValue` → `SocketOptionImpl(state, level+name, int_value)`
  - `kBufValue` → `SocketOptionImpl(state, level+name, buf_value)`
- Returns `Network::Socket::OptionsSharedPtr`

### 3.3 Pre-built Options from Named Proto Fields

**Standard options built from high-level proto fields:**
- `buildTcpKeepaliveOptions()` → `core.v3.TcpKeepalive` → `SO_KEEPALIVE`, `TCP_KEEPCNT/IDLE/INTVL`
- `buildIpFreebindOptions()` → listener `freebind` → `IP_FREEBIND`
- `buildIpTransparentOptions()` → listener `transparent` → `IP_TRANSPARENT`
- `buildSocketMarkOptions()` → upstream bind config → `SO_MARK`
- `buildTcpFastOpenOptions()` → listener `tcp_fast_open_queue_length` → `TCP_FASTOPEN`
- `buildReusePortOptions()` → listener `enable_reuse_port` → `SO_REUSEPORT`
- `buildUdpGroOptions()` → UDP listener → `UDP_GRO`
- `buildMptcpOptions()` → listener `enable_mptcp` → `IPPROTO_MPTCP`

### 3.4 Application Timing via `state` Field

**Socket lifecycle stages:**
- `PREBIND` → before `bind()` → `SO_REUSEPORT`, `IP_FREEBIND`, `TCP_KEEPALIVE`
- `BOUND` → after `bind()` → `IP_TRANSPARENT`, `SO_MARK`
- `LISTENING` → after `listen()` → `TCP_FASTOPEN`

**Applied in**: `ConnectionImpl` via `Network::Socket::applyOptions(socket->options(), *socket, STATE_*)`

---

## 4. UDP Config Proto

**`ResolvedUdpSocketConfig` constructor (`network/utility.cc`):**
- Reads `core.v3.UdpSocketConfig` proto
- Extracts `max_rx_datagram_size` (default: `DEFAULT_UDP_MAX_DATAGRAM_SIZE`) via `PROTOBUF_GET_WRAPPED_OR_DEFAULT`
- Extracts `prefer_gro` (default: function parameter) via same macro
- Both fields are nullable wrappers (`UInt64Value`, `BoolValue`)

---

## 5. Transport Socket Proto (`TypedExtensionConfig`)

**Standard extension pattern in `filter_chain_manager_impl.cc`:**
- Proto: `filter_chain.transport_socket.typed_config` (Any message, e.g., `DownstreamTlsContext`)
- Lookup factory by type: `Config::Utility::getFactory<DownstreamTransportSocketConfigFactory>()`
- Unpack proto: `MessageUtil::unpackTo(typed_config, *config_message)`
- Create factory: `factory->createTransportSocketFactory(*config_message, context, server_names)`
- Result: `Network::TransportSocketFactory` wraps raw I/O with TLS/raw buffer/ALTS at connection time

---

## 6. `ConnectionImpl` and Proto at Runtime

**No proto consumption at construction:**
- Operates on resolved objects (addresses, socket options)
- Only proto usage: socket option state enum constants (`STATE_PREBIND`, `STATE_BOUND`, `STATE_LISTENING`)
- Called via `Network::Socket::applyOptions(socket->options(), *socket, STATE_*)`
- Diagnostics: `setFailureReason()` composes error strings from transport socket failures

---

## 7. Filter Manager and Network Filters

**Proto consumption front-loaded at startup:**
- `FilterManagerImpl` doesn't consume proto directly
- Filter factories created from proto during `ListenerImpl::buildFilterChains()`
- At connection time: `filter_chain.networkFilterFactories()->createNetworkFilterChain(connection)`
- Factory creation: `factory->createFilterFactoryFromProto(*typed_config_message, *context)`
- Runtime: only C++ objects exist, no proto parsing

---

## 8. Address Types and Proto Mapping

**Proto to Runtime Mappings:**
- `kSocketAddress` (TCP/UDP) → `Address::Ipv4Instance` / `Ipv6Instance` → TCP/UDP listeners, upstream endpoints, QUIC
- `kPipe` → `Address::PipeInstance` → Unix domain socket listeners
- `kEnvoyInternalAddress` → `Address::EnvoyInternalInstance` → Internal listeners (sidecar bypass)

---

## 9. End-to-End Flow for Network Proto

**Bootstrap/xDS → Runtime objects:**

1. **Source**: Bootstrap config or xDS resource with `core.v3.Address`, `core.v3.SocketOption`, `core.v3.TcpKeepalive`
2. **Consumers**: `ListenerImpl`, `UpstreamImpl`, `ClusterImplBase`
3. **Conversion paths**:
   - Address: `Utility::protobufAddressToAddressNoThrow(address_proto)`
     - `kSocketAddress` → `parseInternetAddressNoThrow(ip, port)` → IPv4/IPv6 instance
     - `kPipe` → `PipeInstance::create(path, mode)`
     - `kEnvoyInternalAddress` → `EnvoyInternalInstance(name, id)`
   - Socket options: `SocketOptionFactory::buildLiteralOptions(socket_options_proto)` → `SocketOptionImpl[]`
   - TCP keepalive: `parseTcpKeepaliveConfig(tcp_keepalive_proto)` → `SocketOptionImpl` with `SO_KEEPALIVE/TCP_KEEPCNT/IDLE/INTVL`
   - Named options: `buildTcpFastOpenOptions()`, `buildReusePortOptions()`, `buildIpFreebindOptions()`, etc.
4. **Runtime application**: `Socket::applyOptions()` at `PREBIND` / `BOUND` / `LISTENING` stages

---

## 10. Key Utilities Summary

**Core functions and locations:**
- `Utility::protobufAddressToAddressNoThrow()` (`network/utility.cc`) - Proto address → runtime instance
- `Utility::resolveProtoAddress()` (`network/address_impl.cc`) - Validated address resolution
- `Utility::protobufAddressSocketType()` (`network/utility.cc`) - Extract TCP vs UDP from proto
- `Utility::addressToProtobufAddress()` (`network/utility.cc`) - Runtime → proto (admin API dumps)
- `SocketOptionFactory::buildLiteralOptions()` (`socket_option_factory.cc`) - Proto options → `Socket::Options`
- `parseTcpKeepaliveConfig()` (`socket_option_factory.cc`) - Keepalive proto → socket options
- `ResolvedUdpSocketConfig` (`network/utility.cc`) - UDP proto → config struct
- `Socket::applyOptions()` (`socket_interface.cc`) - Apply options at lifecycle stages
- `PROTOBUF_GET_WRAPPED_OR_DEFAULT` (`protobuf/utility.h`) - Read nullable wrapper fields safely

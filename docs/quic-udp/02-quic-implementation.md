# QUIC Implementation Details

This document provides a deep dive into Envoy's QUIC implementation, covering internal architecture, key optimizations, and implementation patterns.

## Core Components

### QuicServerTransportSocketFactory

The factory responsible for creating and managing QUIC server transport sockets.

```cpp
class QuicServerTransportSocketFactory : 
    public Network::DownstreamTransportSocketFactory
{
public:
  static absl::StatusOr<std::unique_ptr<QuicServerTransportSocketFactory>> 
  create(bool enable_early_data, 
         Stats::Scope& store,
         Ssl::ServerContextConfigPtr config,
         Envoy::Ssl::ContextManager& manager);
  
  // Get TLS certificate chain and private key based on SNI
  std::pair<quiche::QuicheReferenceCountedPointer<quic::ProofSource::Chain>,
            std::shared_ptr<quic::CertificatePrivateKey>>
  getTlsCertificateAndKey(absl::string_view sni, 
                         bool* cert_matched_sni) const;
  
  bool earlyDataEnabled() const { return enable_early_data_; }

protected:
  // Handle certificate rotation
  absl::Status onSecretUpdated() override;

private:
  Envoy::Ssl::ContextManager& manager_;
  Ssl::ServerContextConfigPtr config_;
  mutable absl::Mutex ssl_ctx_mu_;
  Envoy::Ssl::ServerContextSharedPtr ssl_ctx_ ABSL_GUARDED_BY(ssl_ctx_mu_);
  bool enable_early_data_;
};
```

#### Key Responsibilities

1. **TLS Context Management**
   - Creates and maintains SSL server contexts
   - Handles certificate rotation without connection interruption
   - Thread-safe access to TLS configuration

2. **Certificate Selection**
   - SNI-based certificate selection
   - Wildcard certificate matching
   - Fallback to default certificate

3. **Early Data Configuration**
   - Controls 0-RTT support
   - Coordinates with TLS 1.3 session tickets

#### Certificate Rotation Flow

```
Secret Update Detected
          │
          ▼
onSecretUpdated() called
          │
          ▼
Create new SSL context
          │
          ▼
Acquire ssl_ctx_mu_ lock
          │
          ▼
Swap ssl_ctx_ pointer
          │
          ▼
Release lock
          │
          ▼
Old context reference-counted deletion
          │
          ▼
New connections use new certificates
```

### QuicFilterManagerConnectionImpl

The bridge between QUIC sessions and Envoy's Network::Connection interface.

```cpp
class QuicFilterManagerConnectionImpl : 
    public Network::ConnectionImplBase,
    public SendBufferMonitor,
    public QuicWriteEventCallback
{
public:
  QuicFilterManagerConnectionImpl(
      QuicNetworkConnection& connection,
      const quic::QuicConnectionId& connection_id,
      Event::Dispatcher& dispatcher,
      uint32_t send_buffer_limit,
      std::shared_ptr<QuicSslConnectionInfo>&& info,
      std::unique_ptr<StreamInfo::StreamInfo>&& stream_info,
      QuicStatNames& quic_stat_names,
      Stats::Scope& stats_scope);
  
  // Network::FilterManager interface
  void addWriteFilter(Network::WriteFilterSharedPtr filter) override;
  void addReadFilter(Network::ReadFilterSharedPtr filter) override;
  bool initializeReadFilters() override;
  
  // SendBufferMonitor - track bytes buffered across all streams
  void updateBytesBuffered(uint64_t old_buffered_bytes, 
                          uint64_t new_buffered_bytes) override;
  
  // Network::Connection interface
  void close(Network::ConnectionCloseType type) override;
  bool connecting() const override;
  void setBufferLimits(uint32_t limit) override;
  bool aboveHighWatermark() const override;

protected:
  virtual bool hasDataToWrite() PURE;
  virtual const quic::QuicConnection* quicConnection() const PURE;
  virtual quic::QuicConnection* quicConnection() PURE;
  
  void onSendBufferHighWatermark();
  void onSendBufferLowWatermark();
  
  QuicNetworkConnection* network_connection_;
  std::unique_ptr<Network::FilterManagerImpl> filter_manager_;
  std::unique_ptr<StreamInfo::StreamInfo> stream_info_;
  
  // Simulates watermark buffer behavior for QUIC
  EnvoyQuicSimulatedWatermarkBuffer write_buffer_watermark_simulation_;
  uint64_t bytes_to_send_{0};
};
```

#### Buffer Management and Watermarks

QUIC doesn't buffer at the connection level; instead, each stream has its own buffer. The `QuicFilterManagerConnectionImpl` simulates connection-level watermarks:

```cpp
void QuicFilterManagerConnectionImpl::updateBytesBuffered(
    uint64_t old_buffered_bytes,
    uint64_t new_buffered_bytes) 
{
  // Track total bytes across all streams
  write_buffer_watermark_simulation_.checkLowWatermark(old_buffered_bytes);
  write_buffer_watermark_simulation_.checkHighWatermark(new_buffered_bytes);
  
  if (write_buffer_watermark_simulation_.isAboveHighWatermark()) {
    onSendBufferHighWatermark();  // Trigger backpressure
  } else if (write_buffer_watermark_simulation_.isBelowLowWatermark()) {
    onSendBufferLowWatermark();   // Release backpressure
  }
}
```

**Watermark Flow:**

```
Stream 1 buffers 50KB
Stream 2 buffers 40KB
Stream 3 buffers 30KB
═══════════════════════
Total: 120KB

High Watermark (100KB) Exceeded
          │
          ▼
onSendBufferHighWatermark()
          │
          ▼
Notify upstream filters (backpressure)
          │
          ▼
Upstream pauses sending data
          │
          ▼
Streams drain to 80KB
          │
          ▼
Low Watermark (50KB) crossed
          │
          ▼
onSendBufferLowWatermark()
          │
          ▼
Resume upstream data flow
```

### EnvoyQuicServerSession

The server-side QUIC session implementation that manages connection lifecycle and stream creation.

```cpp
class EnvoyQuicServerSession : 
    public quic::QuicServerSessionBase,
    public QuicFilterManagerConnectionImpl,
    public Envoy::Http::IdleSessionInterface
{
public:
  EnvoyQuicServerSession(
      const quic::QuicConfig& config,
      const quic::ParsedQuicVersionVector& supported_versions,
      std::unique_ptr<EnvoyQuicServerConnection> connection,
      quic::QuicSession::Visitor* visitor,
      quic::QuicCryptoServerStreamBase::Helper* helper,
      const quic::QuicCryptoServerConfig* crypto_config,
      quic::QuicCompressedCertsCache* compressed_certs_cache,
      Event::Dispatcher& dispatcher,
      uint32_t send_buffer_limit,
      QuicStatNames& quic_stat_names,
      Stats::Scope& listener_scope,
      EnvoyQuicCryptoServerStreamFactoryInterface& crypto_server_stream_factory,
      std::unique_ptr<StreamInfo::StreamInfo>&& stream_info,
      QuicConnectionStats& connection_stats,
      EnvoyQuicConnectionDebugVisitorFactoryInterfaceOptRef debug_visitor_factory,
      Http::SessionIdleListInterface* session_idle_list);
  
  // quic::QuicSession overrides
  void OnConnectionClosed(const quic::QuicConnectionCloseFrame& frame,
                         quic::ConnectionCloseSource source) override;
  void Initialize() override;
  void OnCanWrite() override;
  void OnTlsHandshakeComplete() override;
  void ProcessUdpPacket(const quic::QuicSocketAddress& self_address,
                       const quic::QuicSocketAddress& peer_address,
                       const quic::QuicReceivedPacket& packet) override;
  
  // Create HTTP/3 streams
  quic::QuicSpdyStream* CreateIncomingStream(quic::QuicStreamId id) override;
  
  // IdleSessionInterface
  void TerminateIdleSession() override;

protected:
  std::unique_ptr<quic::QuicCryptoServerStreamBase> 
  CreateQuicCryptoServerStream(
      const quic::QuicCryptoServerConfig* crypto_config,
      quic::QuicCompressedCertsCache* compressed_certs_cache) override;

private:
  void setUpRequestDecoder(EnvoyQuicServerStream& stream);
  void MaybeAddSessionToIdleList();
  void MaybeRemoveSessionFromIdleList();
  
  std::unique_ptr<EnvoyQuicServerConnection> quic_connection_;
  Http::ServerConnectionCallbacks* http_connection_callbacks_{nullptr};
  EnvoyQuicCryptoServerStreamFactoryInterface& crypto_server_stream_factory_;
  QuicConnectionStats& connection_stats_;
  Http::SessionIdleListInterface* session_idle_list_;
  bool is_in_idle_list_ = false;
};
```

#### Stream Creation Flow

```
Client sends HEADERS frame
          │
          ▼
QUIC parses stream ID
          │
          ▼
CreateIncomingStream(stream_id)
          │
          ▼
new EnvoyQuicServerStream()
          │
          ▼
ActivateStream()
          │
          ▼
setUpRequestDecoder()
          │
          ▼
Notify HCM of new stream
          │
          ▼
HTTP filter chain processes request
```

#### Connection Migration Detection

```cpp
void EnvoyQuicServerSession::ProcessUdpPacket(
    const quic::QuicSocketAddress& self_address,
    const quic::QuicSocketAddress& peer_address,
    const quic::QuicReceivedPacket& packet)
{
  // Detect client address changes
  if (peer_address != last_peer_address_) {
    connection_stats_.num_server_migration_detected_.inc();
    ENVOY_CONN_LOG(debug, "Client migrated from {} to {}",
                   last_peer_address_.ToString(),
                   peer_address.ToString());
  }
  
  // Process packet with QUICHE
  quic::QuicServerSessionBase::ProcessUdpPacket(self_address, 
                                                peer_address, 
                                                packet);
}
```

### EnvoyQuicClientSession

Client-side QUIC session with connection pooling and migration support.

```cpp
class EnvoyQuicClientSession :
    public QuicFilterManagerConnectionImpl,
    public quic::QuicSpdyClientSession,
    public Network::ClientConnection,
    public PacketsToReadDelegate
{
public:
  EnvoyQuicClientSession(
      const quic::QuicConfig& config,
      const quic::ParsedQuicVersionVector& supported_versions,
      std::unique_ptr<EnvoyQuicClientConnection> connection,
      quic::QuicForceBlockablePacketWriter* absl_nullable writer,
      EnvoyQuicClientConnection::EnvoyQuicMigrationHelper* migration_helper,
      const quic::QuicConnectionMigrationConfig& migration_config,
      const quic::QuicServerId& server_id,
      std::shared_ptr<quic::QuicCryptoClientConfig> crypto_config,
      Event::Dispatcher& dispatcher,
      uint32_t send_buffer_limit,
      EnvoyQuicCryptoClientStreamFactoryInterface& crypto_stream_factory,
      QuicStatNames& quic_stat_names,
      OptRef<Http::HttpServerPropertiesCache> rtt_cache,
      Stats::Scope& scope,
      const Network::TransportSocketOptionsConstSharedPtr& transport_socket_options,
      OptRef<Network::UpstreamTransportSocketFactory> transport_socket_factory);
  
  // Network::ClientConnection
  void connect() override;
  
  // quic::QuicSession
  void OnConnectionClosed(const quic::QuicConnectionCloseFrame& frame,
                         quic::ConnectionCloseSource source) override;
  void OnHttp3GoAway(uint64_t stream_id) override;
  void OnTlsHandshakeComplete() override;
  
  // Connection pool integration
  void OnCanCreateNewOutgoingStream(bool unidirectional) override;
  
  // Server preferred address migration
  void OnServerPreferredAddressAvailable(
      const quic::QuicSocketAddress& server_preferred_address) override;
  
  // Network observation for migration
  void registerNetworkObserver(EnvoyQuicNetworkObserverRegistry& registry);

protected:
  std::unique_ptr<quic::QuicSpdyClientStream> CreateClientStream() override;
  std::unique_ptr<quic::QuicCryptoClientStreamBase> CreateQuicCryptoStream() override;

private:
  uint64_t streamsAvailable();
  
  Http::ConnectionCallbacks* http_connection_callbacks_{nullptr};
  std::shared_ptr<quic::QuicCryptoClientConfig> crypto_config_;
  OptRef<Http::HttpServerPropertiesCache> rtt_cache_;
  bool disable_keepalive_{false};
  bool session_handles_migration_;
  QuicNetworkConnectivityObserverPtr network_connectivity_observer_;
};
```

#### 0-RTT (Early Data) Support

```cpp
void EnvoyQuicClientSession::connect()
{
  // Attempt 0-RTT if we have cached session ticket
  if (crypto_config_->HasCachedSessionTicket(server_id_)) {
    // Start connection in 0-RTT mode
    CryptoConnect();
    
    // Can send data immediately without waiting for handshake
    if (IsEncryptionEstablished()) {
      OnCanWrite();  // Start sending early data
    }
  } else {
    // Full 1-RTT handshake
    CryptoConnect();
  }
}
```

**0-RTT Flow:**

```
Client has cached session ticket
          │
          ▼
Send Initial packet + 0-RTT packet (request data)
          │
          ├─────────────────────────┐
          │                         │
          ▼                         ▼
Server processes          Server rejects 0-RTT
0-RTT data                        │
          │                         │
          ▼                         ▼
Send response           Downgrade to 1-RTT
                       Client resends data
```

## Address Caching for QUIC Client Sockets

### The Problem

QUIC client sockets receive frequent UDP packets from the same source address (the server). Each received packet requires converting the socket address to an Envoy `Address::Instance`. Creating these objects repeatedly for the same address is expensive:

1. Memory allocation for each address instance
2. String formatting for IP addresses
3. Repeated parsing of sockaddr structures

### The Solution: Address Cache

The `IoSocketHandleImpl` implements an LRU (Least Recently Used) address cache specifically optimized for QUIC client sockets.

```cpp
class IoSocketHandleImpl : public IoSocketHandleBaseImpl {
public:
  explicit IoSocketHandleImpl(
      os_fd_t fd = INVALID_SOCKET,
      bool socket_v6only = false,
      absl::optional<int> domain = absl::nullopt,
      size_t address_cache_max_capacity = 0)  // Key parameter
    : IoSocketHandleBaseImpl(fd, socket_v6only, domain),
      address_cache_max_capacity_(address_cache_max_capacity) 
  {
    if (address_cache_max_capacity > 0) {
      // Only allocate cache if explicitly requested
      recent_received_addresses_ = std::vector<QuicEnvoyAddressPair>();
    }
  }

private:
  // Caches the address instances of the most recently received packets
  // Should only be used by QUIC client sockets to avoid creating multiple
  // address instances for the same address in each read operation.
  // Since the QUIC client sockets are connected via a connect() call,
  // the total number of expected addresses are 2 (source and destination),
  // and the cache size defaults to 4.
  size_t address_cache_max_capacity_;
  
  // Only non-null if address_cache_max_capacity_ is greater than 0
  absl::optional<std::vector<QuicEnvoyAddressPair>> recent_received_addresses_;
};
```

### Cache Implementation

```cpp
// Type definition
using QuicEnvoyAddressPair = 
    std::pair<quic::QuicSocketAddress, Address::InstanceConstSharedPtr>;

Address::InstanceConstSharedPtr 
IoSocketHandleImpl::getOrCreateEnvoyAddressInstance(
    sockaddr_storage ss, 
    socklen_t ss_len) 
{
  // If cache is disabled, create new address every time
  if (!recent_received_addresses_) {
    return Address::addressFromSockAddrOrDie(ss, ss_len, fd_, socket_v6only_);
  }
  
  // Convert to QuicSocketAddress for comparison
  quic::QuicSocketAddress quic_address(ss);
  
  // Linear search through cache (small size makes this efficient)
  auto it = std::find_if(
      recent_received_addresses_->begin(),
      recent_received_addresses_->end(),
      [&quic_address](const QuicEnvoyAddressPair& pair) {
        return pair.first == quic_address;
      });
  
  if (it != recent_received_addresses_->end()) {
    // Cache hit - return cached address
    Address::InstanceConstSharedPtr cached_addr = it->second;
    
    // Move to back (LRU: most recent at end)
    std::rotate(it, it + 1, recent_received_addresses_->end());
    
    return cached_addr;
  }
  
  // Cache miss - create new address
  Address::InstanceConstSharedPtr new_address = 
      Address::addressFromSockAddrOrDie(ss, ss_len, fd_, socket_v6only_);
  
  // Add to cache
  recent_received_addresses_->push_back(
      QuicEnvoyAddressPair(quic_address, new_address));
  
  // Evict oldest if over capacity
  if (recent_received_addresses_->size() > address_cache_max_capacity_) {
    recent_received_addresses_->erase(recent_received_addresses_->begin());
  }
  
  return new_address;
}
```

### Cache Characteristics

| Property | Value | Rationale |
|----------|-------|-----------|
| Default Size | 4 | QUIC client sees 2 addresses (local + remote), 2x for safety |
| Eviction Policy | LRU | Most recently used addresses stay in cache |
| Lookup | O(n) linear | Small cache makes linear search faster than hash |
| Storage | Vector | Contiguous memory, cache-friendly |
| Thread Safety | Single-threaded | Each socket owned by one thread |

### When Address Cache is Used

```cpp
// In recvmsg() - called for every received UDP packet
Api::IoCallUint64Result IoSocketHandleImpl::recvmsg(
    Buffer::RawSlice* slices,
    const uint64_t num_slice,
    uint32_t self_port,
    const UdpSaveCmsgConfig& save_cmsg_config,
    RecvMsgOutput& output)
{
  // ... receive packet ...
  
  // Get peer address - uses cache if enabled
  output.msg_[0].peer_address_ = 
      getOrCreateEnvoyAddressInstance(peer_addr, hdr.msg_namelen);
  
  // Get local address from control message - uses cache if enabled
  if (hdr.msg_controllen > 0) {
    for (struct cmsghdr* cmsg = CMSG_FIRSTHDR(&hdr); 
         cmsg != nullptr; 
         cmsg = CMSG_NXTHDR(&hdr, cmsg)) 
    {
      Address::InstanceConstSharedPtr addr = 
          maybeGetDstAddressFromHeader(*cmsg, self_port);
      if (addr != nullptr) {
        output.msg_[0].local_address_ = std::move(addr);
      }
    }
  }
}
```

### Performance Impact

**Without Cache** (per packet):
- Allocate Address::InstanceImpl: ~200 bytes
- Format IP string: ~50ns
- Shared pointer overhead: ~16 bytes
- Total per packet: ~216 bytes, ~100ns

**With Cache** (cache hit):
- Lookup in 4-entry cache: ~10ns
- Return shared pointer: ~0 bytes allocated
- Total per packet: ~0 bytes, ~10ns

**For 10,000 packets/sec:**
- Without cache: 2.16 MB/s allocated, 1ms CPU
- With cache: ~0 KB/s allocated, ~0.1ms CPU
- **Savings: >90% reduction in allocations and CPU**

### Cache Usage Example

```
QUIC Client Connection Established
          │
          ▼
Socket created with address_cache_max_capacity=4
          │
          ▼
Packet 1 arrives from 192.0.2.1:443
  Cache: [192.0.2.1:443]
          │
          ▼
Packet 2 arrives from 192.0.2.1:443 (cache hit!)
  Cache: [192.0.2.1:443] (reused)
          │
          ▼
Packet 3 arrives from 203.0.113.5:443 (server preferred address)
  Cache: [192.0.2.1:443, 203.0.113.5:443]
          │
          ▼
Packets 4-100 from 203.0.113.5:443 (all cache hits!)
  Cache: [192.0.2.1:443, 203.0.113.5:443]
          │
          ▼
Connection closed, cache destroyed
```

### Configuration

The address cache is enabled automatically for QUIC client connections:

```cpp
// In EnvoyQuicClientConnection constructor
IoSocketHandleImpl(
    fd,
    socket_v6only,
    domain,
    4  // address_cache_max_capacity for QUIC client
);
```

For non-QUIC sockets, the cache is disabled (capacity = 0):

```cpp
// Regular UDP socket
IoSocketHandleImpl(fd, socket_v6only, domain, 0);  // No cache
```

## Stream Management

### Stream Lifecycle

```cpp
// Server-side stream creation
quic::QuicSpdyStream* EnvoyQuicServerSession::CreateIncomingStream(
    quic::QuicStreamId id)
{
  auto stream = std::make_unique<EnvoyQuicServerStream>(
      id, this, quic::BIDIRECTIONAL, http_connection_callbacks_);
  
  EnvoyQuicServerStream* stream_ptr = stream.get();
  ActivateStream(std::move(stream));
  setUpRequestDecoder(*stream_ptr);
  
  return stream_ptr;
}
```

### Stream States

```
┌──────────┐
│   IDLE   │ Stream ID reserved but not yet opened
└────┬─────┘
     │ HEADERS frame received/sent
     ▼
┌──────────┐
│   OPEN   │ Bidirectional data transfer
└────┬─────┘
     │ FIN sent/received on one direction
     ▼
┌──────────┐
│HALF_CLOSE│ One direction closed
└────┬─────┘
     │ FIN sent/received on other direction
     ▼
┌──────────┐
│  CLOSED  │ Stream completed
└──────────┘
```

### Stream Prioritization

HTTP/3 uses a different prioritization scheme than HTTP/2:

```cpp
// HTTP/3 extensible priorities (RFC 9218)
struct Priority {
  uint8_t urgency;        // 0-7, lower is more urgent
  bool incremental;       // true for incremental resources
};

// Example priorities
const Priority URGENT = {urgency: 0, incremental: false};      // Critical JS
const Priority HIGH = {urgency: 3, incremental: false};        // Important CSS
const Priority MEDIUM = {urgency: 4, incremental: true};       // Images
const Priority LOW = {urgency: 6, incremental: true};          // Analytics
```

## Connection Migration Implementation

### Client-Side Port Migration

```cpp
void EnvoyQuicClientConnection::OnPathDegradingDetected()
{
  if (!migrate_port_on_path_degrading_) {
    return;
  }
  
  if (num_socket_switches_ >= kMaxNumSocketSwitches) {
    ENVOY_CONN_LOG(warn, "Exceeded max socket switches ({}), not migrating",
                   kMaxNumSocketSwitches);
    return;
  }
  
  ENVOY_CONN_LOG(debug, "Path degrading detected, attempting port migration");
  maybeMigratePort();
}

void EnvoyQuicClientConnection::maybeMigratePort()
{
  // Create new socket with same local IP but different port
  auto peer_address = connection_socket_->connectionInfoProvider()
                          .remoteAddress();
  probeWithNewPort(peer_address->asQuicSocketAddress(), 
                   quic::PathValidationReason::kPathDegrading);
}

void EnvoyQuicClientConnection::probeWithNewPort(
    const quic::QuicSocketAddress& peer_addr,
    quic::PathValidationReason reason)
{
  // Create validation context with new socket
  auto probing_socket = createProbingSocket();
  auto writer = createPacketWriter(probing_socket.get());
  
  auto context = std::make_unique<EnvoyQuicPathValidationContext>(
      connection_socket_->connectionInfoProvider().localAddress()
          ->asQuicSocketAddress(),
      peer_addr,
      std::move(writer),
      std::move(probing_socket));
  
  // Start path validation
  ValidatePath(std::move(context),
              std::make_unique<EnvoyPathValidationResultDelegate>(*this),
              reason);
}
```

### Path Validation Flow

```
Path Degradation Detected (packet loss > threshold)
          │
          ▼
Create new socket (same IP, new port)
          │
          ▼
Send PATH_CHALLENGE frame on new path
          │
          ├─────────────────────────┐
          │                         │
          ▼                         ▼
PATH_RESPONSE received    Timeout (no response)
          │                         │
          ▼                         │
Validate peer reachability         │
          │                         │
          ▼                         ▼
onPathValidationSuccess   onPathValidationFailure
          │                         │
          ▼                         ▼
Switch to new socket      Destroy probe socket
          │                 Stay on old path
          ▼
Update connection socket
Close old socket
Connection uses new port
```

### Server Preferred Address Migration

```cpp
void EnvoyQuicClientSession::OnServerPreferredAddressAvailable(
    const quic::QuicSocketAddress& server_preferred_address)
{
  ENVOY_CONN_LOG(info, "Server provided preferred address: {}",
                 server_preferred_address.ToString());
  
  // Start probing server's preferred address
  connection_->probeAndMigrateToServerPreferredAddress(
      server_preferred_address);
}
```

## Packet Writing

### EnvoyQuicPacketWriter

```cpp
class EnvoyQuicPacketWriter : public quic::QuicPacketWriter {
public:
  WriteResult WritePacket(
      const char* buffer,
      size_t buf_len,
      const quic::QuicIpAddress& self_address,
      const quic::QuicSocketAddress& peer_address,
      quic::PerPacketOptions* options) override
  {
    // Convert to Envoy address
    auto envoy_peer_address = quicAddressToEnvoyAddress(peer_address);
    auto envoy_self_address = quicAddressToEnvoyAddress(self_address);
    
    // Prepare buffer
    Buffer::RawSlice slice{const_cast<char*>(buffer), buf_len};
    
    // Send via UDP socket
    Api::IoCallUint64Result result = io_handle_->sendmsg(
        &slice,
        1,  // num_slice
        0,  // flags
        envoy_self_address.ip(),
        *envoy_peer_address);
    
    if (result.ok()) {
      return WriteResult(WRITE_STATUS_OK, result.return_value_);
    } else if (result.err_->getErrorCode() == 
               Api::IoError::IoErrorCode::Again) {
      return WriteResult(WRITE_STATUS_BLOCKED, EAGAIN);
    } else {
      return WriteResult(WRITE_STATUS_ERROR, result.err_->getErrorCode());
    }
  }
};
```

### GSO (Generic Segmentation Offload)

For improved performance, Envoy supports GSO for sending multiple QUIC packets:

```cpp
class EnvoyQuicGsoBatchWriter : public EnvoyQuicPacketWriter {
  WriteResult WritePacket(...) override {
    // Batch multiple packets together
    if (can_batch_ && gso_size_ > 0) {
      batch_buffer_->add(buffer, buf_len);
      
      if (batch_buffer_->length() < max_batch_size_) {
        return WriteResult(WRITE_STATUS_OK, buf_len);
      }
      
      // Send batched packets with GSO
      return sendBatch();
    }
    
    // Fall back to single packet send
    return EnvoyQuicPacketWriter::WritePacket(...);
  }
};
```

## Flow Control

### Connection-Level Flow Control

```cpp
class QuicFilterManagerConnectionImpl {
  void setBufferLimits(uint32_t limit) override {
    // Set per-connection send buffer limit
    write_buffer_watermark_simulation_.setWatermarks(limit / 2, limit);
    
    // Propagate to QUIC connection
    if (quicConnection()) {
      // QUIC will enforce this via flow control
      quicConnection()->set_session_send_window_limit(limit);
    }
  }
};
```

### Stream-Level Flow Control

Each QUIC stream has independent flow control:

```cpp
// QUIC stream flow control
const uint64_t initial_stream_flow_control_window = 16 * 1024;  // 16 KB
const uint64_t initial_session_flow_control_window = 128 * 1024; // 128 KB

// Flow control update when data is consumed
void EnvoyQuicStream::OnDataConsumed(size_t bytes_consumed) {
  quic::QuicSpdyStream::OnDataConsumed(bytes_consumed);
  
  // Send WINDOW_UPDATE if needed
  MaybeSendWindowUpdate();
}
```

## Congestion Control

Envoy/QUIC supports multiple congestion control algorithms:

### Cubic (default)

```cpp
quic::QuicConfig config;
config.SetCongestionControlAlgorithm(quic::kCubicBytes);
```

### BBR (Bottleneck Bandwidth and RTT)

```cpp
quic::QuicConfig config;
config.SetCongestionControlAlgorithm(quic::kBBR);
```

### Configuration

```yaml
# Enable BBR congestion control
clusters:
- name: quic_cluster
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  typed_extension_protocol_options:
    envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
      explicit_http_config:
        http3_protocol_options:
          quic_protocol_options:
            max_concurrent_streams: 100
            initial_stream_window_size: 65536
            initial_connection_window_size: 1048576
```

## Debugging and Observability

### QUIC-Specific Logs

```cpp
// Connection-level logging
ENVOY_CONN_LOG(debug, "QUIC connection established, version: {}",
               connection_->version().ToString());

// Stream-level logging
ENVOY_STREAM_LOG(trace, "Stream {} sending {} bytes",
                 id(), buffer.length());

// Packet-level logging (very verbose)
ENVOY_LOG(trace, "Sent packet {}, size: {}, encryption_level: {}",
          packet_number, packet_size, encryption_level);
```

### Connection Debug Visitor

```cpp
class EnvoyQuicConnectionDebugVisitor : 
    public quic::QuicConnectionDebugVisitor 
{
  void OnPacketSent(...) override {
    // Track sent packets
  }
  
  void OnPacketLoss(...) override {
    // Track packet loss
  }
  
  void OnPingSent() override {
    // Track keepalive pings
  }
};
```

### Admin Interface Queries

```bash
# Get QUIC connection stats
curl http://localhost:9901/stats | grep quic

# Example output:
# cluster.quic_cluster.upstream_cx_quic_total: 42
# listener.0.0.0.0_443.downstream_cx_quic_total: 156
# listener.0.0.0.0_443.http3.codec_stats.headers_cb_no_stream: 0
```

## Testing Considerations

### Unit Testing QUIC Code

```cpp
// Mock QUIC connection
class MockQuicConnection : public quic::QuicConnection {
  MOCK_METHOD(void, SendControlFrame, (const quic::QuicFrame&));
  MOCK_METHOD(void, OnCanWrite, ());
};

// Test address caching
TEST(IoSocketHandleImplTest, AddressCachingForQuicClient) {
  IoSocketHandleImpl handle(fd, false, AF_INET, 4);  // Cache size 4
  
  sockaddr_storage addr1 = createAddress("192.0.2.1", 443);
  
  auto address1 = handle.getOrCreateEnvoyAddressInstance(addr1, sizeof(addr1));
  auto address2 = handle.getOrCreateEnvoyAddressInstance(addr1, sizeof(addr1));
  
  // Should return same instance (cache hit)
  EXPECT_EQ(address1.get(), address2.get());
}
```

## Next Steps

- [UDP Listeners](03-udp-listeners.md) - Understanding the UDP layer
- [Configuration Examples](04-configuration-examples.md) - Practical configurations
- [QUIC Overview](01-quic-overview.md) - High-level architecture

# UDP Listener Support in Envoy

This document covers Envoy's UDP listener architecture, UDP proxy functionality, and session management for UDP-based protocols.

## UDP Listener Architecture

Unlike TCP, UDP is connectionless, but Envoy creates the concept of "sessions" to provide stateful processing of UDP traffic.

```
┌─────────────────────────────────────────────────────────┐
│                    UDP Listener                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │         UdpListenerImpl                           │ │
│  │  - Socket management                              │ │
│  │  - Packet reception (recvmsg/recvmmsg)            │ │
│  │  - Event loop integration                         │ │
│  │  - Packet processor delegation                    │ │
│  └───────────────────────────────────────────────────┘ │
│                         │                               │
│                         ▼                               │
│  ┌───────────────────────────────────────────────────┐ │
│  │    UdpListenerCallbacks (Filter)                  │ │
│  │  - Creates sessions                               │ │
│  │  - Routes packets to sessions                     │ │
│  │  - Manages session lifecycle                      │ │
│  └───────────────────────────────────────────────────┘ │
│                         │                               │
│                         ▼                               │
│  ┌───────────────────────────────────────────────────┐ │
│  │         UDP Sessions                              │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │ │
│  │  │Session 1 │  │Session 2 │  │Session 3 │  ...  │ │
│  │  └──────────┘  └──────────┘  └──────────┘       │ │
│  └───────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

## UdpListenerImpl

The core UDP listener implementation handles socket I/O and event management.

```cpp
class UdpListenerImpl : 
    public BaseListenerImpl,
    public virtual UdpListener,
    public UdpPacketProcessor
{
public:
  UdpListenerImpl(
      Event::Dispatcher& dispatcher,
      SocketSharedPtr socket,
      UdpListenerCallbacks& cb,
      TimeSource& time_source,
      const envoy::config::core::v3::UdpSocketConfig& config);
  
  // Network::Listener interface
  void disable() override;
  void enable() override;
  
  // UdpListener interface
  Event::Dispatcher& dispatcher() override;
  const Address::InstanceConstSharedPtr& localAddress() const override;
  Api::IoCallUint64Result send(const UdpSendData& data) override;
  Api::IoCallUint64Result flush() override;
  void activateRead() override;
  
  // UdpPacketProcessor interface
  void processPacket(
      Address::InstanceConstSharedPtr local_address,
      Address::InstanceConstSharedPtr peer_address,
      Buffer::InstancePtr buffer,
      MonotonicTime receive_time,
      uint8_t tos,
      Buffer::OwnedImpl saved_cmsg) override;

private:
  void onSocketEvent(short flags);
  void handleReadCallback();
  void handleWriteCallback();
  
  UdpListenerCallbacks& cb_;
  uint32_t packets_dropped_{0};
  TimeSource& time_source_;
  const ResolvedUdpSocketConfig config_;
};
```

### Socket Configuration

```cpp
struct ResolvedUdpSocketConfig {
  // Maximum size of a received UDP packet
  uint32_t max_rx_datagram_size_{9000};
  
  // Enable Generic Receive Offload (GRO)
  bool prefer_gro_{false};
  
  // Control message configuration
  UdpSaveCmsgConfig save_cmsg_config_;
};
```

### Packet Reception Flow

```
UDP packet arrives at socket
          │
          ▼
Kernel wakes up listener thread
          │
          ▼
File event triggered (READ)
          │
          ▼
handleReadCallback()
          │
          ▼
recvmmsg() - batch receive packets
          │
          ▼
For each received packet:
  │
  ├─ Extract peer address
  ├─ Extract local address (from cmsg)
  ├─ Extract TOS byte
  └─ Extract GSO size (if GRO enabled)
          │
          ▼
processPacket() - forward to filter
          │
          ▼
UdpListenerCallbacks::onData()
```

### Batch Packet Processing (recvmmsg)

For high-performance packet processing, Envoy uses `recvmmsg()` to receive multiple packets in a single syscall:

```cpp
void UdpListenerImpl::handleReadCallback()
{
  // Prepare buffers for multiple packets
  const size_t num_packets = cb_.numPacketsExpectedPerEventLoop();
  RawSliceArrays slices(num_packets);
  
  // Receive up to num_packets in one syscall
  Api::IoCallUint64Result result = 
      socket_->ioHandle().recvmmsg(slices, self_port_, 
                                   save_cmsg_config_, output);
  
  if (!result.ok()) {
    return;
  }
  
  // Process each received packet
  for (size_t i = 0; i < result.return_value_; i++) {
    if (output.msg_[i].truncated_and_dropped_) {
      packets_dropped_++;
      continue;
    }
    
    // Forward packet to callbacks
    cb_.onData(std::move(output.msg_[i]));
  }
}
```

**Performance Benefits:**

| Metric | recvmsg() | recvmmsg() (32 packets) |
|--------|-----------|------------------------|
| Syscalls/1000 packets | 1000 | ~32 |
| Context switches | ~1000 | ~32 |
| CPU overhead | High | Low |
| Throughput | Baseline | 2-3x higher |

## UDP Session Management

### Session Concept

Since UDP is connectionless, Envoy creates "sessions" to provide stateful processing:

```cpp
class ActiveSession {
  // Session is identified by 4-tuple
  struct LocalPeerAddresses {
    Address::InstanceConstSharedPtr local_address_;
    Address::InstanceConstSharedPtr peer_address_;
  };
  
  // Each session has:
  - Idle timeout
  - Upstream connection (or tunnel)
  - Filter chain
  - StreamInfo for observability
  - Access logs
};
```

### Session Lifecycle

```
First packet from (peer_ip:peer_port → local_ip:local_port)
          │
          ▼
Create new ActiveSession
          │
          ▼
Select upstream host (load balancing)
          │
          ▼
Create upstream socket
          │
          ▼
Initialize filter chain
          │
          ▼
Start idle timer
          │
          ▼
Forward packets bidirectionally
          │
          ├─ Downstream → Upstream
          └─ Upstream → Downstream
          │
          ▼
Idle timeout OR explicit close
          │
          ▼
Flush access logs
          │
          ▼
Destroy session
```

### Session Types

#### 1. UdpActiveSession (Direct UDP Forwarding)

```cpp
class UdpActiveSession : 
    public Network::UdpPacketProcessor,
    public ActiveSession 
{
public:
  // Create upstream UDP socket
  bool createUpstream() override {
    udp_socket_ = createUdpSocket(host_);
    
    if (use_original_src_ip_) {
      // Bind to downstream's source IP (transparent proxy)
      udp_socket_->bind(addresses_.peer_address_);
    }
    
    // Connect socket to upstream (avoids port exhaustion)
    udp_socket_->connect(host_->address());
    connected_ = true;
    
    // Register for read events
    udp_socket_->ioHandle().initializeFileEvent(
        filter_.dispatcher_,
        [this](uint32_t events) { onReadReady(); },
        Event::FileTriggerType::Edge,
        Event::FileReadyType::Read);
    
    return true;
  }
  
  // Send packet upstream
  void writeUpstream(Network::UdpRecvData& data) override {
    if (!connected_) {
      createUpstream();
    }
    
    Buffer::RawSliceVector slices = data.buffer_->getRawSlices();
    Api::IoCallUint64Result result = 
        udp_socket_->ioHandle().writev(slices.data(), slices.size());
    
    if (result.ok()) {
      cluster_->cluster_stats_.sess_tx_datagrams_.inc();
    } else {
      cluster_->cluster_stats_.sess_tx_errors_.inc();
    }
  }
  
  // Receive packet from upstream
  void processPacket(
      Address::InstanceConstSharedPtr local_address,
      Address::InstanceConstSharedPtr peer_address,
      Buffer::InstancePtr buffer,
      MonotonicTime receive_time,
      uint8_t tos,
      Buffer::OwnedImpl saved_cmsg) override 
  {
    // Create downstream datagram
    Network::UdpRecvData data{
        {addresses_.local_address_, addresses_.peer_address_},
        std::move(buffer),
        receive_time,
        tos,
        std::move(saved_cmsg)};
    
    // Send through write filter chain
    processUpstreamDatagram(data);
    
    // Send to downstream
    writeDownstream(data);
  }

private:
  Network::SocketPtr udp_socket_;
  bool connected_{false};
  bool use_original_src_ip_;
};
```

#### 2. TunnelingActiveSession (UDP-over-HTTP)

```cpp
class TunnelingActiveSession : 
    public ActiveSession,
    public UpstreamTunnelCallbacks,
    public HttpStreamCallbacks
{
public:
  // Create HTTP connection for tunneling
  bool createUpstream() override {
    if (!conn_pool_) {
      createConnectionPool();
    }
    
    establishUpstreamConnection();
    return true;
  }
  
  // Send UDP datagram over HTTP stream
  void writeUpstream(Network::UdpRecvData& data) override {
    if (!can_send_upstream_) {
      maybeBufferDatagram(data);
      return;
    }
    
    // Encode datagram in HTTP DATA frame
    upstream_->encodeData(*data.buffer_);
  }
  
  // Receive UDP datagram from HTTP stream
  void onUpstreamData(Buffer::Instance& data, bool end_stream) override {
    // Decode datagram from HTTP DATA frame
    Network::UdpRecvData recv_data{
        {addresses_.local_address_, addresses_.peer_address_},
        std::make_unique<Buffer::OwnedImpl>(data),
        filter_.time_source_.monotonicTime()};
    
    processUpstreamDatagram(recv_data);
    writeDownstream(recv_data);
  }

private:
  TunnelingConnectionPoolPtr conn_pool_;
  std::unique_ptr<HttpUpstream> upstream_;
  std::queue<BufferedDatagramPtr> datagrams_buffer_;
  bool can_send_upstream_{false};
};
```

## UDP Proxy Filter

The UDP proxy filter manages sessions and provides load balancing for UDP traffic.

```cpp
class UdpProxyFilter : 
    public Network::UdpListenerReadFilter,
    public Upstream::ClusterUpdateCallbacks
{
protected:
  // Process incoming downstream packet
  Network::FilterStatus onDataInternal(Network::UdpRecvData& data) PURE;
  
  // Find or create session for this 4-tuple
  ActiveSession* getOrCreateSession(
      const Network::UdpRecvData::LocalPeerAddresses& addresses);
  
  // Session management
  SessionStorageType sessions_;
  absl::flat_hash_map<std::string, ClusterInfoPtr> cluster_infos_;
};
```

### Session Hashing

Sessions can be hashed using configurable policies:

```cpp
class UdpLoadBalancerContext : public Upstream::LoadBalancerContextBase {
public:
  UdpLoadBalancerContext(
      const Udp::HashPolicy* hash_policy,
      const Network::Address::InstanceConstSharedPtr& peer_address,
      StreamInfo::StreamInfo* stream_info)
    : stream_info_(stream_info) 
  {
    if (hash_policy) {
      // Compute hash from source IP:port
      hash_ = hash_policy->generateHash(*peer_address);
    }
  }
  
  absl::optional<uint64_t> computeHashKey() override { return hash_; }

private:
  absl::optional<uint64_t> hash_;
  StreamInfo::StreamInfo* const stream_info_;
};
```

### Load Balancing Modes

#### 1. Sticky Session (Default)

```cpp
class StickySessionUdpProxyFilter : public UdpProxyFilter {
  Network::FilterStatus onDataInternal(Network::UdpRecvData& data) override {
    // Look up existing session by 4-tuple
    auto session_it = sessions_.find(data.addresses_);
    
    if (session_it != sessions_.end()) {
      // Reuse existing session and upstream host
      (*session_it)->onData(data);
    } else {
      // Create new session, select upstream host
      auto* session = createSession(
          std::move(data.addresses_),
          nullptr,  // Let session select host
          false);   // Create socket immediately
      
      session->onData(data);
    }
    
    return Network::FilterStatus::StopIteration;
  }
};
```

**Session Persistence:**
```
Client 192.0.2.1:12345 sends packets
          │
          ▼
First packet → Create session → Select upstream 10.0.0.5:53
          │
          ▼
Session maps (192.0.2.1:12345 → 10.0.0.5:53)
          │
          ▼
All subsequent packets from 192.0.2.1:12345
          │
          └──→ Route to same upstream 10.0.0.5:53
```

#### 2. Per-Packet Load Balancing

```cpp
class PerPacketLoadBalancingUdpProxyFilter : public UdpProxyFilter {
  Network::FilterStatus onDataInternal(Network::UdpRecvData& data) override {
    // Select new upstream host for EACH packet
    ClusterInfo* cluster = getClusterInfo(data.addresses_);
    auto host_selection = cluster->chooseHost(data.addresses_.peer_address_);
    
    if (!host_selection.host_) {
      return Network::FilterStatus::StopIteration;
    }
    
    // Create ephemeral session for this packet
    auto* session = createSession(
        std::move(data.addresses_),
        host_selection.host_,
        false);
    
    session->onData(data);
    
    // Session destroyed after packet is sent
    return Network::FilterStatus::StopIteration;
  }
};
```

**Use Cases:**
- Stateless protocols (e.g., DNS)
- Improved load distribution
- Automatic failover

## UDP Session Filters

Similar to HTTP filters, UDP sessions support extensible filter chains:

```cpp
class UdpSessionFilter {
public:
  virtual ~UdpSessionFilter() = default;
  
  // Read filter interface
  virtual ReadFilterStatus onNewSession() { return ReadFilterStatus::Continue; }
  virtual ReadFilterStatus onData(Network::UdpRecvData& data) { 
    return ReadFilterStatus::Continue; 
  }
  
  // Write filter interface
  virtual WriteFilterStatus onWrite(Network::UdpRecvData& data) {
    return WriteFilterStatus::Continue;
  }
};
```

### Filter Chain Execution

```
Downstream Packet Received
          │
          ▼
┌─────────────────────┐
│  Read Filter Chain  │
│                     │
│  ┌───────────────┐ │
│  │ Filter 1      │ │
│  │ (Auth)        │ │
│  └───────┬───────┘ │
│          │ Continue │
│  ┌───────▼───────┐ │
│  │ Filter 2      │ │
│  │ (Rate Limit)  │ │
│  └───────┬───────┘ │
│          │ Continue │
│  ┌───────▼───────┐ │
│  │ Filter 3      │ │
│  │ (Logging)     │ │
│  └───────────────┘ │
└─────────┬───────────┘
          │
          ▼
Forward to Upstream
          │
          ▼
Upstream Response
          │
          ▼
┌─────────────────────┐
│ Write Filter Chain  │
│                     │
│  ┌───────────────┐ │
│  │ Filter 3      │ │
│  │ (Logging)     │ │
│  └───────┬───────┘ │
│          │ Continue │
│  ┌───────▼───────┐ │
│  │ Filter 2      │ │
│  │ (Encryption)  │ │
│  └───────┬───────┘ │
│          │ Continue │
│  ┌───────▼───────┐ │
│  │ Filter 1      │ │
│  │ (Metrics)     │ │
│  └───────────────┘ │
└─────────┬───────────┘
          │
          ▼
Send to Downstream
```

### Example Filter Implementation

```cpp
class UdpLoggingFilter : public UdpSessionFilter {
public:
  ReadFilterStatus onNewSession() override {
    ENVOY_LOG(info, "New UDP session created: {}",
              callbacks_->sessionId());
    return ReadFilterStatus::Continue;
  }
  
  ReadFilterStatus onData(Network::UdpRecvData& data) override {
    ENVOY_LOG(debug, "Session {} received {} bytes from {}",
              callbacks_->sessionId(),
              data.buffer_->length(),
              data.addresses_.peer_address_->asStringView());
    return ReadFilterStatus::Continue;
  }
  
  WriteFilterStatus onWrite(Network::UdpRecvData& data) override {
    ENVOY_LOG(debug, "Session {} sending {} bytes to {}",
              callbacks_->sessionId(),
              data.buffer_->length(),
              data.addresses_.peer_address_->asStringView());
    return WriteFilterStatus::Continue;
  }

private:
  ReadFilterCallbacks* callbacks_;
};
```

## UDP Tunneling

Envoy supports tunneling UDP traffic over HTTP connections for firewall traversal.

### CONNECT-UDP Method

```
Client                    Envoy (Proxy)              Upstream
  │                            │                         │
  │ CONNECT-UDP                │                         │
  │ udp://target:1234          │                         │
  │ HTTP/2 or HTTP/3           │                         │
  ├───────────────────────────>│                         │
  │                            │                         │
  │ HTTP 200 OK                │                         │
  │<───────────────────────────┤                         │
  │                            │                         │
  │ HTTP DATA frame            │ UDP packet              │
  │ (contains UDP datagram)    │ (extracted from frame)  │
  ├───────────────────────────>├────────────────────────>│
  │                            │                         │
  │                            │ UDP packet              │
  │ HTTP DATA frame            │ (wrapped in frame)      │
  │<───────────────────────────┤<────────────────────────┤
  │                            │                         │
```

### HTTP POST Method

Alternative tunneling using HTTP POST requests:

```
Client                    Envoy (Proxy)              Upstream
  │                            │                         │
  │ POST /target?port=1234     │                         │
  │ Content-Type: application/ │                         │
  │   connect-udp-datagram     │                         │
  ├───────────────────────────>│                         │
  │                            │                         │
  │ HTTP 200 OK                │                         │
  │<───────────────────────────┤                         │
  │                            │                         │
  │ (Stream remains open)      │                         │
  │                            │                         │
  │ POST body (UDP datagram)   │ UDP packet              │
  ├───────────────────────────>├────────────────────────>│
  │                            │                         │
```

### Tunneling Configuration

```cpp
class UdpTunnelingConfig {
  // Target host for UDP packets
  virtual const std::string targetHost(
      const StreamInfo::StreamInfo& stream_info) const PURE;
  
  // Use POST instead of CONNECT-UDP
  virtual bool usePost() const PURE;
  
  // POST path template
  virtual const std::string& postPath() const PURE;
  
  // Maximum connection attempts
  virtual uint32_t maxConnectAttempts() const PURE;
  
  // Buffer datagrams during connection establishment
  virtual bool bufferEnabled() const PURE;
  virtual uint32_t maxBufferedDatagrams() const PURE;
  virtual uint64_t maxBufferedBytes() const PURE;
};
```

## DNS Resolution over UDP

Envoy can act as a DNS proxy, forwarding DNS queries to upstream resolvers:

```
DNS Client                Envoy                  DNS Server
  │                         │                         │
  │ DNS Query (UDP)         │                         │
  │ example.com A?          │                         │
  ├────────────────────────>│                         │
  │                         │ DNS Query (UDP)         │
  │                         ├────────────────────────>│
  │                         │                         │
  │                         │ DNS Response            │
  │                         │ 93.184.216.34           │
  │ DNS Response            │<────────────────────────┤
  │<────────────────────────┤                         │
  │                         │                         │
```

### DNS Proxy Configuration

```yaml
listeners:
- name: dns_listener
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 53
      protocol: UDP
  
  listener_filters:
  - name: envoy.filters.udp_listener.udp_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.UdpProxyConfig
      stat_prefix: dns_proxy
      
      cluster: dns_cluster
      
      # Session timeout for DNS queries
      session_timeout: 5s
      
      # Hash policy for load balancing
      hash_policies:
      - source_ip: true

clusters:
- name: dns_cluster
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  load_assignment:
    cluster_name: dns_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: 8.8.8.8
              port_value: 53
      - endpoint:
          address:
            socket_address:
              address: 8.8.4.4
              port_value: 53
```

## Performance Optimizations

### Generic Receive Offload (GRO)

GRO coalesces multiple packets into a single large buffer:

```yaml
udp_listener_config:
  downstream_socket_config:
    prefer_gro: true
```

**Benefits:**
- Reduced syscall overhead
- Better CPU cache utilization
- Higher throughput for bulk transfers

### Generic Segmentation Offload (GSO)

GSO allows sending large buffers that the kernel segments into packets:

```cpp
// Send large buffer (e.g., 16KB)
Buffer::OwnedImpl large_buffer;
// Fill buffer...

// Kernel segments into MTU-sized packets
socket->send(large_buffer);
```

**Benefits:**
- Fewer application-level send calls
- Kernel-level optimization
- Improved throughput

### Packet Batching

```yaml
udp_listener_config:
  # Number of packets to process per event loop iteration
  packets_per_event_loop: 32
```

### Original Source IP

Preserve client IP address when forwarding:

```yaml
use_original_src_ip: true
```

**Requirements:**
- Transparent proxy support (`IP_TRANSPARENT`)
- Proper iptables/routing configuration
- CAP_NET_ADMIN capability

## Observability

### UDP Proxy Stats

```cpp
#define ALL_UDP_PROXY_DOWNSTREAM_STATS(COUNTER, GAUGE)
  COUNTER(downstream_sess_no_route)
  COUNTER(downstream_sess_rx_bytes)
  COUNTER(downstream_sess_rx_datagrams)
  COUNTER(downstream_sess_rx_errors)
  COUNTER(downstream_sess_total)
  COUNTER(downstream_sess_tx_bytes)
  COUNTER(downstream_sess_tx_datagrams)
  COUNTER(downstream_sess_tx_errors)
  COUNTER(idle_timeout)
  GAUGE(downstream_sess_active, Accumulate)
```

### Access Logging

```yaml
session_access_log:
- name: envoy.access_loggers.file
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
    path: /var/log/envoy/udp_sessions.log
    format: "[%START_TIME%] session_id=%REQ(:authority)% "
            "bytes_received=%BYTES_RECEIVED% "
            "bytes_sent=%BYTES_SENT% "
            "duration=%DURATION%\n"
```

### Monitoring Queries

```bash
# Active UDP sessions
curl -s http://localhost:9901/stats | grep downstream_sess_active

# Total sessions created
curl -s http://localhost:9901/stats | grep downstream_sess_total

# Packet drop rate
curl -s http://localhost:9901/stats | grep packets_dropped
```

## Troubleshooting

### Common Issues

1. **Packet Drops**
   - Check `packets_dropped` metric
   - Increase `max_rx_datagram_size`
   - Enable GRO for better performance

2. **Session Exhaustion**
   - Review `downstream_sess_active`
   - Adjust `session_timeout`
   - Check for session leaks

3. **NAT/Firewall Issues**
   - Verify UDP ports are open
   - Check stateful firewall timeouts
   - Consider using UDP tunneling over HTTP

4. **Performance Bottlenecks**
   - Enable GRO/GSO
   - Increase `packets_per_event_loop`
   - Use per-packet load balancing for stateless protocols

## Next Steps

- [Configuration Examples](04-configuration-examples.md) - Practical UDP configurations
- [QUIC Implementation](02-quic-implementation.md) - QUIC-specific details
- [QUIC Overview](01-quic-overview.md) - High-level QUIC architecture

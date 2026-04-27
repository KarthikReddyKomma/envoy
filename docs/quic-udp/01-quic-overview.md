# QUIC Protocol Support in Envoy

## What is QUIC?

QUIC (Quick UDP Internet Connections) is a modern transport protocol that provides the functionality of TCP, TLS, and HTTP/2 while operating over UDP. It was originally developed by Google and is now an IETF standard (RFC 9000).

### Key Benefits

1. **Reduced Latency**: 0-RTT and 1-RTT connection establishment
2. **No Head-of-Line Blocking**: Independent stream multiplexing
3. **Connection Migration**: Seamless handoff between networks
4. **Built-in Encryption**: TLS 1.3 integrated into the protocol
5. **Improved Congestion Control**: Modern loss detection and recovery

## QUIC in Envoy Architecture

Envoy's QUIC implementation is built on top of Google's QUICHE library and integrates seamlessly with the existing HTTP connection manager infrastructure.

```
┌────────────────────────────────────────────────────────────────┐
│                     HTTP Connection Manager                     │
│                  (Protocol-Agnostic Layer)                      │
└────────────────────────────────────────────────────────────────┘
                              │
                              │ Delegates to codec
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                   HTTP/3 Codec Implementation                   │
│        (ServerCodecImpl / ClientCodecImpl)                      │
│  - Translates HCM calls to QUIC session operations             │
│  - Manages HTTP/3 semantics                                     │
└────────────────────────────────────────────────────────────────┘
                              │
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│              QUIC Session Layer                                 │
│   ┌──────────────────────┐      ┌──────────────────────┐      │
│   │EnvoyQuicServerSession│      │EnvoyQuicClientSession│      │
│   │- Connection lifecycle│      │- Connection lifecycle│      │
│   │- Stream management   │      │- Connection pooling  │      │
│   │- ALPN: h3            │      │- 0-RTT support       │      │
│   └──────────────────────┘      └──────────────────────┘      │
└────────────────────────────────────────────────────────────────┘
                              │
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│           QuicFilterManagerConnectionImpl                       │
│  - Implements Network::Connection interface                     │
│  - Buffer management and watermarks                             │
│  - Network filter chain execution                               │
│  - Stats and metrics                                            │
└────────────────────────────────────────────────────────────────┘
                              │
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│              QUIC Connection (QUICHE Library)                   │
│  - Packet framing and parsing                                   │
│  - Encryption/Decryption (TLS 1.3)                             │
│  - Flow control                                                 │
│  - Congestion control                                           │
│  - Loss detection and recovery                                  │
│  - Connection ID management                                     │
└────────────────────────────────────────────────────────────────┘
                              │
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│              Transport Layer (UDP)                              │
│  - EnvoyQuicPacketWriter                                        │
│  - UdpListener (server) / IoSocketHandleImpl (client)          │
│  - Socket I/O operations                                        │
└────────────────────────────────────────────────────────────────┘
```

## Server-Side QUIC (HTTP/3)

### Components

#### 1. QuicServerTransportSocketFactory

The transport socket factory creates and manages QUIC server connections:

```cpp
class QuicServerTransportSocketFactory : 
    public Network::DownstreamTransportSocketFactory
{
  - Creates SSL server contexts
  - Manages TLS certificates and keys
  - Configures early data (0-RTT) support
  - Handles certificate rotation
};
```

**Responsibilities:**
- Initialize TLS configuration for QUIC connections
- Provide certificate chains and private keys based on SNI
- Enable/disable 0-RTT (early data) support
- React to certificate updates (secret rotation)

#### 2. EnvoyQuicServerSession

The server session manages the lifecycle of a single QUIC connection:

```cpp
class EnvoyQuicServerSession : 
    public quic::QuicServerSessionBase,
    public QuicFilterManagerConnectionImpl
{
  - HTTP/3 stream creation and management
  - Connection state tracking
  - Filter chain integration
  - Connection migration handling
  - GoAway signaling for load shedding
};
```

**Key Features:**
- **Stream Management**: Creates bidirectional HTTP/3 streams
- **Filter Integration**: Works with network filters
- **Load Shedding**: Supports H3 GoAway for graceful shutdown
- **Session Idle List**: Tracks idle sessions for cleanup
- **ALPN**: Negotiates "h3" or "h3-29" protocols

#### 3. EnvoyQuicServerStream

Individual HTTP/3 streams that carry request/response pairs:

```cpp
class EnvoyQuicServerStream : public quic::QuicSpdyServerStreamBase
{
  - Request/response processing
  - Header and data frame handling
  - Stream reset handling
  - Backpressure management
};
```

### Connection Lifecycle

```
1. Client sends QUIC Initial packet
          │
          ▼
2. UdpListener receives packet
          │
          ▼
3. QuicServerTransportSocketFactory creates connection
          │
          ▼
4. TLS 1.3 handshake (0-RTT or 1-RTT)
          │
          ▼
5. ALPN negotiation (h3)
          │
          ▼
6. EnvoyQuicServerSession initialized
          │
          ▼
7. HTTP/3 streams created on demand
          │
          ▼
8. Request/response processing
          │
          ▼
9. Stream completion or connection close
```

### Configuration

Basic HTTP/3 listener configuration:

```yaml
listeners:
- name: quic_listener
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 443
      protocol: UDP
  
  udp_listener_config:
    quic_options: {}
    downstream_socket_config:
      prefer_gro: true
  
  filter_chains:
  - filter_chain_match:
      transport_protocol: "quic"
    
    transport_socket:
      name: envoy.transport_sockets.quic
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.quic.v3.QuicDownstreamTransport
        downstream_tls_context:
          common_tls_context:
            tls_certificates:
            - certificate_chain:
                filename: "/path/to/cert.pem"
              private_key:
                filename: "/path/to/key.pem"
          enable_early_data: true
    
    filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: ingress_http
        codec_type: HTTP3
        route_config:
          # ... route configuration
        http_filters:
        - name: envoy.filters.http.router
```

## Client-Side QUIC

### Components

#### 1. QuicClientTransportSocketFactory

Creates QUIC client connections:

```cpp
class QuicClientTransportSocketFactory :
    public Network::UpstreamTransportSocketFactory
{
  - Manages TLS client configuration
  - Handles certificate verification
  - Supports custom QUIC versions
};
```

#### 2. EnvoyQuicClientSession

Client-side QUIC session with connection pooling support:

```cpp
class EnvoyQuicClientSession :
    public quic::QuicSpdyClientSession,
    public QuicFilterManagerConnectionImpl
{
  - Connection establishment
  - Stream creation for requests
  - Connection migration support
  - RTT caching
  - Server preferred address handling
};
```

**Key Features:**
- **Connection Pooling**: Integrates with Envoy's connection pool
- **0-RTT Support**: Can send early data when resuming connections
- **Migration**: Handles network changes (WiFi to cellular)
- **RTT Cache**: Stores server RTT for connection optimization

#### 3. EnvoyQuicClientConnection

Manages the underlying QUIC connection and socket:

```cpp
class EnvoyQuicClientConnection : 
    public quic::QuicConnection,
    public Network::UdpPacketProcessor
{
  - Packet I/O operations
  - Path validation for migration
  - Port migration on path degradation
  - Address caching (optimization)
};
```

### Connection Lifecycle

```
1. Cluster selects upstream host
          │
          ▼
2. Connection pool requests QUIC connection
          │
          ▼
3. EnvoyQuicClientConnection created
          │
          ▼
4. Socket opened with address cache
          │
          ▼
5. TLS handshake (0-RTT if resumed)
          │
          ▼
6. ALPN confirms h3
          │
          ▼
7. EnvoyQuicClientSession ready
          │
          ▼
8. HTTP/3 streams created for requests
          │
          ▼
9. Optional: Connection migration on network change
          │
          ▼
10. Connection reused or closed
```

## QUIC Connection Management

### Connection IDs

QUIC uses connection IDs instead of 5-tuples (IP:port pairs) for connection identification:

- **Purpose**: Enable connection migration
- **Format**: Opaque byte strings (typically 8-20 bytes)
- **Rotation**: New CIDs issued during connection lifetime
- **Generator**: Envoy uses `EnvoyQuicConnectionIdGeneratorFactory`

### Connection Migration

QUIC supports changing IP addresses or ports mid-connection:

#### Server-Side Migration Detection

```cpp
void EnvoyQuicServerSession::ProcessUdpPacket(
    const quic::QuicSocketAddress& self_address,
    const quic::QuicSocketAddress& peer_address,
    const quic::QuicReceivedPacket& packet)
{
  // Detects when client changes address
  // Updates connection stats
  // Validates new path
}
```

#### Client-Side Migration

1. **Path Degradation**: Triggers port migration when packet loss detected
2. **Server Preferred Address**: Migrates to server-provided address
3. **Network Change**: Handles WiFi to cellular transitions

```
Path Degradation Detected
          │
          ▼
Create new socket with different port
          │
          ▼
Path validation (ping/pong)
          │
          ├─ Success → Migrate to new path
          │
          └─ Failure → Revert to old path
```

### Stream Multiplexing

QUIC provides true multiplexing without head-of-line blocking:

```
Connection ID: 0x123456789abcdef0
│
├─ Stream 0 (client-initiated bidirectional)
│  └─ HTTP Request/Response #1
│
├─ Stream 4 (client-initiated bidirectional)
│  └─ HTTP Request/Response #2
│
├─ Stream 8 (client-initiated bidirectional)
│  └─ HTTP Request/Response #3
│
└─ Control Streams (unidirectional)
   ├─ QPACK encoder/decoder streams
   └─ HTTP/3 control stream
```

**Stream ID Allocation:**
- Client-initiated bidirectional: 0, 4, 8, 12, ...
- Server-initiated bidirectional: 1, 5, 9, 13, ...
- Client-initiated unidirectional: 2, 6, 10, 14, ...
- Server-initiated unidirectional: 3, 7, 11, 15, ...

## Integration with HTTP Connection Manager

The HTTP Connection Manager (HCM) is protocol-agnostic and works with QUIC through the codec interface:

```cpp
// HCM creates codec based on protocol
codec_type: HTTP3

// Codec implementation delegates to QUIC session
class ServerCodecImpl : public Http::ServerConnection {
  EnvoyQuicServerSession& quic_session_;
  
  // HCM calls newStream()
  Http::RequestDecoder& newStream(Http::ResponseEncoder&) override {
    // Creates new QUIC stream
    return quic_session_.CreateIncomingStream();
  }
};
```

### Request Flow

```
1. Client sends HTTP/3 HEADERS frame over QUIC
          │
          ▼
2. EnvoyQuicServerStream receives frame
          │
          ▼
3. Stream notifies ServerCodecImpl
          │
          ▼
4. Codec converts to Envoy headers
          │
          ▼
5. HCM receives decoded request
          │
          ▼
6. HCM processes through filter chain
          │
          ▼
7. Response sent back through codec
          │
          ▼
8. ServerCodecImpl writes to QUIC stream
          │
          ▼
9. QUIC stream sends DATA frame
```

## QUIC Version Support

Envoy supports multiple QUIC versions through QUICHE:

- **IETF QUIC (v1)**: RFC 9000 - Production standard
- **Draft versions**: h3-29, h3-27 (legacy support)
- **Version Negotiation**: Automatic selection based on client support

### ALPN (Application-Layer Protocol Negotiation)

QUIC uses ALPN to negotiate HTTP/3:

- **"h3"**: HTTP/3 over IETF QUIC v1
- **"h3-29"**: HTTP/3 over draft-29

## Statistics and Observability

### Connection Stats

```cpp
#define QUIC_CONNECTION_STATS(COUNTER)
  COUNTER(num_server_migration_detected)
  COUNTER(num_packets_rx_on_preferred_address)
```

### Session Stats

Exposed through standard Envoy stats:
- `downstream_cx_total`: Total QUIC connections
- `downstream_cx_active`: Active QUIC connections
- `downstream_cx_destroy`: Connection closures
- `downstream_cx_protocol_error`: Protocol errors

### Stream Stats

Standard HTTP stats apply:
- `downstream_rq_total`: Total requests
- `downstream_rq_xx`: Response codes
- `downstream_rq_time`: Request latency

## Security Considerations

### TLS 1.3 Integration

QUIC mandates TLS 1.3:
- Handshake protection from packet loss
- 0-RTT replay protection considerations
- Certificate verification

### 0-RTT (Early Data)

Enabling 0-RTT reduces latency but requires care:

**Risks:**
- Replay attacks for non-idempotent requests
- Application must handle replayed data

**Mitigation:**
- Only enable for idempotent operations
- Use anti-replay tokens
- Configure carefully per route

```yaml
enable_early_data: true  # Use with caution
```

### Connection Security

- All QUIC connections are encrypted
- No plaintext QUIC support
- TLS 1.3 is mandatory

## Performance Characteristics

### Latency Improvements

| Connection Type | Handshake RTTs | Total Latency |
|----------------|----------------|---------------|
| TCP + TLS 1.3 | 2-RTT | 2-RTT + request |
| QUIC (1-RTT) | 1-RTT | 1-RTT + request |
| QUIC (0-RTT) | 0-RTT | 0-RTT + request |

### Throughput Considerations

- **Congestion Control**: Modern algorithms (Cubic, BBR)
- **Loss Recovery**: Separate per-stream
- **Flow Control**: Connection and stream level
- **GSO/GRO**: Kernel offload support

## Troubleshooting

### Common Issues

1. **Connection Refused**
   - Check UDP port is open (typically 443)
   - Verify QUIC is enabled in listener config
   - Check firewall rules for UDP

2. **Version Mismatch**
   - Client and server must share common QUIC version
   - Check ALPN negotiation in logs

3. **Certificate Errors**
   - QUIC requires valid TLS 1.3 certificates
   - Verify certificate chain and SNI

4. **Migration Failures**
   - Check network path validation
   - Review firewall rules for UDP from different ports

### Debug Logging

Enable detailed QUIC logging:

```yaml
admin:
  access_log_path: /dev/null
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901

# Set log level via admin API:
# POST /logging?level=trace
# POST /logging?quic=trace
```

## Next Steps

- [QUIC Implementation Details](02-quic-implementation.md) - Deep dive into implementation
- [Configuration Examples](04-configuration-examples.md) - Practical configuration examples
- [UDP Listeners](03-udp-listeners.md) - Understanding the UDP layer

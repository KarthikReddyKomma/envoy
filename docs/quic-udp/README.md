# QUIC and UDP Support in Envoy

This directory contains comprehensive documentation about Envoy's QUIC (HTTP/3) and UDP support.

## Overview

Envoy provides full support for QUIC protocol (HTTP/3) and UDP-based communications, enabling modern, high-performance networking capabilities. QUIC combines the benefits of TCP, TLS, and HTTP/2 while operating over UDP, providing:

- Reduced connection establishment latency (0-RTT and 1-RTT)
- Improved multiplexing without head-of-line blocking
- Connection migration support
- Built-in encryption

## Documentation Structure

### 1. [QUIC Overview](01-quic-overview.md)
- QUIC/HTTP3 architecture in Envoy
- Connection management and lifecycle
- Integration with HTTP connection manager
- QUIC versions and protocol support

### 2. [QUIC Implementation](02-quic-implementation.md)
- Deep dive into QUIC implementation details
- Transport socket factories
- Connection and session management
- Address caching optimization for client connections
- Stream management and multiplexing

### 3. [UDP Listeners](03-udp-listeners.md)
- UDP listener architecture
- UDP session management
- UDP proxy functionality
- DNS resolution over UDP
- Packet processing and filtering

### 4. [Configuration Examples](04-configuration-examples.md)
- HTTP/3 listener setup
- QUIC client configuration
- UDP proxy configuration
- Performance tuning guidelines
- TLS/certificate configuration

## Key Features

### QUIC/HTTP3 Support
- **Server-side**: Full HTTP/3 server implementation with TLS 1.3
- **Client-side**: HTTP/3 client with connection pooling and retry logic
- **Connection Migration**: Support for IP and port migration
- **0-RTT**: Optional early data support for reduced latency

### UDP Support
- **UDP Listeners**: High-performance UDP packet processing
- **UDP Proxy**: Session-based UDP forwarding with load balancing
- **UDP Tunneling**: UDP-over-HTTP (CONNECT-UDP and HTTP POST)
- **Session Filters**: Extensible filter chain for UDP sessions

## Architecture Highlights

```
┌─────────────────────────────────────────────────────────────┐
│                    Envoy QUIC/UDP Stack                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  HTTP/3 Layer                                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ EnvoyQuicServerSession / EnvoyQuicClientSession      │  │
│  │ - HTTP/3 stream management                            │  │
│  │ - QPACK header compression                            │  │
│  │ - HTTP semantics over QUIC                            │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                 │
│  QUIC Layer                                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ QuicFilterManagerConnectionImpl                       │  │
│  │ - Connection state management                         │  │
│  │ - Stream multiplexing                                 │  │
│  │ - Flow control                                        │  │
│  │ - Congestion control                                  │  │
│  │ - Loss detection and recovery                         │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                 │
│  UDP Layer                                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ UdpListener / IoSocketHandleImpl                      │  │
│  │ - UDP socket management                               │  │
│  │ - Packet I/O (recvmsg/sendmsg)                        │  │
│  │ - Address caching for QUIC clients                    │  │
│  │ - GSO/GRO support                                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                 │
│  OS Network Stack (UDP)                                     │
└─────────────────────────────────────────────────────────────┘
```

## Use Cases

### 1. HTTP/3 Gateway
Serve HTTP/3 traffic to modern clients while maintaining HTTP/2 and HTTP/1.1 compatibility for older clients.

### 2. QUIC-based Service Mesh
Use QUIC for service-to-service communication to reduce latency and improve connection reliability.

### 3. UDP Proxy and Load Balancing
Load balance UDP traffic (DNS, gaming protocols, VoIP) with session affinity and health checking.

### 4. UDP Tunneling
Tunnel UDP traffic over HTTP/3 or HTTP/2 for traversing firewalls and proxies.

## Getting Started

1. Start with [01-quic-overview.md](01-quic-overview.md) to understand the architecture
2. Review [04-configuration-examples.md](04-configuration-examples.md) for practical examples
3. Dive into [02-quic-implementation.md](02-quic-implementation.md) for implementation details
4. Explore [03-udp-listeners.md](03-udp-listeners.md) for UDP-specific functionality

## Performance Considerations

- **Address Caching**: QUIC client sockets use address caching (default size: 4) to avoid repeated address allocations
- **Batch Processing**: UDP listeners support recvmmsg for efficient packet batching
- **GSO/GRO**: Generic Segmentation/Receive Offload support for improved throughput
- **Zero-Copy**: Minimize data copies through efficient buffer management

## Related Resources

- [Envoy HTTP/3 Documentation](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http3)
- [QUIC Protocol (RFC 9000)](https://www.rfc-editor.org/rfc/rfc9000.html)
- [HTTP/3 (RFC 9114)](https://www.rfc-editor.org/rfc/rfc9114.html)
- [QUICHE Library](https://github.com/google/quiche)

## Version Support

Envoy's QUIC implementation is based on Google's QUICHE library and supports:
- IETF QUIC (RFC 9000)
- HTTP/3 (RFC 9114)
- QPACK (RFC 9204)
- TLS 1.3 (RFC 8446)

## Contributing

When modifying QUIC or UDP code, please ensure:
1. Performance benchmarks are run
2. Connection migration tests pass
3. Address caching behavior is validated for client connections
4. Documentation is updated accordingly

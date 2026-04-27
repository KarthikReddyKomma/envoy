# QUIC and UDP Configuration Examples

This document provides practical configuration examples for QUIC/HTTP3 and UDP functionality in Envoy.

## Table of Contents

1. [HTTP/3 Listener Configuration](#http3-listener-configuration)
2. [QUIC Client Configuration](#quic-client-configuration)
3. [UDP Proxy Configuration](#udp-proxy-configuration)
4. [UDP Tunneling Configuration](#udp-tunneling-configuration)
5. [Performance Tuning](#performance-tuning)
6. [TLS Configuration for QUIC](#tls-configuration-for-quic)
7. [Complete Examples](#complete-examples)

## HTTP/3 Listener Configuration

### Basic HTTP/3 Listener

```yaml
static_resources:
  listeners:
  - name: http3_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 443
        protocol: UDP  # QUIC runs over UDP
    
    # UDP listener configuration
    udp_listener_config:
      quic_options: {}
      downstream_socket_config:
        # Enable Generic Receive Offload for better performance
        prefer_gro: true
        max_rx_datagram_size: 1500
    
    filter_chains:
    # QUIC/HTTP3 filter chain
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
                  filename: "/etc/envoy/certs/server.crt"
                private_key:
                  filename: "/etc/envoy/certs/server.key"
              alpn_protocols:
              - h3
            
            # Enable 0-RTT (use with caution for non-idempotent operations)
            enable_early_data: true
      
      filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: http3_ingress
          codec_type: HTTP3
          
          # HTTP/3 specific options
          http3_protocol_options:
            quic_protocol_options:
              max_concurrent_streams: 100
              initial_stream_window_size: 65536    # 64 KB
              initial_connection_window_size: 1048576  # 1 MB
            
            # Allow extended CONNECT for UDP tunneling
            allow_extended_connect: true
            
            # Override stream idle timeout
            override_stream_error_on_invalid_http_message: true
          
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: backend_cluster
          
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
    
    # TCP fallback filter chain (for HTTP/2, HTTP/1.1)
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: tcp_ingress
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: backend_cluster
          http_filters:
          - name: envoy.filters.http.router

  clusters:
  - name: backend_cluster
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: backend_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: backend.example.com
                port_value: 8080
```

### HTTP/3 with Alt-Svc Advertisement

Advertise HTTP/3 support to HTTP/2 clients:

```yaml
listeners:
- name: https_listener
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 443
  
  filter_chains:
  - filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: https
        codec_type: AUTO
        
        route_config:
          name: local_route
          
          # Add Alt-Svc header to responses
          response_headers_to_add:
          - header:
              key: "alt-svc"
              value: 'h3=":443"; ma=86400'  # Advertise HTTP/3 on same port
          
          virtual_hosts:
          - name: backend
            domains: ["*"]
            routes:
            - match:
                prefix: "/"
              route:
                cluster: backend_cluster
        
        http_filters:
        - name: envoy.filters.http.router

# Also configure HTTP/3 listener as shown above
- name: http3_listener
  # ... (configuration from previous example)
```

### HTTP/3 with Connection Migration

```yaml
listeners:
- name: http3_listener
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 443
      protocol: UDP
  
  udp_listener_config:
    quic_options:
      # Enable connection migration
      quic_connection_options:
        - NCON  # No connection options
        - NSTP  # No stop waiting frame
        - CHLO  # Include client hello
        
      # Server preferred address (optional)
      # server_preferred_address:
      #   ipv4_address: "10.0.0.100:8443"
      #   ipv6_address: "[2001:db8::1]:8443"
      
      enabled: true
    
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
                filename: "/etc/envoy/certs/server.crt"
              private_key:
                filename: "/etc/envoy/certs/server.key"
    
    filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: http3
        codec_type: HTTP3
        
        http3_protocol_options:
          quic_protocol_options:
            max_concurrent_streams: 100
            initial_stream_window_size: 262144      # 256 KB
            initial_connection_window_size: 1048576  # 1 MB
        
        route_config:
          name: local_route
          virtual_hosts:
          - name: backend
            domains: ["*"]
            routes:
            - match:
                prefix: "/"
              route:
                cluster: backend_cluster
        
        http_filters:
        - name: envoy.filters.http.router
```

## QUIC Client Configuration

### Basic QUIC Upstream Cluster

```yaml
clusters:
- name: quic_upstream
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  
  # Use HTTP/3 for upstream connections
  typed_extension_protocol_options:
    envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
      "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
      explicit_http_config:
        http3_protocol_options:
          quic_protocol_options:
            max_concurrent_streams: 100
            initial_stream_window_size: 65536
            initial_connection_window_size: 1048576
          
          # Override stream idle timeout
          override_stream_error_on_invalid_http_message: true
  
  # QUIC transport socket
  transport_socket:
    name: envoy.transport_sockets.quic
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.quic.v3.QuicUpstreamTransport
      upstream_tls_context:
        common_tls_context:
          # Validate server certificate
          validation_context:
            trusted_ca:
              filename: "/etc/ssl/certs/ca-certificates.crt"
          
          alpn_protocols:
          - h3
        
        # SNI for certificate validation
        sni: backend.example.com
  
  load_assignment:
    cluster_name: quic_upstream
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: backend.example.com
              port_value: 443
  
  # Connection pool settings
  circuit_breakers:
    thresholds:
    - priority: DEFAULT
      max_connections: 1024
      max_pending_requests: 1024
      max_requests: 1024
      max_retries: 3
```

### QUIC with Connection Pooling and RTT Cache

```yaml
clusters:
- name: quic_upstream_with_cache
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  
  typed_extension_protocol_options:
    envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
      "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
      explicit_http_config:
        http3_protocol_options:
          quic_protocol_options:
            max_concurrent_streams: 100
            initial_stream_window_size: 262144
            initial_connection_window_size: 1048576
          
          # Enable 0-RTT for resumption
          allow_extended_connect: true
  
  transport_socket:
    name: envoy.transport_sockets.quic
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.quic.v3.QuicUpstreamTransport
      upstream_tls_context:
        common_tls_context:
          validation_context:
            trusted_ca:
              filename: "/etc/ssl/certs/ca-certificates.crt"
          alpn_protocols:
          - h3
        sni: backend.example.com
  
  # HTTP server properties cache for RTT and 0-RTT
  http_server_properties_cache_options:
    max_entries: 1000
    cache_entry_timeout: 3600s
  
  load_assignment:
    cluster_name: quic_upstream_with_cache
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: backend.example.com
              port_value: 443
```

## UDP Proxy Configuration

### Basic UDP Proxy (DNS Example)

```yaml
listeners:
- name: dns_proxy
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
      
      # Route to DNS cluster
      cluster: dns_servers
      
      # Session timeout
      session_timeout: 10s
      
      # Hash policy for session affinity
      hash_policies:
      - source_ip: true

clusters:
- name: dns_servers
  type: STATIC
  lb_policy: ROUND_ROBIN
  
  load_assignment:
    cluster_name: dns_servers
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
  
  # Health checks for UDP
  health_checks:
  - timeout: 1s
    interval: 10s
    unhealthy_threshold: 3
    healthy_threshold: 2
    udp_health_check:
      send:
        text: "0000"  # Health check payload
      receive:
      - text: "0000"  # Expected response
```

### UDP Proxy with Session Filters

```yaml
listeners:
- name: udp_proxy_with_filters
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 10000
      protocol: UDP
  
  listener_filters:
  - name: envoy.filters.udp_listener.udp_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.UdpProxyConfig
      stat_prefix: udp_proxy
      cluster: udp_backend
      session_timeout: 60s
      
      # Session filters (applied per UDP session)
      session_filters:
      - name: envoy.filters.udp.session.dynamic_forward_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.session.dynamic_forward_proxy.v3.FilterConfig
          stat_prefix: dynamic_forward
          dns_cache_config:
            name: dynamic_forward_proxy_cache_config
            dns_lookup_family: V4_ONLY
            max_hosts: 1024
      
      # Access logging per session
      session_access_log:
      - name: envoy.access_loggers.file
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
          path: /var/log/envoy/udp_sessions.log
          log_format:
            text_format_source:
              inline_string: "[%START_TIME%] peer=%DOWNSTREAM_REMOTE_ADDRESS% 
                              local=%DOWNSTREAM_LOCAL_ADDRESS% 
                              bytes_rx=%BYTES_RECEIVED% 
                              bytes_tx=%BYTES_SENT% 
                              duration=%DURATION%\n"

clusters:
- name: udp_backend
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  load_assignment:
    cluster_name: udp_backend
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: backend.example.com
              port_value: 10000
```

### UDP Proxy with Original Source IP

```yaml
listeners:
- name: transparent_udp_proxy
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8080
      protocol: UDP
  
  # Required for transparent proxy
  transparent: true
  
  listener_filters:
  - name: envoy.filters.udp_listener.udp_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.UdpProxyConfig
      stat_prefix: transparent_proxy
      cluster: udp_backend
      
      # Preserve client source IP
      use_original_src_ip: true
      
      session_timeout: 30s

clusters:
- name: udp_backend
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  
  # Upstream bind configuration for original source IP
  upstream_bind_config:
    source_address:
      address: 0.0.0.0
      port_value: 0
  
  load_assignment:
    cluster_name: udp_backend
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: 10.0.0.100
              port_value: 8080
```

### Per-Packet Load Balancing

```yaml
listeners:
- name: per_packet_lb
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 5353
      protocol: UDP
  
  listener_filters:
  - name: envoy.filters.udp_listener.udp_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.UdpProxyConfig
      stat_prefix: per_packet_lb
      cluster: stateless_backend
      
      # Load balance each packet independently
      use_per_packet_load_balancing: true
      
      # Short session timeout for stateless protocols
      session_timeout: 1s

clusters:
- name: stateless_backend
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  load_assignment:
    cluster_name: stateless_backend
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: backend1.example.com
              port_value: 5353
      - endpoint:
          address:
            socket_address:
              address: backend2.example.com
              port_value: 5353
      - endpoint:
          address:
            socket_address:
              address: backend3.example.com
              port_value: 5353
```

## UDP Tunneling Configuration

### CONNECT-UDP Tunneling

```yaml
listeners:
- name: udp_tunnel_listener
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9090
      protocol: UDP
  
  listener_filters:
  - name: envoy.filters.udp_listener.udp_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.UdpProxyConfig
      stat_prefix: udp_tunnel
      cluster: tunnel_cluster
      session_timeout: 300s
      
      # UDP tunneling configuration
      tunneling_config:
        proxy_host: "tunnel-proxy.example.com"
        target_host: "target-backend.example.com"
        
        # Default target port if not specified by client
        default_target_port: 8080
        
        # Number of connection attempts
        max_connect_attempts: 3
        
        # Backoff strategy for retries
        retry_options:
          max_interval: 10s
        
        # Buffer datagrams during connection establishment
        buffer_options:
          max_buffered_datagrams: 1024
          max_buffered_bytes: 16384

clusters:
- name: tunnel_cluster
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  
  # Use HTTP/2 for CONNECT-UDP
  typed_extension_protocol_options:
    envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
      "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
      explicit_http_config:
        http2_protocol_options: {}
  
  load_assignment:
    cluster_name: tunnel_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: tunnel-server.example.com
              port_value: 443
  
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      common_tls_context:
        validation_context:
          trusted_ca:
            filename: "/etc/ssl/certs/ca-certificates.crt"
      sni: tunnel-server.example.com
```

### HTTP POST Tunneling

```yaml
listeners:
- name: udp_post_tunnel
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9091
      protocol: UDP
  
  listener_filters:
  - name: envoy.filters.udp_listener.udp_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.UdpProxyConfig
      stat_prefix: udp_post_tunnel
      cluster: post_tunnel_cluster
      session_timeout: 300s
      
      tunneling_config:
        proxy_host: "post-proxy.example.com"
        target_host: "target-backend.example.com"
        default_target_port: 8080
        
        # Use POST instead of CONNECT-UDP
        use_post: true
        post_path: "/udp-tunnel"
        
        # Custom headers
        headers_to_add:
        - header:
            key: "X-Custom-Header"
            value: "udp-tunnel"
        
        buffer_options:
          max_buffered_datagrams: 512
          max_buffered_bytes: 8192

clusters:
- name: post_tunnel_cluster
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  
  # HTTP/2 or HTTP/3 for POST tunneling
  typed_extension_protocol_options:
    envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
      "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
      explicit_http_config:
        http2_protocol_options: {}
  
  load_assignment:
    cluster_name: post_tunnel_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: post-server.example.com
              port_value: 443
  
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      common_tls_context:
        validation_context:
          trusted_ca:
            filename: "/etc/ssl/certs/ca-certificates.crt"
      sni: post-server.example.com
```

## Performance Tuning

### High-Performance HTTP/3 Configuration

```yaml
listeners:
- name: high_perf_http3
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 443
      protocol: UDP
  
  # Listener options for performance
  per_connection_buffer_limit_bytes: 32768
  
  udp_listener_config:
    quic_options:
      quic_connection_options:
      - NCON
      - NSTP
      enabled: true
    
    downstream_socket_config:
      # Enable GRO for coalescing packets
      prefer_gro: true
      
      # Larger datagram size for jumbo frames
      max_rx_datagram_size: 9000
  
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
                filename: "/etc/envoy/certs/server.crt"
              private_key:
                filename: "/etc/envoy/certs/server.key"
            
            # TLS session cache for resumption
            tls_params:
              tls_minimum_protocol_version: TLSv1_3
              tls_maximum_protocol_version: TLSv1_3
          
          enable_early_data: true
    
    filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: http3_perf
        codec_type: HTTP3
        
        http3_protocol_options:
          quic_protocol_options:
            # Increased stream limit
            max_concurrent_streams: 1000
            
            # Larger flow control windows
            initial_stream_window_size: 1048576    # 1 MB
            initial_connection_window_size: 15728640  # 15 MB
            
            # Increased packet buffer
            max_inbound_header_list_size: 32768
          
          # Allow HTTP/3 datagrams
          allow_extended_connect: true
        
        # Connection management
        common_http_protocol_options:
          idle_timeout: 300s
          max_connection_duration: 0s
          max_requests_per_connection: 0
        
        # Disable unnecessary features for performance
        stream_idle_timeout: 300s
        request_timeout: 300s
        
        route_config:
          name: local_route
          virtual_hosts:
          - name: backend
            domains: ["*"]
            routes:
            - match:
                prefix: "/"
              route:
                cluster: backend_cluster
                timeout: 15s
        
        http_filters:
        - name: envoy.filters.http.router
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
            # Disable upstream connection timeout for performance
            suppress_envoy_headers: true

clusters:
- name: backend_cluster
  type: STRICT_DNS
  lb_policy: LEAST_REQUEST
  
  # Connection pool tuning
  circuit_breakers:
    thresholds:
    - priority: DEFAULT
      max_connections: 10000
      max_pending_requests: 10000
      max_requests: 100000
      max_retries: 3
  
  # Upstream connection options
  common_http_protocol_options:
    idle_timeout: 300s
  
  load_assignment:
    cluster_name: backend_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: backend.example.com
              port_value: 8080
```

### UDP Proxy Performance Tuning

```yaml
listeners:
- name: high_perf_udp
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 10000
      protocol: UDP
  
  # Increase per-connection buffer
  per_connection_buffer_limit_bytes: 65536
  
  # UDP socket options
  socket_options:
  - description: "Enable socket reuse"
    level: 1  # SOL_SOCKET
    name: 2   # SO_REUSEADDR
    int_value: 1
    state: STATE_PREBIND
  
  - description: "Increase receive buffer"
    level: 1  # SOL_SOCKET
    name: 8   # SO_RCVBUF
    int_value: 16777216  # 16 MB
    state: STATE_PREBIND
  
  - description: "Increase send buffer"
    level: 1  # SOL_SOCKET
    name: 7   # SO_SNDBUF
    int_value: 16777216  # 16 MB
    state: STATE_PREBIND
  
  listener_filters:
  - name: envoy.filters.udp_listener.udp_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.UdpProxyConfig
      stat_prefix: udp_perf
      cluster: udp_backend
      
      # Aggressive session timeout
      session_timeout: 30s
      
      # Hash for better distribution
      hash_policies:
      - source_ip: true
      - key: "x-session-id"
      
      # Upstream socket configuration
      upstream_socket_config:
        max_rx_datagram_size: 9000
        prefer_gro: true

clusters:
- name: udp_backend
  type: STRICT_DNS
  
  # Least connection for UDP
  lb_policy: LEAST_REQUEST
  
  # Larger connection pool
  circuit_breakers:
    thresholds:
    - max_connections: 10000
  
  load_assignment:
    cluster_name: udp_backend
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: backend1.example.com
              port_value: 10000
      - endpoint:
          address:
            socket_address:
              address: backend2.example.com
              port_value: 10000
      - endpoint:
          address:
            socket_address:
              address: backend3.example.com
              port_value: 10000
  
  # Health checking
  health_checks:
  - timeout: 1s
    interval: 5s
    unhealthy_threshold: 2
    healthy_threshold: 2
    udp_health_check:
      send:
        text: "health"
      receive:
      - text: "ok"
```

## TLS Configuration for QUIC

### Mutual TLS (mTLS) for HTTP/3

```yaml
listeners:
- name: mtls_http3
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
                filename: "/etc/envoy/certs/server.crt"
              private_key:
                filename: "/etc/envoy/certs/server.key"
            
            # Require client certificate
            validation_context:
              trusted_ca:
                filename: "/etc/envoy/certs/ca.crt"
              
              # Verify certificate chain
              verify_certificate_spki:
              - "base64-encoded-spki-hash"
              
              # Verify SAN
              match_subject_alt_names:
              - exact: "client.example.com"
            
            alpn_protocols:
            - h3
          
          # Require client certificate
          require_client_certificate: true
    
    filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: mtls_http3
        codec_type: HTTP3
        
        route_config:
          name: local_route
          virtual_hosts:
          - name: backend
            domains: ["*"]
            routes:
            - match:
                prefix: "/"
              route:
                cluster: backend_cluster
        
        http_filters:
        - name: envoy.filters.http.router
```

### Certificate Rotation with SDS

```yaml
listeners:
- name: http3_with_sds
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
            # Certificate from SDS
            tls_certificate_sds_secret_configs:
            - name: server_cert
              sds_config:
                resource_api_version: V3
                api_config_source:
                  api_type: GRPC
                  transport_api_version: V3
                  grpc_services:
                  - envoy_grpc:
                      cluster_name: sds_cluster
            
            # Validation context from SDS
            validation_context_sds_secret_config:
              name: validation_context
              sds_config:
                resource_api_version: V3
                api_config_source:
                  api_type: GRPC
                  transport_api_version: V3
                  grpc_services:
                  - envoy_grpc:
                      cluster_name: sds_cluster
    
    filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: http3_sds
        codec_type: HTTP3
        
        route_config:
          name: local_route
          virtual_hosts:
          - name: backend
            domains: ["*"]
            routes:
            - match:
                prefix: "/"
              route:
                cluster: backend_cluster
        
        http_filters:
        - name: envoy.filters.http.router

clusters:
- name: sds_cluster
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  typed_extension_protocol_options:
    envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
      "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
      explicit_http_config:
        http2_protocol_options: {}
  load_assignment:
    cluster_name: sds_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: sds.example.com
              port_value: 8080
```

## Complete Examples

### Full HTTP/3 Gateway with Fallback

```yaml
admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901

static_resources:
  listeners:
  # HTTP/3 (QUIC) listener
  - name: http3_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 443
        protocol: UDP
    
    udp_listener_config:
      quic_options:
        enabled: true
      downstream_socket_config:
        prefer_gro: true
        max_rx_datagram_size: 1500
    
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
                  filename: "/etc/envoy/certs/server.crt"
                private_key:
                  filename: "/etc/envoy/certs/server.key"
              alpn_protocols:
              - h3
            enable_early_data: true
      
      filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: http3_ingress
          codec_type: HTTP3
          http3_protocol_options:
            quic_protocol_options:
              max_concurrent_streams: 100
              initial_stream_window_size: 65536
              initial_connection_window_size: 1048576
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: backend_cluster
          http_filters:
          - name: envoy.filters.http.router
  
  # HTTPS (HTTP/2, HTTP/1.1) listener with Alt-Svc
  - name: https_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 443
    
    filter_chains:
    - transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_certificates:
            - certificate_chain:
                filename: "/etc/envoy/certs/server.crt"
              private_key:
                filename: "/etc/envoy/certs/server.key"
            alpn_protocols:
            - h2
            - http/1.1
      
      filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: https_ingress
          codec_type: AUTO
          route_config:
            name: local_route
            response_headers_to_add:
            - header:
                key: "alt-svc"
                value: 'h3=":443"; ma=86400'
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: backend_cluster
          http_filters:
          - name: envoy.filters.http.router
  
  # HTTP listener (redirect to HTTPS)
  - name: http_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 80
    
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: http_ingress
          codec_type: AUTO
          route_config:
            name: redirect_route
            virtual_hosts:
            - name: redirect
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                redirect:
                  https_redirect: true
          http_filters:
          - name: envoy.filters.http.router

  clusters:
  - name: backend_cluster
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: backend_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: backend.example.com
                port_value: 8080
```

### Production UDP Proxy with Monitoring

```yaml
admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901

static_resources:
  listeners:
  - name: udp_proxy
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 10000
        protocol: UDP
    
    socket_options:
    - description: "Increase receive buffer"
      level: 1
      name: 8
      int_value: 8388608  # 8 MB
      state: STATE_PREBIND
    
    listener_filters:
    - name: envoy.filters.udp_listener.udp_proxy
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.v3.UdpProxyConfig
        stat_prefix: udp_proxy
        cluster: udp_backend
        session_timeout: 60s
        
        hash_policies:
        - source_ip: true
        
        session_access_log:
        - name: envoy.access_loggers.file
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
            path: /var/log/envoy/udp_access.log
            log_format:
              json_format:
                start_time: "%START_TIME%"
                peer_address: "%DOWNSTREAM_REMOTE_ADDRESS%"
                local_address: "%DOWNSTREAM_LOCAL_ADDRESS%"
                bytes_received: "%BYTES_RECEIVED%"
                bytes_sent: "%BYTES_SENT%"
                duration_ms: "%DURATION%"
                upstream_host: "%UPSTREAM_HOST%"

  clusters:
  - name: udp_backend
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    
    circuit_breakers:
      thresholds:
      - max_connections: 5000
    
    load_assignment:
      cluster_name: udp_backend
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: backend1.example.com
                port_value: 10000
        - endpoint:
            address:
              socket_address:
                address: backend2.example.com
                port_value: 10000
    
    health_checks:
    - timeout: 2s
      interval: 10s
      unhealthy_threshold: 3
      healthy_threshold: 2
      udp_health_check:
        send:
          text: "health"
        receive:
        - text: "ok"
```

## Monitoring and Debugging

### Key Metrics to Monitor

```bash
# HTTP/3 metrics
curl -s http://localhost:9901/stats | grep http3

# QUIC connection metrics
curl -s http://localhost:9901/stats | grep quic

# UDP proxy metrics
curl -s http://localhost:9901/stats | grep udp_proxy

# Session metrics
curl -s http://localhost:9901/stats | grep downstream_sess
```

### Enable Debug Logging

```bash
# Via admin API
curl -X POST http://localhost:9901/logging?level=debug
curl -X POST http://localhost:9901/logging?quic=trace
```

## Next Steps

- [QUIC Overview](01-quic-overview.md) - Understanding QUIC architecture
- [QUIC Implementation](02-quic-implementation.md) - Deep dive into implementation
- [UDP Listeners](03-udp-listeners.md) - UDP architecture details

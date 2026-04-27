# Production Hardening

This document covers security hardening, monitoring, alerting, and operational best practices for running Envoy in production.

## Table of Contents

1. [Security Hardening](#security-hardening)
2. [Resource Limits and Performance](#resource-limits-and-performance)
3. [Monitoring and Observability](#monitoring-and-observability)
4. [Alert Configuration](#alert-configuration)
5. [Disaster Recovery](#disaster-recovery)
6. [SRE Runbooks](#sre-runbooks)
7. [Production Checklist](#production-checklist)

---

## Security Hardening

### Principle of Least Privilege

#### Run as Non-Root User

```yaml
# Kubernetes security context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  
  # Drop all capabilities
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE  # Only if binding to ports < 1024
  
  # Read-only root filesystem
  readOnlyRootFilesystem: true
  
  # Prevent privilege escalation
  allowPrivilegeEscalation: false
```

#### File System Permissions

```bash
# Create envoy user
groupadd -r envoy
useradd -r -g envoy -s /sbin/nologin -c "Envoy user" envoy

# Set ownership
chown -R envoy:envoy /etc/envoy /var/log/envoy

# Secure permissions
chmod 750 /etc/envoy
chmod 640 /etc/envoy/envoy.yaml
chmod 600 /etc/envoy/certs/*
```

### TLS/SSL Configuration

#### Strong TLS Configuration

```yaml
# envoy-tls-hardened.yaml
static_resources:
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
          stat_prefix: ingress_https
          codec_type: AUTO
          
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
          
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
      
      # TLS configuration
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          
          # Require client certificates (mTLS)
          require_client_certificate: true
          
          common_tls_context:
            # Server certificates
            tls_certificates:
            - certificate_chain:
                filename: /etc/envoy/certs/server-cert.pem
              private_key:
                filename: /etc/envoy/certs/server-key.pem
            
            # Client certificate validation
            validation_context:
              trusted_ca:
                filename: /etc/envoy/certs/ca-cert.pem
              # Optional: CRL checking
              crl:
                filename: /etc/envoy/certs/crl.pem
            
            # TLS parameters
            tls_params:
              tls_minimum_protocol_version: TLSv1_3
              tls_maximum_protocol_version: TLSv1_3
              
              # Strong cipher suites (TLS 1.3)
              cipher_suites:
              - TLS_AES_256_GCM_SHA384
              - TLS_AES_128_GCM_SHA256
              - TLS_CHACHA20_POLY1305_SHA256
            
            # ALPN protocols
            alpn_protocols:
            - h2
            - http/1.1
  
  clusters:
  - name: backend_cluster
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    
    # Upstream TLS
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        
        common_tls_context:
          # Client certificate for mTLS
          tls_certificates:
          - certificate_chain:
              filename: /etc/envoy/certs/client-cert.pem
            private_key:
              filename: /etc/envoy/certs/client-key.pem
          
          # Validate backend certificates
          validation_context:
            trusted_ca:
              filename: /etc/envoy/certs/backend-ca.pem
            
            # Verify certificate hostname
            match_subject_alt_names:
            - exact: "backend.internal.example.com"
          
          tls_params:
            tls_minimum_protocol_version: TLSv1_3
            tls_maximum_protocol_version: TLSv1_3
        
        # SNI
        sni: backend.internal.example.com
    
    load_assignment:
      cluster_name: backend_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: backend.internal.example.com
                port_value: 8443
```

#### Certificate Rotation

```yaml
# Kubernetes secret for certificates
apiVersion: v1
kind: Secret
metadata:
  name: envoy-tls-certs
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
  ca.crt: <base64-encoded-ca>

---
# Certificate rotation with cert-manager
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: envoy-tls
  namespace: production
spec:
  secretName: envoy-tls-certs
  duration: 2160h  # 90 days
  renewBefore: 360h  # 15 days before expiry
  
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  
  dnsNames:
  - api.example.com
  - "*.api.example.com"
```

### Secrets Management

#### Using External Secrets

```yaml
# Kubernetes with External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: envoy-secrets
  namespace: production
spec:
  refreshInterval: 1h
  
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  
  target:
    name: envoy-secrets
    creationPolicy: Owner
  
  data:
  - secretKey: tls.crt
    remoteRef:
      key: secret/data/envoy/tls
      property: certificate
  
  - secretKey: tls.key
    remoteRef:
      key: secret/data/envoy/tls
      property: private_key
```

#### Envoy Secret Discovery Service (SDS)

```yaml
# Bootstrap configuration with SDS
node:
  cluster: production
  id: envoy-proxy-1

admin:
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 9901

dynamic_resources:
  # ... other xDS configs ...

static_resources:
  clusters:
  # SDS cluster for certificate management
  - name: sds_cluster
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
    load_assignment:
      cluster_name: sds_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: sds-server.internal.example.com
                port_value: 8234
  
  listeners:
  - name: listener_with_sds
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 443
    
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        # ... HCM config ...
      
      # TLS with SDS
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            # Certificate from SDS
            tls_certificate_sds_secret_configs:
            - name: server_cert
              sds_config:
                api_config_source:
                  api_type: GRPC
                  grpc_services:
                  - envoy_grpc:
                      cluster_name: sds_cluster
            
            # CA from SDS
            validation_context_sds_secret_config:
              name: validation_context
              sds_config:
                api_config_source:
                  api_type: GRPC
                  grpc_services:
                  - envoy_grpc:
                      cluster_name: sds_cluster
```

### Access Control

#### IP Whitelisting

```yaml
http_filters:
- name: envoy.filters.http.rbac
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.rbac.v3.RBAC
    rules:
      action: ALLOW
      policies:
        "allow-internal":
          permissions:
          - any: true
          principals:
          - remote_ip:
              address_prefix: 10.0.0.0
              prefix_len: 8
          - remote_ip:
              address_prefix: 172.16.0.0
              prefix_len: 12
          - remote_ip:
              address_prefix: 192.168.0.0
              prefix_len: 16
```

#### JWT Authentication

```yaml
http_filters:
- name: envoy.filters.http.jwt_authn
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.jwt_authn.v3.JwtAuthentication
    providers:
      auth0:
        issuer: "https://your-tenant.auth0.com/"
        audiences:
        - "https://api.example.com"
        
        # JWKS from remote endpoint
        remote_jwks:
          http_uri:
            uri: "https://your-tenant.auth0.com/.well-known/jwks.json"
            cluster: auth0_cluster
            timeout: 5s
          cache_duration:
            seconds: 300
        
        # Extract claims to headers
        payload_in_metadata: jwt_payload
        
        # Forward JWT to backend
        forward: true
        
        # JWT locations
        from_headers:
        - name: Authorization
          value_prefix: "Bearer "
        from_params:
        - access_token
    
    rules:
    - match:
        prefix: "/api"
      requires:
        provider_name: auth0
```

### Rate Limiting

#### Local Rate Limiting

```yaml
http_filters:
- name: envoy.filters.http.local_ratelimit
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
    stat_prefix: http_local_rate_limiter
    
    # 100 requests per second per client IP
    token_bucket:
      max_tokens: 100
      tokens_per_fill: 100
      fill_interval: 1s
    
    # Apply to all requests
    filter_enabled:
      runtime_key: local_rate_limit_enabled
      default_value:
        numerator: 100
        denominator: HUNDRED
    
    filter_enforced:
      runtime_key: local_rate_limit_enforced
      default_value:
        numerator: 100
        denominator: HUNDRED
    
    # Custom response when rate limited
    response_headers_to_add:
    - append: false
      header:
        key: x-ratelimit-limit
        value: "100"
```

#### Global Rate Limiting

```yaml
http_filters:
- name: envoy.filters.http.ratelimit
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.ratelimit.v3.RateLimit
    domain: production_ratelimit
    
    # Fail open if rate limit service is down
    failure_mode_deny: false
    
    # Rate limit service
    rate_limit_service:
      grpc_service:
        envoy_grpc:
          cluster_name: ratelimit_service
      
      transport_api_version: V3
    
    # Timeout for rate limit check
    timeout: 0.2s

# In route configuration
routes:
- match:
    prefix: "/api"
  route:
    cluster: api_cluster
    
    # Rate limit descriptors
    rate_limits:
    # Per-IP rate limit
    - actions:
      - remote_address: {}
    
    # Per-user rate limit (from JWT)
    - actions:
      - metadata:
          descriptor_key: "user_id"
          metadata_key:
            key: "envoy.filters.http.jwt_authn"
            path:
            - key: "jwt_payload"
            - key: "sub"
    
    # Per-path rate limit
    - actions:
      - request_headers:
          header_name: ":path"
          descriptor_key: "path"
```

### Security Headers

```yaml
# Add security headers to responses
http_filters:
- name: envoy.filters.http.lua
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
    inline_code: |
      function envoy_on_response(response_handle)
        -- Security headers
        response_handle:headers():add("X-Frame-Options", "DENY")
        response_handle:headers():add("X-Content-Type-Options", "nosniff")
        response_handle:headers():add("X-XSS-Protection", "1; mode=block")
        response_handle:headers():add("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        response_handle:headers():add("Content-Security-Policy", "default-src 'self'")
        response_handle:headers():add("Referrer-Policy", "strict-origin-when-cross-origin")
        response_handle:headers():add("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        
        -- Remove server header
        response_handle:headers():remove("Server")
        response_handle:headers():remove("X-Powered-By")
      end
```

### Network Policies (Kubernetes)

```yaml
# Restrict ingress to Envoy pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: envoy-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: envoy
  
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  # Allow from LoadBalancer
  - from:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 10000
    - protocol: TCP
      port: 443
  
  # Allow from monitoring
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 9901
  
  egress:
  # Allow to backend services
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 8080
  
  # Allow to DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

---

## Resource Limits and Performance

### Connection Limits

```yaml
clusters:
- name: backend_cluster
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  
  # Circuit breaker settings
  circuit_breakers:
    thresholds:
    - priority: DEFAULT
      max_connections: 1000          # Max concurrent connections
      max_pending_requests: 100      # Max queued requests
      max_requests: 1000             # Max concurrent requests
      max_retries: 3                 # Max concurrent retries
      max_connection_pools: 1        # Connection pool limit
    
    # High priority traffic (if using priority)
    - priority: HIGH
      max_connections: 2000
      max_pending_requests: 200
      max_requests: 2000
      max_retries: 5
  
  # Connection pool settings
  common_lb_config:
    healthy_panic_threshold:
      value: 50.0  # Panic mode at 50% unhealthy
  
  # Per-host limits
  max_requests_per_connection: 100  # HTTP/1.1 connection reuse
  
  load_assignment:
    cluster_name: backend_cluster
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: backend.internal.example.com
              port_value: 8080
```

### Buffer Limits

```yaml
listeners:
- name: listener_0
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 10000
  
  # Per-connection buffer limit
  per_connection_buffer_limit_bytes: 32768  # 32 KB
  
  filter_chains:
  - filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: ingress
        
        # HTTP/1.1 settings
        http_protocol_options:
          accept_http_10: false
          default_host_for_http_10: ""
          max_connection_duration:
            seconds: 300  # 5 minutes max connection duration
        
        # HTTP/2 settings
        http2_protocol_options:
          max_concurrent_streams: 100
          initial_stream_window_size: 65536      # 64 KB
          initial_connection_window_size: 1048576  # 1 MB
          max_inbound_window_update_frames_per_data_frame_sent: 10
        
        # Request/response limits
        common_http_protocol_options:
          max_headers_count: 100
          max_stream_duration:
            seconds: 300
          idle_timeout:
            seconds: 300
          headers_with_underscores_action: REJECT_REQUEST
        
        # Buffer limits
        codec_type: AUTO
        
        # ... rest of HCM config ...
```

### Worker Threads

```yaml
# Bootstrap configuration
# Set worker threads based on CPU cores
# Rule of thumb: 1 worker per CPU core, max 8 workers

static_resources:
  # ... listeners and clusters ...

# Number of worker threads
# Set via command line: --concurrency <num_threads>
# Or in bootstrap:
layered_runtime:
  layers:
  - name: static_layer
    static_layer:
      overload:
        global_downstream_max_connections: 50000

# File: envoy-bootstrap.yaml
concurrency: 4  # 4 worker threads
```

### Memory Management

```yaml
# Overload manager - protect against OOM
overload_manager:
  refresh_interval:
    seconds: 1
  
  resource_monitors:
  # Monitor memory usage
  - name: "envoy.resource_monitors.fixed_heap"
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.resource_monitors.fixed_heap.v3.FixedHeapConfig
      max_heap_size_bytes: 536870912  # 512 MB
  
  actions:
  # Stop accepting new connections at 95% memory
  - name: "envoy.overload_actions.stop_accepting_connections"
    triggers:
    - name: "envoy.resource_monitors.fixed_heap"
      threshold:
        value: 0.95
  
  # Stop accepting new requests at 90% memory
  - name: "envoy.overload_actions.stop_accepting_requests"
    triggers:
    - name: "envoy.resource_monitors.fixed_heap"
      threshold:
        value: 0.90
  
  # Disable HTTP keep-alive at 85% memory
  - name: "envoy.overload_actions.disable_http_keepalive"
    triggers:
    - name: "envoy.resource_monitors.fixed_heap"
      threshold:
        value: 0.85
  
  # Reduce timeouts at 80% memory
  - name: "envoy.overload_actions.reduce_timeouts"
    triggers:
    - name: "envoy.resource_monitors.fixed_heap"
      threshold:
        value: 0.80
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.overload_actions.reduce_timeouts.v3.ReduceTimeoutsConfig
      timeout_reductions:
      - timer: HTTP_DOWNSTREAM_CONNECTION_IDLE
        min_timeout:
          seconds: 5
```

### Timeouts

```yaml
clusters:
- name: backend_cluster
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  
  # Cluster-wide timeouts
  connect_timeout: 5s
  
  # TCP settings
  tcp_keepalive:
    keepalive_probes: 3
    keepalive_time: 60
    keepalive_interval: 10
  
  # HTTP/2 settings
  http2_protocol_options:
    connection_keepalive:
      interval: 30s
      timeout: 10s
  
  load_assignment:
    # ... endpoints ...

# Route-level timeouts
routes:
- match:
    prefix: "/api"
  route:
    cluster: backend_cluster
    
    # Request timeout
    timeout: 30s
    
    # Idle timeout (stream)
    idle_timeout: 60s
    
    # Retry policy
    retry_policy:
      retry_on: "5xx,reset,connect-failure,refused-stream"
      num_retries: 3
      per_try_timeout: 10s
      retry_host_predicate:
      - name: envoy.retry_host_predicates.previous_hosts
      host_selection_retry_max_attempts: 5
```

---

## Monitoring and Observability

### Metrics Export (Prometheus)

```yaml
admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901

# StatsSinks for external metrics
stats_sinks:
# Prometheus (via admin endpoint)
- name: envoy.stat_sinks.metrics_service
  typed_config:
    "@type": type.googleapis.com/envoy.config.metrics.v3.MetricsServiceConfig
    grpc_service:
      envoy_grpc:
        cluster_name: metrics_cluster

# StatsD
- name: envoy.stat_sinks.statsd
  typed_config:
    "@type": type.googleapis.com/envoy.config.metrics.v3.StatsdSink
    tcp_cluster_name: statsd_cluster
    prefix: envoy

# Kubernetes ServiceMonitor
---
apiVersion: v1
kind: Service
metadata:
  name: envoy-metrics
  namespace: production
  labels:
    app: envoy
spec:
  selector:
    app: envoy
  ports:
  - name: metrics
    port: 9901
    targetPort: 9901
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: envoy
  namespace: production
spec:
  selector:
    matchLabels:
      app: envoy
  endpoints:
  - port: metrics
    path: /stats/prometheus
    interval: 30s
    scrapeTimeout: 10s
```

### Key Metrics to Monitor

```
# Connection metrics
envoy_cluster_upstream_cx_active              # Active connections
envoy_cluster_upstream_cx_total               # Total connections
envoy_cluster_upstream_cx_connect_fail        # Connection failures

# Request metrics
envoy_cluster_upstream_rq_total               # Total requests
envoy_cluster_upstream_rq_active              # Active requests
envoy_cluster_upstream_rq_pending_active      # Queued requests
envoy_cluster_upstream_rq_time_bucket         # Request latency histogram

# Response codes
envoy_cluster_upstream_rq_xx{envoy_response_code_class="2"}  # 2xx responses
envoy_cluster_upstream_rq_xx{envoy_response_code_class="4"}  # 4xx responses
envoy_cluster_upstream_rq_xx{envoy_response_code_class="5"}  # 5xx responses

# Health check metrics
envoy_cluster_health_check_healthy            # Healthy hosts
envoy_cluster_health_check_unhealthy          # Unhealthy hosts

# Circuit breaker metrics
envoy_cluster_circuit_breakers_default_cx_open       # Connections rejected
envoy_cluster_circuit_breakers_default_rq_open       # Requests rejected
envoy_cluster_circuit_breakers_default_rq_pending_open  # Queue full

# Memory metrics
envoy_server_memory_allocated                 # Allocated memory
envoy_server_memory_heap_size                 # Heap size

# Hot restart
envoy_server_hot_restart_generation           # Restart generation
```

### Distributed Tracing

```yaml
# Zipkin tracing
tracing:
  http:
    name: envoy.tracers.zipkin
    typed_config:
      "@type": type.googleapis.com/envoy.config.trace.v3.ZipkinConfig
      collector_cluster: zipkin
      collector_endpoint: "/api/v2/spans"
      collector_endpoint_version: HTTP_JSON
      shared_span_context: false
      trace_id_128bit: true

clusters:
- name: zipkin
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  load_assignment:
    cluster_name: zipkin
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: zipkin.tracing.svc.cluster.local
              port_value: 9411

# Enable tracing in HCM
http_connection_manager:
  tracing:
    provider:
      name: envoy.tracers.zipkin
    
    # Sampling rate (100% = all requests)
    random_sampling:
      value: 100.0
    
    # Tag requests
    custom_tags:
    - tag: environment
      literal:
        value: production
    
    - tag: service
      environment:
        name: SERVICE_NAME
        default_value: unknown
```

### Access Logging

```yaml
http_connection_manager:
  access_log:
  # File access log
  - name: envoy.access_loggers.file
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
      path: /var/log/envoy/access.log
      
      # JSON format for easy parsing
      json_format:
        start_time: "%START_TIME%"
        method: "%REQ(:METHOD)%"
        path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
        protocol: "%PROTOCOL%"
        response_code: "%RESPONSE_CODE%"
        response_flags: "%RESPONSE_FLAGS%"
        bytes_received: "%BYTES_RECEIVED%"
        bytes_sent: "%BYTES_SENT%"
        duration: "%DURATION%"
        upstream_service_time: "%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%"
        x_forwarded_for: "%REQ(X-FORWARDED-FOR)%"
        user_agent: "%REQ(USER-AGENT)%"
        request_id: "%REQ(X-REQUEST-ID)%"
        authority: "%REQ(:AUTHORITY)%"
        upstream_host: "%UPSTREAM_HOST%"
        upstream_cluster: "%UPSTREAM_CLUSTER%"
  
  # gRPC access log (send to centralized logging)
  - name: envoy.access_loggers.http_grpc
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.grpc.v3.HttpGrpcAccessLogConfig
      common_config:
        log_name: envoy_access_log
        transport_api_version: V3
        grpc_service:
          envoy_grpc:
            cluster_name: access_log_cluster
      
      # Filter: only log errors
      filter:
        status_code_filter:
          comparison:
            op: GE
            value:
              default_value: 400
              runtime_key: access_log_filter_code
```

---

## Alert Configuration

### Prometheus Alert Rules

```yaml
# prometheus-alerts.yaml
groups:
- name: envoy_alerts
  interval: 30s
  rules:
  
  # High error rate
  - alert: EnvoyHighErrorRate
    expr: |
      (
        sum(rate(envoy_cluster_upstream_rq{envoy_response_code_class="5"}[5m]))
        /
        sum(rate(envoy_cluster_upstream_rq[5m]))
      ) > 0.05
    for: 5m
    labels:
      severity: warning
      component: envoy
    annotations:
      summary: "High 5xx error rate on {{ $labels.envoy_cluster_name }}"
      description: "{{ $labels.envoy_cluster_name }} has {{ $value | humanizePercentage }} error rate"
  
  # Circuit breaker tripped
  - alert: EnvoyCircuitBreakerOpen
    expr: |
      sum by (envoy_cluster_name) (
        envoy_cluster_circuit_breakers_default_rq_open
      ) > 0
    for: 2m
    labels:
      severity: warning
      component: envoy
    annotations:
      summary: "Circuit breaker open for {{ $labels.envoy_cluster_name }}"
      description: "Cluster {{ $labels.envoy_cluster_name }} circuit breaker is rejecting requests"
  
  # No healthy hosts
  - alert: EnvoyNoHealthyHosts
    expr: |
      envoy_cluster_membership_healthy == 0
    for: 1m
    labels:
      severity: critical
      component: envoy
    annotations:
      summary: "No healthy hosts in {{ $labels.envoy_cluster_name }}"
      description: "Cluster {{ $labels.envoy_cluster_name }} has no healthy hosts"
  
  # High latency
  - alert: EnvoyHighLatency
    expr: |
      histogram_quantile(0.99,
        sum(rate(envoy_cluster_upstream_rq_time_bucket[5m])) by (le, envoy_cluster_name)
      ) > 1000
    for: 10m
    labels:
      severity: warning
      component: envoy
    annotations:
      summary: "High latency on {{ $labels.envoy_cluster_name }}"
      description: "P99 latency is {{ $value }}ms for {{ $labels.envoy_cluster_name }}"
  
  # Memory usage high
  - alert: EnvoyHighMemoryUsage
    expr: |
      envoy_server_memory_allocated / envoy_server_memory_heap_size > 0.90
    for: 5m
    labels:
      severity: warning
      component: envoy
    annotations:
      summary: "High memory usage on {{ $labels.instance }}"
      description: "Memory usage is {{ $value | humanizePercentage }}"
  
  # Envoy down
  - alert: EnvoyDown
    expr: up{job="envoy"} == 0
    for: 1m
    labels:
      severity: critical
      component: envoy
    annotations:
      summary: "Envoy instance {{ $labels.instance }} is down"
      description: "Envoy has been down for more than 1 minute"
  
  # Connection pool exhaustion
  - alert: EnvoyConnectionPoolExhausted
    expr: |
      (
        envoy_cluster_upstream_cx_active
        /
        envoy_cluster_circuit_breakers_default_cx_pool_open
      ) > 0.90
    for: 5m
    labels:
      severity: warning
      component: envoy
    annotations:
      summary: "Connection pool near exhaustion for {{ $labels.envoy_cluster_name }}"
      description: "{{ $labels.envoy_cluster_name }} connection pool is {{ $value | humanizePercentage }} full"
```

### Alert Manager Configuration

```yaml
# alertmanager.yaml
global:
  resolve_timeout: 5m

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  
  routes:
  # Critical alerts to PagerDuty
  - match:
      severity: critical
    receiver: pagerduty
    continue: true
  
  # Warning alerts to Slack
  - match:
      severity: warning
    receiver: slack

receivers:
- name: 'default'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/XXX'
    channel: '#envoy-alerts'
    title: 'Envoy Alert'
    text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}'

- name: 'pagerduty'
  pagerduty_configs:
  - service_key: 'XXX'
    description: '{{ .GroupLabels.alertname }}'

- name: 'slack'
  slack_configs:
  - api_url: 'https://hooks.slack.com/services/XXX'
    channel: '#envoy-warnings'
```

---

## Disaster Recovery

### Backup and Restore

#### Configuration Backup

```bash
#!/bin/bash
# backup-envoy-config.sh

BACKUP_DIR="/var/backups/envoy"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/envoy-config-$TIMESTAMP.tar.gz"

mkdir -p "$BACKUP_DIR"

# Backup configuration
tar -czf "$BACKUP_FILE" \
  /etc/envoy/*.yaml \
  /etc/envoy/certs/*

# Backup Kubernetes resources
kubectl get configmap -n production envoy-config -o yaml > "$BACKUP_DIR/configmap-$TIMESTAMP.yaml"
kubectl get deployment -n production envoy -o yaml > "$BACKUP_DIR/deployment-$TIMESTAMP.yaml"
kubectl get service -n production envoy -o yaml > "$BACKUP_DIR/service-$TIMESTAMP.yaml"

# Keep only last 30 days
find "$BACKUP_DIR" -name "envoy-config-*.tar.gz" -mtime +30 -delete

echo "Backup complete: $BACKUP_FILE"
```

#### Restore Procedure

```bash
#!/bin/bash
# restore-envoy-config.sh

BACKUP_FILE="$1"

if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: $0 <backup-file>"
  exit 1
fi

# Extract backup
tar -xzf "$BACKUP_FILE" -C /tmp/

# Validate configuration
envoy --mode validate -c /tmp/etc/envoy/envoy.yaml || {
  echo "Configuration validation failed!"
  exit 1
}

# Apply configuration
cp -r /tmp/etc/envoy/* /etc/envoy/

# Restart Envoy (with hot restart)
systemctl reload envoy

echo "Restore complete"
```

### Multi-Region Failover

```yaml
# DNS-based failover with multiple regions
clusters:
- name: backend_cluster
  type: STRICT_DNS
  lb_policy: ROUND_ROBIN
  
  # Health checking
  health_checks:
  - timeout: 5s
    interval: 10s
    unhealthy_threshold: 3
    healthy_threshold: 2
    http_health_check:
      path: /healthz
  
  # Multiple regions
  load_assignment:
    cluster_name: backend_cluster
    endpoints:
    # Primary region (us-west-2)
    - locality:
        region: us-west-2
        zone: us-west-2a
      lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: backend.us-west-2.internal
              port_value: 8080
        health_check_config:
          port_value: 8081
      priority: 0  # Primary
      load_balancing_weight: 100
    
    # Failover region (us-east-1)
    - locality:
        region: us-east-1
        zone: us-east-1a
      lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: backend.us-east-1.internal
              port_value: 8080
      priority: 1  # Failover
      load_balancing_weight: 100
  
  # Locality-weighted load balancing
  common_lb_config:
    locality_weighted_lb_config: {}
    
    # Trigger failover when 70% of hosts are unhealthy
    healthy_panic_threshold:
      value: 70.0
```

### Database Backup (Stats)

```bash
# If using persistent stats storage
#!/bin/bash
# backup-envoy-stats.sh

STATS_URL="http://localhost:9901/stats/prometheus"
BACKUP_DIR="/var/backups/envoy/stats"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

curl -s "$STATS_URL" > "$BACKUP_DIR/stats-$TIMESTAMP.txt"

# Compress
gzip "$BACKUP_DIR/stats-$TIMESTAMP.txt"

# Keep only last 7 days
find "$BACKUP_DIR" -name "stats-*.txt.gz" -mtime +7 -delete
```

---

## SRE Runbooks

### Runbook: High Error Rate

**Symptoms:**
- 5xx error rate > 5%
- Alert: `EnvoyHighErrorRate`

**Diagnosis:**

```bash
# 1. Check cluster health
curl http://localhost:9901/clusters | grep health_flags

# 2. Check recent errors
curl http://localhost:9901/stats/prometheus | grep envoy_cluster_upstream_rq | grep 5

# 3. Check logs for errors
kubectl logs -n production deploy/envoy -c envoy --tail=100 | grep -i error

# 4. Check backend health
for backend in $(curl -s http://localhost:9901/clusters | grep '::address::' | awk '{print $2}'); do
  echo "Checking $backend"
  curl -I "http://$backend/healthz"
done
```

**Resolution:**

1. **If backends are unhealthy:**
   ```bash
   # Scale up backend replicas
   kubectl scale deployment backend -n production --replicas=6
   
   # Check autoscaling
   kubectl get hpa -n production
   ```

2. **If circuit breaker is open:**
   ```bash
   # Check circuit breaker stats
   curl http://localhost:9901/stats/prometheus | grep circuit_breakers
   
   # Temporarily increase limits (requires config change)
   # Edit envoy.yaml and reload
   ```

3. **If timeouts occurring:**
   ```bash
   # Check upstream latency
   curl http://localhost:9901/stats/prometheus | grep upstream_rq_time
   
   # Increase timeout (requires config change)
   ```

---

### Runbook: Memory Exhaustion

**Symptoms:**
- High memory usage > 90%
- Alert: `EnvoyHighMemoryUsage`
- OOMKilled events

**Diagnosis:**

```bash
# 1. Check current memory usage
curl http://localhost:9901/stats/prometheus | grep memory

# 2. Check connection counts
curl http://localhost:9901/stats/prometheus | grep downstream_cx_active
curl http://localhost:9901/stats/prometheus | grep upstream_cx_active

# 3. Check buffer usage
curl http://localhost:9901/stats/prometheus | grep buffer

# 4. Heap dump (if available)
curl http://localhost:9901/memory > heap.dump
```

**Resolution:**

1. **Immediate mitigation:**
   ```bash
   # Scale up replicas to distribute load
   kubectl scale deployment envoy -n production --replicas=5
   
   # Or increase memory limits
   kubectl patch deployment envoy -n production -p '{"spec":{"template":{"spec":{"containers":[{"name":"envoy","resources":{"limits":{"memory":"1Gi"}}}]}}}}'
   ```

2. **Long-term fixes:**
   - Review connection limits and timeouts
   - Enable overload manager
   - Tune buffer sizes
   - Check for connection leaks

---

### Runbook: Certificate Expiration

**Symptoms:**
- TLS handshake failures
- Alert: Certificate expiring soon

**Diagnosis:**

```bash
# 1. Check certificate expiration
openssl s_client -connect localhost:443 -servername api.example.com </dev/null 2>/dev/null | \
  openssl x509 -noout -dates

# 2. Check Envoy TLS stats
curl http://localhost:9901/stats/prometheus | grep ssl

# 3. Check cert-manager (Kubernetes)
kubectl get certificate -n production
kubectl describe certificate envoy-tls -n production
```

**Resolution:**

```bash
# 1. If using cert-manager, trigger renewal
kubectl delete secret envoy-tls-certs -n production
# cert-manager will automatically recreate

# 2. If manual certs, update secret
kubectl create secret tls envoy-tls-certs \
  --cert=new-cert.pem \
  --key=new-key.pem \
  --dry-run=client -o yaml | kubectl apply -f -

# 3. Reload Envoy (hot restart)
kubectl rollout restart deployment envoy -n production

# 4. Verify new certificate
openssl s_client -connect localhost:443 -servername api.example.com </dev/null 2>/dev/null | \
  openssl x509 -noout -dates
```

---

### Runbook: No Healthy Hosts

**Symptoms:**
- All requests failing
- Alert: `EnvoyNoHealthyHosts`
- 503 Service Unavailable responses

**Diagnosis:**

```bash
# 1. Check cluster membership
curl http://localhost:9901/clusters | grep -A5 "cluster_name::backend"

# 2. Check health check status
curl http://localhost:9901/stats/prometheus | grep health_check

# 3. Test backend health directly
curl http://backend.internal:8080/healthz

# 4. Check network connectivity
kubectl exec -it deploy/envoy -n production -c envoy -- ping backend.internal
```

**Resolution:**

1. **If backends are actually unhealthy:**
   ```bash
   # Check backend logs
   kubectl logs -n production deploy/backend --tail=100
   
   # Restart backends
   kubectl rollout restart deployment backend -n production
   ```

2. **If health checks are misconfigured:**
   ```bash
   # Update health check path/port in envoy.yaml
   # Apply configuration
   kubectl apply -f envoy-config.yaml
   
   # Reload Envoy
   kubectl rollout restart deployment envoy -n production
   ```

3. **If network issues:**
   ```bash
   # Check NetworkPolicy
   kubectl get networkpolicy -n production
   
   # Check Service/Endpoints
   kubectl get endpoints backend -n production
   ```

---

## Production Checklist

### Pre-Deployment Checklist

- [ ] **Configuration validated** with `envoy --mode validate`
- [ ] **TLS certificates** valid and not expiring soon
- [ ] **Resource limits** set appropriately (CPU, memory)
- [ ] **Health checks** configured and tested
- [ ] **Monitoring** integrated (Prometheus, Grafana)
- [ ] **Alerts** configured and tested
- [ ] **Logs** exported to centralized logging
- [ ] **Tracing** enabled (if applicable)
- [ ] **Circuit breakers** configured
- [ ] **Rate limiting** configured
- [ ] **Security hardening** applied (non-root, read-only FS)
- [ ] **Network policies** configured (Kubernetes)
- [ ] **Backup procedures** documented and tested
- [ ] **Rollback plan** documented
- [ ] **Runbooks** created for common issues
- [ ] **Load testing** completed
- [ ] **Disaster recovery** tested

### Post-Deployment Checklist

- [ ] **Health checks** passing
- [ ] **Metrics** being collected
- [ ] **Logs** flowing to centralized system
- [ ] **Alerts** not firing
- [ ] **Traffic** routing correctly
- [ ] **Latency** within SLOs
- [ ] **Error rate** within SLOs
- [ ] **Resource usage** within limits
- [ ] **Certificate rotation** working
- [ ] **Hot restart** tested

### Regular Maintenance

**Daily:**
- Check dashboards for anomalies
- Review error logs
- Check alert history

**Weekly:**
- Review resource usage trends
- Check certificate expiration dates
- Test backup/restore procedures

**Monthly:**
- Perform hot restart test
- Review and update runbooks
- Security patch updates
- Load testing

**Quarterly:**
- Disaster recovery drill
- Review and update monitoring/alerts
- Capacity planning review
- Security audit

---

## Related Documentation

- [Deployment Patterns](01-deployment-patterns.md) - Common deployment scenarios
- [Hot Restart](02-hot-restart.md) - Zero-downtime updates
- [Container Deployment](03-container-deployment.md) - Kubernetes deployment
- [Admin Interface](../admin-operations/01-admin-interface.md) - Admin API reference

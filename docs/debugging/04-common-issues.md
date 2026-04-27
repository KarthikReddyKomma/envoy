# Common Issues and Solutions

This document provides troubleshooting guides for common problems encountered when running and debugging Envoy.

## Table of Contents

1. [Connection Failures](#connection-failures)
2. [TLS/Certificate Issues](#tlscertificate-issues)
3. [Configuration Errors](#configuration-errors)
4. [Performance Problems](#performance-problems)
5. [Memory Leaks](#memory-leaks)
6. [Crash Troubleshooting](#crash-troubleshooting)
7. [Timeout Issues](#timeout-issues)
8. [Health Check Failures](#health-check-failures)
9. [Route Matching Problems](#route-matching-problems)
10. [Filter Chain Issues](#filter-chain-issues)

---

## Connection Failures

### Symptoms

```bash
# Stats showing connection failures
curl http://localhost:9901/stats | grep cx_connect_fail
cluster.my_cluster.upstream_cx_connect_fail: 10

# Logs showing
[error][upstream] upstream connect error or disconnect/reset before headers
```

### Common Causes and Solutions

#### 1. Upstream Service Not Running

**Diagnosis**:
```bash
# Check if upstream is reachable
nc -zv upstream-host 8080

# Check cluster status
curl http://localhost:9901/clusters | grep my_cluster
```

**Solution**:
- Ensure upstream service is running
- Verify firewall rules allow connection
- Check network connectivity

#### 2. DNS Resolution Failure

**Diagnosis**:
```bash
# Check DNS resolution
curl http://localhost:9901/stats | grep dns
cluster.my_cluster.update_attempt: 5
cluster.my_cluster.update_failure: 5

# Logs showing
[error][config] DNS resolution failed for host: example.com
```

**Solution**:
```yaml
# Use IP address instead of hostname
clusters:
- name: my_cluster
  type: STRICT_DNS  # or LOGICAL_DNS
  load_assignment:
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: 192.168.1.100  # Use IP
              port_value: 8080

# Or configure DNS
dns_resolution_config:
  resolvers:
  - socket_address:
      address: 8.8.8.8
      port_value: 53
```

#### 3. Wrong Port or Host

**Diagnosis**:
```bash
# Verify cluster configuration
curl -s http://localhost:9901/config_dump | \
  jq '.configs[1].dynamic_active_clusters[] | 
      select(.cluster.name == "my_cluster") | 
      .cluster.load_assignment.endpoints[].lb_endpoints[].endpoint.address'
```

**Solution**:
- Double-check host and port in configuration
- Test direct connection: `curl http://host:port`

#### 4. Connection Pool Exhaustion

**Diagnosis**:
```bash
# Check pool stats
curl http://localhost:9901/stats | grep cx_pool
cluster.my_cluster.upstream_cx_overflow: 100
cluster.my_cluster.upstream_cx_pool_max_exceeded: 50
```

**Solution**:
```yaml
# Increase connection pool limits
clusters:
- name: my_cluster
  circuit_breakers:
    thresholds:
    - priority: DEFAULT
      max_connections: 2048      # Increase
      max_pending_requests: 2048
      max_requests: 2048
```

#### 5. Firewall/Security Groups

**Diagnosis**:
```bash
# Test connectivity
telnet upstream-host 8080
# Or
nc -zv upstream-host 8080

# Check iptables
sudo iptables -L -n | grep 8080
```

**Solution**:
- Update firewall rules to allow traffic
- Update security groups (AWS, GCP, etc.)
- Check SELinux/AppArmor policies

---

## TLS/Certificate Issues

### Symptoms

```bash
# Logs showing
[error][connection] TLS error: ... certificate verify failed
[error][connection] SSL_ERROR_SSL: error:14094410:SSL routines:ssl3_read_bytes:sslv3 alert handshake failure
```

### Common Causes and Solutions

#### 1. Certificate Expired

**Diagnosis**:
```bash
# Check certificate expiry
curl http://localhost:9901/certs

# Or manually
openssl s_client -connect upstream-host:443 < /dev/null 2>&1 | \
  openssl x509 -noout -dates
```

**Solution**:
- Renew certificates
- Update certificate in Envoy configuration
- Implement certificate rotation

#### 2. Certificate Validation Failure

**Diagnosis**:
```bash
# Logs showing
certificate verify failed: self signed certificate in certificate chain
certificate verify failed: unable to get local issuer certificate
```

**Solution**:
```yaml
# Add trusted CA certificates
clusters:
- name: my_cluster
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      common_tls_context:
        validation_context:
          trusted_ca:
            filename: /etc/ssl/certs/ca-certificates.crt
          # Or for self-signed during development
          trust_chain_verification: ACCEPT_UNTRUSTED
```

#### 3. SNI Mismatch

**Diagnosis**:
```bash
# Check SNI in logs
[error][connection] TLS error: ... certificate verify failed: Hostname mismatch
```

**Solution**:
```yaml
# Configure SNI
clusters:
- name: my_cluster
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
      sni: example.com  # Match certificate CN/SAN
```

#### 4. Protocol Version Mismatch

**Diagnosis**:
```bash
# Logs showing
SSL routines:ssl3_read_bytes:tlsv1 alert protocol version
```

**Solution**:
```yaml
# Configure TLS version
transport_socket:
  typed_config:
    common_tls_context:
      tls_params:
        tls_minimum_protocol_version: TLSv1_2
        tls_maximum_protocol_version: TLSv1_3
```

#### 5. Missing Client Certificate

**Diagnosis**:
```bash
# Logs showing
peer did not return a certificate
```

**Solution**:
```yaml
# Configure client certificate (mTLS)
common_tls_context:
  tls_certificates:
  - certificate_chain:
      filename: /etc/envoy/client-cert.pem
    private_key:
      filename: /etc/envoy/client-key.pem
```

---

## Configuration Errors

### Symptoms

```bash
# Envoy fails to start
[critical][main] error initializing configuration: ...

# Or dynamic config rejected
[warning][config] gRPC config for ... rejected: ...
```

### Common Causes and Solutions

#### 1. Invalid YAML Syntax

**Diagnosis**:
```bash
# Validate YAML syntax
./envoy-static --mode validate -c config.yaml

# Use YAML linter
yamllint config.yaml
```

**Solution**:
- Fix YAML indentation (use spaces, not tabs)
- Check for missing colons, quotes
- Validate with YAML parser

#### 2. Missing Required Fields

**Diagnosis**:
```bash
# Validation error
[critical][main] Proto constraint validation failed: value is required
```

**Solution**:
- Check Envoy documentation for required fields
- Use schema validation tools
- Review error message for specific field

#### 3. Type Mismatch

**Diagnosis**:
```bash
# Error showing
invalid value "string" for type int
```

**Solution**:
```yaml
# Correct types
timeout: 30s              # Duration (not "30" or 30)
port_value: 8080          # Integer
address: "127.0.0.1"      # String
enabled: true             # Boolean
```

#### 4. Deprecated Fields

**Diagnosis**:
```bash
# Warning in logs
[warning][config] Using deprecated field envoy.config...
```

**Solution**:
```bash
# Check for deprecated fields
bazel run //tools:deprecated_features -- config.yaml

# Update to new API version
# v2 → v3 migration
```

#### 5. Circular Dependencies

**Diagnosis**:
```bash
# Error showing
circular dependency detected in filter chain
```

**Solution**:
- Review filter chain order
- Check for filters that depend on each other
- Simplify configuration

---

## Performance Problems

### Symptoms

```bash
# High latency
curl http://localhost:9901/stats | grep duration
cluster.my_cluster.upstream_rq_time: P99(500.0)

# High CPU usage
top -p <envoy-pid>
```

### Common Causes and Solutions

#### 1. Connection Pool Limits

**Diagnosis**:
```bash
# Check pool stats
curl http://localhost:9901/stats | grep cx_pool
cluster.my_cluster.upstream_cx_pool_overflow: 1000
```

**Solution**:
```yaml
# Increase limits
circuit_breakers:
  thresholds:
  - max_connections: 4096
    max_pending_requests: 4096
    max_requests: 4096
    max_retries: 1024
```

#### 2. Too Many Stats

**Diagnosis**:
```bash
# Count stats
curl -s http://localhost:9901/stats | wc -l
# If > 100000, might be too many
```

**Solution**:
```yaml
# Disable unused stats
stats_config:
  stats_matcher:
    exclusion_list:
      patterns:
      - prefix: "cluster.*.upstream_rq_"
      - prefix: "listener.*.downstream_cx_"
```

#### 3. Slow Filters

**Diagnosis**:
```bash
# Enable filter timing
curl -X POST http://localhost:9901/logging?filter=trace

# Look for slow filter execution in logs
```

**Solution**:
- Profile filter code
- Optimize or remove expensive operations
- Consider async processing

#### 4. Memory Fragmentation

**Diagnosis**:
```bash
# Check memory usage over time
watch -n 5 'curl -s http://localhost:9901/memory'
```

**Solution**:
```bash
# Use tcmalloc (usually default)
# Or build with jemalloc
bazel build --define tcmalloc=disabled //source/exe:envoy-static
```

#### 5. Too Many Active Connections

**Diagnosis**:
```bash
# Check connection counts
curl http://localhost:9901/stats | grep cx_active
listener.0.0.0.0_8000.downstream_cx_active: 5000
```

**Solution**:
```yaml
# Add connection limits
listeners:
- name: listener_0
  per_connection_buffer_limit_bytes: 32768
  listener_filters_timeout: 5s
  filter_chains:
  - filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        stream_idle_timeout: 300s
        request_timeout: 300s
        drain_timeout: 5s
```

---

## Memory Leaks

### Symptoms

```bash
# Memory grows over time
curl -s http://localhost:9901/memory | jq '.total_allocated_bytes'
# Keeps increasing

# OOM killer triggers
dmesg | grep -i "out of memory"
```

### Diagnosis Steps

#### 1. Enable ASAN Leak Detection

```bash
# Build with ASAN
bazel build -c dbg --config=asan //source/exe:envoy-static

# Run with leak detection
export ASAN_OPTIONS=detect_leaks=1
./bazel-bin/source/exe/envoy-static -c config.yaml
# Ctrl+C to exit and see leak report
```

#### 2. Use Heap Profiler

```bash
# Enable heap profiling
curl -X POST http://localhost:9901/heapprofiler?enable=y

# Run workload for a while
sleep 300

# Disable and analyze
curl -X POST http://localhost:9901/heapprofiler?enable=n
pprof --text ./envoy-static /tmp/envoy.hprof
```

#### 3. Monitor Memory Stats

```bash
# Create monitoring script
#!/bin/bash
while true; do
  echo "$(date) $(curl -s http://localhost:9901/memory | jq '.total_allocated_bytes')"
  sleep 60
done
```

### Common Leak Sources

#### 1. Filter State Not Cleaned

**Problem**: Filter state accumulates without cleanup

**Solution**:
```cpp
class MyFilter : public Http::StreamDecoderFilter {
  ~MyFilter() override {
    // Ensure cleanup
    cleanup();
  }
  
  void cleanup() {
    // Release resources
    connection_.reset();
    buffer_.clear();
  }
  
  void onDestroy() override {
    cleanup();
  }
};
```

#### 2. Circular References

**Problem**: Shared pointers create cycles

**Solution**:
```cpp
// Use weak_ptr to break cycles
class Parent {
  std::shared_ptr<Child> child_;
};

class Child {
  std::weak_ptr<Parent> parent_;  // Weak reference
};
```

#### 3. Event Dispatcher Callbacks

**Problem**: Callbacks not cancelled

**Solution**:
```cpp
class Component {
  ~Component() {
    // Cancel timers
    if (timer_) {
      timer_->disableTimer();
    }
  }
  
  Event::TimerPtr timer_;
};
```

---

## Crash Troubleshooting

### Symptoms

```bash
# Segmentation fault
Segmentation fault (core dumped)

# Or in logs
[critical][assert] assert failure: ...
```

### Diagnosis Steps

#### 1. Get Stack Trace

```bash
# Enable core dumps
ulimit -c unlimited

# After crash, analyze
gdb ./envoy-static core.12345
(gdb) backtrace full
```

#### 2. Run with ASAN

```bash
# Build and run with ASAN
bazel build -c dbg --config=asan //source/exe:envoy-static
./bazel-bin/source/exe/envoy-static -c config.yaml

# ASAN will catch issues like:
# - Buffer overflows
# - Use after free
# - Memory leaks
```

#### 3. Check for NULL Dereference

```cpp
// Common pattern
if (ptr != nullptr) {
  ptr->method();
}

// Or use optional
if (optional_value.has_value()) {
  use(optional_value.value());
}
```

### Common Crash Causes

#### 1. Null Pointer Dereference

**Stack trace shows**:
```
#0  ClassName::method (this=0x..., ptr=0x0)
```

**Solution**:
```cpp
// Always check pointers
if (upstream_host != nullptr) {
  upstream_host->stats().increment();
}

// Or use OptRef
if (auto host = upstream_info.upstreamHost(); host.has_value()) {
  host->stats().increment();
}
```

#### 2. Use After Free

**ASAN shows**:
```
ERROR: AddressSanitizer: heap-use-after-free
```

**Solution**:
- Use shared_ptr for shared ownership
- Ensure callbacks check object lifetime
- Use weak_ptr where appropriate

#### 3. Stack Overflow

**Very deep stack trace**

**Solution**:
- Check for infinite recursion
- Reduce recursion depth
- Use iteration instead

#### 4. Assertion Failures

**Logs show**:
```
[critical][assert] assert failure: stream_info != nullptr
```

**Solution**:
- Review assertion condition
- Check why precondition not met
- Fix calling code

---

## Timeout Issues

### Symptoms

```bash
# Stats showing timeouts
curl http://localhost:9901/stats | grep timeout
cluster.my_cluster.upstream_rq_timeout: 50

# Logs showing
[warning][router] upstream timeout
```

### Types of Timeouts

#### 1. Connection Timeout

**Diagnosis**:
```bash
curl http://localhost:9901/stats | grep connect_timeout
cluster.my_cluster.upstream_cx_connect_timeout: 10
```

**Solution**:
```yaml
clusters:
- name: my_cluster
  connect_timeout: 5s  # Increase if needed
```

#### 2. Request Timeout

**Diagnosis**:
```bash
curl http://localhost:9901/stats | grep rq_timeout
cluster.my_cluster.upstream_rq_timeout: 25
```

**Solution**:
```yaml
# Route-level timeout
routes:
- match: { prefix: "/" }
  route:
    cluster: my_cluster
    timeout: 30s  # Increase

# Or cluster-level
clusters:
- name: my_cluster
  common_http_protocol_options:
    idle_timeout: 60s
```

#### 3. Stream Idle Timeout

**Diagnosis**:
```bash
# Logs showing
[debug][connection] stream idle timeout
```

**Solution**:
```yaml
http_connection_manager:
  stream_idle_timeout: 300s  # 5 minutes
  request_timeout: 0s        # Disable if needed
```

#### 4. Per-Try Timeout

**Diagnosis**:
```bash
# Check retry stats
curl http://localhost:9901/stats | grep retry
cluster.my_cluster.upstream_rq_per_try_timeout: 15
```

**Solution**:
```yaml
routes:
- match: { prefix: "/" }
  route:
    cluster: my_cluster
    retry_policy:
      per_try_timeout: 10s  # Adjust
```

---

## Health Check Failures

### Symptoms

```bash
# Cluster shows unhealthy
curl http://localhost:9901/clusters | grep healthy
my_cluster::10.0.1.100:8080::health_flags::/failed_active_hc

# Stats
curl http://localhost:9901/stats | grep health_check
cluster.my_cluster.health_check.failure: 10
```

### Common Causes and Solutions

#### 1. Health Check Endpoint Not Responding

**Diagnosis**:
```bash
# Test health check endpoint directly
curl http://upstream:8080/health
```

**Solution**:
- Ensure health endpoint exists
- Check upstream service logs
- Verify endpoint returns correct status

#### 2. Health Check Too Aggressive

**Diagnosis**:
```bash
# Frequent health check failures
curl http://localhost:9901/stats | grep health_check.attempt
```

**Solution**:
```yaml
health_checks:
- timeout: 5s              # Increase
  interval: 10s            # Increase
  unhealthy_threshold: 3   # Increase
  healthy_threshold: 2
  http_health_check:
    path: /health
```

#### 3. Wrong Health Check Configuration

**Diagnosis**:
```bash
# Check health check config
curl -s http://localhost:9901/config_dump | \
  jq '.configs[1].dynamic_active_clusters[] | 
      select(.cluster.name == "my_cluster") | 
      .cluster.health_checks'
```

**Solution**:
```yaml
# Verify settings match upstream
health_checks:
- http_health_check:
    path: /health           # Correct path
    expected_statuses:      # Accept these codes
    - start: 200
      end: 299
```

---

## Route Matching Problems

### Symptoms

```bash
# 404 Not Found
HTTP/1.1 404 Not Found

# Stats showing no route
curl http://localhost:9901/stats | grep no_route
http.ingress.downstream_rq_4xx: 100
```

### Diagnosis

```bash
# Check route configuration
curl -s http://localhost:9901/config_dump | \
  jq '.configs[3].dynamic_route_configs[].route_config'

# Enable router debug logging
curl -X POST "http://localhost:9901/logging?router=trace"

# Make test request and check logs
curl -v http://localhost:8000/api/test
```

### Common Causes and Solutions

#### 1. Route Order Wrong

**Problem**: More specific route after catch-all

**Solution**:
```yaml
# Put specific routes FIRST
routes:
- match:
    prefix: "/api/v2"           # More specific
  route:
    cluster: api_v2
- match:
    prefix: "/api"              # Less specific
  route:
    cluster: api_v1
- match:
    prefix: "/"                 # Catch-all LAST
  route:
    cluster: default
```

#### 2. Case Sensitivity

**Problem**: Path case doesn't match

**Solution**:
```yaml
routes:
- match:
    path: "/API/test"
    case_sensitive: false  # Ignore case
  route:
    cluster: my_cluster
```

#### 3. Host/Authority Mismatch

**Problem**: Virtual host not matching

**Solution**:
```yaml
virtual_hosts:
- name: api
  domains:
  - "api.example.com"
  - "api.example.com:*"  # Include port variants
  - "*.example.com"      # Wildcard if needed
  routes:
  - match: { prefix: "/" }
    route: { cluster: api }
```

#### 4. Method Mismatch

**Problem**: HTTP method not allowed

**Solution**:
```yaml
routes:
- match:
    prefix: "/api"
    headers:
    - name: ":method"
      exact_match: "POST"  # Only POST allowed
  route:
    cluster: api
```

---

## Filter Chain Issues

### Symptoms

```bash
# Connection rejected
[warning][filter] no matching filter chain found

# Or filters not executing
[debug][filter] filter chain matched
```

### Common Causes and Solutions

#### 1. No Matching Filter Chain

**Diagnosis**:
```bash
# Check filter chain matchers
curl -s http://localhost:9901/config_dump | \
  jq '.configs[0].dynamic_listeners[].active_state.listener.filter_chains'
```

**Solution**:
```yaml
listeners:
- filter_chains:
  # Add default filter chain (no matcher)
  - filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config: { ... }
```

#### 2. TLS vs Non-TLS Mismatch

**Problem**: Client uses TLS but filter chain expects plain

**Solution**:
```yaml
filter_chains:
# TLS filter chain
- filter_chain_match:
    transport_protocol: "tls"
  transport_socket:
    name: envoy.transport_sockets.tls
    typed_config: { ... }
  filters: [ ... ]

# Plain text filter chain
- filter_chain_match:
    transport_protocol: "raw_buffer"
  filters: [ ... ]
```

#### 3. SNI Mismatch

**Problem**: SNI doesn't match filter chain

**Solution**:
```yaml
filter_chains:
- filter_chain_match:
    server_names:
    - "*.example.com"
    - "example.com"
  filters: [ ... ]
```

---

## Quick Troubleshooting Checklist

### When Things Go Wrong

1. **Check if Envoy is running**:
   ```bash
   curl http://localhost:9901/ready
   ```

2. **Look for errors in logs**:
   ```bash
   grep -i error /var/log/envoy/envoy.log | tail -20
   ```

3. **Check basic stats**:
   ```bash
   curl http://localhost:9901/stats | grep -E "error|failure|timeout" | grep -v ": 0"
   ```

4. **Verify configuration**:
   ```bash
   ./envoy-static --mode validate -c config.yaml
   ```

5. **Check cluster health**:
   ```bash
   curl http://localhost:9901/clusters | grep -A 5 "my_cluster"
   ```

6. **Enable debug logging**:
   ```bash
   curl -X POST "http://localhost:9901/logging?level=debug"
   ```

7. **Capture traffic**:
   ```bash
   sudo tcpdump -i any -w /tmp/debug.pcap port 8000
   ```

---

## Getting Help

### Collect Diagnostic Information

```bash
#!/bin/bash
# Create debug bundle
mkdir -p envoy-debug
cd envoy-debug

# Get config
curl -s http://localhost:9901/config_dump > config_dump.json

# Get stats
curl -s http://localhost:9901/stats > stats.txt

# Get clusters
curl -s http://localhost:9901/clusters > clusters.txt

# Get server info
curl -s http://localhost:9901/server_info > server_info.json

# Get logs (last 1000 lines)
tail -1000 /var/log/envoy/envoy.log > envoy.log

# Create tarball
cd ..
tar czf envoy-debug-$(date +%Y%m%d-%H%M%S).tar.gz envoy-debug/
```

### Where to Ask

- GitHub Issues: https://github.com/envoyproxy/envoy/issues
- Slack: https://envoyproxy.slack.com
- Mailing List: envoy-users@googlegroups.com

---

## Related Documentation

- [Error Handling Patterns](01-error-handling-patterns.md)
- [Logging and Tracing](02-logging-and-tracing.md)
- [Debugging Tools](03-debugging-tools.md)

## References

- Envoy Troubleshooting Guide
- Envoy Operations Documentation
- Envoy FAQ

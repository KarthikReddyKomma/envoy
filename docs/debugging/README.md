# Debugging and Error Handling in Envoy

Welcome to the comprehensive debugging and error handling documentation for Envoy. This guide helps you understand, diagnose, and fix issues in Envoy deployments.

## Quick Start

### Is Your Envoy Healthy?

```bash
# Check basic health
curl http://localhost:9901/ready

# Quick error check
curl http://localhost:9901/stats | grep -E "error|failure" | grep -v ": 0"

# View recent logs
tail -f /var/log/envoy/envoy.log
```

### Common Commands

```bash
# Get configuration
curl http://localhost:9901/config_dump | jq '.'

# Check cluster health
curl http://localhost:9901/clusters

# Enable debug logging
curl -X POST "http://localhost:9901/logging?level=debug"

# View stats
curl http://localhost:9901/stats | grep http
```

## Documentation Structure

### 1. [Error Handling Patterns](01-error-handling-patterns.md)

Learn about Envoy's error handling mechanisms:

- **Status and StatusOr**: Type-safe error handling pattern
- **Error Propagation**: How errors flow through the system
- **Response Flags**: Understanding why requests fail
- **Connection Close Reasons**: Diagnosing connection issues
- **Response Code Details**: Detailed error explanations

**When to use**: Understanding error patterns in code, implementing new features, debugging specific failures.

**Quick Example**:
```cpp
StatusOr<Config> loadConfig(const std::string& path) {
  if (path.empty()) {
    return ConfigurationError("Path cannot be empty");
  }
  // ... load and return config
  return config;
}
```

### 2. [Logging and Tracing](02-logging-and-tracing.md)

Master Envoy's logging system:

- **Logger System**: Architecture and components
- **Log Levels**: trace, debug, info, warn, error, critical
- **Logging Macros**: ENVOY_LOG, ENVOY_CONN_LOG, ENVOY_STREAM_LOG
- **Fine-Grain Logging**: File-level log control
- **Connection Tracking**: Following requests through the system
- **Custom Tags**: Adding context to logs

**When to use**: Debugging live systems, understanding request flow, troubleshooting production issues.

**Quick Example**:
```cpp
ENVOY_CONN_LOG(info, "Connection established from {}", 
               connection, peer_address);

ENVOY_LOG_EVERY_NTH(warn, 100, 
                   "High error rate: {}", error_count);
```

### 3. [Debugging Tools](03-debugging-tools.md)

Use powerful debugging tools:

- **Admin Interface**: Runtime visibility and control (/stats, /config_dump, /clusters)
- **GDB**: Interactive debugging with breakpoints
- **Core Dumps**: Post-mortem analysis
- **ASAN/TSAN**: Memory and thread error detection
- **Performance Profiling**: CPU and heap profiling
- **Network Tools**: tcpdump, wireshark, strace

**When to use**: Deep debugging, performance analysis, crash investigation.

**Quick Example**:
```bash
# Check for connection failures
curl http://localhost:9901/stats | grep cx_connect_fail

# Profile for 30 seconds
sudo perf record -F 99 -p <pid> -g -- sleep 30
sudo perf report

# Run with ASAN
bazel build -c dbg --config=asan //source/exe:envoy-static
```

### 4. [Common Issues](04-common-issues.md)

Solve frequently encountered problems:

- **Connection Failures**: DNS, firewall, pool exhaustion
- **TLS Issues**: Certificate validation, expiry, SNI
- **Configuration Errors**: Syntax, validation, deprecation
- **Performance**: Connection pools, memory, filters
- **Memory Leaks**: Detection and prevention
- **Crashes**: NULL pointers, use-after-free, assertions
- **Timeouts**: Request, connection, stream idle
- **Health Checks**: Configuration and troubleshooting

**When to use**: Fixing specific problems, learning from common mistakes.

**Quick Example**:
```bash
# Diagnose timeout issues
curl http://localhost:9901/stats | grep timeout | grep -v ": 0"

# Check cluster health
curl http://localhost:9901/clusters | grep health_flags

# Validate configuration
./envoy-static --mode validate -c config.yaml
```

## Troubleshooting Flowchart

```
                    ┌─────────────────────┐
                    │  Issue Detected?    │
                    └──────────┬──────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
         ┌──────▼─────┐  ┌────▼────┐  ┌─────▼──────┐
         │ Connection │  │ Request │  │   Crash    │
         │  Failure   │  │ Timeout │  │ Segfault   │
         └──────┬─────┘  └────┬────┘  └─────┬──────┘
                │              │              │
         ┌──────▼─────────────▼──────────────▼──────┐
         │  Check Admin Interface                    │
         │  curl localhost:9901/stats               │
         │  curl localhost:9901/clusters            │
         └──────┬──────────────────────────────┬────┘
                │                              │
         ┌──────▼──────────┐          ┌───────▼──────┐
         │ Enable Debug    │          │ Collect Core │
         │ Logging         │          │ Dump         │
         └──────┬──────────┘          └───────┬──────┘
                │                              │
         ┌──────▼──────────┐          ┌───────▼──────┐
         │ Check Response  │          │ Analyze with │
         │ Flags & Details │          │ GDB/ASAN     │
         └──────┬──────────┘          └───────┬──────┘
                │                              │
                └──────────────┬───────────────┘
                               │
                      ┌────────▼─────────┐
                      │  Problem Solved? │
                      └────────┬─────────┘
                               │
                        ┌──────┴──────┐
                      Yes             No
                        │              │
                   ┌────▼───┐    ┌────▼──────────┐
                   │ Done!  │    │ See Common    │
                   └────────┘    │ Issues Guide  │
                                 └───────────────┘
```

## Quick Reference

### Response Flags Decoder

| Flag | Meaning | Check |
|------|---------|-------|
| UH | No healthy upstream | `curl .../clusters` |
| UT | Upstream timeout | `curl .../stats \| grep timeout` |
| UF | Upstream connection failure | Check upstream logs |
| UC | Upstream connection termination | Network issues |
| UO | Upstream overflow | Circuit breaker tripped |
| NR | No route found | Check route config |
| RL | Rate limited | Check rate limit config |
| UAEX | External authz denied | Check authz service |
| DPE | Downstream protocol error | Invalid HTTP |

### Admin Interface Quick Reference

| Endpoint | Purpose | Example |
|----------|---------|---------|
| `/stats` | Get all statistics | Filter with `?filter=http` |
| `/config_dump` | Current configuration | Pipe to `jq` for parsing |
| `/clusters` | Cluster status & health | Check upstream health |
| `/logging` | Control log levels | `POST ?level=debug` |
| `/ready` | Health check | Returns 200 if ready |
| `/server_info` | Version & uptime | Get Envoy version |
| `/memory` | Memory usage | Monitor for leaks |
| `/certs` | Certificate info | Check cert expiry |

### Log Level Guidelines

| Level | When to Use | Example |
|-------|-------------|---------|
| trace | Function entry/exit, detailed flow | Development only |
| debug | Debugging info, state changes | Development/staging |
| info | Important events | Production (default) |
| warn | Unexpected but handled | Production |
| error | Error conditions | Production |
| critical | System-threatening errors | Production |

### Useful Log Patterns

```bash
# Show only errors
tail -f envoy.log | grep "\[error\]"

# Show connection logs
tail -f envoy.log | grep "\[connection\]"

# Show specific connection
tail -f envoy.log | grep "ConnectionId\":\"123\""

# Show timeouts
tail -f envoy.log | grep -i timeout

# Count errors by type
grep "\[error\]" envoy.log | cut -d']' -f5- | sort | uniq -c | sort -rn
```

## Debugging Scenarios

### Scenario 1: 503 Service Unavailable

```bash
# 1. Check cluster health
curl http://localhost:9901/clusters | grep my_cluster

# 2. Check for no healthy upstream
curl http://localhost:9901/stats | grep my_cluster | grep health

# 3. Look at response flags
curl http://localhost:9901/stats | grep response_flags

# 4. Check logs for connection failures
tail -100 /var/log/envoy/envoy.log | grep -i "connect\|health"
```

**Common causes**: All upstreams failed health checks, connection failures, DNS issues.

**See**: [Connection Failures](04-common-issues.md#connection-failures)

### Scenario 2: Slow Responses

```bash
# 1. Check latency stats
curl http://localhost:9901/stats | grep duration

# 2. Check for timeouts
curl http://localhost:9901/stats | grep timeout | grep -v ": 0"

# 3. Check connection pool
curl http://localhost:9901/stats | grep cx_pool

# 4. Enable request tracing
curl -X POST "http://localhost:9901/logging?http=trace"
```

**Common causes**: Slow upstreams, connection pool exhaustion, timeout too short.

**See**: [Performance Problems](04-common-issues.md#performance-problems)

### Scenario 3: Memory Growing

```bash
# 1. Check memory over time
watch -n 10 'curl -s http://localhost:9901/memory | jq .total_allocated_bytes'

# 2. Enable heap profiling
curl -X POST http://localhost:9901/heapprofiler?enable=y
# ... wait ...
curl -X POST http://localhost:9901/heapprofiler?enable=n

# 3. Run with ASAN
bazel build -c dbg --config=asan //source/exe:envoy-static
export ASAN_OPTIONS=detect_leaks=1
./envoy-static -c config.yaml
```

**Common causes**: Filter state not cleaned, circular references, uncancelled callbacks.

**See**: [Memory Leaks](04-common-issues.md#memory-leaks)

### Scenario 4: TLS Handshake Failures

```bash
# 1. Check certificate status
curl http://localhost:9901/certs

# 2. Check for TLS errors in logs
grep -i "tls\|ssl\|certificate" /var/log/envoy/envoy.log | tail -20

# 3. Test certificate manually
openssl s_client -connect upstream:443 -servername hostname

# 4. Check cipher compatibility
openssl s_client -connect upstream:443 -cipher 'ALL'
```

**Common causes**: Expired certificates, SNI mismatch, protocol version mismatch.

**See**: [TLS/Certificate Issues](04-common-issues.md#tlscertificate-issues)

## Best Practices

### Development

1. **Build with debug symbols**: `bazel build -c dbg`
2. **Use ASAN/TSAN**: Catch bugs early
3. **Write tests**: Unit and integration tests
4. **Use StatusOr**: For error handling
5. **Log liberally**: Use appropriate levels
6. **Validate config**: Before deployment

### Production

1. **Monitor stats**: Set up alerting on key metrics
2. **Structured logging**: Use JSON format for parsing
3. **Keep debug symbols**: Separate file for crash analysis
4. **Enable core dumps**: With size limits
5. **Rate limit logs**: Use ENVOY_LOG_EVERY_NTH
6. **Admin interface**: Secure but accessible

### Debugging

1. **Start broad**: Check stats and logs first
2. **Enable debug logging**: Temporarily for specific components
3. **Use admin interface**: Runtime visibility
4. **Capture traffic**: tcpdump when needed
5. **Reproduce locally**: Simplify the scenario
6. **Check common issues**: Before deep diving

## Tools Installation

### GDB

```bash
# Ubuntu/Debian
sudo apt-get install gdb

# CentOS/RHEL
sudo yum install gdb

# macOS
brew install gdb
```

### Sanitizers

```bash
# Usually included with compiler
# Check availability
clang --version | grep -i sanitizer
gcc --version | grep -i sanitizer
```

### Network Tools

```bash
# Ubuntu/Debian
sudo apt-get install tcpdump wireshark netcat-openbsd

# CentOS/RHEL
sudo yum install tcpdump wireshark nc

# macOS
brew install tcpdump wireshark netcat
```

### Performance Tools

```bash
# perf (Linux only)
sudo apt-get install linux-tools-common linux-tools-generic

# pprof
go get github.com/google/pprof

# FlameGraph
git clone https://github.com/brendangregg/FlameGraph
```

## Additional Resources

### Official Documentation

- [Envoy Documentation](https://www.envoyproxy.io/docs)
- [Envoy API Reference](https://www.envoyproxy.io/docs/envoy/latest/api-v3/api)
- [Envoy Operations Guide](https://www.envoyproxy.io/docs/envoy/latest/operations/operations)

### Source Code

- [Envoy GitHub](https://github.com/envoyproxy/envoy)
- [Common Status/StatusOr](https://github.com/envoyproxy/envoy/tree/main/source/common/common)
- [Logger Implementation](https://github.com/envoyproxy/envoy/tree/main/source/common/common)

### Community

- [Envoy Slack](https://envoyproxy.slack.com)
- [Mailing List](https://groups.google.com/forum/#!forum/envoy-users)
- [GitHub Issues](https://github.com/envoyproxy/envoy/issues)

### Related Envoy Docs

- [Admin Interface Operations](../admin-operations/)
- [Request Flow and Filters](../request-flow/)
- [Configuration Reference](../core-runtime-xds-istio/)
- [Architecture Overview](../ENVOY_ARCHITECTURE_REFERENCE.md)

## Contributing

Found an issue or have suggestions? Please contribute:

1. Check existing documentation
2. Test your changes
3. Submit clear examples
4. Update relevant sections
5. Link to source code when applicable

## FAQ

### Q: How do I enable debug logging for just one component?

```bash
curl -X POST "http://localhost:9901/logging?router=debug"
```

### Q: What's the difference between trace and debug logs?

- **trace**: Very detailed, function-level, disabled in production builds
- **debug**: Debugging information, useful for troubleshooting, can be enabled in production

### Q: How do I capture a core dump?

```bash
# Enable core dumps
ulimit -c unlimited

# Set pattern
echo "core.%e.%p" | sudo tee /proc/sys/kernel/core_pattern

# After crash
gdb ./envoy-static core.*
```

### Q: What are the most important stats to monitor?

```
# Errors
*.upstream_cx_connect_fail
*.upstream_cx_connect_timeout
*.upstream_rq_timeout
*.upstream_rq_5xx

# Health
cluster.*.health_flags
cluster.*.membership_healthy

# Performance
*.upstream_rq_time (p50, p95, p99)
*.downstream_cx_active
*.upstream_cx_active
```

### Q: How do I debug "no healthy upstream"?

1. Check cluster health: `curl .../clusters`
2. Check health check config: `curl .../config_dump`
3. Test health endpoint directly
4. Check upstream service logs
5. Verify network connectivity

### Q: Can I use Envoy in production with ASAN?

**No**. ASAN has significant performance overhead. Use it in development/testing only.

---

**Last Updated**: 2024-04-26

**Version**: Envoy 1.x

For questions or issues, consult the [Common Issues](04-common-issues.md) guide or reach out on Envoy Slack.

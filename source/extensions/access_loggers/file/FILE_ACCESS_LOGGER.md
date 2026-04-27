# File Access Logger

## Introduction

The File Access Logger writes access logs to local files with configurable formats. It's the simplest and most performant access logger, ideal for local development, simple deployments, or when used with log shippers.

### Key Features

- **Multiple Formats**: Text, JSON, or custom format strings
- **High Performance**: Direct file I/O, no network overhead
- **Format Substitution**: Rich variable substitution (e.g., %REQ(:METHOD)%)
- **Flexible Output**: stdout, stderr, or regular files
- **Zero Latency Impact**: Writes after response sent

### When to Use

- **Local development and debugging**
- **Simple deployments without centralized logging**
- **Maximum throughput requirements**
- **With log shippers** (Filebeat, Fluentd file tail)
- **Container environments** (write to stdout, container runtime handles forwarding)

## Architecture

```mermaid
graph LR
    A[Access Log Call] --> B[Filter Check]
    B --> C[Format String Substitution]
    C --> D[Write to File]
    D --> E[OS Buffer]
    E --> F[Disk/stdout]
```

## Configuration

### Basic Text Format

```yaml
access_log:
  - name: envoy.access_loggers.file
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
      path: /var/log/envoy/access.log
      format: "[%START_TIME%] \"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%\" %RESPONSE_CODE% %RESPONSE_FLAGS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION% \"%REQ(X-FORWARDED-FOR)%\" \"%REQ(USER-AGENT)%\" \"%REQ(X-REQUEST-ID)%\" \"%REQ(:AUTHORITY)%\" \"%UPSTREAM_HOST%\"\n"
```

### JSON Format

```yaml
access_log:
  - name: envoy.access_loggers.file
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
      path: /var/log/envoy/access.log
      log_format:
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
          upstream_host: "%UPSTREAM_HOST%"
          x_forwarded_for: "%REQ(X-FORWARDED-FOR)%"
          user_agent: "%REQ(USER-AGENT)%"
          request_id: "%REQ(X-REQUEST-ID)%"
          authority: "%REQ(:AUTHORITY)%"
          upstream_cluster: "%UPSTREAM_CLUSTER%"
```

### stdout (Container-Friendly)

```yaml
access_log:
  - name: envoy.access_loggers.file
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
      path: /dev/stdout
      log_format:
        json_format:
          # ... JSON fields ...
```

## Format String Substitution

### Common Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `%START_TIME%` | Request start time | `2024-04-26T10:30:00.123Z` |
| `%REQ(:METHOD)%` | HTTP method | `GET` |
| `%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%` | Original path (with fallback) | `/api/users` |
| `%PROTOCOL%` | HTTP protocol | `HTTP/1.1` |
| `%RESPONSE_CODE%` | Response status code | `200` |
| `%RESPONSE_FLAGS%` | Response flags | `UH` (upstream host) |
| `%BYTES_RECEIVED%` | Bytes received | `1234` |
| `%BYTES_SENT%` | Bytes sent | `5678` |
| `%DURATION%` | Total duration (ms) | `125` |
| `%RESPONSE_DURATION%` | Time to first byte (ms) | `100` |
| `%REQ(HEADER)%` | Request header | `%REQ(USER-AGENT)%` |
| `%RESP(HEADER)%` | Response header | `%RESP(CONTENT-TYPE)%` |
| `%DOWNSTREAM_REMOTE_ADDRESS%` | Client IP | `192.168.1.100:54321` |
| `%DOWNSTREAM_LOCAL_ADDRESS%` | Envoy listen address | `0.0.0.0:8080` |
| `%UPSTREAM_HOST%` | Upstream server | `10.0.1.50:80` |
| `%UPSTREAM_CLUSTER%` | Upstream cluster name | `backend_service` |
| `%UPSTREAM_LOCAL_ADDRESS%` | Envoy upstream address | `10.0.0.5:45678` |

### Full Variable List

See: https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#command-operators

## Performance

### Characteristics

- **Latency**: ~10μs per log (in-memory buffer write)
- **Throughput**: 100,000+ req/s
- **CPU**: <0.5% (most workloads)
- **Memory**: ~Minimal (OS manages file buffer)

### Optimization

```yaml
# Use simple format for high throughput
format: "[%START_TIME%] %RESPONSE_CODE% %DURATION%\n"

# Or use filter to reduce volume
filter:
  status_code_filter:
    comparison:
      op: GE
      value: 400  # Only errors
```

## Best Practices

### 1. Use JSON for Production

Easier to parse by log aggregators:

```yaml
log_format:
  json_format:
    "@timestamp": "%START_TIME%"
    level: "info"
    # ... fields ...
```

### 2. Include Essential Fields

```yaml
json_format:
  timestamp: "%START_TIME%"
  method: "%REQ(:METHOD)%"
  path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
  status: "%RESPONSE_CODE%"
  duration_ms: "%DURATION%"
  upstream: "%UPSTREAM_CLUSTER%"
  request_id: "%REQ(X-REQUEST-ID)%"  # For tracing
  flags: "%RESPONSE_FLAGS%"  # For debugging
```

### 3. Container Best Practice

```yaml
# Write to stdout
path: /dev/stdout

# Let container runtime handle:
# - Log rotation
# - Forwarding to log aggregator
# - Cleanup
```

### 4. File Rotation

For regular files, use external log rotation:

```bash
# logrotate config
/var/log/envoy/*.log {
    daily
    rotate 7
    compress
    delaycompress
    notifempty
    create 0640 envoy envoy
    sharedscripts
    postrotate
        # Envoy handles USR1 for log rotation
        kill -USR1 $(cat /var/run/envoy.pid)
    endscript
}
```

### 5. Avoid Sensitive Data

```yaml
# DON'T log:
# - Authorization headers
# - Cookies
# - Request/response bodies with PII
```

## Related Documentation

- [ACCESS_LOGGERS_OVERVIEW.md](../ACCESS_LOGGERS_OVERVIEW.md)
- [STREAM_ACCESS_LOGGER.md](../stream/STREAM_ACCESS_LOGGER.md) (similar, but stdout)
- [ACCESS_LOG_FILTERS.md](../filters/ACCESS_LOG_FILTERS.md)

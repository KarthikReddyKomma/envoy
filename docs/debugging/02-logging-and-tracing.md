# Logging and Tracing in Envoy

This document covers Envoy's logging system, log levels, tracing capabilities, and debugging techniques using logs.

## Table of Contents

1. [Logger System Overview](#logger-system-overview)
2. [Log Levels](#log-levels)
3. [Logging Macros](#logging-macros)
4. [Logger IDs](#logger-ids)
5. [Fine-Grain Logging](#fine-grain-logging)
6. [Connection and Stream Tracking](#connection-and-stream-tracking)
7. [Request Tracing](#request-tracing)
8. [Custom Log Tags](#custom-log-tags)
9. [Log Configuration](#log-configuration)
10. [Debugging with Logs](#debugging-with-logs)

---

## Logger System Overview

Envoy uses the **spdlog** library with custom wrappers for structured, efficient logging.

### Architecture

```
┌─────────────────┐
│  ENVOY_LOG()    │  Logging Macros
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Logger         │  Component-specific loggers
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  DelegatingLog  │  Sink delegation
│  Sink           │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  StderrSink     │  Output destination
└─────────────────┘
```

### Key Components

From `source/common/common/logger.h`:

1. **Logger Registry**: Manages all loggers by ID
2. **DelegatingLogSink**: Routes logs to appropriate sinks
3. **Context**: Sets log level, format, and threading
4. **Loggable<id>**: Mixin for component-specific logging

---

## Log Levels

### Available Levels

```cpp
namespace spdlog {
  enum level_enum {
    trace,     // Most verbose
    debug,     // Debug information
    info,      // Informational
    warn,      // Warnings
    error,     // Errors
    critical,  // Critical errors
    off        // Disable logging
  };
}
```

### Level Usage Guidelines

| Level | Use Case | Example |
|-------|----------|---------|
| `trace` | Detailed execution flow | Function entry/exit, variable values |
| `debug` | Debugging information | Configuration details, state changes |
| `info` | General information | Connection established, request processed |
| `warn` | Unexpected but handled | Retry attempts, fallback behavior |
| `error` | Error conditions | Connection failures, invalid config |
| `critical` | System-threatening | Fatal errors, assertion failures |

---

## Logging Macros

### Basic Logging

```cpp
#include "source/common/common/logger.h"

class MyFilter : public Logger::Loggable<Logger::Id::filter> {
public:
  void processRequest() {
    ENVOY_LOG(trace, "Entering processRequest");
    ENVOY_LOG(debug, "Processing request for path: {}", path);
    ENVOY_LOG(info, "Request processing completed successfully");
    ENVOY_LOG(warn, "Retry attempt {} of {}", attempt, max_attempts);
    ENVOY_LOG(error, "Failed to connect to upstream: {}", error_msg);
    ENVOY_LOG(critical, "Unrecoverable error: {}", error_msg);
  }
};
```

### Conditional Logging

```cpp
// Log only if level is enabled (optimization)
if (ENVOY_LOG_CHECK_LEVEL(debug)) {
  ENVOY_LOG(debug, "Expensive debug info: {}", computeExpensiveInfo());
}
```

### Logging to Specific Logger

```cpp
// Use a different logger than the class default
ENVOY_LOG_TO_LOGGER(GET_MISC_LOGGER(), info, "General information");
ENVOY_LOG_MISC(info, "Same as above - logs to misc logger");
```

### Rate-Limited Logging

```cpp
// Log only the first N occurrences
ENVOY_LOG_FIRST_N(warn, 5, "This warning appears at most 5 times");

// Log once
ENVOY_LOG_ONCE(error, "This error is logged only once");

// Log every Nth occurrence
ENVOY_LOG_EVERY_NTH(info, 100, "Logged every 100th time");

// Log every power of 2 (1, 2, 4, 8, 16...)
ENVOY_LOG_EVERY_POW_2(debug, "Logged at 1, 2, 4, 8, 16... occurrences");

// Log periodically (time-based)
ENVOY_LOG_PERIODIC(info, std::chrono::seconds(60), 
                   "Logged at most once per minute");
```

### Conditional Rate-Limited Logging

```cpp
// Log first N times if condition is true
ENVOY_LOG_FIRST_N_IF(warn, 3, connection_retry_needed,
                     "Connection retry needed");

// Log once if condition is true
ENVOY_LOG_ONCE_IF(error, !config_valid, "Invalid configuration");
```

---

## Logger IDs

### Available Logger IDs

From `source/common/common/logger.h`, Envoy defines numerous logger IDs:

```cpp
enum class Id {
  admin,              // Admin interface
  aws,                // AWS integration
  assert,             // Assertions
  backtrace,          // Stack traces
  client,             // Client operations
  config,             // Configuration
  connection,         // Connection management
  conn_handler,       // Connection handler
  dns,                // DNS resolution
  dubbo,              // Dubbo protocol
  ext_authz,          // External authorization
  ext_proc,           // External processing
  file,               // File operations
  filter,             // Filter operations
  forward_proxy,      // Forward proxy
  grpc,               // gRPC
  hc,                 // Health check
  health_checker,     // Health checker
  http,               // HTTP
  http2,              // HTTP/2
  init,               // Initialization
  io,                 // I/O operations
  jwt,                // JWT processing
  kafka,              // Kafka
  lua,                // Lua scripting
  main,               // Main thread
  matcher,            // Matchers
  mongo,              // MongoDB
  oauth2,             // OAuth2
  quic,               // QUIC protocol
  quic_stream,        // QUIC streams
  pool,               // Connection pools
  rbac,               // RBAC
  redis,              // Redis
  router,             // Router
  runtime,            // Runtime
  stats,              // Statistics
  secret,             // Secrets
  tap,                // Taps
  testing,            // Testing
  thrift,             // Thrift
  tracing,            // Distributed tracing
  upstream,           // Upstream
  udp,                // UDP
  wasm,               // WebAssembly
  // ... and more
};
```

### Using Logger IDs

```cpp
// Inherit from Loggable with specific logger ID
class RouterFilter : public Logger::Loggable<Logger::Id::router> {
public:
  void routeRequest() {
    ENVOY_LOG(debug, "Routing request to cluster: {}", cluster_name);
  }
};

class ConnectionHandler : public Logger::Loggable<Logger::Id::connection> {
public:
  void handleConnection() {
    ENVOY_LOG(info, "New connection from {}", peer_address);
  }
};
```

---

## Fine-Grain Logging

Fine-grain logging allows file-level log control without explicit logger implementation.

### Enabling Fine-Grain Logging

```cpp
// At initialization
Logger::Context context(
  spdlog::level::info,
  "[%Y-%m-%d %T.%e][%t][%l][%n] %v",
  lock,
  false, // escape logs
  true   // enable fine-grain logging
);
```

### Using Fine-Grain Logs

```cpp
// In any source file, logs automatically use file-specific logger
void someFunction() {
  ENVOY_LOG(debug, "This uses file-specific logger automatically");
}
```

### Fine-Grain Logger Control

```cpp
// Check if fine-grain logging is enabled
if (Logger::Context::useFineGrainLogger()) {
  // Fine-grain logging is active
}

// Enable/disable at runtime
Logger::Context::enableFineGrainLogger();
Logger::Context::disableFineGrainLogger();

// Change all log levels
Logger::Context::changeAllLogLevels(spdlog::level::debug);
```

---

## Connection and Stream Tracking

### Connection Logging

```cpp
class ConnectionHandler : public Logger::Loggable<Logger::Id::connection> {
public:
  void handleConnection(Network::Connection& connection) {
    // Automatically adds ConnectionId tag
    ENVOY_CONN_LOG(info, "Connection established", connection);
    
    ENVOY_CONN_LOG(debug, "Remote address: {}", connection,
                   connection.connectionInfoProvider()
                     .remoteAddress()->asString());
    
    ENVOY_CONN_LOG(warn, "Connection timeout after {}ms", 
                   connection, timeout_ms);
  }
};
```

### Stream Logging

```cpp
class HttpFilter : public Logger::Loggable<Logger::Id::http> {
public:
  void onRequest(Http::RequestHeaderMap& headers, 
                Http::StreamEncoder& stream) {
    // Automatically adds ConnectionId and StreamId tags
    ENVOY_STREAM_LOG(info, "Processing request for path: {}", 
                     stream, headers.Path()->value().getStringView());
    
    ENVOY_STREAM_LOG(debug, "Method: {}, Host: {}", 
                     stream,
                     headers.Method()->value().getStringView(),
                     headers.Host()->value().getStringView());
  }
};
```

### Output Format

```
[2024-04-26 15:30:45.123][12345][info][connection] [Tags: "ConnectionId":"42"] Connection established
[2024-04-26 15:30:45.124][12345][debug][http] [Tags: "ConnectionId":"42","StreamId":"1"] Processing request for path: /api/users
```

---

## Request Tracing

### Stable Event Logging

For important events that need stable identifiers:

```cpp
class Filter : public Logger::Loggable<Logger::Id::filter> {
public:
  void onAuthSuccess() {
    // Logs with stable event name for monitoring/alerting
    ENVOY_LOG_EVENT(info, "auth_success",
                   "User {} authenticated successfully", user_id);
  }
  
  void onAuthFailure() {
    ENVOY_LOG_EVENT(warn, "auth_failure",
                   "Authentication failed for user {}: {}", 
                   user_id, reason);
  }
  
  void onRateLimitHit() {
    ENVOY_LOG_EVENT(warn, "rate_limit_exceeded",
                   "Rate limit exceeded for client {}", client_ip);
  }
};
```

### Connection Event Logging

```cpp
ENVOY_CONN_LOG_EVENT(info, "connection_established",
                    "Connection established", connection);

ENVOY_CONN_LOG_EVENT(warn, "connection_timeout",
                    "Connection timeout after {}ms", 
                    connection, timeout_ms);
```

---

## Custom Log Tags

### Adding Custom Tags

```cpp
void processRequest(Http::RequestHeaderMap& headers) {
  std::map<std::string, std::string> tags;
  tags["RequestId"] = request_id;
  tags["UserId"] = user_id;
  tags["ApiVersion"] = "v2";
  
  ENVOY_TAGGED_LOG(info, tags, "Processing API request");
}
```

### Connection Tags

```cpp
void handleConnection(Network::Connection& connection) {
  std::map<std::string, std::string> tags;
  tags["Environment"] = "production";
  tags["Region"] = "us-west-2";
  
  // Automatically adds ConnectionId
  ENVOY_TAGGED_CONN_LOG(info, tags, connection,
                       "Connection from {}", peer_address);
}
```

### Stream Tags

```cpp
void processStream(Http::StreamEncoder& stream) {
  std::map<std::string, std::string> tags;
  tags["TraceId"] = trace_id;
  tags["SpanId"] = span_id;
  
  // Automatically adds ConnectionId and StreamId
  ENVOY_TAGGED_STREAM_LOG(info, tags, stream,
                         "Stream processing started");
}
```

### Output Format

```
[2024-04-26 15:30:45.123][12345][info][filter] [Tags: "RequestId":"abc-123","UserId":"user456","ApiVersion":"v2"] Processing API request
```

---

## Log Configuration

### Setting Log Level at Runtime

```bash
# Set all loggers to debug
curl -X POST "localhost:9901/logging?level=debug"

# Set specific logger to trace
curl -X POST "localhost:9901/logging?connection=trace"

# Set multiple loggers
curl -X POST "localhost:9901/logging?http=debug&router=trace"

# Reset to default
curl -X POST "localhost:9901/logging?level=info"
```

### Setting Log Format

```cpp
// Plain text format
Logger::Registry::setLogFormat(
  "[%Y-%m-%d %T.%e][%t][%l][%n] %v"
);

// JSON format
Logger::Registry::setLogFormat(
  "{\"timestamp\":\"%Y-%m-%dT%H:%M:%S.%e\",\"level\":\"%l\",\"logger\":\"%n\",\"message\":\"%v\"}"
);
```

### Custom Format Flags

```cpp
// Escape newlines in messages
// %_ flag replaces \n with \\n
Logger::Registry::setLogFormat("[%Y-%m-%d %T.%e][%l] %_");

// JSON-escape messages
// %j flag makes message valid JSON string
Logger::Registry::setLogFormat("{\"message\":\"%j\"}");

// Extract tags
// %* extracts [Tags: ...] for JSON
Logger::Registry::setLogFormat("{\"message\":\"%+\"%*}");
```

### Log Configuration in Code

```cpp
class MyComponent {
public:
  MyComponent() {
    // Create logging context
    Logger::Context context(
      spdlog::level::info,                    // Default level
      "[%Y-%m-%d %T.%e][%t][%l][%n] %v",     // Format
      lock_,                                  // Thread lock
      false                                   // Escape logs
    );
  }
  
private:
  Thread::MutexBasicLockable lock_;
};
```

---

## Debugging with Logs

### Debugging Filter Execution

```cpp
class DebugFilter : public Logger::Loggable<Logger::Id::filter> {
public:
  Http::FilterHeadersStatus decodeHeaders(Http::RequestHeaderMap& headers, 
                                         bool end_stream) {
    ENVOY_LOG(trace, "decodeHeaders called, end_stream={}", end_stream);
    
    // Log all headers
    headers.iterate([](const Http::HeaderEntry& header) -> Http::HeaderMap::Iterate {
      ENVOY_LOG(debug, "Header: {} = {}", 
               header.key().getStringView(),
               header.value().getStringView());
      return Http::HeaderMap::Iterate::Continue;
    });
    
    ENVOY_LOG(trace, "decodeHeaders returning Continue");
    return Http::FilterHeadersStatus::Continue;
  }
  
  Http::FilterDataStatus decodeData(Buffer::Instance& data, bool end_stream) {
    ENVOY_LOG(trace, "decodeData called, length={}, end_stream={}",
             data.length(), end_stream);
    return Http::FilterDataStatus::Continue;
  }
};
```

### Debugging Connection Flow

```cpp
void debugConnection(Network::Connection& connection) {
  ENVOY_CONN_LOG(trace, "Connection state: {}", 
                connection, connection.state());
  
  ENVOY_CONN_LOG(debug, "Local: {}, Remote: {}", 
                connection,
                connection.connectionInfoProvider().localAddress()->asString(),
                connection.connectionInfoProvider().remoteAddress()->asString());
  
  if (auto ssl = connection.ssl()) {
    ENVOY_CONN_LOG(debug, "SSL Protocol: {}, Cipher: {}", 
                  connection, ssl->tlsVersion(), ssl->ciphersuiteString());
  }
  
  ENVOY_CONN_LOG(trace, "Bytes sent: {}, received: {}", 
                connection,
                connection.bytesWritten(),
                connection.bytesReceived());
}
```

### Debugging Cluster Selection

```cpp
void debugRouting(const Router::RouteEntry& route) {
  ENVOY_LOG(debug, "Route matched: {}", route.routeName());
  ENVOY_LOG(debug, "Cluster: {}", route.clusterName());
  ENVOY_LOG(debug, "Timeout: {}ms", route.timeout().count());
  
  if (route.retryPolicy()) {
    ENVOY_LOG(debug, "Retry policy: {} retries", 
             route.retryPolicy()->numRetries());
  }
}
```

### Debugging Configuration Loading

```cpp
StatusOr<Config> loadConfig(const std::string& yaml) {
  ENVOY_LOG_MISC(debug, "Loading configuration from YAML");
  
  auto parsed = parseYaml(yaml);
  if (!parsed.ok()) {
    ENVOY_LOG_MISC(error, "YAML parsing failed: {}", 
                  parsed.status().message());
    return parsed.status();
  }
  
  ENVOY_LOG_MISC(trace, "Parsed YAML: {}", parsed.value().DebugString());
  
  auto validated = validateConfig(parsed.value());
  if (!validated.ok()) {
    ENVOY_LOG_MISC(error, "Config validation failed: {}", 
                  validated.status().message());
    return validated.status();
  }
  
  ENVOY_LOG_MISC(info, "Configuration loaded successfully");
  return validated.value();
}
```

### Debugging Performance Issues

```cpp
class PerformanceLogger : public Logger::Loggable<Logger::Id::filter> {
public:
  Http::FilterHeadersStatus decodeHeaders(Http::RequestHeaderMap& headers,
                                         bool end_stream) {
    auto start = std::chrono::steady_clock::now();
    
    // Process headers...
    auto result = processHeaders(headers);
    
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(
      std::chrono::steady_clock::now() - start);
    
    if (duration.count() > 1000) {  // > 1ms
      ENVOY_LOG(warn, "Slow header processing: {}μs", duration.count());
    }
    
    // Log periodically for high-traffic debugging
    ENVOY_LOG_EVERY_NTH(debug, 1000, 
                       "Average processing time: {}μs", 
                       getAverageTime());
    
    return result;
  }
};
```

### Capturing Request/Response Flow

```cpp
class TraceFilter : public Logger::Loggable<Logger::Id::http>,
                   public Http::StreamDecoderFilter,
                   public Http::StreamEncoderFilter {
public:
  // Request path
  Http::FilterHeadersStatus decodeHeaders(Http::RequestHeaderMap& headers,
                                         bool end_stream) override {
    ENVOY_STREAM_LOG(trace, "→ Request headers received", *encoder_callbacks_);
    logHeaders("→ Request", headers);
    return Http::FilterHeadersStatus::Continue;
  }
  
  Http::FilterDataStatus decodeData(Buffer::Instance& data,
                                   bool end_stream) override {
    ENVOY_STREAM_LOG(trace, "→ Request data: {} bytes, end_stream={}",
                    *encoder_callbacks_, data.length(), end_stream);
    return Http::FilterDataStatus::Continue;
  }
  
  // Response path
  Http::FilterHeadersStatus encodeHeaders(Http::ResponseHeaderMap& headers,
                                         bool end_stream) override {
    ENVOY_STREAM_LOG(trace, "← Response headers received", *decoder_callbacks_);
    logHeaders("← Response", headers);
    return Http::FilterHeadersStatus::Continue;
  }
  
  Http::FilterDataStatus encodeData(Buffer::Instance& data,
                                   bool end_stream) override {
    ENVOY_STREAM_LOG(trace, "← Response data: {} bytes, end_stream={}",
                    *decoder_callbacks_, data.length(), end_stream);
    return Http::FilterDataStatus::Continue;
  }
  
private:
  void logHeaders(const char* prefix, const Http::HeaderMap& headers) {
    headers.iterate([this, prefix](const Http::HeaderEntry& header) {
      ENVOY_LOG(trace, "{} {}: {}", prefix,
               header.key().getStringView(),
               header.value().getStringView());
      return Http::HeaderMap::Iterate::Continue;
    });
  }
};
```

---

## Log Analysis Tools

### Filtering Logs

```bash
# Show only errors
grep "\[error\]" envoy.log

# Show connection-related logs
grep "\[connection\]" envoy.log

# Show specific connection
grep "ConnectionId\":\"42\"" envoy.log

# Show logs for specific time range
awk '/2024-04-26 15:30:00/,/2024-04-26 15:31:00/' envoy.log
```

### Parsing JSON Logs

```bash
# Using jq for JSON logs
cat envoy.log | jq 'select(.level == "error")'
cat envoy.log | jq 'select(.logger == "router")'
cat envoy.log | jq 'select(.message | contains("timeout"))'
```

### Aggregating Logs

```bash
# Count errors by logger
grep "\[error\]" envoy.log | cut -d']' -f4 | sort | uniq -c

# Top error messages
grep "\[error\]" envoy.log | cut -d']' -f5- | sort | uniq -c | sort -rn | head -10
```

---

## Best Practices

1. **Use appropriate log levels**:
   - `trace` for detailed debugging (disabled in production)
   - `debug` for development and troubleshooting
   - `info` for important events
   - `warn`/`error` for problems

2. **Include context in log messages**:
   ```cpp
   // Good
   ENVOY_LOG(error, "Connection to {}:{} failed: {}", host, port, reason);
   
   // Bad
   ENVOY_LOG(error, "Connection failed");
   ```

3. **Use connection/stream logging for HTTP**:
   ```cpp
   // Automatically includes IDs
   ENVOY_CONN_LOG(info, "Message", connection);
   ENVOY_STREAM_LOG(info, "Message", stream);
   ```

4. **Use rate limiting for high-frequency logs**:
   ```cpp
   ENVOY_LOG_EVERY_NTH(debug, 100, "Processed {} requests", total);
   ```

5. **Check log level before expensive operations**:
   ```cpp
   if (ENVOY_LOG_CHECK_LEVEL(debug)) {
     ENVOY_LOG(debug, "Details: {}", computeExpensiveDebugInfo());
   }
   ```

6. **Use stable event names for monitoring**:
   ```cpp
   ENVOY_LOG_EVENT(warn, "rate_limit_exceeded", "Details...");
   ```

7. **Flush logs when needed**:
   ```cpp
   ENVOY_FLUSH_LOG();  // Ensure logs are written
   ```

---

## Related Documentation

- [Error Handling Patterns](01-error-handling-patterns.md)
- [Debugging Tools](03-debugging-tools.md)
- [Common Issues](04-common-issues.md)

## References

- Source: `source/common/common/logger.h`
- Source: `source/common/common/logger_impl.h`
- Source: `source/common/common/base_logger.h`

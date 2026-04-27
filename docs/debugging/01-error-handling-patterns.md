# Error Handling Patterns in Envoy

This document covers the core error handling patterns and mechanisms used throughout Envoy's codebase.

## Table of Contents

1. [Status and StatusOr](#status-and-statusor)
2. [Error Propagation Patterns](#error-propagation-patterns)
3. [Response Flags](#response-flags)
4. [Connection Close Reasons](#connection-close-reasons)
5. [Response Code Details](#response-code-details)
6. [Common Error Scenarios](#common-error-scenarios)

---

## Status and StatusOr

### Overview

Envoy uses `Status` and `StatusOr<T>` for explicit error handling, built on top of Abseil's status utilities. This pattern allows functions to return either a valid value or an error without exceptions.

### Basic Usage

```cpp
#include "source/common/common/statusor.h"
#include "source/common/common/status.h"

// Function that returns StatusOr
Envoy::StatusOr<int> parseValue(const std::string& input) {
  if (input.empty()) {
    return CodecProtocolError("Input cannot be empty");
  }
  
  try {
    int value = std::stoi(input);
    return value;  // Success case
  } catch (...) {
    return CodecProtocolError("Invalid integer format");
  }
}

// Calling code
void processValue() {
  auto status_or = parseValue("123");
  
  if (status_or.ok()) {
    int result = status_or.value();
    // Use result...
  } else {
    ENVOY_LOG(debug, "Parse error: {}", status_or.status().message());
    // Handle error...
  }
}
```

### Important Rules

1. **Never use default constructor**: `StatusOr` default constructor is forbidden in Envoy
   ```cpp
   // BAD - Will cause RELEASE_ASSERT
   StatusOr<int> result;
   
   // GOOD - Always initialize with value or error
   StatusOr<int> result = someFunction();
   ```

2. **Use error creation functions**: Always use helper functions from `common/common/status.h`
   ```cpp
   // Create specific error types
   return CodecProtocolError("Invalid protocol");
   return ConfigurationError("Missing required field");
   return NetworkError("Connection refused");
   ```

3. **Check before accessing**: Always verify `.ok()` before calling `.value()`
   ```cpp
   if (result.ok()) {
     auto value = result.value();
   }
   ```

### Error Categories

Common error creation functions:

- `CodecProtocolError()` - Protocol parsing/encoding errors
- `ConfigurationError()` - Configuration validation errors
- `NetworkError()` - Network/connection errors
- `FilterError()` - Filter processing errors
- `UpstreamError()` - Upstream communication errors

### Practical Example: Header Processing

```cpp
StatusOr<std::string> extractAuthToken(const Http::RequestHeaderMap& headers) {
  auto auth_header = headers.get(Http::CustomHeaders::get().Authorization);
  
  if (auth_header.empty()) {
    return FilterError("Missing Authorization header");
  }
  
  const auto& value = auth_header[0]->value().getStringView();
  if (!absl::StartsWith(value, "Bearer ")) {
    return FilterError("Invalid Authorization format");
  }
  
  return std::string(value.substr(7));  // Return token
}

// Usage in filter
Http::FilterHeadersStatus onRequestHeaders(Http::RequestHeaderMap& headers) {
  auto token_result = extractAuthToken(headers);
  
  if (!token_result.ok()) {
    ENVOY_LOG(warn, "Auth extraction failed: {}", 
              token_result.status().message());
    decoder_callbacks_->sendLocalReply(
      Http::Code::Unauthorized,
      "Authentication required",
      nullptr, absl::nullopt, "missing_auth_token");
    return Http::FilterHeadersStatus::StopIteration;
  }
  
  std::string token = token_result.value();
  // Process token...
  return Http::FilterHeadersStatus::Continue;
}
```

---

## Error Propagation Patterns

### Pattern 1: Early Return

```cpp
StatusOr<Config> loadConfig(const std::string& path) {
  // Validate input
  if (path.empty()) {
    return ConfigurationError("Config path cannot be empty");
  }
  
  // Try to read file
  auto file_result = readFile(path);
  if (!file_result.ok()) {
    return file_result.status();  // Propagate error
  }
  
  // Try to parse
  auto parse_result = parseYaml(file_result.value());
  if (!parse_result.ok()) {
    return parse_result.status();  // Propagate error
  }
  
  return parse_result.value();
}
```

### Pattern 2: Error Wrapping

```cpp
StatusOr<ProcessedData> processRequest(const Request& req) {
  auto validation_result = validateRequest(req);
  if (!validation_result.ok()) {
    return absl::Status(
      validation_result.status().code(),
      absl::StrCat("Request validation failed: ", 
                   validation_result.status().message())
    );
  }
  
  // Continue processing...
}
```

### Pattern 3: Callback Error Handling

```cpp
void asyncOperation(Callback callback) {
  connection_->connect([callback](StatusOr<Connection> result) {
    if (!result.ok()) {
      ENVOY_LOG(error, "Connection failed: {}", result.status().message());
      callback(result.status());
      return;
    }
    
    // Success path
    callback(processConnection(result.value()));
  });
}
```

---

## Response Flags

Response flags indicate why a request failed or was handled in a non-standard way. They are crucial for debugging and monitoring.

### Core Response Flags

Defined in `envoy/stream_info/stream_info.h`:

```cpp
enum CoreResponseFlag : uint16_t {
  FailedLocalHealthCheck,          // Local healthcheck failed
  NoHealthyUpstream,                // No healthy upstream hosts
  UpstreamRequestTimeout,           // Upstream request timed out
  LocalReset,                       // Local codec reset
  UpstreamRemoteReset,              // Remote codec reset
  UpstreamConnectionFailure,        // Initial connection failure
  UpstreamConnectionTermination,    // Connection terminated
  UpstreamOverflow,                 // Resource overflow
  NoRouteFound,                     // No route match
  DelayInjected,                    // Delay fault injection
  FaultInjected,                    // Abort fault injection
  RateLimited,                      // Rate limited
  UnauthorizedExternalService,      // External authz denied
  RateLimitServiceError,            // Rate limit service error
  DownstreamConnectionTermination,  // Downstream disconnected
  UpstreamRetryLimitExceeded,       // Retry limit exceeded
  StreamIdleTimeout,                // Stream idle timeout
  InvalidEnvoyRequestHeaders,       // Invalid x-envoy-* headers
  DownstreamProtocolError,          // Downstream protocol error
  UpstreamMaxStreamDurationReached, // Max duration exceeded
  ResponseFromCacheFilter,          // Served from cache
  NoFilterConfigFound,              // Filter config missing
  DurationTimeout,                  // Connection duration timeout
  UpstreamProtocolError,            // Upstream protocol error
  NoClusterFound,                   // Cluster not found
  OverloadManager,                  // Overload manager terminated
  DnsResolutionFailed,              // DNS resolution failed
  DropOverLoad,                     // Dropped due to overload
  DownstreamRemoteReset,            // Downstream remote reset
  UnconditionalDropOverload,        // Unconditional drop
};
```

### Checking Response Flags

```cpp
void analyzeResponse(const StreamInfo::StreamInfo& stream_info) {
  // Check specific flag
  if (stream_info.hasResponseFlag(
        StreamInfo::CoreResponseFlag::UpstreamRequestTimeout)) {
    ENVOY_LOG(warn, "Request timed out waiting for upstream");
  }
  
  // Check if any flags are set
  if (stream_info.hasAnyResponseFlag()) {
    ENVOY_LOG(info, "Request completed with flags");
    
    // Get all flags
    for (const auto& flag : stream_info.responseFlags()) {
      ENVOY_LOG(debug, "Flag: {}", flag.value());
    }
  }
}
```

### Setting Response Flags

```cpp
void handleUpstreamTimeout(StreamInfo::StreamInfo& stream_info) {
  stream_info.setResponseFlag(
    StreamInfo::CoreResponseFlag::UpstreamRequestTimeout);
  stream_info.setResponseCodeDetails("response_timeout");
  stream_info.setResponseCode(504);  // Gateway Timeout
}
```

### Common Flag Combinations

1. **Connection Failures**:
   ```
   UpstreamConnectionFailure + NoHealthyUpstream
   ```

2. **Timeout Scenarios**:
   ```
   UpstreamRequestTimeout + UpstreamRetryLimitExceeded
   ```

3. **Protocol Errors**:
   ```
   UpstreamProtocolError + UpstreamRemoteReset
   ```

---

## Connection Close Reasons

### Detected Close Types

```cpp
enum class DetectedCloseType {
  Normal,       // Normal close
  LocalReset,   // Local reset initiated by Envoy
  RemoteReset,  // Peer reset
};
```

### Local Close Reasons

Common reasons for local connection close (from `LocalCloseReasonValues`):

```cpp
// Connection management
DeferredCloseOnDrainedConnection
IdleTimeoutOnConnection
MaxConnectionDurationReached

// Protocol violations
Http2PingTimeout
Http2ConnectionProtocolViolation

// Timeout scenarios
TransportSocketTimeout
TcpSessionIdleTimeout
BufferHighWatermarkTimeout

// TCP proxy scenarios
ClosingUpstreamTcpDueToDownstreamRemoteClose
ClosingUpstreamTcpDueToDownstreamLocalClose
ClosingUpstreamTcpDueToDownstreamResetClose

// Health checks
NonPooledTcpConnectionHostHealthFailure
```

### Tracking Connection Close

```cpp
void logConnectionClose(const StreamInfo::StreamInfo& stream_info) {
  auto close_type = stream_info.downstreamDetectedCloseType();
  auto close_reason = stream_info.downstreamLocalCloseReason();
  
  ENVOY_LOG(info, "Connection closed - Type: {}, Reason: {}",
            static_cast<int>(close_type), close_reason);
  
  if (close_type == StreamInfo::DetectedCloseType::RemoteReset) {
    ENVOY_LOG(warn, "Client reset connection unexpectedly");
  }
}
```

---

## Response Code Details

Response code details provide human-readable explanations for why a response was generated.

### Core Response Code Details

From `ResponseCodeDetailValues`:

**Upstream Issues**:
```cpp
ViaUpstream                          // Normal upstream response
NoHealthyUpstream                    // No healthy hosts
ResponseTimeout                      // Upstream timeout
EarlyUpstreamReset                   // Reset before response
LateUpstreamReset                    // Reset after response started
```

**Request Validation**:
```cpp
MissingHost                          // Missing Host/:authority
MissingPath                          // Missing Path/:path
InvalidPath                          // Invalid path format
PathNormalizationFailed              // Path normalization error
InvalidEnvoyRequestHeaders           // Bad x-envoy-* headers
```

**Configuration/Routing**:
```cpp
RouteNotFound                        // No matching route
ClusterNotFound                      // Cluster doesn't exist
NoRouteFound                         // No route configuration
FilterChainNotFound                  // No matching filter chain
```

**Timeouts**:
```cpp
StreamIdleTimeout                    // Stream idle timeout
MaxDurationTimeout                   // Max duration exceeded
RequestOverallTimeout                // Overall request timeout
UpstreamPerTryTimeout                // Per-try timeout
```

**Connection Issues**:
```cpp
DownstreamRemoteDisconnect           // Client disconnected
DownstreamLocalDisconnect("{}")      // Local close (parameterized)
DurationTimeout                      // Connection duration exceeded
```

### Setting Response Code Details

```cpp
void sendTimeoutResponse(Http::StreamDecoderFilterCallbacks* callbacks,
                        StreamInfo::StreamInfo& stream_info) {
  // Set response flag
  stream_info.setResponseFlag(
    StreamInfo::CoreResponseFlag::UpstreamRequestTimeout);
  
  // Set detailed reason
  stream_info.setResponseCodeDetails(
    StreamInfo::ResponseCodeDetails::get().ResponseTimeout);
  
  // Send local reply
  callbacks->sendLocalReply(
    Http::Code::GatewayTimeout,
    "upstream request timeout",
    nullptr, absl::nullopt,
    StreamInfo::ResponseCodeDetails::get().ResponseTimeout);
}
```

---

## Common Error Scenarios

### 1. Configuration Validation Error

```cpp
StatusOr<FilterConfig> validateConfig(const Json::Object& config) {
  if (!config.hasObject("required_field")) {
    return ConfigurationError(
      "Missing required field 'required_field'");
  }
  
  auto timeout = config.getInteger("timeout", 30);
  if (timeout <= 0 || timeout > 3600) {
    return ConfigurationError(
      absl::StrCat("Invalid timeout: ", timeout, 
                   " (must be 1-3600)"));
  }
  
  return FilterConfig{timeout};
}
```

### 2. Upstream Connection Failure

```cpp
void handleConnectionFailure(StreamInfo::StreamInfo& stream_info,
                            const std::string& failure_reason) {
  // Set flags
  stream_info.setResponseFlag(
    StreamInfo::CoreResponseFlag::UpstreamConnectionFailure);
  
  // Set transport failure reason
  if (auto upstream_info = stream_info.upstreamInfo()) {
    upstream_info->setUpstreamTransportFailureReason(failure_reason);
  }
  
  // Set response details
  stream_info.setResponseCodeDetails(
    StreamInfo::ResponseCodeDetails::get().NoHealthyUpstream);
  
  ENVOY_LOG(warn, "Upstream connection failed: {}", failure_reason);
}
```

### 3. Protocol Error Handling

```cpp
void handleProtocolError(Http::StreamDecoderFilterCallbacks* callbacks,
                        StreamInfo::StreamInfo& stream_info,
                        const std::string& error_details) {
  // Set protocol error flag
  stream_info.setResponseFlag(
    StreamInfo::CoreResponseFlag::DownstreamProtocolError);
  
  // Set specific details
  stream_info.setResponseCodeDetails(error_details);
  
  // Log detailed error
  ENVOY_LOG(error, "Protocol error: {}", error_details);
  
  // Send 400 Bad Request
  callbacks->sendLocalReply(
    Http::Code::BadRequest,
    "protocol error",
    nullptr, absl::nullopt, error_details);
}
```

### 4. Rate Limiting

```cpp
void handleRateLimit(Http::StreamDecoderFilterCallbacks* callbacks,
                    StreamInfo::StreamInfo& stream_info) {
  // Set rate limit flag
  stream_info.setResponseFlag(
    StreamInfo::CoreResponseFlag::RateLimited);
  
  // Set response details
  stream_info.setResponseCodeDetails("request_rate_limited");
  
  // Send 429 Too Many Requests
  Http::HeaderMapPtr response_headers{
    Http::createHeaderMap<Http::ResponseHeaderMapImpl>({
      {Http::Headers::get().RetryAfter, "60"}
    })
  };
  
  callbacks->sendLocalReply(
    Http::Code::TooManyRequests,
    "rate limited",
    [](Http::ResponseHeaderMap& headers) {
      headers.addCopy(Http::Headers::get().RetryAfter, "60");
    },
    absl::nullopt,
    "request_rate_limited");
}
```

### 5. Timeout Handling

```cpp
class TimeoutHandler {
public:
  void onTimeout(Event::Dispatcher& dispatcher) {
    if (timeout_timer_) {
      timeout_timer_->disableTimer();
    }
    
    ENVOY_LOG(warn, "Request timeout after {}ms", timeout_ms_);
    
    // Set flags
    stream_info_.setResponseFlag(
      StreamInfo::CoreResponseFlag::UpstreamRequestTimeout);
    stream_info_.setResponseCodeDetails(
      StreamInfo::ResponseCodeDetails::get().ResponseTimeout);
    
    // Send timeout response
    callbacks_->sendLocalReply(
      Http::Code::GatewayTimeout,
      "request timeout",
      nullptr, absl::nullopt,
      StreamInfo::ResponseCodeDetails::get().ResponseTimeout);
  }
  
  void setupTimeout(uint32_t timeout_ms) {
    timeout_ms_ = timeout_ms;
    timeout_timer_ = dispatcher_.createTimer([this]() { onTimeout(); });
    timeout_timer_->enableTimer(std::chrono::milliseconds(timeout_ms));
  }
  
private:
  Event::Dispatcher& dispatcher_;
  StreamInfo::StreamInfo& stream_info_;
  Http::StreamDecoderFilterCallbacks* callbacks_;
  Event::TimerPtr timeout_timer_;
  uint32_t timeout_ms_;
};
```

---

## Best Practices

1. **Always use StatusOr for fallible operations**:
   ```cpp
   // Good
   StatusOr<Config> loadConfig();
   
   // Bad (exception-based)
   Config loadConfig();  // throws exception
   ```

2. **Provide detailed error messages**:
   ```cpp
   // Good
   return ConfigurationError(
     absl::StrCat("Invalid port ", port, ": must be 1-65535"));
   
   // Bad
   return ConfigurationError("Invalid port");
   ```

3. **Set appropriate response flags and details**:
   ```cpp
   // Set both flag and details
   stream_info.setResponseFlag(flag);
   stream_info.setResponseCodeDetails(details);
   ```

4. **Log errors with context**:
   ```cpp
   ENVOY_LOG(error, "Connection failed to {}:{} - {}",
             host, port, error_message);
   ```

5. **Use structured error handling in filters**:
   ```cpp
   auto result = processRequest(request);
   if (!result.ok()) {
     stream_info.setResponseFlag(appropriate_flag);
     stream_info.setResponseCodeDetails(result.status().message());
     return Http::FilterHeadersStatus::StopIteration;
   }
   ```

---

## Debugging Tips

1. **Check response flags in access logs**:
   ```yaml
   access_log:
   - format: "%RESPONSE_FLAGS% %RESPONSE_CODE_DETAILS%"
   ```

2. **Use admin interface to check stats**:
   ```bash
   curl localhost:9901/stats | grep -i error
   curl localhost:9901/stats | grep -i timeout
   ```

3. **Enable debug logging for specific components**:
   ```bash
   curl -X POST localhost:9901/logging?filter=debug
   curl -X POST localhost:9901/logging?connection=trace
   ```

4. **Inspect stream info in debugger**:
   ```
   (gdb) p stream_info.responseFlags()
   (gdb) p stream_info.responseCodeDetails()
   (gdb) p stream_info.upstreamTransportFailureReason()
   ```

---

## Related Documentation

- [Logging and Tracing](02-logging-and-tracing.md)
- [Debugging Tools](03-debugging-tools.md)
- [Common Issues](04-common-issues.md)
- [Envoy Admin Interface](../admin-operations/)

## References

- Source: `source/common/common/statusor.h`
- Source: `source/common/common/status.h`
- Source: `envoy/stream_info/stream_info.h`
- Docs: Official Envoy documentation on error handling

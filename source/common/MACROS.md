# Well-Known Macros in Envoy Common

**Purpose**: Reference guide for frequently used macros across `source/common/` components

---

## 1. Assertion and Error Handling

### 1.1 ASSERT Family (`common/assert.h`)

**Basic Assertions:**
- `ASSERT(condition)` - Debug-only assertion, aborts in debug mode
- `ASSERT(condition, "details")` - Verbose variant with detailed message
- `SLOW_ASSERT(condition)` - May be compiled out for performance
- `RELEASE_ASSERT(condition, "details")` - Always compiled, aborts on failure
- `SECURITY_ASSERT(condition, "details")` - Memory bounds-checking assertions
- `MOBILE_RELEASE_ASSERT(condition, "details")` - Enforced on iOS/Android only
- `KNOWN_ISSUE_ASSERT(condition, "details")` - For triaged issues, can be disabled

**When to use:**
- `ASSERT` - Development invariants, compiled out in release builds
- `RELEASE_ASSERT` - Critical invariants that must hold in production
- `SECURITY_ASSERT` - Memory safety boundaries
- `SLOW_ASSERT` - Expensive checks that impact performance

### 1.2 ENVOY_BUG (`common/assert.h`)

**Usage:**
- `ENVOY_BUG(condition, "details")` - Non-fatal condition check with exponential backoff logging
- `IS_ENVOY_BUG("message")` - Always triggers ENVOY_BUG for unexpected code paths

**Behavior:**
- Debug mode: crashes if condition fails
- Release mode: logs with power-of-two backoff, increments stats
- Coverage mode: logs without crashing (prevents flakiness)

**Example:**
```cpp
if (!validateConfig(config)) {
  ENVOY_BUG(false, "Invalid config should have been caught earlier");
  return defaultConfig();
}
```

### 1.3 PANIC Macros (`common/assert.h`)

**Usage:**
- `PANIC("message")` - Immediate abort with message
- `PANIC_ON_PROTO_ENUM_SENTINEL_VALUES` - Handles proto sentinel values in switch
- `PANIC_DUE_TO_PROTO_UNSET` - For unset proto oneof fields
- `PANIC_DUE_TO_CORRUPT_ENUM` - For corrupted enum values

**Example:**
```cpp
switch (proto.address_case()) {
case kSocketAddress: return handleSocket();
case kPipe: return handlePipe();
PANIC_ON_PROTO_ENUM_SENTINEL_VALUES;
}
```

---

## 2. Status Error Handling

### 2.1 Status Return Macros (`envoy/common/exception.h`)

**Early return on error:**
- `RETURN_IF_NOT_OK(status_fn)` - Returns status if not OK
- `RETURN_IF_NOT_OK_REF(status_var)` - Returns reference status if not OK
- `RETURN_ONLY_IF_NOT_OK_REF(status_var)` - Returns void if not OK
- `SET_AND_RETURN_IF_NOT_OK(check_status, set_status)` - Sets status and returns

**Example:**
```cpp
absl::Status validateAndApply(const Config& config) {
  RETURN_IF_NOT_OK(validateAddresses(config.addresses()));
  RETURN_IF_NOT_OK(applySocketOptions(config.socket_options()));
  return absl::OkStatus();
}
```

### 2.2 HTTP Status Macros (`common/http/status.h`)

**HTTP-specific error handling:**
- `RETURN_IF_ERROR(expr)` - Returns HTTP Status if expression fails

**Usage:**
```cpp
Http::Status parseHeaders() {
  RETURN_IF_ERROR(validateContentLength());
  RETURN_IF_ERROR(checkHeaderSize());
  return Http::okStatus();
}
```

### 2.3 Exception Macros (`envoy/common/exception.h`)

**Throw on error:**
- `THROW_IF_NOT_OK(status_fn)` - Throws EnvoyException if status not OK
- `THROW_IF_NOT_OK_REF(status)` - Throws on status reference
- `THROW_OR_RETURN_VALUE(expr, type)` - Returns value or throws

**Exception toggle:**
- `throwEnvoyExceptionOrPanic(msg)` - Throws EnvoyException or panics (if exceptions disabled)
- `throwExceptionOrPanic(type, msg)` - Throws typed exception or panics

---

## 3. Protobuf Field Accessors

### 3.1 Wrapped Field Macros (`common/protobuf/utility.h`)

**Nullable wrapper fields (UInt32Value, BoolValue, etc.):**
- `PROTOBUF_GET_WRAPPED_OR_DEFAULT(message, field_name, default)` - Returns value or default
- `PROTOBUF_GET_OPTIONAL_WRAPPED(message, field_name)` - Returns `absl::optional`
- `PROTOBUF_GET_WRAPPED_REQUIRED(message, field_name)` - Returns value or throws

**Example:**
```cpp
// Proto: google.protobuf.UInt32Value max_connections = 1;
uint32_t max_conns = PROTOBUF_GET_WRAPPED_OR_DEFAULT(config, max_connections, 1024);
```

### 3.2 Duration Field Macros (`common/protobuf/utility.h`)

**google.protobuf.Duration fields:**
- `PROTOBUF_GET_MS_OR_DEFAULT(message, field_name, default)` - Milliseconds or default
- `PROTOBUF_GET_OPTIONAL_MS(message, field_name)` - Returns `absl::optional<ms>`
- `PROTOBUF_GET_MS_REQUIRED(message, field_name)` - Milliseconds or throws
- `PROTOBUF_GET_SECONDS_OR_DEFAULT(message, field_name, default)` - Seconds or default
- `PROTOBUF_GET_SECONDS_REQUIRED(message, field_name)` - Seconds or throws

**Example:**
```cpp
// Proto: google.protobuf.Duration idle_timeout = 2;
auto timeout = PROTOBUF_GET_MS_OR_DEFAULT(config, idle_timeout, 300000);
```

### 3.3 String Field Macros (`common/protobuf/utility.h`)

**String fields with defaults:**
- `PROTOBUF_GET_STRING_OR_DEFAULT(message, field_name, default)` - Returns string or default

---

## 4. Logging Macros

### 4.1 Basic Logging (`common/logger.h`)

**Standard logging:**
- `ENVOY_LOG(level, format, ...)` - Log to current logger context
- `ENVOY_LOG_TO_LOGGER(logger, level, format, ...)` - Log to specific logger
- `ENVOY_LOG_MISC(level, format, ...)` - Log to misc logger

**Levels**: `trace`, `debug`, `info`, `warn`, `error`, `critical`

**Example:**
```cpp
ENVOY_LOG(debug, "Processing connection from {}", remote_address);
```

### 4.2 Connection/Stream Logging (`common/logger.h`)

**Context-aware logging:**
- `ENVOY_CONN_LOG(level, format, connection, ...)` - Logs with connection ID
- `ENVOY_STREAM_LOG(level, format, stream, ...)` - Logs with stream ID
- `ENVOY_TAGGED_LOG(level, tags, format, ...)` - Logs with custom tags

**Example:**
```cpp
ENVOY_CONN_LOG(debug, "received {} bytes", connection, bytes_received);
```

### 4.3 Rate-Limited Logging (`common/logger.h`)

**Throttled logging:**
- `ENVOY_LOG_ONCE(level, format, ...)` - Logs only first occurrence
- `ENVOY_LOG_ONCE_IF(level, condition, format, ...)` - Conditional once
- `ENVOY_LOG_FIRST_N(level, n, format, ...)` - Logs first N occurrences
- `ENVOY_LOG_EVERY_NTH(level, n, format, ...)` - Logs every Nth call
- `ENVOY_LOG_EVERY_POW_2(level, format, ...)` - Logs at power-of-2 intervals
- `ENVOY_LOG_PERIODIC(level, duration, format, ...)` - Logs at time intervals

**Example:**
```cpp
ENVOY_LOG_EVERY_POW_2(warn, "High connection rate: {}", conn_count);
// Logs at calls: 1, 2, 4, 8, 16, 32, 64...
```

### 4.4 Log Level Checks (`common/logger.h`)

**Conditional execution:**
- `ENVOY_LOG_CHECK_LEVEL(level)` - Returns true if level is enabled
- `ENVOY_LOG_COMP_LEVEL(logger, level)` - Compares logger level

---

## 5. Utility Macros

### 5.1 Array and String Macros (`common/macros.h`)

**Size calculations:**
- `ARRAY_SIZE(X)` - Number of elements in C array
- `STATIC_STRLEN(X)` - Length of string literal (excluding null terminator)

**Example:**
```cpp
static const char* PROTOCOLS[] = {"http1", "http2", "http3"};
for (size_t i = 0; i < ARRAY_SIZE(PROTOCOLS); ++i) { ... }
```

### 5.2 Code Generation Helpers (`common/macros.h`)

**Enum to string:**
- `GENERATE_ENUM(X)` - Generates enum value
- `GENERATE_STRING(X)` - Generates string literal

**Example:**
```cpp
#define PROTOCOL_LIST(X) X(HTTP1) X(HTTP2) X(HTTP3)
enum Protocol { PROTOCOL_LIST(GENERATE_ENUM) };
const char* NAMES[] = { PROTOCOL_LIST(GENERATE_STRING) };
```

### 5.3 Singleton Pattern (`common/macros.h`)

**Static initialization:**
- `CONSTRUCT_ON_FIRST_USE(type, ...)` - Creates const singleton on first call
- `MUTABLE_CONSTRUCT_ON_FIRST_USE(type, ...)` - Creates mutable singleton

**Example:**
```cpp
const Registry& getRegistry() {
  CONSTRUCT_ON_FIRST_USE(Registry, "default_name");
}
```

### 5.4 Control Flow (`common/macros.h`)

**Switch fallthrough:**
- `FALLTHRU` - C++17 `[[fallthrough]]` or compiler-specific equivalent

**Unreferenced parameters:**
- `UNREFERENCED_PARAMETER(X)` - Suppresses unused parameter warnings (non-Windows)

---

## 6. Socket Option Macros

### 6.1 Socket Option Names (`envoy/network/socket.h`)

**Platform-conditional socket options:**
- `ENVOY_SOCKET_SO_KEEPALIVE` - TCP keepalive enable
- `ENVOY_SOCKET_SO_REUSEPORT` - Port reuse
- `ENVOY_SOCKET_SO_MARK` - Socket marking (Linux)
- `ENVOY_SOCKET_SO_NOSIGPIPE` - No SIGPIPE (BSD/macOS)
- `ENVOY_SOCKET_IP_TRANSPARENT` - Transparent proxy (IPv4)
- `ENVOY_SOCKET_IPV6_TRANSPARENT` - Transparent proxy (IPv6)
- `ENVOY_SOCKET_IP_FREEBIND` - Bind to non-local addresses (IPv4)
- `ENVOY_SOCKET_IPV6_FREEBIND` - Bind to non-local addresses (IPv6)
- `ENVOY_SOCKET_IP_RECVTOS` - Receive TOS field (IPv4)
- `ENVOY_SOCKET_IPV6_RECVTCLASS` - Receive traffic class (IPv6)

**Usage:**
```cpp
options->push_back(std::make_shared<SocketOptionImpl>(
    STATE_PREBIND, ENVOY_SOCKET_SO_REUSEPORT, 1));
```

**Platform handling:**
- Available platforms: expands to `ENVOY_MAKE_SOCKET_OPTION_NAME(level, name)`
- Unavailable platforms: expands to empty `SocketOptionName()`

---

## 7. Testing Macros

### 7.1 Test Access (`common/macros.h`)

**Friend declarations for tests:**
- `GTEST_FRIEND_CLASS(test_case_name, test_name)` - Grants friend access to specific test

**Example:**
```cpp
class ConnectionImpl {
  GTEST_FRIEND_CLASS(ConnectionImplTest, ValidatesHeaders);
private:
  void validateHeaders();
};
```

---

## 8. Platform Detection

### 8.1 Compiler Detection (`common/macros.h`)

**Compiler-specific code:**
- `GCC_COMPILER` - Defined for GCC (not Clang)

### 8.2 OS-Specific Defaults (`api/*/os_sys_calls_impl.h`)

**Default pipe types:**
- `ENVOY_DEFAULT_PIPE_TYPE` - `AF_UNIX` on POSIX, `AF_INET` on Windows

---

## 9. Usage Patterns and Best Practices

### 9.1 Status Handling Flow

**Typical proto validation:**
```cpp
absl::Status ListenerImpl::initialize(const Listener& config) {
  // Resolve address
  RETURN_IF_NOT_OK(Utility::resolveProtoAddress(config.address()));

  // Parse socket options
  auto options = SocketOptionFactory::buildLiteralOptions(config.socket_options());

  // Apply TCP keepalive if present
  if (config.has_tcp_keepalive()) {
    RETURN_IF_NOT_OK(parseTcpKeepaliveConfig(config.tcp_keepalive()));
  }

  return absl::OkStatus();
}
```

### 9.2 Defensive Programming

**Invariant checking:**
```cpp
void processRequest(RequestPtr request) {
  ASSERT(request != nullptr, "Request must be non-null");
  ASSERT(request->headers().Method() != nullptr);

  if (!validateRequest(*request)) {
    ENVOY_BUG(false, "Invalid request passed validation earlier");
    return rejectRequest();
  }
}
```

### 9.3 Rate-Limited Logging

**High-frequency code paths:**
```cpp
void ConnectionImpl::onDataReceived(Buffer::Instance& data) {
  bytes_received_ += data.length();

  // Log only on power-of-2 intervals to avoid log spam
  ENVOY_LOG_EVERY_POW_2(trace, "Total bytes received: {}", bytes_received_);
}
```

---

## 10. Macro Categories Summary

**By purpose:**
- **Assertions**: `ASSERT`, `RELEASE_ASSERT`, `SECURITY_ASSERT`, `ENVOY_BUG`, `PANIC`
- **Status flow**: `RETURN_IF_NOT_OK`, `RETURN_IF_ERROR`, `THROW_IF_NOT_OK`
- **Proto access**: `PROTOBUF_GET_WRAPPED_*`, `PROTOBUF_GET_MS_*`, `PROTOBUF_GET_SECONDS_*`
- **Logging**: `ENVOY_LOG*`, `ENVOY_CONN_LOG*`, `ENVOY_STREAM_LOG*`
- **Utilities**: `ARRAY_SIZE`, `CONSTRUCT_ON_FIRST_USE`, `FALLTHRU`
- **Sockets**: `ENVOY_SOCKET_*` family
- **Testing**: `GTEST_FRIEND_CLASS`

**By source file:**
- `common/assert.h` - Assertion and panic macros
- `common/exception.h` - Status return macros
- `common/protobuf/utility.h` - Proto field accessors
- `common/logger.h` - Logging macros
- `common/macros.h` - Utility macros
- `common/http/status.h` - HTTP status macros
- `envoy/network/socket.h` - Socket option macros

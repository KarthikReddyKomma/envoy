# Specialized Access Loggers

This document covers the remaining specialized access logger implementations in Envoy.

## Table of Contents
1. [Stream Access Logger](#stream-access-logger)
2. [Fluentd Access Logger](#fluentd-access-logger)
3. [WASM Access Logger](#wasm-access-logger)
4. [Stats Access Logger](#stats-access-logger)
5. [Dynamic Modules Access Logger](#dynamic-modules-access-logger)

---

## Stream Access Logger

**Location**: `source/extensions/access_loggers/stream/`
**Name**: `envoy.access_loggers.stream` (stdout/stderr)

### Overview

Writes access logs to stdout or stderr. This is the preferred approach for containerized environments following 12-factor app principles.

### When to Use

- **Kubernetes/Docker deployments** (container runtime handles forwarding)
- **Cloud-native applications** (CloudWatch, Stackdriver, etc.)
- **Development/debugging** (immediate console output)
- **Simple deployments** (no file management needed)

### Configuration

```yaml
access_log:
  # stdout
  - name: envoy.access_loggers.stream
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
      log_format:
        json_format:
          timestamp: "%START_TIME%"
          method: "%REQ(:METHOD)%"
          path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
          status: "%RESPONSE_CODE%"
          duration_ms: "%DURATION%"
  
  # stderr (for errors)
  - name: envoy.access_loggers.stream
    filter:
      status_code_filter:
        comparison:
          op: GE
          value: 400
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StderrAccessLog
      log_format:
        json_format:
          level: "error"
          status: "%RESPONSE_CODE%"
          path: "%REQ(:PATH)%"
```

### Architecture

```
Envoy → Format String → stdout/stderr → OS Buffer → Container Runtime → Log Aggregator
```

### Advantages

- **No File Management**: No rotation, no disk space issues
- **Cloud-Native**: Standard pattern for containers
- **Zero Configuration**: Works out of the box
- **Aggregator-Friendly**: Automatic forwarding by container runtime

### Performance

- Latency: ~10μs (same as file logger)
- Throughput: 100,000+ req/s
- CPU: <0.5%
- Memory: Minimal (OS manages buffering)

### Best Practices

1. **Always Use JSON Format** (easier for log aggregators):
```yaml
log_format:
  json_format:
    timestamp: "%START_TIME%"
    level: "info"
    # ...
```

2. **Separate stdout and stderr**:
```yaml
# Normal logs → stdout
- name: envoy.access_loggers.stream
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog

# Error logs → stderr
- name: envoy.access_loggers.stream
  filter:
    status_code_filter:
      comparison:
        op: GE
        value: 400
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StderrAccessLog
```

3. **Configure Container Logging Driver**:
```yaml
# Docker
logging:
  driver: "json-file"
  options:
    max-size: "10m"
    max-file: "3"

# Kubernetes (use FluentD/Fluent Bit)
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: envoy
      image: envoyproxy/envoy:v1.28.0
      # Logs automatically captured by kubelet
```

---

## Fluentd Access Logger

**Location**: `source/extensions/access_loggers/fluentd/`
**Name**: `envoy.access_loggers.fluentd`

### Overview

Sends access logs to Fluentd using the Forward protocol (MessagePack over TCP). Use when you have existing Fluentd infrastructure.

### When to Use

- **Existing Fluentd Deployment** (reuse infrastructure)
- **Tag-Based Routing** (Fluentd tags for routing)
- **Fluentd Plugin Ecosystem** (output to 200+ destinations)

### Architecture

```
Envoy → Format → MessagePack → TCP (Forward Protocol) → Fluentd → Plugins → Destinations
```

### Configuration

```yaml
access_log:
  - name: envoy.access_loggers.fluentd
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.fluentd.v3.FluentdAccessLogConfig
      cluster: fluentd_cluster
      tag: envoy.access
      stat_prefix: fluentd_access_log
      buffer_flush_interval: 1s
      buffer_size_bytes: 16384
      record:
        timestamp: "%START_TIME%"
        method: "%REQ(:METHOD)%"
        path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
        status: "%RESPONSE_CODE%"
        duration_ms: "%DURATION%"
        upstream_cluster: "%UPSTREAM_CLUSTER%"

# Fluentd cluster
clusters:
  - name: fluentd_cluster
    type: STRICT_DNS
    connect_timeout: 1s
    load_assignment:
      cluster_name: fluentd_cluster
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: fluentd.logging.svc.cluster.local
                    port_value: 24224  # Fluentd forward port
```

### Key Features

- **Tag-Based Routing**: Route logs via tags (e.g., `envoy.access.prod`)
- **MessagePack Encoding**: Efficient binary format
- **Batching**: Configurable buffering
- **Substitution Formatter**: Rich variable substitution like file logger

### Fluentd Configuration

```ruby
# Receive from Envoy
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

# Route by tag
<match envoy.access.**>
  @type elasticsearch
  host elasticsearch.logging.svc.cluster.local
  port 9200
  index_name envoy-access-logs
  type_name _doc
</match>
```

### Performance

- Latency: ~50-100μs (network + serialization)
- Throughput: 50,000+ req/s
- CPU: 1-3%
- Network: ~1-2 MB/s at 10k req/s

---

## WASM Access Logger

**Location**: `source/extensions/access_loggers/wasm/`
**Name**: `envoy.access_loggers.wasm`

### Overview

Execute custom logging logic in WebAssembly modules. Provides maximum flexibility for custom log processing, enrichment, or filtering.

### When to Use

- **Custom Log Enrichment** (add business logic)
- **Proprietary Formats** (custom serialization)
- **Complex Filtering** (beyond CEL capabilities)
- **Multi-Language Support** (Rust, C++, Go, etc.)

### Architecture

```
Envoy → WASM VM → Custom WASM Code → Custom Destination
                 ↓
            (Sandboxed, Safe)
```

### Configuration

```yaml
access_log:
  - name: envoy.access_loggers.wasm
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.wasm.v3.WasmAccessLog
      config:
        vm_config:
          runtime: "envoy.wasm.runtime.v8"
          code:
            local:
              filename: "/etc/envoy/my_access_logger.wasm"
        configuration:
          "@type": type.googleapis.com/google.protobuf.StringValue
          value: |
            {
              "destination": "http://logs.example.com/submit",
              "api_key": "secret"
            }
```

### WASM ABI

Implement these functions in your WASM module:

```rust
// Rust example
use proxy_wasm::traits::*;
use proxy_wasm::types::*;

#[no_mangle]
pub fn _start() {
    proxy_wasm::set_log_level(LogLevel::Trace);
    proxy_wasm::set_root_context(|_| -> Box<dyn RootContext> {
        Box::new(MyAccessLogger::new())
    });
}

impl Context for MyAccessLogger {
    fn on_log(&mut self, _id: u32) {
        // Get request data
        let method = self.get_http_request_header(":method");
        let path = self.get_http_request_header(":path");
        let status = self.get_http_response_header(":status");
        
        // Custom processing
        let log_entry = format!("{} {} {}", method, path, status);
        
        // Send to custom destination
        self.send_to_custom_backend(log_entry);
    }
}
```

### Use Cases

1. **Custom Enrichment**:
```rust
// Add custom metadata from external source
fn on_log(&mut self, _id: u32) {
    let user_id = self.get_http_request_header("x-user-id");
    let user_metadata = self.fetch_user_metadata(user_id);
    
    let enriched_log = format!("{} {}", log_entry, user_metadata);
    self.write_log(enriched_log);
}
```

2. **Proprietary Format**:
```rust
// Convert to custom binary format
fn on_log(&mut self, _id: u32) {
    let log_data = self.collect_log_data();
    let proprietary_format = self.encode_proprietary(log_data);
    self.send_to_backend(proprietary_format);
}
```

3. **Complex Aggregation**:
```rust
// Aggregate metrics before logging
fn on_log(&mut self, _id: u32) {
    self.update_aggregates(log_data);
    
    if self.should_flush() {
        let summary = self.compute_summary();
        self.send_summary(summary);
    }
}
```

### Performance

- Overhead: 10-100μs (depends on WASM logic)
- Sandboxed: Safe execution, can't crash Envoy
- Language-agnostic: Write in Rust, C++, Go, etc.

### Security Considerations

- WASM is sandboxed (can't access filesystem, network directly)
- Use Envoy's APIs for external communication
- Validate all inputs in WASM code

---

## Stats Access Logger

**Location**: `source/extensions/access_loggers/stats/`
**Name**: `envoy.access_loggers.stats`

### Overview

Generates custom statistics from access log data. Convert log events into metrics for monitoring and alerting.

### When to Use

- **Custom Metrics** (business-specific monitoring)
- **Aggregation** (count events matching criteria)
- **Alerting** (metric-based alerts)

### Configuration

```yaml
access_log:
  - name: envoy.access_loggers.stats
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.stats.v3.StatsAccessLogConfig
      stat_prefix: custom_access_log
      # Stats are incremented on each log
```

### Example Use Case

Count API endpoint usage:

```yaml
# Use with CEL filter to create per-endpoint counters
access_log:
  # Count /api/users calls
  - name: envoy.access_loggers.stats
    filter:
      extension_filter:
        name: envoy.access_loggers.extension_filters.cel
        typed_config:
          expression: "request.path == '/api/users'"
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.stats.v3.StatsAccessLogConfig
      stat_prefix: api_users
  
  # Count /api/products calls
  - name: envoy.access_loggers.stats
    filter:
      extension_filter:
        name: envoy.access_loggers.extension_filters.cel
        typed_config:
          expression: "request.path == '/api/products'"
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.stats.v3.StatsAccessLogConfig
      stat_prefix: api_products
```

Resulting stats:
```
custom_access_log.api_users.log_count
custom_access_log.api_products.log_count
```

### Performance

- Overhead: ~1-2μs (counter increment)
- Very efficient for generating metrics

---

## Dynamic Modules Access Logger

**Location**: `source/extensions/access_loggers/dynamic_modules/`
**Name**: `envoy.access_loggers.dynamic_modules`

### Overview

Load native shared libraries (.so/.dylib) as access loggers. For maximum performance when WASM overhead is unacceptable or when integrating with existing native libraries.

### When to Use

- **Maximum Performance** (native code, no VM)
- **Legacy Integration** (existing C/C++ libraries)
- **Proprietary Protocols** (closed-source implementations)

### Architecture

```
Envoy → ABI Layer → Native Shared Library (.so) → Custom Logic
```

### Configuration

```yaml
access_log:
  - name: envoy.access_loggers.dynamic_modules
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.access_loggers.dynamic_modules.v3.DynamicModulesAccessLogConfig
      module_path: "/usr/local/lib/my_access_logger.so"
      config:
        # Module-specific configuration
```

### ABI Implementation

Implement these functions in your shared library:

```cpp
// C ABI
extern "C" {
    // Initialize logger
    void* envoy_dynamic_module_on_access_log_init(const char* config) {
        MyLogger* logger = new MyLogger(config);
        return static_cast<void*>(logger);
    }
    
    // Log entry
    void envoy_dynamic_module_on_access_log(
        void* context,
        const envoy_dynamic_module_log_context_t* log_context) {
        MyLogger* logger = static_cast<MyLogger*>(context);
        logger->log(log_context);
    }
    
    // Cleanup
    void envoy_dynamic_module_on_access_log_destroy(void* context) {
        MyLogger* logger = static_cast<MyLogger*>(context);
        delete logger;
    }
}
```

### Security Considerations

⚠️ **WARNING**: Native modules can crash Envoy and access any system resource.

- **Thoroughly test** modules before deploying
- **Code review** all module code
- **Use WASM instead** when possible (safer)
- **Isolate Envoy** if using untrusted modules

### Performance

- Overhead: ~5-20μs (depends on module)
- Native performance (no VM overhead)
- Direct memory access

---

## Comparison Summary

| Logger | Complexity | Performance | Use Case |
|--------|-----------|-------------|----------|
| **Stream** | Low | Excellent | Containers, cloud-native |
| **Fluentd** | Medium | Good | Existing Fluentd infrastructure |
| **WASM** | High | Good | Custom logic, safe extensibility |
| **Stats** | Low | Excellent | Metrics generation |
| **Dynamic** | High | Excellent | Native integrations, max performance |

## Decision Tree

```
Need custom logging?
├─ Yes
│  ├─ Need max performance?
│  │  ├─ Yes → Dynamic Modules (or WASM)
│  │  └─ No → WASM
│  └─ Just need metrics? → Stats Logger
└─ No
   ├─ Have Fluentd? → Fluentd Logger
   ├─ In containers? → Stream Logger (stdout)
   ├─ Need centralized? → gRPC or OpenTelemetry Logger
   └─ Simple local? → File Logger
```

## Related Documentation

- [ACCESS_LOGGERS_OVERVIEW.md](ACCESS_LOGGERS_OVERVIEW.md)
- [FILE_ACCESS_LOGGER.md](file/FILE_ACCESS_LOGGER.md)
- [GRPC_ACCESS_LOGGER.md](grpc/GRPC_ACCESS_LOGGER.md)
- [OPENTELEMETRY_ACCESS_LOGGER.md](open_telemetry/OPENTELEMETRY_ACCESS_LOGGER.md)
- [ACCESS_LOGGER_COMMON.md](ACCESS_LOGGER_COMMON.md)
- [ACCESS_LOG_FILTERS.md](filters/ACCESS_LOG_FILTERS.md)

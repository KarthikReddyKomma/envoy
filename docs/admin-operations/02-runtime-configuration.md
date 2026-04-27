# Runtime Configuration

## Overview

Envoy's runtime configuration system allows dynamic modification of behavior without restarting the process. Runtime flags control features like circuit breaker thresholds, retry policies, and feature flags.

## Architecture

### Runtime Layers

Runtime configuration uses a **layered approach** where values are resolved through multiple layers in order:

```
┌─────────────────────────────────┐
│    Admin Layer (highest)        │  /runtime_modify
├─────────────────────────────────┤
│    Disk Layer                   │  File-based runtime
├─────────────────────────────────┤
│    RTDS Layer                   │  Runtime Discovery Service
├─────────────────────────────────┤
│    Static Layer (lowest)        │  Bootstrap config
└─────────────────────────────────┘
```

**Layer Priority** (highest to lowest):
1. **Admin Layer** - Values set via `/runtime_modify` endpoint
2. **Disk Layer** - Values loaded from filesystem
3. **RTDS Layer** - Values from Runtime Discovery Service (xDS)
4. **Static Layer** - Values from bootstrap configuration

### Key Components

**Files**: `source/common/runtime/runtime_impl.{h,cc}`

**Classes**:
- `RuntimeImpl` - Main runtime implementation
- `Snapshot` - Immutable runtime value snapshot
- `SnapshotImpl` - Concrete snapshot implementation
- `Loader` - Runtime configuration loader

### Runtime Value Types

1. **Integers** - Numeric values (thresholds, limits)
2. **Booleans** - Feature flags (true/false)
3. **Fractional Percentages** - 0-100 for sampling/rollout (e.g., 25.5%)
4. **Strings** - Text values

## Configuration

### Static Layer (Bootstrap)

```yaml
layered_runtime:
  layers:
    - name: static_layer
      static_layer:
        # Circuit breaker thresholds
        upstream.healthy_panic_threshold: 50
        
        # Health check settings
        health_check.min_interval: 1000
        health_check.max_interval: 30000
        
        # Connection pool limits
        upstream.max_connections: 1024
        upstream.max_pending_requests: 1024
        
        # Feature flags
        envoy.reloadable_features.some_feature: true
```

### Disk Layer

```yaml
layered_runtime:
  layers:
    - name: disk_layer
      disk_layer:
        symlink_root: /srv/runtime
        subdirectory: envoy
        append_service_cluster: true
```

**Directory Structure**:
```
/srv/runtime/envoy/
├── upstream.healthy_panic_threshold
├── health_check.min_interval
├── envoy.reloadable_features.new_feature
└── subdirectory/
    └── nested.value
```

**File Contents**:
```bash
$ cat /srv/runtime/envoy/upstream.healthy_panic_threshold
50
```

**Reload Behavior**:
- Envoy watches symlink root
- Detects symlink changes
- Reloads runtime values automatically
- Uses inotify on Linux

### RTDS Layer (Runtime Discovery Service)

```yaml
layered_runtime:
  layers:
    - name: rtds_layer
      rtds_layer:
        name: runtime
        rtds_config:
          resource_api_version: V3
          api_config_source:
            api_type: GRPC
            transport_api_version: V3
            grpc_services:
              - envoy_grpc:
                  cluster_name: rtds_cluster
```

**Use Case**: Centralized runtime management (like Istio control plane)

### Admin Layer

Values set via admin interface (highest priority):

```bash
curl -X POST "http://localhost:9901/runtime_modify?upstream.healthy_panic_threshold=30"
```

**Characteristics**:
- In-memory only (lost on restart)
- Highest priority (overrides all layers)
- Useful for emergency adjustments

## Common Runtime Keys

### Circuit Breaking

```yaml
# Panic threshold (%)
upstream.healthy_panic_threshold: 50

# Connection limits per upstream cluster
circuit_breakers.default.max_connections: 1024
circuit_breakers.default.max_pending_requests: 1024
circuit_breakers.default.max_requests: 1024
circuit_breakers.default.max_retries: 3
```

### Health Checking

```yaml
# Health check intervals (milliseconds)
health_check.min_interval: 1000
health_check.max_interval: 30000

# Unhealthy threshold (failures before marking unhealthy)
health_check.unhealthy_threshold: 3

# Healthy threshold (successes before marking healthy)
health_check.healthy_threshold: 2
```

### Outlier Detection

```yaml
# Outlier detection thresholds
outlier_detection.consecutive_5xx: 5
outlier_detection.consecutive_gateway_failure: 5
outlier_detection.interval_ms: 10000
outlier_detection.base_ejection_time_ms: 30000
outlier_detection.max_ejection_percent: 10
outlier_detection.enforcing_consecutive_5xx: 100
outlier_detection.enforcing_success_rate: 100
```

### Retry Policy

```yaml
# Retry settings
retry.max_retries: 3
retry.base_interval_ms: 25
retry.max_interval_ms: 250
```

### Connection Pool

```yaml
# HTTP connection pool
http1.max_requests_per_connection: 0  # 0 = unlimited
http2.max_concurrent_streams: 2147483647
```

### Load Balancing

```yaml
# Least request load balancer
upstream.least_request.choice_count: 2

# Zone-aware routing
upstream.zone_routing.enabled: true
upstream.zone_routing.min_cluster_size: 6
```

### Feature Flags

Envoy uses runtime flags to enable/disable features:

```yaml
# Reloadable features (can change at runtime)
envoy.reloadable_features.http2_use_oghttp2: false
envoy.reloadable_features.enable_universal_header_validator: true
envoy.reloadable_features.validate_upstream_headers: true

# Deprecated features (temporary backwards compatibility)
envoy.deprecated_features.some_old_feature: true
```

**File**: `source/common/runtime/runtime_features.h` - Exhaustive list of feature flags

## Using Runtime Values in Code

### Reading Runtime Values

```cpp
// Get runtime loader
Runtime::Loader& runtime = context.runtime();

// Get snapshot (immutable view)
const Runtime::Snapshot& snapshot = runtime.snapshot();

// Read integer
uint64_t threshold = snapshot.getInteger(
  "upstream.healthy_panic_threshold",
  50  // default value
);

// Read boolean
bool enabled = snapshot.getBoolean(
  "envoy.reloadable_features.new_feature",
  false  // default value
);

// Read percentage (0-100)
uint64_t percentage = snapshot.getInteger(
  "traffic_routing.canary_percentage",
  0  // default value
);

// Use fractional percentage for precise sampling
Protobuf::RepeatedPtrField<FractionalPercent> fractional;
if (snapshot.featureEnabled("traffic_routing.sample_rate", fractional)) {
  // Sample this request
}
```

### Feature-Gated Code

```cpp
// Feature flag check
if (runtime.snapshot().featureEnabled("envoy.reloadable_features.new_behavior")) {
  // New code path
  useNewImplementation();
} else {
  // Legacy code path (for gradual rollout)
  useOldImplementation();
}
```

### Runtime Guards

**Macro**: `RUNTIME_GUARD(flag_name)`

```cpp
// Declares a runtime feature flag
RUNTIME_GUARD(envoy_reloadable_features_new_feature);

// Usage in code
if (Runtime::runtimeFeatureEnabled(
      "envoy.reloadable_features.new_feature")) {
  // New behavior
}
```

## Admin Interface Operations

### Query Runtime Values

```bash
# Get all runtime values
curl http://localhost:9901/runtime

# Output (JSON):
{
  "layers": [
    {
      "name": "static_layer",
      "values": {
        "upstream.healthy_panic_threshold": "50",
        "health_check.min_interval": "1000"
      }
    },
    {
      "name": "admin",
      "values": {
        "upstream.healthy_panic_threshold": "30"
      }
    }
  ],
  "entries": {
    "upstream.healthy_panic_threshold": {
      "layer_values": ["50", "30"],
      "final_value": "30"
    },
    "health_check.min_interval": {
      "layer_values": ["1000"],
      "final_value": "1000"
    }
  }
}
```

### Modify Runtime Values

```bash
# Set single value
curl -X POST "http://localhost:9901/runtime_modify?key=value"

# Set multiple values
curl -X POST "http://localhost:9901/runtime_modify?key1=value1&key2=value2"

# Delete admin override (set to empty)
curl -X POST "http://localhost:9901/runtime_modify?key="
```

**Examples**:

```bash
# Reduce panic threshold for testing
curl -X POST "http://localhost:9901/runtime_modify?upstream.healthy_panic_threshold=30"

# Enable feature flag
curl -X POST "http://localhost:9901/runtime_modify?envoy.reloadable_features.new_feature=true"

# Adjust health check interval
curl -X POST "http://localhost:9901/runtime_modify?health_check.min_interval=5000"

# Remove override
curl -X POST "http://localhost:9901/runtime_modify?upstream.healthy_panic_threshold="
```

## Common Patterns

### Gradual Feature Rollout

Use percentage-based runtime flags for canary releases:

```yaml
# Start with 0% (disabled)
feature.new_algorithm.percentage: 0
```

```bash
# Gradually increase
curl -X POST "http://localhost:9901/runtime_modify?feature.new_algorithm.percentage=10"
# Monitor metrics...

curl -X POST "http://localhost:9901/runtime_modify?feature.new_algorithm.percentage=50"
# Monitor metrics...

curl -X POST "http://localhost:9901/runtime_modify?feature.new_algorithm.percentage=100"
# Full rollout
```

### Emergency Circuit Breaker Adjustment

```bash
# Cluster experiencing issues, reduce panic threshold
curl -X POST "http://localhost:9901/runtime_modify?upstream.healthy_panic_threshold=20"

# Restore after recovery
curl -X POST "http://localhost:9901/runtime_modify?upstream.healthy_panic_threshold="
```

### Dynamic Retry Tuning

```bash
# Increase retry budget during incident
curl -X POST "http://localhost:9901/runtime_modify?retry.max_retries=5"

# Reduce after resolution
curl -X POST "http://localhost:9901/runtime_modify?retry.max_retries=2"
```

### Zone-Aware Routing Toggle

```bash
# Enable zone-aware routing
curl -X POST "http://localhost:9901/runtime_modify?upstream.zone_routing.enabled=true"

# Disable if causing issues
curl -X POST "http://localhost:9901/runtime_modify?upstream.zone_routing.enabled=false"
```

## Best Practices

### Development

1. **Use Feature Flags**: Gate new features behind runtime flags
   ```cpp
   if (runtime.snapshot().featureEnabled("envoy.reloadable_features.my_new_feature")) {
     // New code
   } else {
     // Legacy code
   }
   ```

2. **Default to Safe Values**: Choose conservative defaults
   ```yaml
   # Default to disabled for new features
   envoy.reloadable_features.experimental_feature: false
   ```

3. **Document Runtime Keys**: Document all custom runtime keys

### Operations

1. **Layer Organization**:
   - Static layer: Stable, well-tested values
   - Disk layer: Environment-specific overrides
   - Admin layer: Emergency adjustments only

2. **Monitoring**: Track runtime value changes in audit logs

3. **Testing**: Test both enabled/disabled states for feature flags

4. **Rollback Plan**: Always know how to revert changes
   ```bash
   # Clear admin overrides
   curl -X POST "http://localhost:9901/runtime_modify?key="
   ```

5. **Persistence**: Admin layer values are lost on restart
   - For permanent changes, update disk or static layer
   - Document temporary admin changes

### Security

1. **Restrict Admin Access**: Admin layer can override all values
2. **Audit Trail**: Enable admin access logging
3. **Validation**: Validate runtime values (e.g., percentages 0-100)
4. **Sensitive Values**: Don't use runtime for secrets

## Troubleshooting

### Value Not Taking Effect

**Check layer resolution**:
```bash
curl http://localhost:9901/runtime | jq '.entries["key"]'
```

Output shows which layer provides the final value.

**Possible Issues**:
1. Lower-priority layer being overridden
2. Key name mismatch (typo)
3. Value type mismatch (integer vs string)
4. Cache not refreshed (symlink not updated)

### File Reload Not Working

**Disk layer issues**:
1. Check symlink root path
2. Verify inotify limits (Linux)
   ```bash
   cat /proc/sys/fs/inotify/max_user_watches
   ```
3. Check Envoy logs for reload errors
4. Ensure subdirectory structure is correct

### Admin Modifications Lost

Admin layer values are **in-memory only**:
- Lost on restart
- Lost on hot restart
- For persistence, update static/disk layer

## Implementation Details

### Snapshot Creation

**Flow**:
1. Loader creates new snapshot
2. Merges all layers (lowest to highest priority)
3. Publishes snapshot atomically
4. Old snapshot reference counted (eventually deleted)

### Thread Safety

- Snapshots are immutable (thread-safe reads)
- Loader updates are serialized
- No locking required for reads

### Hot Restart Behavior

Runtime values from admin layer are **not preserved** across hot restart:
- Static layer: Preserved (in config)
- Disk layer: Preserved (reloaded from files)
- RTDS layer: Re-fetched from control plane
- Admin layer: **Lost** (ephemeral)

## Related Files

- `source/common/runtime/runtime_impl.h` - Runtime implementation
- `source/common/runtime/runtime_impl.cc` - Runtime loader logic
- `source/common/runtime/runtime_features.h` - Feature flag definitions
- `source/common/runtime/runtime_keys.h` - Runtime key constants
- `source/server/admin/runtime_handler.h` - Admin endpoints for runtime
- `envoy/runtime/runtime.h` - Runtime interface

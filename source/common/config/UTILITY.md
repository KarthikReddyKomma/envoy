# Config Utility Functions

**File:** `source/common/config/utility.h` and `source/common/config/utility.cc`

**Purpose:** Collection of utility functions for config validation, transformation, and common operations used throughout Envoy's configuration system.

## Key Utility Functions

### Hash and Version

```cpp
// Compute hash and hex representation for version tracking
static std::pair<std::string, uint64_t> 
computeHashedVersion(const std::string& input);

// Example
auto [version_str, hash] = Utility::computeHashedVersion(proto.SerializeAsString());
// version_str = "hash_1234567890abcdef"
// hash = 0x1234567890abcdef
```

### Cluster Validation

```cpp
// Check if cluster exists and is valid for xDS
static absl::StatusOr<Upstream::ClusterConstOptRef>
checkCluster(absl::string_view error_prefix, 
             absl::string_view cluster_name,
             Upstream::ClusterManager& cm, 
             bool allow_added_via_api = false);

// Example
auto cluster_or_error = Utility::checkCluster(
    "xds_cluster", "my_xds_cluster", cm);
if (!cluster_or_error.ok()) {
  return cluster_or_error.status();
}
```

### Config Source Timeout

```cpp
// Extract initial fetch timeout from config source
static std::chrono::milliseconds
configSourceInitialFetchTimeout(
    const envoy::config::core::v3::ConfigSource& config_source);

// Default is 15 seconds if not specified
auto timeout = Utility::configSourceInitialFetchTimeout(config);
```

### Factory Lookup

```cpp
// Get factory by name with error checking
template<class Factory>
static Factory& getAndCheckFactoryByName(const std::string& name);

// Get factory by protobuf message type
template<class Factory, class ProtoMessage>
static Factory& getAndCheckFactory(const ProtoMessage& message);

// Example
auto& factory = Utility::getAndCheckFactory<NetworkFilterConfigFactory>(config);
```

### gRPC Client Creation

```cpp
// Create gRPC async client factory
static absl::StatusOr<Grpc::AsyncClientFactoryPtr>
factoryForGrpcApiConfigSource(
    Grpc::AsyncClientManager& async_client_manager,
    const envoy::config::core::v3::ApiConfigSource& api_config_source,
    Stats::Scope& scope, 
    bool skip_cluster_check, 
    int grpc_service_idx,
    bool xdstp_config_source);
```

### Backoff Strategy

```cpp
// Create jittered exponential backoff
static absl::StatusOr<JitteredExponentialBackOffStrategyPtr>
prepareJitteredExponentialBackOffStrategy(
    const envoy::config::core::v3::ApiConfigSource& api_config_source,
    Random::RandomGenerator& random, 
    uint32_t default_base_interval_ms,
    absl::optional<uint32_t> default_max_interval_ms);
```

### Stats Generation

```cpp
// Generate subscription stats
static SubscriptionStats generateStats(Stats::Scope& scope);

// Generate control plane stats
static ControlPlaneStats generateControlPlaneStats(Stats::Scope& scope);
```

## Common Patterns

### Validating Config Before Use

```cpp
// Check local info
RETURN_IF_NOT_OK(Utility::checkLocalInfo("my_component", local_info));

// Check cluster
auto cluster_result = Utility::checkCluster("MyFilter", "backend_cluster", cm);
RETURN_IF_NOT_OK(cluster_result.status());

// Check transport version
RETURN_IF_NOT_OK(Utility::checkTransportVersion(api_config_source));
```

### Creating Subscriptions

```cpp
// Validate cluster for API config source
RETURN_IF_NOT_OK(Utility::checkApiConfigSourceSubscriptionBackingCluster(
    cm.primaryClusters(), api_config_source));

// Create gRPC factory
auto factory_or_error = Utility::factoryForGrpcApiConfigSource(
    cm.grpcAsyncClientManager(), api_config_source, 
    scope, false, 0, false);
RETURN_IF_NOT_OK(factory_or_error.status());
```

## Key Takeaways

1. **Validation**: Comprehensive validation functions
2. **Factory Pattern**: Unified factory lookup
3. **Error Handling**: StatusOr for clear errors
4. **Stats**: Consistent stat generation
5. **Reusable**: Common patterns abstracted

## Related Documentation

- [01-xds-manager-impl.md](01-xds-manager-impl.md)
- [02-subscription-factory-impl.md](02-subscription-factory-impl.md)

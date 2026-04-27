# Decoded Resource Implementation

**File:** `source/common/config/decoded_resource_impl.h`

**Purpose:** Wrapper for decoded xDS resources that provides a uniform interface for accessing resource metadata (name, aliases, version, TTL) regardless of the underlying protocol format.

## Overview

DecodedResourceImpl abstracts the differences between:
- Plain protobuf resources
- Wrapped `envoy::service::discovery::v3::Resource` messages
- Collection inline entries (`xds::core::v3::CollectionEntry::InlineEntry`)

## Class Structure

```cpp
class DecodedResourceImpl : public DecodedResource {
public:
  // Factory methods
  static absl::StatusOr<DecodedResourceImplPtr>
  fromResource(OpaqueResourceDecoder& resource_decoder, 
               const Protobuf::Any& resource,
               const std::string& version);
  
  static DecodedResourceImplPtr
  fromResource(OpaqueResourceDecoder& resource_decoder,
               const envoy::service::discovery::v3::Resource& resource);

  // DecodedResource interface
  const std::string& name() const override;
  const std::vector<std::string>& aliases() const override;
  const std::string& version() const override;
  const Protobuf::Message& resource() const override;
  bool hasResource() const override;
  absl::optional<std::chrono::milliseconds> ttl() const override;
  const OptRef<const envoy::config::core::v3::Metadata> 
      metadata() const override;
  
private:
  const ProtobufTypes::MessagePtr resource_;
  const bool has_resource_;
  const std::string name_;
  const std::vector<std::string> aliases_;
  const std::string version_;
  const absl::optional<std::chrono::milliseconds> ttl_;
  const absl::optional<envoy::config::core::v3::Metadata> metadata_;
};
```

## Key Features

### Resource Name and Aliases

```cpp
// Primary name
const std::string& name() const;

// Alternative names (for migration scenarios)
const std::vector<std::string>& aliases() const;
```

### Version Tracking

```cpp
// Version string from xDS server
const std::string& version() const;
```

### TTL Support

```cpp
// Optional per-resource TTL
absl::optional<std::chrono::milliseconds> ttl() const;

// If present, resource should be removed after TTL expires
```

### Metadata

```cpp
// Optional metadata from Resource wrapper
const OptRef<const envoy::config::core::v3::Metadata> metadata() const;
```

## Usage Example

```cpp
// In subscription callback
absl::Status onConfigUpdate(
    const std::vector<DecodedResourceRef>& resources,
    const std::string& version_info) override {
  
  for (const auto& resource_ref : resources) {
    const auto& decoded = resource_ref.get();
    
    ENVOY_LOG(info, "Resource: {} version: {}", 
              decoded.name(), decoded.version());
    
    // Check aliases
    for (const auto& alias : decoded.aliases()) {
      ENVOY_LOG(debug, "  Alias: {}", alias);
    }
    
    // Check TTL
    if (decoded.ttl().has_value()) {
      ENVOY_LOG(info, "  TTL: {}ms", decoded.ttl()->count());
      // Schedule removal after TTL
    }
    
    // Get actual resource proto
    const auto& proto = decoded.resource();
    // Process proto...
  }
  
  return absl::OkStatus();
}
```

## DecodedResourcesWrapper

Helper for batch decoding:

```cpp
struct DecodedResourcesWrapper {
  static absl::StatusOr<std::unique_ptr<DecodedResourcesWrapper>>
  create(OpaqueResourceDecoder& resource_decoder,
         const Protobuf::RepeatedPtrField<Protobuf::Any>& resources, 
         const std::string& version);

  void pushBack(Config::DecodedResourcePtr&& resource);

  std::vector<Config::DecodedResourcePtr> owned_resources_;
  std::vector<Config::DecodedResourceRef> refvec_;
};

// Usage
auto wrapper_or_error = DecodedResourcesWrapper::create(
    decoder, resources, version);
RETURN_IF_NOT_OK(wrapper_or_error.status());

// Pass to callbacks
callbacks.onConfigUpdate(wrapper->refvec_, version);
```

## Key Takeaways

1. **Uniform Interface**: Same API regardless of source format
2. **Resource Metadata**: Name, aliases, version, TTL
3. **Efficient**: Single decode, multiple accessors
4. **TTL Support**: Per-resource expiration
5. **Batch Helper**: DecodedResourcesWrapper for multiple resources

## Related Documentation

- [02-subscription-factory-impl.md](02-subscription-factory-impl.md)
- [CONFIG_ARCHITECTURE.md](../CONFIG_ARCHITECTURE.md)

# DataSource Implementation

**File:** `source/common/config/datasource.h` and `source/common/config/datasource.cc`

**Purpose:** Provides flexible data loading from multiple sources (inline, file, environment variables, remote HTTP) with optional file watching for dynamic updates. Used extensively for loading TLS certificates, secrets, and WASM modules.

## Overview

DataSource supports four data sources:
1. **Inline String/Bytes**: Embedded directly in configuration
2. **Filename**: Load from local filesystem
3. **Environment Variable**: Read from environment
4. **Remote (via watched directory)**: Asynchronous HTTP fetch (future)

## Key Types

```cpp
namespace Config::DataSource {
  // Read data synchronously
  absl::StatusOr<std::string> read(
      const envoy::config::core::v3::DataSource& source,
      bool allow_empty, Api::Api& api, uint64_t max_size = 0);
  
  // Read from file
  absl::StatusOr<std::string> readFile(
      const std::string& path, Api::Api& api, 
      bool allow_empty, uint64_t max_size = 0);
  
  // Provider with file watching
  template<class DataType>
  class DataSourceProvider {
    static absl::StatusOr<DataSourceProviderPtr<DataType>>
    create(const ProtoDataSource& source, 
           Event::Dispatcher& dispatcher,
           ThreadLocal::SlotAllocator& tls,
           Api::Api& api,
           DataTransform<DataType> transform,
           const ProviderOptions& options);
    
    std::shared_ptr<DataType> data() const;
  };
}
```

## Usage Examples

### Example 1: Simple Read

```cpp
envoy::config::core::v3::DataSource source;
source.set_filename("/etc/envoy/cert.pem");

auto data_or_error = Config::DataSource::read(source, false, api);
if (!data_or_error.ok()) {
  ENVOY_LOG(error, "Failed to read: {}", data_or_error.status().message());
  return;
}

std::string cert_data = *data_or_error;
```

### Example 2: Provider with File Watching

```cpp
// Transform function
auto transform = [](absl::string_view data) 
    -> absl::StatusOr<std::shared_ptr<std::string>> {
  return std::make_shared<std::string>(std::string(data));
};

// Create provider
envoy::config::core::v3::DataSource source;
source.set_filename("/etc/envoy/config.yaml");
source.mutable_watched_directory()->set_path("/etc/envoy");

auto provider_or_error = DataSource::DataSourceProvider<std::string>::create(
    source, dispatcher, tls, api, transform, 
    {.allow_empty = false, .modify_watch = true});

if (!provider_or_error.ok()) {
  throw EnvoyException(provider_or_error.status().message());
}

auto provider = std::move(*provider_or_error);

// Access data (thread-safe)
auto config_data = provider->data();
```

## Key Features

1. **Multiple Sources**: Inline, file, env var
2. **File Watching**: Auto-reload on changes
3. **ThreadLocal**: Efficient per-worker storage
4. **Size Limits**: Prevent memory exhaustion
5. **Async Updates**: Non-blocking reload
6. **Singleton Sharing**: Share watchers across instances

## Related Documentation

- [03-config-provider-impl.md](03-config-provider-impl.md) - Similar update pattern
- [CONFIG_ARCHITECTURE.md](../CONFIG_ARCHITECTURE.md)

# Runtime — Class hierarchy (UML)

Every class declared in `source/common/runtime/` plus the closest interfaces from `envoy/runtime/runtime.h` and
`envoy/config/`.

```mermaid
classDiagram
    direction LR

    %% ----------- envoy/runtime interfaces -----------
    class Loader {
      <<interface>>
      +initialize(cm) Status
      +snapshot() Snapshot&
      +threadsafeSnapshot() SnapshotConstSharedPtr
      +mergeValues(values) Status
      +startRtdsSubscriptions(on_done)
      +getRootScope() Scope&
      +countDeprecatedFeatureUse()
      +onWorkerThreadsRegistered() Status
    }

    class Snapshot {
      <<interface>>
      +deprecatedFeatureEnabled(key, default) bool
      +runtimeFeatureEnabled(key) bool
      +featureEnabled(key, ...) bool overloads
      +get(key) ConstStringOptRef
      +getInteger(key, default) uint64
      +getDouble(key, default) double
      +getBoolean(key, value) bool
      +getLayers() vector~OverrideLayerConstPtr~
      +EntryMap : alias
    }

    class OverrideLayer {
      <<interface>>
      +values() EntryMap&
      +name() string&
    }

    class Entry {
      +raw_string_value_ : string
      +uint_value_ : optional~uint64~
      +double_value_ : optional~double~
      +bool_value_ : optional~bool~
      +fractional_percent_value_ : optional~FractionalPercent~
    }

    %% ----------- envoy/config interfaces -----------
    class SubscriptionCallbacks {
      <<interface>>
      +onConfigUpdate(...) Status
      +onConfigUpdateFailed(reason, ex)
    }

    %% ----------- runtime_impl -----------
    class LoaderImpl {
      -generator_ : RandomGenerator&
      -stats_ : RuntimeStats
      -admin_layer_ : AdminLayerPtr
      -tls_ : SlotPtr
      -config_ : LayeredRuntime
      -service_cluster_ : string
      -watcher_ : Filesystem::WatcherPtr
      -on_rtds_initialized_ : ReadyCallback
      -init_watcher_ : Init::WatcherImpl
      -init_manager_ : Init::ManagerImpl
      -subscriptions_ : vector~RtdsSubscriptionPtr~
      -cm_ : ClusterManager*
      -snapshot_mutex_ : Mutex
      -thread_safe_snapshot_ : SnapshotConstSharedPtr
      +create(...) StatusOr unique_ptr
      +initialize(cm) Status override
      +snapshot() Snapshot& override
      +threadsafeSnapshot() Shared override
      +mergeValues(values) Status override
      +startRtdsSubscriptions(on_done) override
      +getRootScope() Scope& override
      +countDeprecatedFeatureUse() override
      +onWorkerThreadsRegistered() Status override
      -initLayers(dispatcher, validator) Status
      -createNewSnapshot() StatusOr SnapshotImplPtr
      -loadNewSnapshot() Status
      -onRtdsReady()
      -generateStats(store) RuntimeStats
    }

    class SnapshotImpl {
      -layers_ : vector~OverrideLayerConstPtr~
      -values_ : EntryMap
      -generator_ : RandomGenerator&
      -stats_ : RuntimeStats&
      +deprecatedFeatureEnabled(key, default) bool override
      +runtimeFeatureEnabled(key) bool override
      +featureEnabled(...) bool overrides
      +get(key) ConstStringOptRef override
      +getInteger(key, default) uint64 override
      +getDouble(key, default) double override
      +getBoolean(key, value) bool override
      +getLayers() vector& override
      +values() EntryMap&
      +createEntry(value, raw) static Entry
      +addEntry(values, key, value, raw) static
    }

    class OverrideLayerImpl {
      #values_ : EntryMap
      #name_ : string
      +values() EntryMap& override
      +name() string& override
    }

    class AdminLayer {
      -stats_ : RuntimeStats&
      +AdminLayer(name, stats)
      +AdminLayer(const AdminLayer&) copy
      +mergeValues(values) Status
    }

    class DiskLayer {
      -path_ : string
      -watcher_ : Filesystem::WatcherPtr
      -walkDirectory(path, prefix, depth, api) Status
    }

    class ProtoLayer {
      -walkProtoValue(value, prefix) Status
    }

    class RtdsSubscription {
      +parent_ : LoaderImpl&
      +config_source_ : ConfigSource
      +store_ : Stats::Store&
      +stats_scope_ : ScopeSharedPtr
      +subscription_ : SubscriptionPtr
      +resource_name_ : string
      +init_target_ : Init::TargetImpl
      +resource_type_helper_ : ResourceTypeHelper
      +proto_ : Protobuf::Struct
      +onConfigUpdate(resources, version) Status override
      +onConfigUpdate(added, removed, sys) Status override
      +onConfigUpdateFailed(reason, ex) override
      +start()
      +validateUpdateSize(added, removed) Status
      +onConfigRemoved(removed) Status
      +createSubscription() Status
    }

    class RuntimeStats {
      +deprecated_feature_use : Counter
      +load_error : Counter
      +load_success : Counter
      +override_dir_exists : Counter
      +override_dir_not_exists : Counter
      +admin_overrides_active : Gauge
      +deprecated_feature_seen_since_process_start : Gauge
      +num_keys : Gauge
      +num_layers : Gauge
    }

    %% ----------- runtime_features -----------
    class RuntimeFeatures {
      -all_features_ : flat_hash_map~string, absl::CommandLineFlag*~
      +RuntimeFeatures()
      +getFlag(feature) absl::CommandLineFlag*
    }

    class FreeFunctions {
      <<utility>>
      +runtimeFeatureEnabled(feature) bool
      +getInteger(feature, default) uint64
      +maybeSetRuntimeGuard(name, value)
      +maybeSetDeprecatedInts(name, value)
      +markRuntimeInitialized()
      +isRuntimeInitialized() bool
      +isRuntimeFeature(feature) bool
      +isLegacyRuntimeFeature(feature) bool
      +hasRuntimePrefix(feature) bool
    }

    %% ----------- inheritance & composition -----------
    Loader <|.. LoaderImpl
    Snapshot <|.. SnapshotImpl
    OverrideLayer <|.. OverrideLayerImpl
    OverrideLayerImpl <|-- AdminLayer
    OverrideLayerImpl <|-- DiskLayer
    OverrideLayerImpl <|-- ProtoLayer
    SubscriptionCallbacks <|.. RtdsSubscription

    LoaderImpl *-- AdminLayer : owns single instance
    LoaderImpl *-- RtdsSubscription : vector
    LoaderImpl *-- RuntimeStats
    LoaderImpl o-- SnapshotImpl : thread_safe_snapshot_ + TLS
    SnapshotImpl *-- OverrideLayerImpl : layers_
    SnapshotImpl *-- Entry : values_
    LoaderImpl --> RuntimeFeatures : via free fns

    RuntimeFeatures --> FreeFunctions
```

## How to read it

- **Solid arrow with empty triangle (`<|..`)** — interface implementation.
- **Solid arrow with filled triangle (`<|--`)** — class inheritance.
- **Diamond (`*--`)** — strong ownership.
- **Open diamond (`o--`)** — non-owning (TLS slot pointer / mutex-guarded shared_ptr).

Three structural observations:

1. **One `Loader`, many snapshots.** `LoaderImpl` lives as long as the server. Each `SnapshotImpl` is created
   per-update and outlives in-flight reads on workers via its `shared_ptr` reference count.
2. **Layers are immutable post-construction.** `ProtoLayer` walks the input once; `DiskLayer` walks the filesystem
   once; `AdminLayer` is the only layer whose values mutate — and snapshots see a **copy** of it.
3. **`RuntimeFeatures` is independent of `LoaderImpl`.** The flag store is a `ConstSingleton` built from `absl::Flag`
   reflection; it works even before a loader exists (e.g. in unit tests that call `runtimeFeatureEnabled` without
   spinning up a `LoaderImpl`).

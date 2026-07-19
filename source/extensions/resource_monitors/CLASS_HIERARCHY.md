# Resource Monitors — Class Hierarchy

UML-style class diagrams for the resource monitor extensions. Documentation aids, not exhaustive.

## 1. The two interface models

```mermaid
classDiagram
    class ResourceMonitor {
        <<interface>>
        +updateResourceUsage(callbacks) void
    }
    class ResourceUpdateCallbacks {
        <<interface>>
        +onSuccess(ResourceUsage) void
        +onFailure(EnvoyException) void
    }
    class ResourceUsage {
        +resource_pressure_ : double
    }
    class ProactiveResourceMonitor {
        <<interface>>
        +tryAllocateResource(inc) bool
        +tryDeallocateResource(dec) bool
        +currentResourceUsage() int64
        +maxResourceUsage() int64
    }
    class ProactiveResource {
        -monitor_ : ProactiveResourceMonitor
        -failed_updates_ : Counter
        -pressure_gauge_ : Gauge
        +updateResourcePressure() double
    }

    ResourceMonitor ..> ResourceUpdateCallbacks : reports via
    ResourceUpdateCallbacks ..> ResourceUsage
    ProactiveResource o-- ProactiveResourceMonitor
```

## 2. Factory bases (common/)

```mermaid
classDiagram
    class ResourceMonitorFactory {
        <<interface>>
        +createResourceMonitor(cfg, ctx) ResourceMonitorPtr
        +createEmptyConfigProto() MessagePtr
        +name() string
    }
    class ProactiveResourceMonitorFactory {
        <<interface>>
        +createProactiveResourceMonitor(cfg, ctx) ProactiveResourceMonitorPtr
    }
    class FactoryBase~ConfigProto~ {
        +createResourceMonitor(...) downcastAndValidate
        #createResourceMonitorFromProtoTyped(cfg, ctx)* PURE
    }
    class ProactiveFactoryBase~ConfigProto~ {
        #createProactiveResourceMonitorFromProtoTyped(cfg, ctx)* PURE
    }

    ResourceMonitorFactory <|-- FactoryBase
    ProactiveResourceMonitorFactory <|-- ProactiveFactoryBase
```

## 3. Regular monitors

```mermaid
classDiagram
    class ResourceMonitor { <<interface>> }
    class FixedHeapMonitor {
        -stats_ : MemoryStatsReader
        -max_heap_ : variant
    }
    class CpuUtilizationMonitor {
        -cpu_stats_reader_ : CpuStatsReader
        -utilization_ : double (EWMA)
    }
    class CgroupMemoryMonitor {
        -stats_reader_ : CgroupMemoryStatsReader
        -max_memory_bytes_ : uint64
    }
    class InjectedResourceMonitor {
        -filename_ : string
        -watcher_ : Filesystem::Watcher
        -pressure_ : optional~double~
    }

    ResourceMonitor <|-- FixedHeapMonitor
    ResourceMonitor <|-- CpuUtilizationMonitor
    ResourceMonitor <|-- CgroupMemoryMonitor
    ResourceMonitor <|-- InjectedResourceMonitor
```

## 4. Stats-reader seams

```mermaid
classDiagram
    class MemoryStatsReader {
        +reservedHeapBytes() uint64
        +unmappedHeapBytes() uint64
        +freeMappedHeapBytes() uint64
        +allocatedHeapBytes() uint64
    }
    class CpuStatsReader {
        <<interface>>
        +getUtilization() StatusOr~double~
    }
    class LinuxCpuStatsReader
    class CgroupV1CpuStatsReader
    class CgroupV2CpuStatsReader
    class CgroupMemoryStatsReader {
        +readMemoryStats() MemoryStats
    }
    class CgroupV1StatsReader
    class CgroupV2StatsReader

    CpuStatsReader <|-- LinuxCpuStatsReader
    CpuStatsReader <|-- CgroupV1CpuStatsReader
    CpuStatsReader <|-- CgroupV2CpuStatsReader
    CgroupMemoryStatsReader <|-- CgroupV1StatsReader
    CgroupMemoryStatsReader <|-- CgroupV2StatsReader

    FixedHeapMonitor ..> MemoryStatsReader
    CpuUtilizationMonitor ..> CpuStatsReader
    CgroupMemoryMonitor ..> CgroupMemoryStatsReader
```

## 5. The proactive monitor

```mermaid
classDiagram
    class ProactiveResourceMonitor { <<interface>> }
    class ActiveDownstreamConnectionsResourceMonitor {
        -current_ : atomic~int64~
        -max_ : int64
        -synchronizer_ : ThreadSynchronizer
        +tryAllocateResource(inc) bool
        +tryDeallocateResource(dec) bool
    }
    ProactiveResourceMonitor <|-- ActiveDownstreamConnectionsResourceMonitor
```

## 6. Each monitor's factory

```mermaid
classDiagram
    class FactoryBase~ConfigProto~
    class ProactiveFactoryBase~ConfigProto~
    class FixedHeapMonitorFactory
    class CpuUtilizationMonitorFactory
    class CgroupMemoryMonitorFactory
    class InjectedResourceMonitorFactory
    class ActiveDownstreamConnectionsMonitorFactory

    FactoryBase <|-- FixedHeapMonitorFactory
    FactoryBase <|-- CpuUtilizationMonitorFactory
    FactoryBase <|-- CgroupMemoryMonitorFactory
    FactoryBase <|-- InjectedResourceMonitorFactory
    ProactiveFactoryBase <|-- ActiveDownstreamConnectionsMonitorFactory
```

All regular factories register against `ResourceMonitorFactory`; the proactive one registers
against `ProactiveResourceMonitorFactory`. Category for both: `envoy.resource_monitors`.

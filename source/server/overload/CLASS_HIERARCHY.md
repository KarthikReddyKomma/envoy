# Overload Manager — Class Hierarchy

UML-style class diagrams for the overload subsystem. Documentation aids, not exhaustive.

## 1. The manager

```mermaid
classDiagram
    class LoadShedPointProvider {
        <<interface>>
        +getLoadShedPoint(name) LoadShedPoint*
    }
    class OverloadManager {
        <<interface>>
        +start() void
        +stop() void
        +registerForAction(action, dispatcher, cb) bool
        +getThreadLocalOverloadState() ThreadLocalOverloadState&
        +scaledTimerFactory() ScaledRangeTimerManagerFactory
        +getShrinkHeapConfig() optional~ShrinkHeapConfig~
    }
    class OverloadManagerImpl {
        -dispatcher_ : Dispatcher&
        -tls_ : TypedSlot
        -action_symbol_table_ : NamedOverloadActionSymbolTable
        -refresh_interval_ : ms
        -timer_ : TimerPtr
        -resources_ : map~string,Resource~
        -proactive_resources_ : map
        -actions_ : map~Symbol,OverloadAction~
        -loadshed_points_ : map
        +flushResourceUpdates() void
        +updateResourcePressure(name, pressure, epoch) void
    }
    class NullOverloadManager {
        -permissive_ : bool
        note: never overloaded
    }

    LoadShedPointProvider <|-- OverloadManager
    OverloadManager <|-- OverloadManagerImpl
    OverloadManager <|-- NullOverloadManager
```

## 2. Resources and monitors

```mermaid
classDiagram
    class ResourceMonitor {
        <<interface>>
        +updateResourceUsage(ResourceUpdateCallbacks&) void
    }
    class ProactiveResourceMonitor {
        <<interface>>
        +tryAllocateResource(increment) bool
        +tryDeallocateResource(decrement) bool
        +currentResourceUsage() int64
        +maxResourceUsage() int64
    }
    class ResourceUpdateCallbacks {
        <<interface>>
        +onSuccess(ResourceUsage&) void
        +onFailure(error) void
    }
    class Resource {
        -monitor_ : ResourceMonitorPtr
        -pending_update_ : bool
        +update(epoch) void
        +onSuccess(usage) void
    }
    class ProactiveResource {
        -monitor_ : ProactiveResourceMonitorPtr
        +tryAllocateResource(n) bool
        +updateResourcePressure() double
    }

    ResourceUpdateCallbacks <|-- Resource
    Resource o-- ResourceMonitor
    ProactiveResource o-- ProactiveResourceMonitor
    OverloadManagerImpl *-- Resource
    OverloadManagerImpl *-- ProactiveResource
```

## 3. Actions, triggers, and the symbol table

```mermaid
classDiagram
    class Trigger {
        <<interface>>
        +updateValue(value) bool
        +actionState() OverloadActionState
    }
    class ThresholdTriggerImpl {
        -threshold_ : double
    }
    class ScaledTriggerImpl {
        -scaling_threshold_ : double
        -saturated_threshold_ : double
    }
    class OverloadAction {
        -triggers_ : map~string,TriggerPtr~
        -state_ : OverloadActionState
        +updateResourcePressure(name, pressure) bool
        +getState() OverloadActionState
    }
    class OverloadActionState {
        -value_ : UnitFloat
        +inactive()$ OverloadActionState
        +saturated()$ OverloadActionState
        +isSaturated() bool
        +value() UnitFloat
    }
    class NamedOverloadActionSymbolTable {
        +get(name) Symbol
        +lookup(name) optional~Symbol~
        +name(symbol) string
        +size() size_t
    }

    Trigger <|-- ThresholdTriggerImpl
    Trigger <|-- ScaledTriggerImpl
    OverloadAction *-- Trigger
    OverloadAction --> OverloadActionState
```

## 4. Thread-local state

```mermaid
classDiagram
    class ThreadLocalObject { <<interface>> }
    class ThreadLocalOverloadState {
        <<interface>>
        +getState(action) OverloadActionState&
        +tryAllocateResource(name, inc) bool
        +tryDeallocateResource(name, dec) bool
        +isResourceMonitorEnabled(name) bool
    }
    class ThreadLocalOverloadStateImpl {
        -actions_ : vector~OverloadActionState~
        -action_symbol_table_ : ref
        -proactive_resources_ : ref
        +setState(Symbol, state) void
    }

    ThreadLocalObject <|-- ThreadLocalOverloadState
    ThreadLocalOverloadState <|-- ThreadLocalOverloadStateImpl
    OverloadManagerImpl ..> ThreadLocalOverloadStateImpl : runOnAllThreads(setState)
```

The flat `actions_` vector indexed by interned `Symbol` is what makes the hot-path
`getState(action)` an O(1), lock-free array read on each worker thread.

## 5. Scaled-timer link

```mermaid
flowchart LR
    OM["OverloadManagerImpl"] -->|scaledTimerFactory| STM["ScaledRangeTimerManager<br/>(per worker dispatcher)"]
    OM -->|reduce_timeouts callback| STM
    STM -->|setScaleFactor(state.invert())| Timers["scaled timers fire at<br/>min + (max-min)*factor"]
```

# Config Validation — Class Hierarchy

UML-style class diagrams for the validation stubs versus their production counterparts.
Documentation aids, not exhaustive.

## 1. The validation instance

```mermaid
classDiagram
    class Instance { <<interface>> }
    class ServerLifecycleNotifier { <<interface>> }
    class WorkerFactory { <<interface>> }

    class ValidationInstance {
        -init_manager_ : Init::ManagerImpl
        -api_ : ValidationImpl
        -dispatcher_ : ValidationDispatcher
        -admin_ : ValidationAdmin
        -config_ : MainImpl
        -cluster_manager_factory_ : ValidationClusterManagerFactory
        -listener_manager_ : ListenerManager
        -overload_manager_ : OverloadManagerImpl
        -nop_hot_restart_ : HotRestartNopImpl
        -server_contexts_ : ServerFactoryContextImpl
        +run() PANIC
        +createWorker(...) nullptr
        +initialize(...) void
        +shutdown() void
    }

    Instance <|-- ValidationInstance
    ServerLifecycleNotifier <|-- ValidationInstance
    WorkerFactory <|-- ValidationInstance
```

`ValidationInstance` is its own `WorkerFactory` (it returns `nullptr` workers, since none run)
and its own lifecycle notifier (callbacks return `nullptr` handles).

## 2. Stub vs. production: dispatcher and API

```mermaid
classDiagram
    class DispatcherImpl
    class ValidationDispatcher {
        +createClientConnection(...) override
        note: blocks outbound network
    }
    class Impl {
        <<Api::Impl>>
    }
    class ValidationImpl {
        +allocateDispatcher(name) override
        note: hands out ValidationDispatcher
    }

    DispatcherImpl <|-- ValidationDispatcher
    Impl <|-- ValidationImpl
    ValidationImpl ..> ValidationDispatcher : creates
```

## 3. Stub vs. production: cluster manager

```mermaid
classDiagram
    class ClusterManagerImpl
    class ValidationClusterManager {
        +getThreadLocalCluster(name) nullptr
        note: no upstream connections
    }
    class ProdClusterManagerFactory
    class ValidationClusterManagerFactory {
        +clusterManagerFromProto(bootstrap) override
        +createCds(...) nullptr
        note: builds then discards CDS
    }

    ClusterManagerImpl <|-- ValidationClusterManager
    ProdClusterManagerFactory <|-- ValidationClusterManagerFactory
    ValidationClusterManagerFactory ..> ValidationClusterManager : creates
```

## 4. Stub vs. production: admin and hot restart

```mermaid
classDiagram
    class Admin { <<interface>> }
    class AdminImpl {
        note: full HTTP server
    }
    class ValidationAdmin {
        -config_tracker_ : ConfigTrackerImpl
        +startHttpListener(...) no-op
        +makeRequest(...) nullptr
        note: no listener opened
    }
    class HotRestart { <<interface>> }
    class HotRestartImpl
    class HotRestartNopImpl {
        note: no shared memory / takeover
    }

    Admin <|-- AdminImpl
    Admin <|-- ValidationAdmin
    HotRestart <|-- HotRestartImpl
    HotRestart <|-- HotRestartNopImpl
    ValidationInstance ..> ValidationAdmin
    ValidationInstance ..> HotRestartNopImpl
```

## 5. The substitution map

```mermaid
flowchart LR
    subgraph Production
      P1["DispatcherImpl"]
      P2["Api::Impl"]
      P3["ClusterManagerImpl"]
      P4["AdminImpl"]
      P5["HotRestartImpl"]
    end
    subgraph Validation
      V1["ValidationDispatcher"]
      V2["ValidationImpl"]
      V3["ValidationClusterManager"]
      V4["ValidationAdmin"]
      V5["HotRestartNopImpl"]
    end
    P1 -. swapped for .-> V1
    P2 -. swapped for .-> V2
    P3 -. swapped for .-> V3
    P4 -. swapped for .-> V4
    P5 -. swapped for .-> V5
```

Everything not in this map (config parsing, factories, filter-chain building, runtime,
overload manager, secret manager, SSL context manager) is the **same code** the serving server
runs — that's what gives validation its fidelity.

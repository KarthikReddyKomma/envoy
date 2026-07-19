# Server Lifecycle — Class Hierarchy

UML-style class diagrams for the server instance, configuration, and factory context.
These are documentation aids, not exhaustive header transcriptions.

## 1. The instance

```mermaid
classDiagram
    class Instance {
        <<interface>>
        +dispatcher() Dispatcher&
        +threadLocal() ThreadLocal::Instance&
        +clusterManager() ClusterManager&
        +stats() Store&
        +runtime() Loader&
        +admin() OptRef~Admin~
        +listenerManager() ListenerManager&
        +overloadManager() OverloadManager&
        +singletonManager() Singleton::Manager&
        +shutdown() void
        +isShutdown() bool
    }

    class ServerLifecycleNotifier {
        <<interface>>
        +registerCallback(Stage, StageCallback) HandlePtr
    }

    class InstanceBase {
        -dispatcher_ : DispatcherPtr
        -thread_local_ : ThreadLocal::Instance&
        -stats_store_ : Store&
        -runtime_ : LoaderPtr
        -cluster_manager_factory_ : ...
        -listener_manager_ : ListenerManagerPtr
        -admin_ : AdminPtr
        -overload_manager_ : OverloadManagerPtr
        -main_thread_guard_dog_ : GuardDogPtr
        -worker_guard_dog_ : GuardDogPtr
        -drain_manager_ : DrainManagerPtr
        -restarter_ : HotRestart&
        -init_manager_ : Init::ManagerImpl
        -workers_started_ : bool
        -shutdown_ : bool
        -terminated_ : bool
        +initialize(addr, ComponentFactory&) void
        +run() void
        +terminate() void
        +shutdown() void
        +flushStats() void
        #initializeOrThrow() Status
        #onRuntimeReady() void
        #startWorkers() void
    }

    class InstanceImpl {
        -heap_shrinker_ : HeapShrinkerPtr
        #createOverloadManager() OverloadManagerPtr
        #createNullOverloadManager() OverloadManagerPtr
        #maybeCreateGuardDog(...) GuardDogPtr
        #maybeCreateHdsDelegate(...) HdsDelegateApiPtr
        #maybeCreateHeapShrinker() void
    }

    Instance <|-- InstanceBase
    ServerLifecycleNotifier <|-- InstanceBase
    InstanceBase <|-- InstanceImpl
```

`InstanceBase` is abstract: it defines the entire lifecycle but defers a handful of
"how do I build the production version of X" decisions to virtual methods. `InstanceImpl`
provides the production answers. Test/validation servers provide different ones (see
`../config_validation` and the test fixtures).

## 2. Bootstrap helpers and run helper

```mermaid
classDiagram
    class InstanceUtil {
        <<utility>>
        +loadBootstrapConfig(Bootstrap&, Options, ...) Status$
        +createRuntime(Instance&, Initial&) LoaderPtr$
        +flushMetricsToSinks(sinks, Store&, ...) void$
    }

    class RunHelper {
        -init_watcher_ : Init::WatcherImpl
        -sigterm_ : SignalEventPtr
        -sigint_ : SignalEventPtr
        -sig_usr_1_ : SignalEventPtr
        -sig_hup_ : SignalEventPtr
        +RunHelper(instance, options, dispatcher, xds, cm, ..., post_init_cb)
    }

    class ComponentFactory {
        <<interface>>
        +createRuntime(server, config) LoaderPtr
        +createDrainManager(server) DrainManagerPtr
    }

    InstanceBase ..> InstanceUtil : uses
    InstanceBase ..> RunHelper : creates in run()
    InstanceBase ..> ComponentFactory : uses in initialize()
```

`RunHelper` exists primarily so the early-shutdown-during-startup behavior can be unit
tested in isolation. Its constructor performs all the "about to run the loop" wiring.

## 3. Configuration objects

```mermaid
classDiagram
    class Initial {
        <<interface>>
        +admin() Admin&
        +runtime() Runtime*
        +flagsPath() optional~string~
        +layeredRuntime() LayeredRuntime&
    }

    class Main {
        <<interface>>
        +clusterManager() ClusterManager*
        +statsSinks() list~SinkPtr~
        +statsFlushInterval() ms
        +tracer() ...
        +listenerManager() ...
    }

    class InitialImpl {
        -admin_ : AdminImpl
        -runtime_ : RuntimeImpl
        -layered_runtime_ : LayeredRuntime
        -flags_path_ : optional~string~
    }

    class MainImpl {
        -cluster_manager_ : ClusterManagerPtr
        -stats_sinks_ : list~SinkPtr~
        -stats_flush_interval_ : ms
        +initialize(Bootstrap&, Instance&, ClusterManagerFactory&) Status
    }

    Initial <|-- InitialImpl
    Main <|-- MainImpl
```

`InitialImpl` is parsed first (its fields are needed before the main config). `MainImpl`
holds the rest, and its `initialize()` is what actually builds the cluster manager and the
stats sinks from the bootstrap proto.

## 4. The factory context

```mermaid
classDiagram
    class ServerFactoryContext {
        <<interface>>
        +clusterManager() ClusterManager&
        +mainThreadDispatcher() Dispatcher&
        +scope() Scope&
        +threadLocal() ThreadLocal::Instance&
        +runtime() Loader&
        +singletonManager() Singleton::Manager&
        +api() Api&
        +options() Options&
        +messageValidationContext() ...
    }

    class ServerFactoryContextImpl {
        -server_ : Instance&
        -server_scope_ : ScopePtr
    }

    ServerFactoryContext <|-- ServerFactoryContextImpl
    ServerFactoryContextImpl ..> Instance : delegates to
```

`ServerFactoryContextImpl` is a thin adapter: it forwards each accessor to the underlying
`Instance` (or a scoped view of it). Extensions receive this context instead of the raw
server so the extension API stays decoupled from the concrete server implementation.

# Workers & Listeners — Class Hierarchy

UML-style class diagrams for the worker, listener-acceptance, and guard/watch dog code.
Documentation aids, not exhaustive.

## 1. Worker and factory

```mermaid
classDiagram
    class Worker {
        <<interface>>
        +addListener(...) void
        +removeListener(...) void
        +stopListener(...) void
        +start(guard_dog, cb) void
        +stop() void
        +numConnections() uint64
    }
    class WorkerImpl {
        -dispatcher_ : DispatcherPtr
        -handler_ : ConnectionHandlerPtr
        -thread_ : ThreadPtr
        -watch_dog_ : WatchDogSharedPtr
        -tls_ : ThreadLocal::Instance&
        -threadRoutine(guard_dog, cb) void
    }
    class WorkerFactory {
        <<interface>>
        +createWorker(index, overload, null_overload, name) WorkerPtr
    }
    class ProdWorkerFactory {
        -api_ : Api&
        -tls_ : ThreadLocal::Instance&
        -hooks_ : ListenerHooks&
    }

    Worker <|-- WorkerImpl
    WorkerFactory <|-- ProdWorkerFactory
    ProdWorkerFactory ..> WorkerImpl : creates
    WorkerImpl o-- ConnectionHandler
```

## 2. Connection handler

```mermaid
classDiagram
    class TcpConnectionHandler { <<interface>> }
    class UdpConnectionHandler { <<interface>> }
    class InternalListenerManager { <<interface>> }
    class ConnectionHandler {
        note: combined handler type used by workers
    }
    TcpConnectionHandler <|-- ConnectionHandler
    UdpConnectionHandler <|-- ConnectionHandler
    InternalListenerManager <|-- ConnectionHandler

    class ConnectionHandlerFactory {
        <<UntypedFactory>>
        category = "envoy.connection_handler"
    }
    ConnectionHandlerFactory ..> ConnectionHandler : creates
```

## 3. Active listeners

```mermaid
classDiagram
    class ActiveListener {
        <<interface>>
        +listenerTag() uint64
    }
    class ActiveListenerImplBase {
        -stats_ : ListenerStats
        -per_worker_stats_ : PerHandlerListenerStats
        -config_ : ListenerConfig*
        +listenerTag() uint64
    }
    class ActiveUdpListener { <<interface>> }
    class ActiveUdpListenerBase {
        -worker_index_ : uint32
        -concurrency_ : uint32
        -udp_listener_worker_router_ : ref
        +onData(UdpRecvData&&) void
        +destination(data) uint32
    }
    class ActiveRawUdpListener {
        -udp_packet_writer_ : UdpPacketWriterPtr
        +onDataWorker(UdpRecvData&&) void
    }

    ActiveListener <|-- ActiveListenerImplBase
    ActiveListenerImplBase <|-- ActiveUdpListenerBase
    ActiveUdpListener <|-- ActiveUdpListenerBase
    ActiveUdpListenerBase <|-- ActiveRawUdpListener
```

(The TCP active listener, `ActiveTcpListener`, lives in
`source/common/listener_manager/` and also derives from `ActiveListenerImplBase` via a
stream-listener base.)

## 4. Stats and hooks

```mermaid
classDiagram
    class ListenerStats {
        +downstream_cx_total
        +downstream_cx_active
        +downstream_cx_destroy
        +downstream_cx_overflow
        +no_filter_chain_match
        +downstream_cx_length_ms
    }
    class PerHandlerListenerStats {
        +downstream_cx_total
        +downstream_cx_active
    }
    ActiveListenerImplBase *-- ListenerStats
    ActiveListenerImplBase *-- PerHandlerListenerStats

    class ListenerHooks {
        <<interface>>
        +onWorkerListenerAdded() void
        +onWorkerListenerRemoved() void
        +onWorkersStarted() void
    }
    class DefaultListenerHooks {
        note: production; all empty
    }
    ListenerHooks <|-- DefaultListenerHooks
```

## 5. Listener manager factory

```mermaid
classDiagram
    class ListenerManagerFactory {
        <<TypedFactory>>
        category = "envoy.listener_manager_impl"
        +createListenerManager(config, server, factory, worker_factory, ...) ListenerManagerPtr
    }
```

## 6. Guard dog and watch dog

```mermaid
classDiagram
    class WatchDog {
        <<interface>>
        +touch() void
        +threadId() ThreadId
    }
    class WatchDogImpl {
        -thread_id_ : ThreadId
        -touched_ : atomic~bool~
        +getTouchedAndReset() bool
    }
    class GuardDog {
        <<interface>>
        +createWatchDog(tid, name, dispatcher) WatchDogSharedPtr
        +stopWatching(wd) void
    }
    class GuardDogImpl {
        -watched_dogs_ : list~WatchedDog~
        -loop_timer_ : TimerPtr
        -miss_timeout_ / megamiss_timeout_
        -kill_timeout_ / multi_kill_timeout_
        -multi_kill_fraction_ : double
        -step() void
        -invokeGuardDogActions(event, threads, now) void
    }
    class WatchedDog {
        -dog_ : WatchDogImplPtr
        -last_checkin_ : MonotonicTime
        -miss_alerted_ : bool
        -megamiss_alerted_ : bool
    }

    WatchDog <|-- WatchDogImpl
    GuardDog <|-- GuardDogImpl
    GuardDogImpl *-- WatchedDog
    WatchedDog *-- WatchDogImpl
```

The dispatcher's `WatchdogRegistration` (in `source/common/event/`) holds the `WatchDog` and
arms the touch timer; see `guarddog.md` for the touch + escalation mechanics.

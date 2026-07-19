# Hot Restart — Class Hierarchy

UML-style class diagrams for the hot restart subsystem. Documentation aids, not exhaustive.

## 1. The interface and implementations

```mermaid
classDiagram
    class HotRestart {
        <<interface>>
        +drainParentListeners() void
        +duplicateParentListenSocket(addr, worker, ns) int
        +registerUdpForwardingListener(addr, cfg) void
        +parentDrainedCallbackRegistrar() OptRef
        +initialize(Dispatcher&, Instance&) void
        +sendParentAdminShutdownRequest() optional~AdminShutdownResponse~
        +sendParentTerminateRequest() void
        +mergeParentStatsIfAny(StoreRoot&) ServerStatsFromParent
        +shutdown() void
        +baseId() uint32
        +version() string
        +logLock() BasicLockable&
        +accessLogLock() BasicLockable&
        +isInitializing() bool
    }

    class HotRestartImpl {
        -base_id_ : uint32
        -scaled_base_id_ : uint32
        -as_child_ : HotRestartingChild
        -as_parent_ : HotRestartingParent
        -shmem_ : SharedMemory*
        -log_lock_ : ProcessSharedMutex
        -access_log_lock_ : ProcessSharedMutex
        +hotRestartVersion()$ string
    }

    class HotRestartNopImpl {
        -log_lock_ : MutexBasicLockable
        -access_log_lock_ : MutexBasicLockable
        note: all ops are stubs; version()="disabled"
    }

    HotRestart <|-- HotRestartImpl
    HotRestart <|-- HotRestartNopImpl
```

## 2. Shared memory and process-shared mutex

```mermaid
classDiagram
    class SharedMemory {
        +size_ : uint64
        +version_ : uint64
        +log_lock_ : pthread_mutex_t
        +access_log_lock_ : pthread_mutex_t
        +flags_ : atomic~uint64~
        +SHMEM_FLAGS_INITIALIZING = 0x1
    }
    class ProcessSharedMutex {
        <<BasicLockable>>
        -mutex_ : pthread_mutex_t&
        +lock() void
        +tryLock() bool
        +unlock() void
        note: PROCESS_SHARED + ROBUST;<br/>recovers on EOWNERDEAD
    }
    HotRestartImpl *-- SharedMemory
    HotRestartImpl *-- ProcessSharedMutex
    ProcessSharedMutex ..> SharedMemory : references mutexes in shmem
```

## 3. Transport classes

```mermaid
classDiagram
    class HotRestartingBase {
        #main_rpc_stream_ : RpcStream
        #udp_forwarding_rpc_stream_ : RpcStream
        +hotRestartGeneration() uint64
    }
    class RpcStream {
        -domain_socket_ : int
        +bindDomainSocket(id, role, path, mode) void
        +createDomainSocketAddress(id, role, path, mode) sockaddr_un
        +sendHotRestartMessage(addr, proto, allow_failure) bool
        +receiveHotRestartMessage(blocking) HotRestartMessagePtr
        -getPassedFdIfPresent(out, msghdr) void
    }
    HotRestartingBase *-- RpcStream : two channels
```

## 4. Child and parent roles

```mermaid
classDiagram
    class HotRestartingBase
    class HotRestartingChild {
        -restart_epoch_ : int
        -parent_terminated_ : bool
        -parent_drained_ : bool
        -parent_address_ : sockaddr_un
        -stat_merger_ : StatMergerPtr
        +duplicateParentListenSocket(addr, worker) int
        +getParentStats() ...
        +drainParentListeners() void
        +sendParentAdminShutdownRequest() optional
        +sendParentTerminateRequest() void
        +mergeParentStatsIfAny(StoreRoot&) ...
        +registerParentDrainedCallback(cb) void
    }
    class HotRestartingParent {
        -child_address_ : sockaddr_un
        -internal_ : Internal
        +initialize(Dispatcher&, Instance&) void
        +onSocketEvent() void
        +shutdown() void
    }
    class Internal {
        +shutdownAdmin() HotRestartMessage
        +getListenSocketsForChild(req) HotRestartMessage
        +exportStatsToChild(Stats*) void
        +drainListeners() void
    }

    HotRestartingBase <|-- HotRestartingChild
    HotRestartingBase <|-- HotRestartingParent
    HotRestartingParent *-- Internal
```

## 5. Ownership at runtime

```mermaid
flowchart TD
    HRI["HotRestartImpl (this process)"] --> Child["as_child_ : HotRestartingChild<br/>(talks to predecessor, epoch-1)"]
    HRI --> Parent["as_parent_ : HotRestartingParent<br/>(serves successor, epoch+1)"]
    HRI --> SHM["SharedMemory* (mmap)"]
    Child --> CMain["main_rpc_stream_"]
    Child --> CUdp["udp_forwarding_rpc_stream_"]
    Parent --> PMain["main_rpc_stream_"]
    Parent --> PUdp["udp_forwarding_rpc_stream_"]
```

Each process is both a child (of the older process it is replacing) and a parent (ready to
serve a future replacement), which is why both halves are always constructed.

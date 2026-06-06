# Thread-local storage — class hierarchy (UML)

UML for `envoy/thread_local/` (interfaces) and `source/common/thread_local/` (implementation).

---

## Interfaces + implementation

```mermaid
classDiagram
    direction TB

    class ThreadLocalObject {
        <<base>>
        +~ThreadLocalObject()
        +asType~T~() T&
    }

    class SlotAllocator {
        <<interface>>
        +allocateSlot() SlotPtr
    }

    class Slot {
        <<interface>>
        +currentThreadRegistered() bool
        +get() ThreadLocalObjectSharedPtr
        +getTyped~T~() T&
        +set(InitializeCb) void
        #runOnAllThreads(UpdateCb) void
        #isShutdown() bool
    }

    class Instance {
        <<interface>>
        +registerThread(dispatcher, main) void
        +shutdownGlobalThreading() void
        +shutdownThread() void
        +dispatcher() Dispatcher&
        +isShutdown() bool
    }

    class TypedSlot~T~ {
        <<template>>
        -SlotPtr slot_
        +set(InitializeCb) void
        +get() OptRef~T~
        +operator*() T&
        +operator->() T*
        +runOnAllThreads(UpdateCb) void
        +makeUnique(allocator)$ TypedSlotPtr~T~
    }

    class InstanceImpl {
        -MainThread main_thread_
        -vector~Slot*~ slots_
        -vector~uint32~ free_slot_indexes_
        -list~Dispatcher&~ registered_threads_
        -Dispatcher* main_thread_dispatcher_
        -atomic~bool~ shutdown_
        -static thread_local ThreadLocalData thread_local_data_
        +allocateSlot() SlotPtr
        -removeSlot(index) void
        -runOnAllThreads(cb) void
        -setThreadLocal(index, obj)$ void
    }

    class SlotImpl {
        -InstanceImpl& parent_
        -uint32 index_
        -shared_ptr~bool~ still_alive_guard_
        +get() ThreadLocalObjectSharedPtr
        +set(InitializeCb) void
        +runOnAllThreads(UpdateCb) void
        -wrapCallback(cb) function
        -dataCallback(cb) function
    }

    class ThreadLocalData {
        +Dispatcher* dispatcher_
        +vector~ThreadLocalObjectSharedPtr~ data_
    }

    SlotAllocator <|-- Instance
    Instance <|.. InstanceImpl
    Slot <|.. SlotImpl
    InstanceImpl *-- SlotImpl : nested, owns indices
    InstanceImpl *-- ThreadLocalData : one per thread (static)
    TypedSlot~T~ o-- Slot : wraps SlotPtr
    Slot ..> ThreadLocalObject : stores shared_ptr to
    TypedSlot~T~ ..> ThreadLocalObject : T derives from
```

---

## Smart-pointer & callback aliases

| Alias | Definition | Meaning |
|---|---|---|
| `ThreadLocalObjectSharedPtr` | `std::shared_ptr<ThreadLocalObject>` | what a slot stores per thread |
| `SlotPtr` | `std::unique_ptr<Slot>` | a raw (untyped) slot |
| `TypedSlotPtr<T>` | `std::unique_ptr<TypedSlot<T>>` | a typed slot |
| `Slot::InitializeCb` | `function<ThreadLocalObjectSharedPtr(Dispatcher&)>` | runs per thread, returns object to store |
| `Slot::UpdateCb` | `function<void(ThreadLocalObjectSharedPtr)>` | mutates the existing per-thread object |
| `TypedSlot<T>::InitializeCb` | `function<shared_ptr<T>(Dispatcher&)>` | typed init |
| `TypedSlot<T>::UpdateCb` | `function<void(OptRef<T>)>` | typed update |

---

## The slot-index relationship

```mermaid
classDiagram
    direction LR
    class InstanceImpl {
        vector~Slot*~ slots_
        vector~uint32~ free_slot_indexes_
    }
    class SlotImpl {
        uint32 index_
    }
    class ThreadLocalData {
        vector~shared_ptr~ data_
    }
    InstanceImpl "1" o-- "*" SlotImpl : tracks live slots
    SlotImpl "1" ..> "1 per thread" ThreadLocalData : index_ selects data_[index_]
    note for SlotImpl "index_ is the same number\non every thread; it selects\nthat thread's object"
```

---

## Legend

- `<<interface>>` — pure virtual, under `envoy/thread_local/`.
- `<<template>>` — header-only template (`TypedSlot<T>`).
- `<<base>>` — concrete base class all stored objects derive from.
- `*--` — composition/ownership; `o--` — aggregation; `..>` — uses/references.

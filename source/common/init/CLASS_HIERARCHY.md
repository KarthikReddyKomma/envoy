# Init framework — class hierarchy (UML)

UML-style class diagrams for every type in `envoy/init/` (interfaces) and `source/common/init/`
(implementations).

---

## Interfaces + implementations

```mermaid
classDiagram
    direction TB

    class Manager {
        <<interface>>
        +State state() const
        +add(Target) void
        +initialize(Watcher) void
        +updateWatcher(Watcher) void
        +dumpUnreadyTargets(dumps) void
    }
    class ManagerImpl {
        -string name_
        -State state_
        -uint32_t count_
        -WatcherHandlePtr watcher_handle_
        -WatcherImpl watcher_
        -list~TargetHandlePtr~ target_handles_
        -flat_hash_map~string,uint32~ target_names_count_
        -onTargetReady(name) void
        -ready() void
    }
    Manager <|.. ManagerImpl

    class Target {
        <<interface>>
        +name() string_view
        +createHandle(name) TargetHandlePtr
    }
    class TargetHandle {
        <<interface>>
        +initialize(Watcher) bool
        +name() string_view
    }
    class TargetImpl {
        -string name_
        -WatcherHandlePtr watcher_handle_
        -shared_ptr~InternalInitializeFn~ fn_
        +ready() bool
    }
    class SharedTargetImpl {
        -string name_
        -vector~WatcherHandlePtr~ watcher_handles_
        -shared_ptr~InternalInitializeFn~ fn_
        -bool initialized_
        -once_flag once_flag_
        +ready() bool
    }
    class TargetHandleImpl {
        -string handle_name_
        -string name_
        -weak_ptr~InternalInitializeFn~ fn_
        +initialize(Watcher) bool
    }
    Target <|.. TargetImpl
    Target <|.. SharedTargetImpl
    TargetHandle <|.. TargetHandleImpl

    class Watcher {
        <<interface>>
        +name() string_view
        +createHandle(name) WatcherHandlePtr
    }
    class WatcherHandle {
        <<interface>>
        +ready() bool
    }
    class WatcherImpl {
        -string name_
        -shared_ptr~TargetAwareReadyFn~ fn_
    }
    class WatcherHandleImpl {
        -string handle_name_
        -string name_
        -weak_ptr~TargetAwareReadyFn~ fn_
        +ready() bool
    }
    Watcher <|.. WatcherImpl
    WatcherHandle <|.. WatcherHandleImpl
```

---

## Ownership & reference relationships

Solid = owns (`shared_ptr`/value/`unique_ptr`); dashed = weak reference (`weak_ptr`) created via a handle.

```mermaid
classDiagram
    direction LR

    class ManagerImpl {
        list~TargetHandlePtr~ target_handles_
        WatcherImpl watcher_
        WatcherHandlePtr watcher_handle_
    }
    class TargetImpl {
        shared_ptr~InternalInitializeFn~ fn_
        WatcherHandlePtr watcher_handle_
    }
    class TargetHandleImpl {
        weak_ptr~InternalInitializeFn~ fn_
    }
    class WatcherImpl {
        shared_ptr~TargetAwareReadyFn~ fn_
    }
    class WatcherHandleImpl {
        weak_ptr~TargetAwareReadyFn~ fn_
    }

    ManagerImpl o-- "1" WatcherImpl : owns internal watcher_
    ManagerImpl o-- "*" TargetHandleImpl : owns handles (to targets)
    ManagerImpl o-- "1" WatcherHandleImpl : owns handle (to owner's watcher)

    TargetHandleImpl ..> TargetImpl : weak_ptr to fn_
    TargetImpl o-- "1" WatcherHandleImpl : owns handle (to manager's watcher_)
    WatcherHandleImpl ..> WatcherImpl : weak_ptr to fn_

    note for TargetHandleImpl "lock() fails safely if\nTargetImpl was destroyed"
    note for WatcherHandleImpl "lock() fails safely if\nWatcherImpl was destroyed"
```

---

## Callback type aliases

| Alias | Type | Where | Meaning |
|---|---|---|---|
| `InitializeFn` | `std::function<void()>` | `target_impl.h` | user-supplied "go initialize" body |
| `InternalInitializeFn` | `std::function<void(WatcherHandlePtr)>` | `target_impl.h` | internal wrapper that also captures the manager's watcher handle |
| `ReadyFn` | `std::function<void()>` | `watcher_impl.h` | user-supplied "all done" body |
| `TargetAwareReadyFn` | `std::function<void(absl::string_view)>` | `watcher_impl.h` | "all done" body that also receives the finishing target's name |

`ManagerImpl` always builds its internal `watcher_` with a `TargetAwareReadyFn` so it learns *which* target
finished (used to maintain `target_names_count_`). A plain `WatcherImpl(name, ReadyFn)` simply ignores the
name argument.

---

## Smart-pointer aliases

| Alias | Definition |
|---|---|
| `TargetHandlePtr` | `std::unique_ptr<TargetHandle>` |
| `WatcherHandlePtr` | `std::unique_ptr<WatcherHandle>` |

---

## Legend

- `<<interface>>` — pure-virtual type declared under `envoy/init/`.
- Solid arrow `<|..` — implements interface.
- `o--` — ownership.
- `..>` — weak reference (the "handle" safety mechanism).

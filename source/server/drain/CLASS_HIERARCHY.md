# Drain Manager — Class Hierarchy

UML-style class diagrams for the drain subsystem. Documentation aids, not exhaustive.

## 1. Interface and implementation

```mermaid
classDiagram
    class DrainDecision {
        <<interface>>
        +drainClose(direction) bool
        +addOnDrainCloseCb(direction, cb) CallbackHandlePtr
    }
    class DrainManager {
        <<interface>>
        +startDrainSequence(direction, complete_cb) void
        +draining(direction) bool
        +startParentShutdownSequence() void
        +createChildManager(dispatcher, drain_type) DrainManagerPtr
        +createChildManager(dispatcher) DrainManagerPtr
    }
    class DrainManagerImpl {
        -server_ : Instance&
        -dispatcher_ : Dispatcher&
        -drain_type_ : Listener::DrainType
        -draining_ : atomic~DrainPair~
        -drain_tick_timers_ : map~DrainDirection,TimerPtr~
        -drain_deadlines_ : map~DrainDirection,MonotonicTime~
        -cbs_ : CallbackManager
        -drain_complete_cbs_ : vector~function~
        -children_ : ThreadSafeCallbackManager
        -parent_callback_handle_ : CallbackHandlePtr
        -parent_shutdown_timer_ : TimerPtr
    }

    DrainDecision <|-- DrainManager
    DrainManager <|-- DrainManagerImpl
```

## 2. The drain tree

```mermaid
classDiagram
    class DrainManagerImpl_Root {
        children_ : ThreadSafeCallbackManager
    }
    class DrainManagerImpl_ChildA {
        parent_callback_handle_
    }
    class DrainManagerImpl_ChildB {
        parent_callback_handle_
    }

    DrainManagerImpl_Root o-- DrainManagerImpl_ChildA : createChildManager
    DrainManagerImpl_Root o-- DrainManagerImpl_ChildB : createChildManager
    DrainManagerImpl_Root ..> DrainManagerImpl_ChildA : runCallbacks() cascades drain
    DrainManagerImpl_Root ..> DrainManagerImpl_ChildB : runCallbacks() cascades drain
```

Each child registers a callback in the root's `children_` `ThreadSafeCallbackManager`; the
root's `startDrainSequence` invokes them all, starting each child's own drain sequence.

## 3. Key collaborators

```mermaid
classDiagram
    class DrainManagerImpl
    class Options {
        +drainTime() Duration
        +drainStrategy() DrainStrategy
        +parentShutdownTime() Duration
        +hotRestartDisabled() bool
    }
    class HotRestart {
        +sendParentTerminateRequest() void
    }
    class RandomGenerator {
        +random() uint64
    }
    class Timer

    DrainManagerImpl ..> Options : reads drain settings
    DrainManagerImpl ..> RandomGenerator : salts gradual close
    DrainManagerImpl ..> HotRestart : parent terminate
    DrainManagerImpl *-- Timer : tick + parent shutdown
```

## 4. `DrainPair` state

```mermaid
classDiagram
    class DrainPair {
        +first : bool       %% draining active?
        +second : DrainDirection  %% covered direction
    }
    class DrainDirection {
        <<enum>>
        None
        InboundOnly
        All
    }
    DrainManagerImpl *-- DrainPair : atomic
    DrainPair ..> DrainDirection
```

`drainClose(direction)` and `draining(direction)` both gate on `direction <= draining_.second`,
so the ordered enum encodes "is this traffic covered by the current drain?".

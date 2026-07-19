# Event — class hierarchy (UML)

UML-style Mermaid for the dispatcher, scheduler, and primitives. See [`OVERVIEW.md`](OVERVIEW.md),
[`primitives.md`](primitives.md), and [`lifecycle_and_threading.md`](lifecycle_and_threading.md) for behavior.

---

## Dispatcher & scheduler

```mermaid
classDiagram
    class DispatcherBase {
        <<interface>>
        +post(cb)*
        +isThreadSafe()* bool
    }
    class ScopeTracker {
        <<interface>>
        +pushTrackedObject(obj)*
        +popTrackedObject(obj)*
        +trackedObjectStackIsEmpty()* bool
    }
    class Dispatcher {
        <<interface>>
        +createFileEvent(fd, cb, trigger, events)*
        +createTimer(cb)*
        +createScaledTimer(...)*
        +createSchedulableCallback(cb)*
        +createServerConnection(...)* / createClientConnection(...)*
        +createFilesystemWatcher()*
        +deferredDelete(obj)*
        +deleteInDispatcherThread(obj)*
        +listenForSignal(num, cb)*
        +registerWatchdog(...)*
        +run(RunType)* / exit()* / shutdown()*
        +timeSource()* / approximateMonotonicTime()*
    }
    class DispatcherImpl {
        -base_scheduler_ : LibeventScheduler
        -scheduler_ : SchedulerPtr
        -post_callbacks_ / post_lock_
        -to_delete_1_ / to_delete_2_ / current_to_delete_
        -tracked_object_stack_
        -run_tid_ : ThreadId
        -scaled_timer_manager_
    }
    class FatalErrorHandlerInterface {
        <<interface>>
        +onFatalError(os)*
        +runFatalActionsOnTrackedObject(actions)*
    }

    DispatcherBase <|-- Dispatcher
    ScopeTracker <|-- Dispatcher
    Dispatcher <|.. DispatcherImpl
    FatalErrorHandlerInterface <|.. DispatcherImpl
    DispatcherImpl *-- LibeventScheduler

    note for DispatcherImpl "one per thread\nfactory + event loop\nASSERT(isThreadSafe) everywhere"
```

```mermaid
classDiagram
    class Scheduler {
        <<interface>>
        +createTimer(cb, dispatcher)*
    }
    class CallbackScheduler {
        <<interface>>
        +createSchedulableCallback(cb)*
    }
    class LibeventScheduler {
        -libevent_ : Libevent::BasePtr
        -stats_ : DispatcherStats*
        +run(RunType) / loopExit()
        +base() event_base&
        +registerOnPrepareCallback(cb) / registerOnCheckCallback(cb)
        +initializeStats(stats)
    }
    class TimeSystem {
        <<interface>>
        +createScheduler(base, cb_sched)*
    }
    class RealTimeSystem {
        -time_source_ : RealTimeSource
        +systemTime() / monotonicTime()
    }

    Scheduler <|.. LibeventScheduler
    CallbackScheduler <|.. LibeventScheduler
    TimeSystem <|.. RealTimeSystem
    RealTimeSystem ..> LibeventScheduler : createScheduler

    note for LibeventScheduler "owns event_base\nprepare/check hooks → loop stats"
```

---

## Event primitives

```mermaid
classDiagram
    class ImplBase {
        #raw_event_ : event
    }

    class FileEvent {
        <<interface>>
        +activate(events)*
        +setEnabled(events)*
        +registerEventIfEmulatedEdge(e)* / unregisterEventIfEmulatedEdge(e)*
    }
    class FileEventImpl {
        -fd_ / trigger_ / enabled_events_
        -injected_activation_events_
        -activation_cb_ : SchedulableCallbackPtr
        -assignEvents() / updateEvents() / mergeInjectedEventsAndRunCb()
    }

    class Timer {
        <<interface>>
        +enableTimer(ms, scope)* / enableHRTimer(us, scope)*
        +disableTimer()* / enabled()*
    }
    class TimerImpl {
        -cb_ / dispatcher_
        -object_ : atomic~ScopeTrackedObject*~
        -internalEnableTimer(tv, scope)
    }
    class TimerUtils {
        +durationToTimeval(d, tv)$
    }

    class SchedulableCallback {
        <<interface>>
        +scheduleCallbackCurrentIteration()*
        +scheduleCallbackNextIteration()*
        +cancel()* / enabled()*
    }
    class SchedulableCallbackImpl {
        -cb_
    }

    class SignalEvent { <<interface>> }
    class SignalEventImpl

    ImplBase <|-- FileEventImpl
    ImplBase <|-- TimerImpl
    ImplBase <|-- SchedulableCallbackImpl
    ImplBase <|-- SignalEventImpl
    FileEvent <|.. FileEventImpl
    Timer <|.. TimerImpl
    SchedulableCallback <|.. SchedulableCallbackImpl
    SignalEvent <|.. SignalEventImpl
    FileEventImpl ..> SchedulableCallbackImpl : activation_cb_
    TimerImpl ..> TimerUtils

    note for ImplBase "embeds the libevent event struct"
    note for FileEventImpl "EV_PERSIST; edge/level/emulated-edge"
```

---

## Scaled timers

```mermaid
classDiagram
    class ScaledRangeTimerManager { <<interface>> }
    class ScaledRangeTimerManagerImpl {
        -queues_ : set~Queue~
        -scale_factor_ : UnitFloat
        +createTimer(min, cb) / createTimer(type, cb)
        +setScaleFactor(f)
    }
    class Queue {
        +duration_ : ms (max-min)
        +range_timers_ : list~Item~
        +timer_ : TimerPtr (one real timer for head)
    }
    class RangeTimerImpl

    ScaledRangeTimerManager <|.. ScaledRangeTimerManagerImpl
    ScaledRangeTimerManagerImpl *-- Queue
    Queue *-- RangeTimerImpl : tracks
    RangeTimerImpl ..|> Timer

    note for ScaledRangeTimerManagerImpl "N timers per range → 1 real libevent timer"
```

---

## Deferred deletion & cross-thread

```mermaid
classDiagram
    class DeferredDeletable { <<interface>> +deleteIsPending() }
    class DispatcherThreadDeletable { <<interface>> }
    class DeferredTaskUtil {
        +deferredRun(dispatcher, fn)$
    }
    class DeferredTask {
        -task_
        +~DeferredTask() runs task_
    }

    DeferredDeletable <|.. DeferredTask
    DeferredTaskUtil ..> DeferredTask : wraps fn
    DeferredTaskUtil ..> DeferredDeletable : via deferredDelete

    note for DeferredTask "task runs in destructor →\nordered with deferred deletes"
```

---

## Type & alias reference

| Symbol | Meaning |
|---|---|
| `DispatcherPtr` | `unique_ptr<Dispatcher>` — one per thread. |
| `FileEventPtr` / `TimerPtr` / `SchedulableCallbackPtr` / `SignalEventPtr` | owning handles to primitives. |
| `PostCb` | `absl::AnyInvocable<void()>` — a posted callback. |
| `FileReadyCb` | `function<Status(uint32_t events)>` — fd-event callback. |
| `TimerCb` | `function<void()>` — timer callback. |
| `Libevent::BasePtr` | `CSmartPtr<event_base, event_base_free>` — RAII `event_base`. |
| `RunType` | `Block` / `NonBlock` / `RunUntilExit`. |

---

## Relationship summary

| Relationship | Type | Meaning |
|---|---|---|
| `DispatcherImpl` → `LibeventScheduler` | composition | Owns the `event_base` + loop. |
| `DispatcherImpl` → primitives | factory | Creates file events / timers / callbacks / connections. |
| `*Impl` → `ImplBase` | inheritance | Embeds the raw libevent `event`. |
| `FileEventImpl` → `SchedulableCallbackImpl` | composition | `activate()` uses a next-iteration callback. |
| `RealTimeSystem` → `LibeventScheduler` | factory | `createScheduler` (swap for SimulatedTime in tests). |
| `ScaledRangeTimerManagerImpl` → `Queue` → real `Timer` | composition | Load-scaled timers share one real timer per range. |
| `DeferredTaskUtil` → `deferredDelete` | reuse | Run a task after pending deletions. |

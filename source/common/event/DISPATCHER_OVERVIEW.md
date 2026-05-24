# Event Loop & Dispatcher — Overview

A high-level guided tour of `source/common/event/`: the **event loop**, the **dispatcher**, and the primitives they expose. The companion document `EVENT_LOOP_ARCHITECTURE.md` covers internals in depth; this one stays at the conceptual level with mermaid diagrams.

For "who uses the dispatcher and how", see `DISPATCHER_USAGE.md`.

---

## 1. What Lives Here

```mermaid
graph TB
    subgraph "Public Interfaces (envoy/event/*.h)"
        I_Disp[Dispatcher]
        I_Tim[Timer]
        I_FE[FileEvent]
        I_Sched[SchedulableCallback]
        I_Sig[SignalEvent]
        I_DD[DeferredDeletable]
    end

    subgraph "source/common/event implementations"
        D[DispatcherImpl<br/>main orchestrator]
        LS[LibeventScheduler<br/>libevent base wrapper]
        T[TimerImpl<br/>ms / us timers]
        FE[FileEventImpl<br/>fd readiness]
        SC[SchedulableCallbackImpl<br/>this/next iteration cb]
        SR[ScaledRangeTimerManagerImpl<br/>scaled timers]
        DT[DeferredTaskUtil<br/>run after deletions]
        RTS[RealTimeSystem<br/>monotonic clock]
        EB[event_impl_base<br/>libevent shim base]
    end

    I_Disp -.implemented by.-> D
    I_Tim -.-> T
    I_FE -.-> FE
    I_Sched -.-> SC
    I_DD -.helper.-> DT

    D --> LS
    D --> SR
    D --> RTS
    LS --> T
    LS --> FE
    LS --> SC
    T --> EB
    FE --> EB
    SC --> EB

    style D fill:#e1f5ff
    style LS fill:#fff9c4
    style EB fill:#cccccc
```

| File | Role |
|------|------|
| `dispatcher_impl.{h,cc}` | `DispatcherImpl` — the public per-thread event loop API |
| `libevent_scheduler.{h,cc}` | Wraps libevent `event_base`, exposes prepare/check hooks, run/exit |
| `libevent.{h,cc}` | Type-safe ownership wrappers for libevent C structs |
| `timer_impl.{h,cc}` | `TimerImpl` — duration → `timeval` clamping, scope tracking |
| `file_event_impl.{h,cc}` | `FileEventImpl` — fd read/write/closed monitoring, edge-trigger emulation |
| `schedulable_cb_impl.{h,cc}` | `SchedulableCallbackImpl` — current vs next iteration callbacks |
| `scaled_range_timer_manager_impl.{h,cc}` | Time-scaled timers driven by overload manager |
| `event_impl_base.{h,cc}` | RAII base class for libevent event registration |
| `deferred_task.h` | `DeferredTaskUtil::deferredRun` helper |
| `real_time_system.{h,cc}` | Default `TimeSystem` using real wall/monotonic clocks |
| `posix/`, `win32/` | Platform-specific signal handling |

---

## 2. Conceptual Model

```mermaid
graph LR
    subgraph "Thread"
        D[Dispatcher]
        D --> EL[Event Loop]
    end

    subgraph "Inputs handled per iteration"
        FD[fd readiness<br/>read/write/close]
        TM[timer expiry]
        ZT[zero-delay timers<br/>activate / scheduleNextIter]
        PC[post callbacks<br/>cross-thread]
        DD[deferredDelete<br/>safe deletion]
        SI[scheduleCurrentIter<br/>same-iter cb]
        SIG[signals<br/>SIGTERM, …]
    end

    EL --> FD
    EL --> TM
    EL --> ZT
    EL --> PC
    EL --> DD
    EL --> SI
    EL --> SIG

    style D fill:#e1f5ff
    style EL fill:#c8e6c9
```

- **One dispatcher per thread.** Worker threads, the main thread, and admin all have their own.
- **Single-threaded execution model inside one dispatcher.** Almost every method assumes the caller is the run thread; `post()` is the lone exception.
- **Time, I/O, and scheduled work share the same loop.** No separate timer thread, no I/O thread pool.

---

## 3. The Event Loop Cycle

```mermaid
sequenceDiagram
    autonumber
    participant App
    participant D as DispatcherImpl
    participant LS as LibeventScheduler
    participant K as Kernel<br/>(epoll/kqueue)
    participant W as Work List

    App->>D: run(RunUntilExit / Block / NonBlock)
    D->>LS: base_scheduler_.run(mode)
    loop until exit
        LS->>LS: 1) compute poll timeout<br/>(closest timer deadline)
        LS->>LS: 2) on-prepare callbacks<br/>(stats, time)
        LS->>K: 3) epoll_wait / kevent (BLOCKS)
        K-->>LS: ready fds + timeout fired
        LS->>D: 4) on-check callbacks<br/>(updateApproximateMonotonicTime)
        LS->>LS: 5) move expired timers → work list
        LS->>W: 6) drain work list
        Note right of W: order:<br/>fd events →<br/>timers / activate / nextIter →<br/>same-iter (post / dD / currentIter)
        LS->>D: touchWatchdog after each item
    end
    LS-->>D: loopExit (when exit() / RunType ends)
    D-->>App: run returns
```

Highlights:
- **Step 3 is the only place the loop blocks.** Everything else is synchronous CPU work.
- **Step 6 can grow during execution** — same-iteration mechanisms (`post`, `deferredDelete`, `scheduleCallbackCurrentIteration`) add to the list while it is being drained.
- **No ordering guarantees** between `post`, `deferredDelete`, and `scheduleCallbackCurrentIteration`. Use `DeferredTaskUtil::deferredRun` if you must run after a deletion.

---

## 4. DispatcherImpl Construction

```mermaid
flowchart TD
    A[DispatcherImpl ctor] --> B[Pick concrete deps]
    B --> C[name_, thread_factory_, time_source_, file_system_]
    B --> D[buffer_factory_<br/>WatermarkBufferFactory by default]
    B --> E[base_scheduler_ = LibeventScheduler]
    B --> F[scheduler_ = TimeSystem.createScheduler]

    A --> G[Schedulable callbacks created up front]
    G --> G1[thread_local_delete_cb_<br/>runThreadLocalDelete]
    G --> G2[deferred_delete_cb_<br/>clearDeferredDeleteList]
    G --> G3[post_cb_<br/>runPostCallbacks]

    A --> H[scaled_timer_manager_<br/>ScaledRangeTimerManagerImpl]
    A --> I[Register fatal error handler]
    A --> J[updateApproximateMonotonicTimeInternal]
    A --> K[register OnCheckCallback to update time each iter]

    style E fill:#fff9c4
    style G fill:#e1f5ff
```

The three pre-built `SchedulableCallback`s — `deferred_delete_cb_`, `post_cb_`, `thread_local_delete_cb_` — are how the dispatcher *itself* gets work onto the loop.

---

## 5. Event Primitives (At a Glance)

```mermaid
graph TB
    subgraph "Timer"
        T1[createTimer cb]
        T2[enableTimer ms / enableHRTimer us]
        T3[disableTimer]
        T1 --> Tx[Timer fires once<br/>cb runs in event loop]
    end

    subgraph "FileEvent"
        F1[createFileEvent fd, cb, trigger, events]
        F2[setEnabled new mask]
        F3[activate manual trigger via 0-delay timer]
        F1 --> Fx[cb invoked when fd is read/write/closed]
    end

    subgraph "SchedulableCallback"
        S1[createSchedulableCallback cb]
        S2[scheduleCallbackCurrentIteration]
        S3[scheduleCallbackNextIteration]
        S1 --> Sx[cb runs in current or next loop pass]
    end

    subgraph "Deferred Deletion"
        D1[deferredDelete obj]
        D2[DeferredTaskUtil::deferredRun fn]
        D1 --> Dx[obj destroyed after current call stack unwinds]
        D2 --> Dx
    end

    subgraph "Cross-thread"
        P1[post cb]
        P2[deleteInDispatcherThread obj]
        P1 --> Px[cb runs on this dispatcher's thread]
        P2 --> Px
    end

    style Tx fill:#c8e6c9
    style Fx fill:#c8e6c9
    style Sx fill:#c8e6c9
    style Dx fill:#c8e6c9
    style Px fill:#c8e6c9
```

Selection guide:

| Need | Reach for |
|------|-----------|
| One-shot delay | `Timer` |
| Wait for fd to be ready | `FileEvent` |
| Run "soon, not now" without delay | `SchedulableCallback::scheduleCallbackCurrentIteration` |
| Yield to I/O before continuing | `SchedulableCallback::scheduleCallbackNextIteration` |
| Hand work to another thread's loop | `Dispatcher::post` |
| Delete `this` safely from a callback | `Dispatcher::deferredDelete` |
| Run code *after* deferred deletions | `DeferredTaskUtil::deferredRun` |
| Time-scaled timeouts under overload | `Dispatcher::createScaledTimer` |

---

## 6. Deferred Deletion (Double-Buffered)

```mermaid
graph LR
    A[Caller in event cb] -->|deferredDelete obj| B[current_to_delete_<br/>= to_delete_1_]
    B --> C[deferred_delete_cb_->scheduleCallbackCurrentIteration]
    C --> D[Loop reaches same-iter cb]
    D --> E[clearDeferredDeleteList]
    E --> F[Swap: current_to_delete_ → to_delete_2_]
    F --> G[Iterate to_delete_1 in FIFO,<br/>reset() each → destructors run]
    G --> H{New deferredDelete<br/>during destruction?}
    H -- yes --> I[Goes into to_delete_2_<br/>handled in next iteration]
    H -- no --> J[Done]

    style B fill:#fff9c4
    style F fill:#e1f5ff
    style I fill:#ffe1e1
```

Why double-buffer? A destructor may itself call `deferredDelete`. Without the swap, you would mutate the vector you are iterating.

---

## 7. Cross-Thread Post

```mermaid
sequenceDiagram
    participant T1 as Thread A
    participant L as post_lock_
    participant Q as post_callbacks_
    participant CB as post_cb_
    participant T2 as Thread B (dispatcher's run thread)
    participant EL as Event Loop

    T1->>L: lock
    T1->>Q: push_back(cb)
    Note right of T1: do_post = (queue was empty)
    T1->>L: unlock
    alt do_post
        T1->>CB: scheduleCallbackCurrentIteration<br/>(thread-safe libevent call)
    end

    EL->>CB: fires on T2's loop
    EL->>L: lock
    EL->>Q: std::move(post_callbacks_)
    EL->>L: unlock
    EL->>T2: run callbacks one by one
```

Properties:
- **Lock held only around the queue swap** — callbacks run lock-free.
- **First post wakes the loop** (`do_post = queue.empty()`); subsequent posts piggyback.
- **FIFO** within a single batch.
- **Cross-thread safe** because `scheduleCallbackCurrentIteration` is the only Dispatcher method documented as thread-safe.

---

## 8. Run Modes

```mermaid
stateDiagram-v2
    [*] --> Idle: ctor
    Idle --> Running: run(mode)
    Running --> Idle: exit() or NonBlock returns
    Running --> [*]: dtor

    note right of Running
      RunType:
      • NonBlock — drain pending work, return
      • Block — block once, process, return
      • RunUntilExit — loop until exit()
    end note
```

`RunType` maps onto libevent flags:
- `NonBlock` → `EVLOOP_NONBLOCK` (+ `EVLOOP_ONCE` on level-triggered platforms to avoid live-lock).
- `Block` → standard one-shot wait.
- `RunUntilExit` → continuous loop until `loopExit()`.

---

## 9. Time Management

```mermaid
graph LR
    subgraph "Per-iteration time refresh"
        A[on-check callback registered in ctor]
        A --> B[updateApproximateMonotonicTime]
        B --> C[approximate_monotonic_time_ = time_source_.monotonicTime]
    end

    subgraph "Reads"
        R1[approximateMonotonicTime] --> X[returns cached value]
        R2[updateApproximateMonotonicTime + read] --> Y[forces refresh]
    end

    style C fill:#c8e6c9
```

- All callbacks within one loop iteration see **the same** approximate time, because the on-check hook runs after `epoll_wait` and before work-list draining.
- Skips the `clock_gettime` syscall on every read — important for hot paths.
- For scenarios that need a fresh stamp (e.g. measuring CPU time), call `updateApproximateMonotonicTime()` explicitly.

---

## 10. Watchdog Integration

```mermaid
graph TB
    A[Worker thread health is monitored by GuardDog]
    A --> B[registerWatchdog watchdog, interval]
    B --> C[Create touch_timer_]
    C --> D[Timer fires every interval]
    D --> E[watchdog->touch]
    D --> F[re-arm touch_timer_]

    G[After every fd cb / timer / sched cb / post / dD]
    G --> H[touchWatchdog]
    H --> I{watchdog_registration_?}
    I -- yes --> J[watchdog->touch]
    I -- no --> K[no-op]

    style E fill:#c8e6c9
    style J fill:#c8e6c9
```

Two complementary mechanisms:
- **Periodic timer** (every `min_touch_interval`) covers the case where the loop is otherwise idle.
- **Per-callback `touchWatchdog`** keeps a busy loop visible as alive even if it never reaches its periodic timer.

---

## 11. Crash Diagnostics — Tracked Object Stack

```mermaid
sequenceDiagram
    participant Filter
    participant ST as ScopeTracker (RAII)
    participant D as Dispatcher
    participant Stack as tracked_object_stack_
    participant Crash as Fatal Handler

    Filter->>ST: ScopeTracker tracker(*this, dispatcher_)
    ST->>D: pushTrackedObject(this)
    D->>Stack: push_back

    Filter->>Filter: process work
    Note over Filter: SEGFAULT here…

    Crash->>D: onFatalError(os)
    D->>Stack: iterate
    D->>Filter: dumpState(os)

    Filter->>ST: tracker dtor
    ST->>D: popTrackedObject(this)
    D->>Stack: pop_back (assert top == this)
```

Why it matters: Envoy is single-threaded per worker, so a crash in any callback comes with a chain of "what was this thread doing" objects ready to dump structured state.

---

## 12. Lifecycle of a Single Worker

```mermaid
sequenceDiagram
    autonumber
    participant Bootstrap
    participant W as Worker thread
    participant D as DispatcherImpl
    participant L as Listeners + Conns
    participant K as Kernel

    Bootstrap->>W: spawn thread
    W->>D: api.allocateDispatcher(name)
    Bootstrap->>D: createListener / addCluster / …
    Bootstrap->>D: registerWatchdog(...)
    Bootstrap->>D: post(initialize stats)
    W->>D: run(RunType::RunUntilExit)
    loop event loop
        D->>K: epoll_wait
        K-->>D: ready fds / signals
        D->>L: invoke fd / timer / post callbacks
        L->>D: createTimer / deferredDelete / post
    end
    Bootstrap->>D: exit()  (during shutdown)
    D->>W: run() returns
    W->>D: dtor
    W->>Bootstrap: thread joins
```

This pattern is replicated for every worker thread and (in a slightly trimmed form) for the main and admin threads.

---

## 13. Where to Go Next

- **`EVENT_LOOP_ARCHITECTURE.md`** — deep dive into ordering rules, edge-trigger emulation, deferred-delete double buffering, time clamping, and testing patterns.
- **`DISPATCHER_USAGE.md`** — how connections, conn pools, routers, cluster manager, TLS, and other modules use the dispatcher's primitives.
- **`include/envoy/event/dispatcher.h`** — the public interface.
- **`source/common/event/dispatcher_impl.cc`** — the canonical reference implementation.


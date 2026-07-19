# Signal handling — class hierarchy (UML)

UML-style Mermaid for the signal/fatal-error types. See [`OVERVIEW.md`](OVERVIEW.md) for behavior.

---

## Core types

```mermaid
classDiagram
    class SignalAction {
        -guard_size_ / altstack_size_
        -altstack_ : char*
        -previous_handlers_ : sigaction[]
        +sigHandler(sig, info, ctx)$
        -mapAndProtectStackMemory()
        -installSigHandlers() / removeSigHandlers()
    }
    note for SignalAction "RAII: ctor installs, dtor restores\nFATAL_SIGS = {ABRT,BUS,FPE,ILL,SEGV}\nruns handler on guard-paged alt stack"

    class FatalErrorHandlerInterface {
        <<interface>>
        +onFatalError(os)*
        +runFatalActionsOnTrackedObject(actions)*
    }
    class DispatcherImpl
    FatalErrorHandlerInterface <|.. DispatcherImpl
    note for FatalErrorHandlerInterface "must be async-signal-safe\n(no allocation)"
```

---

## The registry (free functions + atomics)

```mermaid
classDiagram
    class FatalErrorHandler {
        <<namespace>>
        +registerFatalErrorHandler(h)$
        +removeFatalErrorHandler(h)$
        +callFatalErrorHandlers(os)$
        +registerFatalActions(safe, unsafe, tf)$
        +runSafeActions()$ Status
        +runUnsafeActions()$ Status
        +clearFatalActionsOnTerminate()$
    }
    class fatal_error_handlers {
        <<atomic ptr>>
        FailureFunctionList*
    }
    class fatal_action_manager {
        <<atomic ptr>>
        FatalActionManager*
    }
    class failure_tid {
        <<atomic int64>>
        winning thread id
    }
    FatalErrorHandler ..> fatal_error_handlers : exchange/store
    FatalErrorHandler ..> fatal_action_manager : exchange
    FatalErrorHandler ..> failure_tid : compare_exchange
```

---

## Fatal actions

```mermaid
classDiagram
    class FatalActionManager {
        -safe_actions_ : FatalActionPtrList
        -unsafe_actions_ : FatalActionPtrList
        -thread_factory_
        +getSafeActions() / getUnsafeActions()
    }
    class Status {
        <<enum>>
        Success
        ActionManagerUnset
        RunningOnAnotherThread
        AlreadyRanOnThisThread
    }
    class FatalActionPtr { <<extension point>> }
    FatalActionManager o-- FatalActionPtr : safe + unsafe
    FatalErrorHandler ..> Status : returns
    FatalErrorHandler ..> FatalActionManager
```

---

## Relationship summary

| Relationship | Type | Meaning |
|---|---|---|
| `SignalAction` (RAII) | lifecycle | Installs/removes handlers + alt stack. |
| `DispatcherImpl` → `FatalErrorHandlerInterface` | implements | Dumps tracked objects on crash. |
| `FatalErrorHandler` → `fatal_error_handlers` (atomic) | lock-free access | Read handler list during a crash. |
| `FatalErrorHandler` → `failure_tid` (atomic) | election | Only one thread runs fatal actions. |
| `FatalActionManager` → `FatalActionPtr` | composition | Safe + unsafe action buckets from extensions. |
| `runSafeActions/runUnsafeActions` → `Status` | result | Tells caller how to proceed. |

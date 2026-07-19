# Api — class hierarchy (UML)

UML-style Mermaid for the `Api::Api` facade and the `OsSysCalls` wrapper. See [`OVERVIEW.md`](OVERVIEW.md) for
behavior.

---

## The capability facade

```mermaid
classDiagram
    class Api {
        <<interface>>
        +allocateDispatcher(name)* DispatcherPtr
        +allocateDispatcher(name, scaledTimerFactory)* DispatcherPtr
        +allocateDispatcher(name, watermarkFactory)* DispatcherPtr
        +threadFactory()* ThreadFactory&
        +fileSystem()* Filesystem::Instance&
        +timeSource()* TimeSource&
        +rootScope()* Stats::Scope&
        +randomGenerator()* RandomGenerator&
        +bootstrap()* const Bootstrap&
        +processContext()* ProcessContextOptRef
        +customStatNamespaces()* CustomStatNamespaces&
    }
    class Impl {
        -thread_factory_ / store_ / time_system_
        -file_system_ / random_generator_ / bootstrap_
        -custom_stat_namespaces_ / process_context_
        -watermark_factory_
    }
    Api <|.. Impl
    Impl ..> DispatcherImpl : allocateDispatcher creates

    note for Impl "holds references only;\nthe one object threaded\nthrough the codebase"
```

---

## The OS syscall wrapper

```mermaid
classDiagram
    class OsSysCalls {
        <<interface>>
        +socket/bind/connect/listen/accept(...)* SysCall*Result
        +readv/writev/pread/pwrite(...)* SysCallSizeResult
        +recv/recvmsg/recvmmsg/send/sendmsg(...)* SysCall*Result
        +setsockopt/getsockopt/ioctl(...)* SysCallIntResult
        +mmap/ftruncate/stat/fstat/open/unlink(...)* SysCall*Result
        +supportsMmsg/UdpGro/UdpGso/IpTransparent/Mptcp()* bool
    }
    class OsSysCallsImpl_posix
    class OsSysCallsImpl_win32
    class LinuxOsSysCalls { <<interface>> }

    OsSysCalls <|.. OsSysCallsImpl_posix
    OsSysCalls <|.. OsSysCallsImpl_win32
    OsSysCallsImpl_posix ..> LinuxOsSysCalls : Linux extras

    note for OsSysCallsImpl_posix "selected at BUILD time;\nexposed via ThreadSafeSingleton\n(OsSysCallsSingleton)"
```

---

## The result type

```mermaid
classDiagram
    class SysCallResult~T~ {
        +return_value_ : T
        +errno_ : int
    }
    class SysCallIntResult { T = int }
    class SysCallSizeResult { T = ssize_t }
    class SysCallSocketResult { T = os_fd_t }
    class SysCallPtrResult { T = void* }
    class SysCallBoolResult { T = bool }

    SysCallResult <|.. SysCallIntResult
    SysCallResult <|.. SysCallSizeResult
    SysCallResult <|.. SysCallSocketResult
    SysCallResult <|.. SysCallPtrResult
    SysCallResult <|.. SysCallBoolResult

    note for SysCallResult "value + errno captured together\n→ no errno-clobber bugs"
```

---

## Relationship summary

| Relationship | Type | Meaning |
|---|---|---|
| `Impl` → `Api` | implements | The concrete facade. |
| `Impl` → `DispatcherImpl` | factory | `allocateDispatcher` builds per-thread loops. |
| `OsSysCallsImpl_*` → `OsSysCalls` | implements | Per-platform syscall bindings (build-time selected). |
| `OsSysCallsImpl_posix` → `LinuxOsSysCalls` | extends | Linux-only syscalls. |
| `SysCall*Result` → `SysCallResult<T>` | alias | Typed result carrying value + errno. |
| `OsSysCallsSingleton` → `OsSysCallsImpl` | singleton | Global access point (swappable in tests). |

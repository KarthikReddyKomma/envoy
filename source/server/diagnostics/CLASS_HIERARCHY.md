# Diagnostics & Process Support — Class Hierarchy

UML-style class diagrams for the diagnostics utilities. Documentation aids, not exhaustive.

## 1. `BackwardsTrace`

```mermaid
classDiagram
    class BackwardsTrace {
        -stack_trace_ : void*[64]
        -stack_depth_ : int
        +capture() void
        +captureFrom(context) void
        +logTrace() void
        +printTrace(os) void
        +logFault(signame, addr) void
        +addrMapping(setup)$ string_view
        +setLogToStderr(bool)$ void
        -visitTrace(visitor) void
    }
    class absl_debugging {
        <<library>>
        GetStackTrace()
        GetStackTraceWithContext()
        Symbolize()
    }
    BackwardsTrace ..> absl_debugging : uses
```

### Crash-path collaborators

```mermaid
classDiagram
    class SignalAction {
        +sigHandler(sig, info, context)$ void
    }
    class TerminateHandler {
        +logOnTerminate() function
    }
    class BackwardsTrace
    SignalAction ..> BackwardsTrace : captureFrom + logTrace
    TerminateHandler ..> BackwardsTrace : BACKTRACE_LOG()
```

## 2. cgroup CPU detection

```mermaid
classDiagram
    class CgroupDetector {
        <<interface>>
        +getCpuLimit(fs) optional~uint32~
    }
    class CgroupDetectorImpl {
        +getCpuLimit(fs) optional~uint32~
    }
    class CgroupCpuUtil {
        +getCpuLimit(fs)$ optional~uint32~
        -discoverCgroupMount(fs)$
        -getCurrentCgroupPath(fs)$
        -accessCgroupV1Files(...)$
        -accessCgroupV2Files(...)$
        -readActualLimitsV1(...)$
        -readActualLimitsV2(...)$
    }
    class OptionsImplPlatform {
        +getCpuCount()$ uint32
    }

    CgroupDetector <|-- CgroupDetectorImpl
    CgroupDetectorImpl ..> CgroupCpuUtil : delegates
    OptionsImplPlatform ..> CgroupDetector : via CgroupDetectorSingleton
```

`CgroupDetectorSingleton = ThreadSafeSingleton<CgroupDetectorImpl>`.

## 3. Regex engine wiring

```mermaid
classDiagram
    class Engine { <<interface>> }
    class GoogleReEngine {
        note: default (RE2)
    }
    class EngineFactory {
        <<interface>>
        +createEngine(config, ctx) EnginePtr
    }
    class createRegexEngine {
        <<free function>>
        +createRegexEngine(bootstrap, visitor, server_ctx) EnginePtr
    }
    class InstanceBase {
        -regex_engine_ : EnginePtr
        +regexEngine() Engine&
    }

    Engine <|-- GoogleReEngine
    EngineFactory ..> Engine : creates
    createRegexEngine ..> EngineFactory : if configured
    createRegexEngine ..> GoogleReEngine : default
    InstanceBase ..> createRegexEngine : at initialize()
    InstanceBase o-- Engine : owns regex_engine_
```

## 4. Proto descriptor check & startup helpers (free functions)

```mermaid
classDiagram
    class proto_descriptors {
        <<free function>>
        +validateProtoDescriptors() void
        note: RELEASE_ASSERT each xDS method + type
    }
    class ProcessWide {
        note: ctor calls validateProtoDescriptors()<br/>(ENVOY_ENABLE_FULL_PROTOS)
    }
    class Server_Utility {
        <<namespace>>
        +serverState(init_state, hc_failed) State
        +assertExclusiveLogFormatMethod(opts, cfg) Status
        +maybeSetApplicationLogFormat(cfg) Status
    }
    ProcessWide ..> proto_descriptors : invokes
```

These are namespaced free functions rather than classes — they hold no state, just perform a
startup check or a one-shot mapping/format operation.

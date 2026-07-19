# Command-Line Options — Class Hierarchy

UML-style class diagrams for the options subsystem. Documentation aids, not exhaustive.

## 1. Interface, storage, and parser

```mermaid
classDiagram
    class Options {
        <<interface>>
        +configPath() string
        +configYaml() string
        +configProto() Bootstrap&
        +concurrency() uint32
        +mode() Mode
        +restartEpoch() uint64
        +baseId() uint64
        +drainTime() Duration
        +drainStrategy() DrainStrategy
        +logLevel() level
        +toCommandLineOptions() CommandLineOptionsPtr
    }

    class OptionsImplBase {
        #concurrency_ : uint32 = 1
        #mode_ : Mode = Serve
        #drain_time_ : sec = 600
        #parent_shutdown_time_ : sec = 900
        #socket_path_ : string = "@envoy_domain_socket"
        #signal_handling_enabled_ : bool = true
        +setConcurrency(uint32) void
        +setConfigProto(Bootstrap) void
        +setMode(Mode) void
        +setLogLevel(string) Status
        +disableExtensions(list) void
        +count() uint32
    }

    class OptionsImpl {
        +OptionsImpl(argc, argv, hot_restart_cb, level)
        +OptionsImpl(vector~string~, hot_restart_cb, level)
        +OptionsImpl(cluster, node, zone, level)
        -parseComponentLogLevels(string) void
    }

    Options <|-- OptionsImplBase
    OptionsImplBase <|-- OptionsImpl
    OptionsImpl ..> OptionsImplBase : friend writes private fields
```

`OptionsImplBase` is fully usable on its own (Envoy Mobile path). `OptionsImpl` only adds the
TCLAP front-end.

## 2. Exceptions and enums

```mermaid
classDiagram
    class NoServingException {
        note: --help / --version -> exit(0)
    }
    class MalformedArgvException {
        note: bad flag -> exit(1)
    }
    class Mode {
        <<enum>>
        Serve
        Validate
        InitOnly
    }
    class DrainStrategy {
        <<enum>>
        Gradual
        Immediate
    }
    OptionsImpl ..> NoServingException : throws
    OptionsImpl ..> MalformedArgvException : throws
    Options ..> Mode
    Options ..> DrainStrategy
```

## 3. The platform seam

```mermaid
classDiagram
    class OptionsImplPlatform {
        <<static>>
        +getCpuCount()$ uint32
    }
    class OptionsImplPlatformLinux {
        +getCpuAffinityCount(hw_threads)$ uint32
        note: min(hw, affinity, cgroup_limit)
    }
    class CgroupDetector {
        <<interface>>
        +getCpuLimit(fs) optional~uint32~
    }
    class CgroupCpuUtil {
        +getCpuLimit(fs)$ optional~uint32~
        note: cgroup v1 (cfs_quota/period)<br/>+ v2 (cpu.max)
    }

    OptionsImplPlatform <.. OptionsImplPlatformLinux : Linux build
    OptionsImplPlatformLinux ..> CgroupDetector : uses
    CgroupDetector <|-- CgroupCpuUtil
    OptionsImpl ..> OptionsImplPlatform : getCpuCount() on cpuset path
```

The two `getCpuCount()` bodies (`options_impl_platform_default.cc` vs.
`options_impl_platform_linux.cc`) are selected by the build system, so the rest of the code
stays `#ifdef`-free.

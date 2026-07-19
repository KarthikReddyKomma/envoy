# Command-Line Options

> Documentation for Envoy's CLI options parsing.
> Source lives in `source/server/options_impl.{h,cc}`, `source/server/options_impl_base.{h,cc}`,
> `source/server/options_impl_platform.h`, `source/server/options_impl_platform_default.cc`,
> and `source/server/options_impl_platform_linux.{h,cc}`. The interface is
> `envoy/server/options.h`.

This subsystem turns `argc`/`argv` into the **read-only `Options` object** that the rest of
the server consults at runtime: where the config comes from, how many worker threads to run,
hot-restart parameters, drain timing, logging, and which top-level **mode** to run in
(serve / validate / init-only).

## The split: storage vs. parsing

There are two layers, deliberately separated:

| Layer | Class | Responsibility |
|-------|-------|----------------|
| Storage | `OptionsImplBase` | Holds every parsed field with sane defaults; implements all `Options` accessors. **No CLI dependency.** |
| Parsing | `OptionsImpl` | Adds TCLAP command-line parsing on top of the base. |

Why split? So embedders like **Envoy Mobile** can build a fully-populated options object
**programmatically** (via the base's setters) without dragging in TCLAP or a command line.
`OptionsImpl` is a `friend` of the base so its parser can write the private fields directly.

## The one computed field: concurrency

Most options map 1:1 to a flag. The exception is `concurrency()` (the worker-thread count),
which is genuinely *computed*:

- If `--concurrency` is set → use it (clamped to ≥ 1).
- Else if `--cpuset-threads` is set → use `OptionsImplPlatform::getCpuCount()` (cpuset +
  cgroup aware).
- Else → default to `std::thread::hardware_concurrency()`.

This is why there's a platform abstraction (`getCpuCount`) — counting "CPUs available to
*this* process" is OS-specific (Linux uses `sched_getaffinity` + cgroup quotas).

## The three modes

`Options::mode()` returns a `Mode` enum that is the top-level switch for what Envoy does:

| Mode | Behavior |
|------|----------|
| `Serve` | Normal: run the event loop and serve traffic. |
| `Validate` | Parse + initialize the config, print OK/FAIL, and exit (see [`../config_validation/`](../config_validation/README.md)). |
| `InitOnly` | Initialize then exit (used for profiling/perf dumps). |

## Documentation map

| Document | Contents |
|----------|----------|
| `OVERVIEW.md` | The `Options` interface by theme, the base/parser split, the TCLAP parse + validation flow, concurrency computation, the platform/cgroup abstraction, and how options feed the rest of the server. |
| `CLASS_HIERARCHY.md` | UML diagrams for `Options`, `OptionsImplBase`, `OptionsImpl`, the exceptions, and the platform seam. |

## One-paragraph mental model

`main()` constructs an `OptionsImpl` from `argc`/`argv`. The constructor defines a TCLAP arg
per flag, parses, converts string flags into enums (`mode`, `drain-strategy`,
`ip-version`), validates mutually-exclusive combinations (e.g. `--use-dynamic-base-id` with a
non-zero `--restart-epoch`), and computes `concurrency_`. `--help`/`--version` throw
`NoServingException` (exit 0); bad flags throw `MalformedArgvException` (exit 1). The
resulting `const Options&` is threaded everywhere — the server scaffolding switches on
`mode()`, and `InstanceBase`, the hot restarter, the drain manager, and the logging context
all read their settings from it.

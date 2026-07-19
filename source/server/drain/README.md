# Drain Manager

> Documentation for Envoy's connection-draining subsystem.
> Source lives in `source/server/drain_manager_impl.{h,cc}`. The interface is
> `envoy/server/drain_manager.h` (which extends `Network::DrainDecision`).

Draining is how Envoy **gracefully winds down** — during shutdown, during a hot restart, or
when an operator hits `/drain_listeners`. Instead of slamming connections closed, the drain
manager nudges them shut gradually so in-flight requests can complete and downstreams can
notice the host going away.

## Two things it controls

| Responsibility | Mechanism |
|----------------|-----------|
| **Should I close this connection now?** | `drainClose(direction)` — consulted on the connection hot path. Returns true with a rising probability as the drain window elapses. |
| **Shut down the hot-restart parent** | `startParentShutdownSequence()` — arms a timer that eventually tells the parent process to terminate. |

## The drain "tree"

There is one **root** drain manager (owned by the server) and a **child** drain manager per
listener, created via `createChildManager()`. When the root starts draining, it cascades the
state change to all children, so a server-wide drain fans out to every listener. Children can
also drain independently (e.g. a single listener being removed).

```mermaid
flowchart TD
    Root["Root DrainManager<br/>(server-wide)"] -->|createChildManager| L1["Listener A drain mgr"]
    Root -->|createChildManager| L2["Listener B drain mgr"]
    Root -->|createChildManager| L3["Listener C drain mgr"]
    Root -. "startDrainSequence cascades" .-> L1
    Root -. .-> L2
    Root -. .-> L3
```

## Drain strategies

`--drain-strategy` (an `Options::DrainStrategy`) picks how connections are closed during the
drain window (`--drain-time`, default 10 min):

| Strategy | Behavior |
|----------|----------|
| `Immediate` | `drainClose()` returns `true` right away — close as soon as asked. |
| `Gradual` (default) | `drainClose()` returns `true` with probability `elapsed / drain_time`, so closes ramp up smoothly over the window. |

A separate timer, `--parent-shutdown-time` (default 15 min), governs when a hot-restart child
finally tells its parent to terminate.

## Documentation map

| Document | Contents |
|----------|----------|
| `OVERVIEW.md` | The `DrainManager` interface, the gradual-close probability math, per-direction draining, the child cascade, the drain + parent-shutdown sequences, and integration points. |
| `CLASS_HIERARCHY.md` | UML diagrams for `DrainDecision`, `DrainManager`, and `DrainManagerImpl`. |

## One-paragraph mental model

When a drain is triggered for a `DrainDirection` (InboundOnly or All), the manager records a
deadline (`now + drain_time`), flips an atomic `draining_` flag, cascades to child managers,
and arms a timer that fires the completion callbacks at the end of the window. Meanwhile, every
connection consults `drainClose(direction)`: under the gradual strategy it returns true when a
random number falls below the fraction of the window elapsed, so connections close at an
increasing rate. If the server's health check has failed (and the listener is `DEFAULT` drain
type), `drainClose` returns true immediately. Separately, in a hot restart, the child arms
`parent_shutdown_timer_`; when it fires, it calls `hotRestart().sendParentTerminateRequest()`.

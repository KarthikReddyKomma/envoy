# Hot Restart

> Documentation for Envoy's zero-downtime hot restart machinery.
> Source lives in `source/server/hot_restart_impl.{h,cc}`,
> `source/server/hot_restart_nop_impl.h`, `source/server/hot_restarting_base.{h,cc}`,
> `source/server/hot_restarting_child.{h,cc}`, `source/server/hot_restarting_parent.{h,cc}`,
> and the proto `source/server/hot_restart.proto`. The interface is
> `envoy/server/hot_restart.h`.

Hot restart lets a brand-new Envoy binary (and/or config) take over from a running one
**without dropping connections and without losing listening sockets**. This is how you
upgrade Envoy in production without a load-balancer dance.

## The problem it solves

A naive restart closes the listen sockets, so in-flight connections drop and there is a
window where the port is unbound and new connections are refused. Hot restart avoids both:
the new process (the **child**) asks the old process (the **parent**) to **hand over its
open listen socket file descriptors** over a unix domain socket. The kernel keeps the
sockets open the whole time, so no connection is ever refused.

## The three actors

| Role | Class | What it is |
|------|-------|-----------|
| Coordinator | `HotRestartImpl` | The per-process object. Owns shared memory + both halves below. |
| New process | `HotRestartingChild` | Requests sockets/stats from the parent; takes over. |
| Old process | `HotRestartingParent` | Serves the child's requests; then drains and dies. |
| Transport | `HotRestartingBase` / `RpcStream` | Datagram domain socket, message framing, fd passing. |
| Disabled | `HotRestartNopImpl` | No-op used when hot restart is off or in config validation. |

Because up to **three** Envoy processes can coexist during an upgrade (old, current, new),
a single process is *simultaneously* a child (of its predecessor) and a parent (of its
successor). That's why `HotRestartImpl` owns **both** an `as_child_` and an `as_parent_`.

## Key concepts

- **restart epoch** (`--restart-epoch`): a generation counter. **Epoch 0 is the first
  process and has no parent.** Each hot restart launches the next process with epoch N+1.
- **base id** (`--base-id` / `--use-dynamic-base-id`): lets multiple independent Envoy
  groups share a host without colliding. It prefixes the shared-memory and socket names.
- **version hash** (`--hot-restart-version`): a compatibility string. If the new binary's
  hot-restart layout differs, the takeover is refused and you must do a full restart.

## Documentation map

| Document | Contents |
|----------|----------|
| `OVERVIEW.md` | Architecture: shared memory layout, the two-channel domain socket, fd passing via `SCM_RIGHTS`, the child/parent roles, versioning, dynamic base id. |
| `protocol.md` | The wire protocol and the full takeover handshake, message by message, with sequence diagrams. |
| `CLASS_HIERARCHY.md` | UML diagrams for the interface, `HotRestartImpl`, the transport classes, and the child/parent. |

## One-paragraph mental model

The child binds a domain socket, computes the parent's socket name (epoch − 1), and over
that socket: asks the parent to release its admin port, requests each listen socket's fd
(passed as ancillary `SCM_RIGHTS` data), and pulls the parent's stats so metrics stay
continuous. Once the child's own workers are up and accepting on every port, it tells the
parent to drain; after a configurable parent-shutdown delay it tells the parent to
terminate. A small POSIX shared-memory region holds cross-process robust mutexes (for log
file serialization) and a version/initialization guard.

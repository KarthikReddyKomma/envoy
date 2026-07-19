# Overload Manager

> Documentation for Envoy's overload manager and resource-monitoring subsystem.
> Source lives in `source/server/overload_manager_impl.{h,cc}` and
> `source/server/null_overload_manager.h`. Interfaces are under
> `envoy/server/overload/` and `envoy/server/resource_monitor.h`.

The overload manager keeps an Envoy instance from collapsing under resource pressure. It
**monitors resources** (heap, CPU, file descriptors, active downstream connections, ...),
converts each reading into a normalized **pressure** value in `[0.0, 1.0]`, and uses
configured **triggers** to drive **overload actions** that shed load before the process
falls over.

## What it can do under pressure

Well-known overload actions (names in `envoy/server/overload/overload_manager.h`):

| Action | Effect |
|--------|--------|
| `stop_accepting_requests` | Reject new HTTP requests. |
| `disable_http_keepalive` | Drop HTTP/1.x keep-alive (force connection close). |
| `stop_accepting_connections` | Stop accepting new connections. |
| `reject_incoming_connections` | Accept then immediately close new connections. |
| `shrink_heap` | Release free memory back to the OS. |
| `reduce_timeouts` | Scale configured timeouts down toward their minimums. |
| `reset_high_memory_stream` | Reset streams using excessive memory. |
| `close_idle_http_connections` | Terminate idle downstream HTTP connections. |

There is also a related **LoadShedPoint** mechanism (`load_shed_point.h`): hot-path code
calls `shouldShedLoad()` at named lifecycle points (e.g. `tcp_listener_accept`) and the same
resource pressure decides whether to shed.

## Two kinds of resources

| Kind | Polled? | Example |
|------|---------|---------|
| **Regular** | Yes — on a periodic timer. | fixed heap, CPU utilization, injected resource. |
| **Proactive** | No — checked synchronously on the hot path. | global downstream max connections (atomic allocate/deallocate). |

## Documentation map

| Document | Contents |
|----------|----------|
| `OVERVIEW.md` | The periodic refresh loop, how pressure becomes action state, threshold vs scaled triggers, the thread-local publish, scaled-timer integration, and proactive resources. |
| `CLASS_HIERARCHY.md` | UML diagrams for the interfaces, `OverloadManagerImpl`, triggers, actions, the symbol table, and the thread-local state. |

## One-paragraph mental model

On a timer (default 1s) the manager asks each regular monitor to update; each monitor
reports a pressure in `[0,1]`. For every action subscribed to that resource, the manager
recomputes the action's state as the **maximum over its triggers** (a threshold trigger is
binary; a scaled trigger ramps linearly between a scaling and a saturation threshold). When
an action's state changes, the new state is **published to every worker thread's
thread-local copy** and registered callbacks are posted to their dispatchers. Hot-path code
then reads `getThreadLocalOverloadState().getState(action)` with no locks. Proactive
resources skip the timer entirely: the hot path atomically allocates/deallocates against a
bound.

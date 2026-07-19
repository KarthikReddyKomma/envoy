# Workers & Listeners — Overview

This document covers the worker thread lifecycle, how listeners are added to workers, the
TCP accept + balancing path, UDP routing, and the factory / stats / hooks seams.

## 1. The worker thread

`WorkerImpl` (`worker_impl.{h,cc}`) is a thin wrapper that **owns**:

- `Event::DispatcherPtr dispatcher_` — its own event loop.
- `Network::ConnectionHandlerPtr handler_` — does the actual listener/connection work.
- `Thread::ThreadPtr thread_` — the OS thread the loop runs on.
- `WatchDogSharedPtr watch_dog_` — created lazily once the loop is running.

Nearly every functional call (`addListener`, `removeListener`, `stopListener`,
`numConnections`, `closeIdleHttpConnections`, ...) forwards to `handler_`. `WorkerImpl` is
really about thread + lifecycle management.

### Construction

`ProdWorkerFactory::createWorker()` allocates a dispatcher (named like `worker_3`), creates
the `ConnectionHandler` via the registered factory `envoy.connection_handler.default`, and
constructs the `WorkerImpl`. The `WorkerImpl` constructor registers the dispatcher with TLS
(`tls_.registerThread(*dispatcher_, false)`) and registers four overload-manager actions
(stop-accepting, reject-incoming, reset-streams, close-idle-http).

### Start and the thread routine

`start()` spawns the thread (named `wrk:<dispatcher>`); the thread runs `threadRoutine`:

```cpp
void WorkerImpl::threadRoutine(OptRef<GuardDog> guard_dog, const std::function<void()>& cb) {
  dispatcher_->post([this, &guard_dog, cb]() {
    cb();                                  // signal "this worker is up"
    if (guard_dog.has_value()) {
      watch_dog_ = guard_dog->createWatchDog(currentThreadId(), dispatcher_->name(), *dispatcher_);
    }
  });
  dispatcher_->run(Event::Dispatcher::RunType::Block);   // the worker event loop
  if (guard_dog.has_value()) guard_dog->stopWatching(watch_dog_);
  dispatcher_->shutdown();
  handler_.reset();        // close all connections before TLS teardown (avoid races)
  tls_.shutdownThread();
  watch_dog_.reset();
}
```

The watchdog is created **inside a posted callback** so it only registers after the loop is
running and TLS stat scopes are valid.

### Lifecycle rule: post, don't touch

All mutating operations on a worker are **posted onto that worker's dispatcher** for
thread-safety, then delegate to the handler:

```cpp
void WorkerImpl::addListener(..., Network::ListenerConfig& listener, AddListenerCompletion completion, ...) {
  dispatcher_->post([this, ..., completion]() {
    handler_->addListener(overridden_listener, listener, runtime, random);
    hooks_.onWorkerListenerAdded();
    completion();
  });
}
```

`stop()` calls `dispatcher_->exit()` then `thread_->join()` (guarded by `if (thread_)` since
shutdown can race startup).

## 2. Adding listeners to workers

The `ListenerManagerImpl` (in `source/common/listener_manager/`) adds the **same** listener
to **every** worker:

```cpp
void ListenerManagerImpl::addListenerToWorker(Worker& worker, ..., ListenerImpl& listener, cb) {
  worker.addListener(overridden_listener, listener,
    [this, cb]() {
      // completion runs on the worker thread; post back to main to avoid locking
      server_.dispatcher().post([this, cb]() { stats_.listener_create_success_.inc(); if (cb) cb(); });
    }, server_.runtime(), server_.api().randomGenerator());
}
```

So every worker independently listens on the shared / reuse-port / duplicated socket.

## 3. TCP accept and connection balancing

Per-worker accepts are handled by `ActiveTcpListener` (in `source/common/listener_manager/`).
On accept, the socket may be **rebalanced** to a different worker:

```cpp
void ActiveTcpListener::onAcceptWorker(ConnectionSocketPtr&& socket, ..., bool rebalanced, ...) {
  Network::BalancedConnectionHandler& target = connection_balancer_.pickTargetHandler(*this);
  if (&target != this) {
    target.post(std::move(socket));   // hand the socket to another worker's listener
    return;
  }
  // ... otherwise create the connection on this worker
}
```

- The listener registers/unregisters with the `ConnectionBalancer` in its ctor/dtor.
- `post()` wraps the socket in a `shared_ptr`, posts it to the target worker's dispatcher,
  then looks up the destination listener via `getBalancedHandlerByTag(...)` and calls
  `onAcceptWorker(..., rebalanced=true)`.

This is how Envoy evens out load when the kernel's accept distribution is lumpy.

## 4. UDP routing

UDP has a two-layer design in `active_udp_listener.{h,cc}`:

- **`ActiveUdpListenerBase`** adds UDP routing on top of `ActiveListenerImplBase`. On each
  datagram, it picks a destination worker and either handles locally or routes:

```cpp
void ActiveUdpListenerBase::onData(Network::UdpRecvData&& data) {
  uint32_t dest = (concurrency_ > 1) ? destination(data) : worker_index_;
  if (dest == worker_index_) onDataWorker(std::move(data));        // local
  else udp_listener_worker_router_.deliver(dest, std::move(data)); // route to peer worker
}
```

  `destination()` defaults to the current worker; QUIC overrides it to route by connection
  ID so all packets for a connection land on the same worker.

- **`ActiveRawUdpListener`** is the concrete non-QUIC listener; it owns a `UdpPacketWriter`
  and runs the UDP listener read-filter chain in `onDataWorker`.

### TCP vs UDP at a glance

| Aspect | TCP | UDP |
|--------|-----|-----|
| Unit | a connection (socket) | a datagram |
| Cross-worker | `ConnectionBalancer::pickTargetHandler` + `post(socket)` | `UdpListenerWorkerRouter::deliver` + `destination()` |
| Filters | network filter chains per connection | UDP read-filter chain per listener |
| State | long-lived connections | connectionless (QUIC adds state) |

## 5. The factory and stats seams

- **`ListenerManagerFactory`** (`listener_manager_factory.h`, category
  `envoy.listener_manager_impl`) makes the listener manager itself pluggable, so a variant
  (e.g. API-listener-only for Envoy Mobile) can be selected. The combined `ConnectionHandler`
  type and its `ConnectionHandlerFactory` (category `envoy.connection_handler`) are defined
  here too.
- **`ListenerStats` / `PerHandlerListenerStats`** (`listener_stats.h`) are stat structs
  built from macros — per-listener counters/gauges (`downstream_cx_total`,
  `downstream_cx_active`, `downstream_cx_overflow`, `no_filter_chain_match`,
  `downstream_cx_length_ms`, ...) and a per-worker subset. They are instantiated in
  `ActiveListenerImplBase`'s constructor.

## 6. The hooks seam

`ListenerHooks` (`listener_hooks.h`) is a test-integration interface with three hooks:
`onWorkerListenerAdded()`, `onWorkerListenerRemoved()`, and `onWorkersStarted()`. The
**production** impl, `DefaultListenerHooks`, makes all three empty; integration tests
subclass it to block/wait on listener lifecycle transitions.

## 7. Startup flow (end to end)

```mermaid
sequenceDiagram
    participant LM as ListenerManagerImpl
    participant W as WorkerImpl (x N)
    participant D as worker dispatcher
    participant GD as GuardDog

    LM->>W: addListenerToWorker (per listener, per worker)
    Note over W: posts handler_->addListener onto D
    LM->>W: worker->start(guard_dog, cb)
    W->>D: createThread(threadRoutine)
    D->>D: post: cb(); watch_dog_ = guard_dog.createWatchDog(...)
    D->>D: run(Block)  -- accepting connections
    LM->>LM: workers_waiting_to_run.Wait()
    GD-->>D: periodically verify watchdog touched
```

The guard dog (already running since server init) begins watching each worker the moment
`createWatchDog` registers it. See `guarddog.md` for that mechanism.

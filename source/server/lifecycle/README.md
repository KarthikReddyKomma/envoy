# Server Lifecycle & Bootstrap

> Documentation for the server instance, bootstrap, and lifecycle code.
> Source lives in `source/server/server.{h,cc}`, `source/server/instance_impl.{h,cc}`,
> `source/server/configuration_impl.{h,cc}`, and `source/server/factory_context_impl.{h,cc}`.

This folder documents the **heart of an Envoy process**: the object that owns every
subsystem and orchestrates the journey from "the process just started" to "we are
serving traffic" and finally to "we have cleanly shut down".

## What this subsystem does

When Envoy starts, *something* has to:

1. Load and validate the bootstrap config.
2. Stand up the global infrastructure: stats, runtime, the thread-local system, the
   singleton manager, the secret manager, the cluster manager, the listener manager,
   the admin console, tracing, access logging, the overload manager, and the guard dog.
3. Wait for all of those to initialize (clusters may need network round-trips).
4. Start the worker threads so listeners begin accepting connections.
5. Run the main event loop until told to stop.
6. Tear everything down in the correct order.

That "something" is **`Server::InstanceBase`** (in `server.h`/`server.cc`) and its
production subclass **`Server::InstanceImpl`** (in `instance_impl.{h,cc}`). It implements
the `Server::Instance` interface that the rest of the codebase uses to reach shared
services (`clusterManager()`, `dispatcher()`, `stats()`, `runtime()`, `admin()`, ...).

## The cast of characters

| Component | File | Role |
|-----------|------|------|
| `InstanceBase` | `server.h` / `server.cc` | The server object. Owns everything, drives `initialize()` → `run()` → `terminate()`. |
| `InstanceImpl` | `instance_impl.{h,cc}` | Production subclass; supplies the real overload manager, guard dog, heap shrinker, HDS delegate. |
| `InstanceUtil` | `server.cc` | Free helpers: `loadBootstrapConfig()`, `createRuntime()`, `flushMetricsToSinks()`. |
| `RunHelper` | `server.cc` | A small RAII object that wires signals + the cluster-manager init callback for `run()`. |
| `MainImpl` / `InitialImpl` | `configuration_impl.{h,cc}` | Parse the bootstrap proto into the live configuration consumed at startup. |
| `ServerFactoryContextImpl` | `factory_context_impl.{h,cc}` | The "give me access to server services" context handed to extension factories. |

## How to read these docs

- **`OVERVIEW.md`** — the big picture: the layered ownership model, the three phases
  (`initialize` / `run` / `terminate`), and the major design rules.
- **`startup_sequence.md`** — a step-by-step walk through `initialize()` and `run()`,
  including the init-manager handshake, cluster warming, and worker startup, with
  sequence diagrams.
- **`CLASS_HIERARCHY.md`** — UML-style class diagrams for `Instance`, `InstanceBase`,
  `InstanceImpl`, the configuration objects, and the factory context.

## The one-paragraph mental model

A process boots into `main()` (see `source/exe/`), which builds an `InstanceImpl`. The
constructor + `initialize()` build the world from the bootstrap config. `run()` posts the
`Startup` lifecycle notification, then blocks in `dispatcher_->run(Block)` — this is the
main thread's event loop. While blocked, the cluster manager finishes initializing, the
init manager fires, workers start, and listeners come up. When a `SIGTERM` (or
`/quitquitquit`) arrives, `shutdown()` causes the loop to exit and `terminate()` runs the
teardown sequence in reverse dependency order.

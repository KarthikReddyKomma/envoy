# Config Validation Mode

> Documentation for Envoy's `--mode validate` config-validation server.
> Source lives in `source/server/config_validation/`: `server.{h,cc}`, `api.h`,
> `dispatcher.h`, `cluster_manager.h`, `admin.h`.

When you run `envoy --mode validate -c config.yaml`, Envoy does **not** start serving. It
loads and initializes the configuration as far as it can **without any observable side
effects** — no ports bound, no upstream connections, no hot restart — and reports whether the
config is valid. This folder implements that mode.

## Why a whole separate server?

The goal is high-fidelity validation: catch the same errors a real startup would, by
exercising the *actual* initialization code paths (JSON/proto parsing, factory construction,
filter-chain building, cluster construction). A shallow "schema check" would miss errors that
only surface when objects are built.

So instead of a special-purpose validator, Envoy runs a **stubbed `Server::Instance`** —
`ValidationInstance` — that reuses the real `MainImpl` config, the real cluster-manager
*factory*, the real listener-manager factory, and so on, but swaps a handful of components for
no-op / no-side-effect variants:

| Real component | Validation stub | What it suppresses |
|----------------|-----------------|--------------------|
| `Event::DispatcherImpl` | `ValidationDispatcher` | `createClientConnection` (no outbound network) |
| `Api::Impl` | `Api::ValidationImpl` | hands out `ValidationDispatcher`s |
| `ClusterManagerImpl` | `ValidationClusterManager` | `getThreadLocalCluster` returns `nullptr` (no upstream conns) |
| `ProdClusterManagerFactory` | `ValidationClusterManagerFactory` | CDS created but discarded |
| `AdminImpl` | `ValidationAdmin` | tracks config handlers but starts no HTTP listener |
| `HotRestartImpl` | `HotRestartNopImpl` | no shared memory, no parent takeover |

If initialization reaches the point where a real Envoy would begin serving, validation passes.

## Entry point

```cpp
bool validateConfig(const Options&, const Address& local_address, ComponentFactory&,
                    ThreadFactory&, Filesystem::Instance&, ProcessContextOptRef = nullopt);
```

It constructs a `ValidationInstance`, prints `configuration '<path>' OK` on success, calls
`shutdown()`, and returns `true`. Any `EnvoyException` during construction is caught and
returns `false`. This is invoked from `MainCommonBase::run()` when `options.mode() == Validate`
(see [`../options/`](../options/README.md)).

## Documentation map

| Document | Contents |
|----------|----------|
| `OVERVIEW.md` | The `ValidationInstance` lifecycle, the stripped-down `initialize()`, and how each stub avoids side effects. |
| `CLASS_HIERARCHY.md` | UML diagrams for `ValidationInstance` and each stub vs. its production counterpart. |

## One-paragraph mental model

`validateConfig()` builds a `ValidationInstance`, which mirrors `InstanceBase::initialize()`
but only runs the side-effect-free subset: load bootstrap, build the regex engine, validate
bootstrap extensions, create runtime, overload manager, listener manager (validation variant),
secret manager, SSL context manager, xDS manager, and the cluster manager (via the validation
factory). The dispatcher and cluster manager are wired so that nothing can open a socket. When
`clusterManager().setInitializedCb` fires, the init manager runs; reaching that point means the
config is valid. Then `shutdown()` does an abbreviated teardown (no workers to stop).

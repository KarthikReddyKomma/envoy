# Smart Pointers in Envoy

This document explains how Envoy uses smart pointers throughout its codebase — which types are used, when each is appropriate, and concrete patterns taken from the actual source.

---

## 1. Overview

Envoy is a modern C++17 codebase that avoids raw owning pointers almost entirely. The policy is:

- **`std::unique_ptr`** — sole, non-shared ownership; the default choice
- **`std::shared_ptr`** — shared ownership; used when lifetime is genuinely shared across threads or callbacks
- **`std::weak_ptr`** — non-owning observer of a `shared_ptr`; used to break cycles and for safe cross-thread callbacks
- **`absl::WrapUnique`** — wraps a raw pointer in a `unique_ptr` (used when calling factories that return `T*`)
- **`OptRef<T>`** — a non-owning nullable reference (Envoy-specific; not a pointer at all)

---

## 2. `std::unique_ptr` — Sole Ownership

### 2.1 When Used

- An object has a single, clear owner
- Object is created and destroyed within one component
- Factory return values (the caller owns the result)
- Sub-objects inside a class that outlive no other entity

### 2.2 Examples in the Codebase

**Listener sub-objects** (`listener_impl.h`):

```cpp
// Each of these is owned exclusively by ListenerImpl
std::unique_ptr<Init::Manager>    dynamic_init_manager_;
std::unique_ptr<FilterChainManagerImpl> filter_chain_manager_;
std::unique_ptr<Network::InternalListenerConfig> internal_listener_config_;
std::unique_ptr<HttpConnectionManagerProto::ProxyStatusConfig> proxy_status_config_;
```

**Factory methods return `unique_ptr`** (`listener_impl.cc`):

```cpp
// Static factory — caller owns the result
absl::StatusOr<std::unique_ptr<ListenerImpl>>
ListenerImpl::create(const envoy::config::listener::v3::Listener& config, ...) {
    auto ret = std::unique_ptr<ListenerImpl>(
        new ListenerImpl(config, ...));    // private ctor → must use raw new
    RETURN_IF_NOT_OK(creation_status);
    return ret;
}
```

**`make_unique` for construction** (preferred style):

```cpp
udp_listener_config_->listener_factory_ =
    std::make_unique<Server::ActiveRawUdpListenerFactory>(concurrency);

udp_listener_config_->writer_factory_ =
    std::make_unique<Network::UdpDefaultWriterFactory>();
```

**Upgrade filter factories** (`http_connection_manager/config.cc`):

```cpp
std::unique_ptr<FilterFactoriesList> factories = std::make_unique<FilterFactoriesList>();
helper.processFilters(upgrade_config.filters(), name, "http upgrade", *factories);
upgrade_filter_factories_.emplace(name, FilterConfig{std::move(factories), enabled});
```

### 2.3 `absl::WrapUnique` for Raw Pointer Adoption

When a legacy API returns a raw `T*` and you need to take ownership:

```cpp
// network/listen_socket_impl.h
return absl::WrapUnique(new NetworkSocket(...));
// Equivalent to: std::unique_ptr<NetworkSocket>(new NetworkSocket(...))
// Used when make_unique can't be used (e.g., private constructor or factory)
```

### 2.4 Transfer with `std::move`

`unique_ptr` is move-only. Ownership transfer is always explicit:

```cpp
auto manager = std::make_unique<Init::ManagerImpl>("name");
parent_.init_manager_.add(*manager);           // pass by ref — no transfer
some_member_ = std::move(manager);             // transfer ownership here
// manager is now null — accessing it would be UB
```

---

## 3. `std::shared_ptr` — Shared Ownership

### 3.1 When Used

- Object lifetime is controlled by multiple owners (e.g., listeners share a factory context)
- Objects captured in lambdas that outlive the creator (common with dispatchers and callbacks)
- Thread-safe lifetime extension when passing objects across threads
- Plugin/factory objects where many components hold references

### 3.2 Examples in the Codebase

**Listener shares factory context across filter chains** (`listener_impl.h`):

```cpp
std::shared_ptr<PerListenerFactoryContextImpl>  listener_factory_context_;
std::shared_ptr<UdpListenerConfigImpl>           udp_listener_config_;
std::shared_ptr<BasicResourceLimitImpl>          open_connections_;
std::shared_ptr<Server::Configuration::TransportSocketFactoryContextImpl>
                                                 transport_factory_context_;
```

**The HCM itself is created as a `shared_ptr`** because it is captured in a lambda:

```cpp
// http_connection_manager/config.cc
return [singletons, filter_config, &context, clear_hop_by_hop_headers]
       (Network::FilterManager& filter_manager) -> void {
    auto hcm = std::make_shared<Http::ConnectionManagerImpl>(
        filter_config, context.drainDecision(), ...);
    filter_manager.addReadFilter(std::move(hcm));
    // hcm is now shared between the lambda (singletons) and the filter manager
};
```

**Access log shared across request streams**:

```cpp
// access_logs_ is vector<AccessLog::InstanceSharedPtr>
using InstanceSharedPtr = std::shared_ptr<Instance>;

AccessLog::InstanceSharedPtr log =
    AccessLog::AccessLogFactory::fromProto(access_log, *listener_factory_context_);
access_logs_.push_back(log);
```

**Connection pool objects shared between cluster and threads**:

```cpp
std::shared_ptr<BasicResourceLimitImpl> open_connections_ =
    std::make_shared<BasicResourceLimitImpl>(
        std::numeric_limits<uint64_t>::max(),
        runtime(),
        cx_limit_runtime_key_);
```

### 3.3 Aliased `shared_ptr` (sub-object sharing)

Envoy uses the aliased `shared_ptr` constructor to share sub-objects while tying their lifetime to the parent:

```cpp
// shared_ptr<Child> that keeps the Parent alive
auto child_view = std::shared_ptr<Child>(parent_shared_ptr, &parent_shared_ptr->child_);
```

This pattern appears in config provider managers where sub-configs share the parent subscription's lifetime.

---

## 4. `std::weak_ptr` — Non-Owning Observer

### 4.1 When Used

- Break reference cycles between two `shared_ptr`-owning objects
- Safe cross-thread callbacks: check if the object still exists before using it
- Caches where the cached value should not prevent destruction

### 4.2 Examples in the Codebase

**Init system — safe callbacks** (`init/target_impl.h`):

```cpp
// The fn_ weak_ptr allows the target to be destroyed while a handle still exists
const std::weak_ptr<InternalInitializeFn> fn_;

// At callback time:
auto fn = fn_.lock();    // returns nullptr if object was destroyed
if (fn) {
    (*fn)();
}
```

**Tracer manager cache** (`tracing/tracer_manager_impl.h`):

```cpp
// Tracers are owned by listeners. The manager caches them weakly
// so destroyed listeners don't prevent cleanup.
absl::flat_hash_map<std::size_t, std::weak_ptr<Tracer>> tracers_;

// Periodically prune dead tracers:
absl::erase_if(tracers_, [](const auto& entry) {
    return entry.second.expired();
});
```

**Config subscription shared pool** (`config/config_provider_impl.h`):

```cpp
// Multiple providers may share one subscription; the subscription is
// owned by the first provider, the rest hold weak refs.
absl::node_hash_map<uint64_t, std::weak_ptr<ConfigSubscriptionCommonBase>> subscriptions_;
```

**Safe async callbacks via `weak_ptr<void>` sentinel** (`network/udp_listener_impl.cc`):

```cpp
// destruction_checker_ is a shared_ptr<void> owned by the listener
// If the listener is destroyed before the callback fires, alive.lock() returns nullptr.
dispatcher.post([this, alive = std::weak_ptr<void>(destruction_checker_)]() {
    if (alive.lock() == nullptr) {
        return;  // listener gone — skip the callback
    }
    handleIncomingPacket();
});
```

---

## 5. `enable_shared_from_this` — Self-Referential Shared Objects

### 5.1 When Used

When an object needs to create a `shared_ptr` to itself (e.g., to pass `this` into a callback where the callback must keep the object alive).

```cpp
class ObjectSharedPool
    : public std::enable_shared_from_this<ObjectSharedPool<T, HashFunc, EqualFunc>> {

    void someMethod() {
        auto self = shared_from_this();  // creates shared_ptr to this
        dispatcher_.post([self]() {      // captures shared_ptr in lambda
            self->doWork();              // safe: object alive as long as lambda lives
        });
    }
};
```

### 5.2 Anti-pattern: `shared_ptr<this>` without `enable_shared_from_this`

Doing `std::shared_ptr<Foo>(this)` without inheriting from `enable_shared_from_this` creates a **second, independent reference count** — when either `shared_ptr` hits zero, it deletes the object while the other still thinks it's alive. This is a use-after-free bug.

---

## 6. Common Aliases in Envoy

Envoy defines type aliases to reduce verbosity and enforce policy:

```cpp
// Common shared_ptr aliases (from envoy/network/...)
using SocketSharedPtr        = std::shared_ptr<Socket>;
using SocketOptionsSharedPtr = std::shared_ptr<Options>;
using FilterSharedPtr        = std::shared_ptr<Filter>;
using ConnectionSharedPtr    = std::shared_ptr<Connection>;
using AddressConstSharedPtr  = std::shared_ptr<const Address::Instance>;

// Common unique_ptr aliases
using SubscriptionPtr   = std::unique_ptr<Subscription>;
using DrainManagerPtr   = std::unique_ptr<Server::DrainManager>;
```

Using aliases also means you can change the underlying smart pointer type in one place.

---

## 7. `OptRef<T>` — Non-Owning Optional Reference

`OptRef<T>` is an Envoy-specific type (wrapping `absl::optional<std::reference_wrapper<T>>`) used as a safer alternative to `T*` for optional parameters/return values:

```cpp
// In listener_impl.h:
Network::UdpListenerConfigOptRef udpListenerConfig() override {
    return udp_listener_config_ != nullptr
               ? Network::UdpListenerConfigOptRef(*udp_listener_config_)
               : Network::UdpListenerConfigOptRef();
}
```

Usage:

```cpp
auto udp_config = listener.udpListenerConfig();
if (udp_config.has_value()) {
    udp_config->get().doSomething();
}
```

**Why `OptRef` instead of `T*`?**
- Caller can't accidentally take ownership (no `delete`)
- `has_value()` makes optionality explicit vs. a nullable raw pointer
- No implicit null dereference — `has_value()` must be checked

---

## 8. Ownership Decision Guide

```
Do you need to transfer/move the object?
    └── yes → unique_ptr

Do multiple parties need the object to stay alive?
    ├── yes, and all parties are in the same thread → shared_ptr
    └── yes, across threads → shared_ptr + ensure thread-safe access

Do you need to observe an object without owning it?
    ├── object is always alive (invariant) → raw ref or raw ptr (T& / T*)
    ├── object may be destroyed → weak_ptr (and lock() before use)
    └── just for the duration of a function call → const ref (T&)

Does an object need shared_ptr to itself?
    └── inherit enable_shared_from_this, use shared_from_this()

Is a pointer optional (may be null)?
    ├── non-owning → OptRef<T>
    └── owning → unique_ptr (null if not set) or absl::optional<T>
```

---

## 9. Common Pitfalls and Envoy Conventions

### 9.1 Avoid `shared_ptr` when `unique_ptr` suffices

Shared ownership adds overhead (atomic ref count) and obscures lifetimes. Prefer `unique_ptr` unless sharing is genuinely needed.

### 9.2 Never store raw owning pointers

Bad:
```cpp
FilterChainManagerImpl* filter_chain_manager_;  // who deletes this?
```

Good:
```cpp
std::unique_ptr<FilterChainManagerImpl> filter_chain_manager_;
```

### 9.3 Use `std::move` when transferring `unique_ptr`

```cpp
// Wrong: won't compile — unique_ptr is not copyable
some_vec.push_back(my_unique_ptr);

// Correct:
some_vec.push_back(std::move(my_unique_ptr));
```

### 9.4 `make_unique` / `make_shared` over `new`

```cpp
// Prefer:
auto ptr = std::make_unique<Foo>(args...);
auto sptr = std::make_shared<Foo>(args...);

// Avoid (except when private constructor requires it):
std::unique_ptr<Foo>(new Foo(args...))
```

### 9.5 Lambdas and `shared_ptr` capture

When a lambda outlives its enclosing scope (e.g., posted to a dispatcher), ensure captured objects are kept alive via `shared_ptr`:

```cpp
// Correct: captures shared_ptr — object alive as long as lambda exists
auto self = shared_from_this();
dispatcher_.post([self]() { self->handleWork(); });

// Incorrect: captures raw this — may dangle if object destroyed before lambda fires
dispatcher_.post([this]() { handleWork(); });  // DANGEROUS
```

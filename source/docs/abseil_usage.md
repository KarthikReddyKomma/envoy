# Abseil Usage in Envoy

Abseil (absl) is a foundational C++ library from Google that Envoy uses extensively in place of (or alongside) the C++ standard library. This document covers every category of Abseil used in Envoy, with real examples from the codebase.

---

## 1. Why Abseil?

- **Performance**: Abseil containers (hash maps, etc.) are consistently faster than `std::unordered_map`
- **Safety**: `absl::Status` / `absl::StatusOr` provide structured error handling without exceptions
- **Portability**: Abseil backports C++17/20 features to C++14 and normalizes platform differences
- **Integration**: Google-internal ecosystem — Envoy's protobuf and gRPC dependencies also use Abseil

---

## 2. Strings (`absl/strings/`)

### 2.1 `absl::string_view` — Non-Owning String Reference

A non-owning view into a string buffer. Preferred over `const std::string&` for read-only string parameters because it accepts string literals, `std::string`, and other buffers without allocation.

```cpp
// connection_impl.cc
constexpr absl::string_view kTransportSocketConnectTimeoutTerminationDetails =
    "transport socket timeout was reached";

void ConnectionImpl::setFailureReason(absl::string_view failure_reason) {
    failure_reason_ = absl::StrCat(failure_reason, ". ", transport_socket_->failureReason());
}

absl::string_view ConnectionImpl::transportFailureReason() const {
    return failure_reason_;
}
```

**Why not `std::string_view`?**  Envoy pre-dates C++17 standardization of `string_view`, and Abseil's version has better compiler support and lifetime checks.

### 2.2 `absl::StrCat` — Efficient String Concatenation

`absl::StrCat` concatenates any number of strings and string-convertible values in a single allocation:

```cpp
// listener_impl.cc
return absl::StrCat("envoy_internal_", config.name());

// connection_impl.cc
setFailureReason(absl::StrCat("delayed connect error: ", errorDetails(error)));
setFailureReason(absl::StrCat("failed to bind to ", source->get()->asString(), ": ", strerror(errno)));
```

**Do not** use `absl::StrCat` in a loop — use `absl::StrAppend` instead:

```cpp
// filter_chain_manager_impl.cc
std::string message;
absl::StrAppend(&message, " filter_chain: ", chain.name());
absl::StrAppend(&message, " error: ", error_detail);
```

### 2.3 `absl::StrJoin` — Join a Range with a Separator

```cpp
// listener_impl.cc — join multiple listener addresses for error messages
absl::StrJoin(addresses_, ",", Network::AddressStrFormatter())
// → "0.0.0.0:8080,0.0.0.0:8443"
```

Custom formatter:

```cpp
struct AddressStrFormatter {
    void operator()(std::string* out, const Network::Address::InstanceConstSharedPtr& addr) const {
        absl::StrAppend(out, addr->asString());
    }
};
```

### 2.4 `absl::StrSplit` — Split a String

```cpp
std::vector<absl::string_view> parts = absl::StrSplit(header_value, ',');
for (auto part : parts) { ... }

// With limit:
std::pair<absl::string_view, absl::string_view> kv =
    absl::StrSplit(line, absl::MaxSplits(':', 1));
```

### 2.5 `absl::AsciiStrToLower` / `absl::EqualsIgnoreCase`

```cpp
// filter_chain_manager_impl.cc
absl::AsciiStrToLower(server_name)   // in-place lowercase for SNI matching

// conn_manager_impl.cc
absl::EqualsIgnoreCase(upgrade_type, "websocket")
```

### 2.6 `absl::StrContains`, `absl::StartsWith`, `absl::EndsWith`

```cpp
#include "absl/strings/match.h"

absl::StrContains(path, "/healthz")
absl::StartsWith(type_url, "type.googleapis.com/")
absl::EndsWith(cluster_name, "_v2")
```

### 2.7 `absl::StrFormat` — Printf-Style Formatting

```cpp
#include "absl/strings/str_format.h"

std::string msg = absl::StrFormat("listener %s: %d connections", name, count);
```

Safer than `sprintf` — format string is checked at compile time.

### 2.8 `absl::Substitute` — Template Substitution

```cpp
#include "absl/strings/substitute.h"

// $0, $1, ... are positional placeholders
std::string result = absl::Substitute(
    "Failed to create listener '$0': $1", listener_name, error_msg);
```

### 2.9 `absl::StripPrefix`, `absl::StripSuffix`

```cpp
#include "absl/strings/strip.h"

absl::string_view path = "/api/v1/clusters";
absl::StripPrefix(&path, "/api");  // path now = "/v1/clusters"
```

---

## 3. Status and Error Handling (`absl/status/`)

This is one of the most important Abseil libraries in Envoy. Envoy's codebase transitioned from exceptions to `absl::Status` for configuration and initialization errors.

### 3.1 `absl::Status` — Error Value

Represents success (`OkStatus()`) or failure (with a code and message):

```cpp
#include "absl/status/status.h"

// Returning errors from build methods:
absl::Status ListenSocketFactoryImpl::doFinalPreWorkerInit() {
    if (rc.return_value_ != 0) {
        return absl::InvalidArgumentError(
            fmt::format("cannot listen() errno={}", rc.errno_));
    }
    return absl::OkStatus();
}

// Checking status:
absl::Status status = doFinalPreWorkerInit();
if (!status.ok()) {
    ENVOY_LOG(error, status.message());
    return status;
}
```

Common error constructors:

| Constructor | gRPC equivalent | Use case |
|---|---|---|
| `absl::OkStatus()` | OK | Success |
| `absl::InvalidArgumentError(msg)` | INVALID_ARGUMENT | Bad config field |
| `absl::NotFoundError(msg)` | NOT_FOUND | Resource missing |
| `absl::UnimplementedError(msg)` | UNIMPLEMENTED | Feature not built |
| `absl::InternalError(msg)` | INTERNAL | Unexpected internal error |
| `absl::UnavailableError(msg)` | UNAVAILABLE | Transient failure |

### 3.2 `absl::StatusOr<T>` — Value or Error

Holds either a value of type `T` or an `absl::Status` error. The Envoy equivalent of a `Result<T, Error>` type.

```cpp
#include "absl/status/statusor.h"

// Factory returns either a valid object or an error
absl::StatusOr<std::unique_ptr<ListenerImpl>>
ListenerImpl::create(const envoy::config::listener::v3::Listener& config, ...) {
    absl::Status creation_status = absl::OkStatus();
    auto ret = std::unique_ptr<ListenerImpl>(new ListenerImpl(config, ..., creation_status));
    RETURN_IF_NOT_OK(creation_status);
    return ret;  // implicitly wraps unique_ptr in StatusOr
}

// Caller usage:
auto listener_or_error = ListenerImpl::create(config, ...);
if (!listener_or_error.ok()) {
    return listener_or_error.status();  // propagate error
}
auto listener = std::move(*listener_or_error);  // extract value
```

### 3.3 Envoy's Status Propagation Macros

Envoy defines macros to reduce boilerplate:

```cpp
// Return early if status is not OK
RETURN_IF_NOT_OK(status);
RETURN_IF_NOT_OK_REF(status_or.status());

// Set an output absl::Status reference and return if not OK
SET_AND_RETURN_IF_NOT_OK(some_operation().status(), creation_status);

// Extract value from StatusOr, or return error — like Rust's ? operator
auto value = THROW_OR_RETURN_VALUE(risky_operation(), ExpectedType);
```

Example combining these:

```cpp
ListenerImpl::ListenerImpl(const Listener& config, ..., absl::Status& creation_status) {
    auto address_or_error = Network::Address::resolveProtoAddress(config.address());
    SET_AND_RETURN_IF_NOT_OK(address_or_error.status(), creation_status);
    address_ = std::move(*address_or_error);

    SET_AND_RETURN_IF_NOT_OK(buildFilterChains(config), creation_status);
    SET_AND_RETURN_IF_NOT_OK(buildConnectionBalancer(config, *address_), creation_status);
}
```

---

## 4. Containers (`absl/container/`)

Abseil containers replace `std::unordered_map` and `std::unordered_set`. They use open-addressing hash tables for better cache performance.

### 4.1 `absl::flat_hash_map` — Fast Hash Map

The go-to replacement for `std::unordered_map`. Stores keys and values inline (flat), so iteration is cache-friendly.

```cpp
#include "absl/container/flat_hash_map.h"

// listener_impl.h — maps address string → connection balancer
absl::flat_hash_map<std::string, Network::ConnectionBalancerSharedPtr> connection_balancers_;

// listener_manager_impl.h — maps listener name → error state
absl::flat_hash_map<std::string, std::unique_ptr<UpdateFailureState>> lds_error_state_tracker_;

// http extensions — set of destination ports
absl::flat_hash_set<uint32_t> https_destination_ports_;
```

**When to use `flat_hash_map` vs `node_hash_map`?**
- `flat_hash_map`: keys/values stored contiguously — fastest for small, stable maps. **Pointers/iterators are invalidated on rehash.**
- `node_hash_map`: each element heap-allocated — slower but **pointer/iterator stability** is preserved across insertions.

### 4.2 `absl::node_hash_map` — Stable-Pointer Hash Map

```cpp
#include "absl/container/node_hash_map.h"

// filter_chain_manager_impl.h — filter chains keyed by match proto
// node_hash_map used because FilterChainMatch contains repeated fields
// and pointer stability across insertion is needed
using FilterChainsByMatcher = absl::node_hash_map<
    envoy::config::listener::v3::FilterChainMatch,
    Network::FilterChainSharedPtr,
    MessageUtil,
    MessageUtil>;
```

### 4.3 `absl::flat_hash_set` / `absl::node_hash_set`

```cpp
// listener_manager_impl.h
absl::flat_hash_set<uint64_t> stopped_listener_tags_;

// lds_api.cc — track listener names seen in this update
absl::node_hash_set<std::string> listener_names;
```

### 4.4 `absl::btree_map` / `absl::btree_set` — Ordered Containers

Drop-in replacement for `std::map` / `std::set`, but based on B-tree for better cache performance on iteration:

```cpp
#include "absl/container/btree_map.h"

// Used when ordered iteration over keys is required
absl::btree_map<std::string, RouteConfigPtr> route_configs_;
```

### 4.5 `absl::InlinedVector` — Small-Buffer Optimization

Stores the first N elements inline (on the stack), falling back to heap for larger sizes:

```cpp
#include "absl/container/inlined_vector.h"

// tracing/trace_context_impl.h
// Most trace context lookups return 0 or 1 results — stored inline
using GetAllResult = absl::InlinedVector<absl::string_view, 1>;
```

### 4.6 `absl::Span<T>` — Non-Owning Range View

Like `absl::string_view` but for arbitrary arrays. Used to pass contiguous sequences without copying:

```cpp
#include "absl/types/span.h"

// crypto/utility.h
std::vector<uint8_t> getSha256Hmac(
    absl::Span<const uint8_t> key,     // view into byte buffer
    absl::Span<const uint8_t> text);

// Usage:
std::vector<uint8_t> key_bytes = ...;
getSha256Hmac(absl::MakeSpan(key_bytes), absl::MakeSpan(text_bytes));
```

---

## 5. Optional and Variant (`absl/types/`)

### 5.1 `absl::optional` — Nullable Value

A backport of `std::optional`. Holds either a value or nothing.

```cpp
#include "absl/types/optional.h"

// connection_impl.h
absl::optional<std::chrono::milliseconds> lastRoundTripTime() const override;
absl::optional<uint64_t> congestionWindowInBytes() const override;

// Usage:
auto rtt = connection.lastRoundTripTime();
if (rtt.has_value()) {
    stats_.rtt_.record(rtt->count());
}

// Return empty optional:
return absl::nullopt;
```

Duration fields from proto often become `absl::optional`:

```cpp
idle_timeout_(PROTOBUF_GET_OPTIONAL_MS(config.common_http_protocol_options(), idle_timeout))
// → absl::optional<std::chrono::milliseconds>

// Later in constructor — treat zero duration as "disabled":
if (idle_timeout_.value().count() == 0) {
    idle_timeout_ = absl::nullopt;
}
```

### 5.2 `absl::variant` — Tagged Union

```cpp
#include "absl/types/variant.h"

// Used for type-safe oneof alternatives
using AddressOrError = absl::variant<Address::InstanceConstSharedPtr, absl::Status>;
```

---

## 6. Synchronization (`absl/synchronization/`)

### 6.1 `absl::Mutex` and `absl::MutexLock`

Abseil's mutex provides deadlock detection, lock-order checking, and static analysis annotations that `std::mutex` lacks.

```cpp
#include "absl/synchronization/mutex.h"

// network/connection_balancer_impl.h
class ExactConnectionBalancerImpl {
    absl::Mutex lock_;
    std::vector<BalancedConnectionHandler*> handlers_ ABSL_GUARDED_BY(lock_);
};

// network/connection_balancer_impl.cc
void ExactConnectionBalancerImpl::registerHandler(BalancedConnectionHandler& handler) {
    absl::MutexLock lock(&lock_);   // RAII lock — released on scope exit
    handlers_.push_back(&handler);
}
```

### 6.2 Thread Safety Annotations

Abseil defines macros for Clang's thread safety analysis:

```cpp
ABSL_GUARDED_BY(lock_)       // member must be accessed under this lock
ABSL_LOCKS_EXCLUDED(lock_)   // function must not hold this lock when called
ABSL_REQUIRES(lock_)         // function requires caller to hold this lock
ABSL_EXCLUSIVE_LOCKS_REQUIRED(lock_)
```

Example from access log manager:

```cpp
// access_log_manager_impl.h
absl::Mutex write_lock_;
bool flush_thread_exit_  ABSL_GUARDED_BY(write_lock_) {false};
bool reopen_file_        ABSL_GUARDED_BY(write_lock_) {false};
Buffer        flush_buffer_ ABSL_GUARDED_BY(write_lock_);
```

The Clang compiler verifies that `flush_thread_exit_` is only accessed while holding `write_lock_`.

### 6.3 `absl::BlockingCounter` — Wait for N Events

```cpp
// listener_manager_impl.cc — wait for all workers to acknowledge new listener
absl::BlockingCounter workers_waiting_to_run(workers_.size());

for (const auto& worker : workers_) {
    worker->addListener(listener, [&workers_waiting_to_run]() {
        workers_waiting_to_run.DecrementCount();
    });
}

workers_waiting_to_run.Wait();  // blocks until all workers have called DecrementCount()
```

---

## 7. Cleanup (`absl/cleanup/`)

`absl::Cleanup` is a scope guard that executes a lambda on scope exit — like `defer` in Go:

```cpp
#include "absl/cleanup/cleanup.h"

// http2/codec_impl.cc — clear current_stream_id_ regardless of how we exit
absl::Cleanup clear_current_stream_id = [this]() {
    parent_.current_stream_id_.reset();
};

// key_value_store_base.cc — reset iteration flag when done
absl::Cleanup restore_under_iterate = [this] {
    under_iterate_ = false;
};
under_iterate_ = true;
// ... do the iteration ...
// restore_under_iterate fires here, setting under_iterate_ = false
```

This is the preferred alternative to `try/finally` for cleanup logic.

---

## 8. Hashing (`absl/hash/`)

### 8.1 `absl::Hash` — Universal Hash Function

Used as the hash function for Abseil containers. Automatically composes across types:

```cpp
#include "absl/hash/hash.h"

// Custom hashable type
struct MyKey {
    std::string name;
    uint32_t port;

    template <typename H>
    friend H AbslHashValue(H h, const MyKey& k) {
        return H::combine(std::move(h), k.name, k.port);
    }
};

absl::flat_hash_map<MyKey, Value> my_map;  // uses AbslHashValue automatically
```

### 8.2 `absl::HashOf` — Compute Hash Directly

```cpp
size_t key_hash = absl::HashOf(config_source_hash, name);
```

---

## 9. Numeric (`absl/numeric/`)

### 9.1 `absl::int128` / `absl::uint128`

Used for 128-bit integer arithmetic (e.g., IPv6 address manipulation):

```cpp
#include "absl/numeric/int128.h"

absl::uint128 ipv6_addr = absl::MakeUint128(high_bits, low_bits);
```

---

## 10. Attributes and Optimization (`absl/base/`)

### 10.1 `ABSL_PREDICT_TRUE` / `ABSL_PREDICT_FALSE`

Compiler branch prediction hints:

```cpp
if (ABSL_PREDICT_FALSE(creation_status != nullptr && !creation_status->ok())) {
    return;
}
```

### 10.2 `ABSL_ATTRIBUTE_ALWAYS_INLINE` / `ABSL_ATTRIBUTE_NOINLINE`

```cpp
ABSL_ATTRIBUTE_ALWAYS_INLINE void fastPath() { ... }
ABSL_ATTRIBUTE_NOINLINE void slowPath() { ... }  // keep out of instruction cache
```

### 10.3 `absl::call_once` — Thread-Safe One-Time Initialization

```cpp
#include "absl/base/call_once.h"

absl::once_flag init_flag_;

void ensureInitialized() {
    absl::call_once(init_flag_, []() {
        // Runs exactly once, even with concurrent calls
        globalInit();
    });
}
```

---

## 11. Erase Utilities

### 11.1 `absl::erase_if` — Conditional Removal from Containers

```cpp
#include "absl/container/flat_hash_map.h"  // includes erase_if

// tracing/tracer_manager_impl.cc — prune expired tracer cache entries
absl::erase_if(tracers_, [](const std::pair<const std::size_t, std::weak_ptr<Tracer>>& entry) {
    return entry.second.expired();  // remove entries whose weak_ptr is dead
});
```

---

## 12. Summary: Abseil Module → Use Case

| Abseil Module | Key Types | Primary Use in Envoy |
|---|---|---|
| `absl/strings/` | `string_view`, `StrCat`, `StrJoin`, `StrSplit` | String manipulation without allocation |
| `absl/status/` | `Status`, `StatusOr` | Error propagation without exceptions |
| `absl/container/flat_hash_map` | `flat_hash_map`, `flat_hash_set` | Fast hash maps (most common) |
| `absl/container/node_hash_map` | `node_hash_map`, `node_hash_set` | Hash maps needing pointer stability |
| `absl/container/btree_map` | `btree_map`, `btree_set` | Ordered maps with fast iteration |
| `absl/container/inlined_vector` | `InlinedVector<T, N>` | Small vectors with inline storage |
| `absl/types/optional` | `optional`, `nullopt` | Nullable values |
| `absl/types/span` | `Span<T>` | Non-owning array views |
| `absl/types/variant` | `variant` | Tagged unions |
| `absl/synchronization/` | `Mutex`, `MutexLock`, `BlockingCounter` | Thread-safe access with static analysis |
| `absl/cleanup/` | `Cleanup` | Scope-exit cleanup (like Go `defer`) |
| `absl/hash/` | `Hash`, `HashOf` | Universal hashing for custom types |
| `absl/base/` | `call_once`, `PREDICT_TRUE/FALSE` | One-time init, branch hints |
| `absl/numeric/` | `int128`, `uint128` | 128-bit integers for IPv6 math |

---

## 13. Abseil vs Standard Library: When to Choose

| Situation | Choose |
|---|---|
| Simple hash map, no pointer stability needed | `absl::flat_hash_map` over `std::unordered_map` |
| Hash map where you store iterators/pointers to elements | `absl::node_hash_map` |
| Ordered map | `absl::btree_map` over `std::map` |
| Optional value | `absl::optional` (same as `std::optional` in C++17) |
| String parameter (read-only) | `absl::string_view` |
| String concatenation | `absl::StrCat` |
| Error return value | `absl::StatusOr<T>` |
| Mutex | `absl::Mutex` (adds thread-safety annotations) |
| Scope cleanup | `absl::Cleanup` |
| Non-owning array view | `absl::Span<T>` |

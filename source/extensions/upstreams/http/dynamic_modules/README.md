# Dynamic-Module HTTP-over-TCP Bridge Upstream

Router upstream that delegates HTTP<->TCP protocol translation to a loaded
dynamic module. The router speaks HTTP to this upstream; the upstream owns a
raw TCP connection, and a dynamic-module bridge decides how to marshal
headers/data/trailers onto the TCP wire and how to synthesize HTTP responses
from TCP bytes.

Proto: `envoy.extensions.upstreams.http.dynamic_modules.v3.Config`

## Files
- `upstream_request.h` / `upstream_request.cc` - `TcpConnPool` and
  `HttpTcpBridge` (the main `GenericUpstream`) declarations and logic.
- `abi_impl.cc` - implementations of the C ABI callbacks the module uses to
  push data upstream or synthesize downstream responses
  (`send_upstream_data`, `send_response`, `send_response_headers`,
  `send_response_data`, `send_response_trailers`, buffer accessors).
- `config.h` / `config.cc` - `DynamicModuleGenericConnPoolFactory`
  registered as `envoy.upstreams.http.dynamic_modules`; also a thread-local
  `BridgeConfig` cache keyed by config hash so
  `on_bridge_config_new` only runs once per unique config per worker.

## Interface
- `DynamicModuleGenericConnPoolFactory` implements
  `Router::GenericConnPoolFactory`.
- `TcpConnPool` implements `Router::GenericConnPool` and
  `Envoy::Tcp::ConnectionPool::Callbacks`.
- `HttpTcpBridge` implements `Router::GenericUpstream` and
  `Envoy::Tcp::ConnectionPool::UpstreamCallbacks`.

## Logic
1. Factory (`config.cc:60`): look up (or create) a `BridgeConfig` for the
   proto via `getOrCreateBridgeConfig`, which loads the module, resolves
   each `envoy_dynamic_module_on_upstream_http_tcp_bridge_*` function, and
   calls `on_bridge_config_new`. Cached per thread by proto hash.
   Then build a `TcpConnPool` with that config.
2. `TcpConnPool::newStream`: acquire a TCP connection from the cluster's
   thread-local TCP pool; on success, build an `HttpTcpBridge` and call
   `onPoolReady`.
3. `HttpTcpBridge::encodeHeaders` / `encodeData` / `encodeTrailers` do not
   write to the TCP socket directly - instead they call the module's
   `on_bridge_encode_headers` / `on_bridge_encode_data` /
   `on_bridge_encode_trailers` hooks. The module inspects the buffers via
   the ABI accessors (`requestHeaders`, `requestBuffer`) and then decides
   whether to call `sendUpstreamData` to actually push bytes to TCP.
4. `onUpstreamData` dispatches to `on_bridge_on_upstream_data`. The module
   decides when to produce a downstream response by calling one of
   `sendResponse`, `sendResponseHeaders`, `sendResponseData`, or
   `sendResponseTrailers` - these are the only paths that reach
   `upstream_request_->decodeHeaders/decodeData/decodeTrailers` on the
   router side.
5. `onEvent`/watermark callbacks propagate the usual TCP connection events
   up the router chain.

Lifetime: the per-stream in-module object (`in_module_bridge_`) is created
in the constructor via `on_bridge_new` and torn down via `on_bridge_destroy`
in the destructor; the per-config object is torn down in `BridgeConfig`'s
destructor.

## Key decision points
- `config.cc:22-56` - the per-thread bridge-config cache.
- `upstream_request.cc:32-71` - resolution of every ABI symbol during
  `BridgeConfig::create`.
- `upstream_request.h:110-161` - ABI surface exposed to the module (flow is
  explicit: no return-code "iteration" semantics).

## Configuration
`dynamic_modules.v3.Config`:
- `dynamic_module_config` - module name + `do_not_close`, `load_globally`.
- `bridge_name` - identifies which bridge inside the module to instantiate.
- `bridge_config` - opaque `google.protobuf.Any` passed to
  `on_bridge_config_new` as a byte buffer.

## Stats
None directly; any accounting is the module's responsibility.

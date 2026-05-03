# Internal Listener Bootstrap Extension

Provides the server-wide plumbing required for Envoy's "internal listener"
feature, which lets one listener hand off virtual connections to another
listener in-process (zero-copy, no sockets). This bootstrap extension
owns the `InternalListenerRegistry` singleton and exposes a thread-local
registry so workers can look up peer listeners without cross-thread
coordination.

Proto: `envoy.extensions.bootstrap.internal_listener.v3.InternalListener`.
Factory name: `envoy.bootstrap.internal_listener`.

## Files
- `internal_listener_registry.h/cc` -
  `TlsInternalListenerRegistry` (singleton wrapper around the thread-local
  slot), `InternalListenerExtension` (bootstrap extension), and
  `InternalListenerFactory` (registration).
- `thread_local_registry.h/cc` - `ThreadLocalRegistryImpl` implementing
  `Network::LocalInternalListenerRegistry` per worker.
- `active_internal_listener.h/cc` - `ActiveInternalListener` subclass of
  `Server::OwnedActiveStreamListenerBase` that handles accepted
  internal sockets.
- `client_connection_factory.h/cc` - `InternalClientConnectionFactory`
  (named `envoy_internal`) that creates a client side of an internal
  connection pair.

## Interface
- Bootstrap extension base: `Server::BootstrapExtension`.
- Factory base: `Server::Configuration::BootstrapExtensionFactory`.
- Registry base: `Network::InternalListenerRegistry` /
  `Network::LocalInternalListenerRegistry`.
- Client connection factory base: `Network::ClientConnectionFactory`
  (name `envoy_internal`).
- Active listener base: `Network::InternalListener` plus
  `Server::OwnedActiveStreamListenerBase`.

## Logic
- `InternalListenerExtension` registers a singleton via
  `SINGLETON_MANAGER_REGISTRATION(internal_listener_registry)` during
  construction so listeners created before server initialization can
  still find it (`internal_listener_registry.cc:27`).
- On `onServerInitialized`, the extension creates the
  `TypedSlot<ThreadLocalRegistryImpl>` and publishes it to the static
  `InternalClientConnectionFactory::registry_tls_slot_` along with the
  configured buffer size.
- `ThreadLocalRegistryImpl::setInternalListenerManager` is invoked by
  the per-silo `ConnectionHandlerImpl` when the first internal listener
  is added through LDS, binding the worker's listener manager into the
  registry.
- `ActiveInternalListener` inherits from
  `OwnedActiveStreamListenerBase`; `disable`/`enable` are no-ops
  because internal listeners are not driven by OS IO events. Accepting
  a socket goes straight through `onAccept` -> `newActiveConnection`.
- `InternalClientConnectionFactory::createClientConnection` looks up the
  destination listener in the thread-local registry and builds an
  in-memory connection pair using `buffer_size_`.

## Key decision points
- `internal_listener_registry.cc:21` - `buffer_size_kb` defaults to
  `InternalClientConnectionFactory::DefaultBufferSize = 1024`, i.e.
  1 MiB per side of the internal connection.
- `active_internal_listener.h:40` - `disable`/`enable` are deliberately
  no-ops since there is no OS accept queue to pause; the TODO notes a
  future user-space accept queue.
- `active_internal_listener.h:56` - `getBalancedHandlerByAddress`
  PANICs because internal listeners cannot migrate connections across
  workers in the current implementation.
- `client_connection_factory.h:36` - the TLS slot pointer is a static
  member because the factory has no owning lifetime; ownership lives
  with the bootstrap extension singleton.

## Configuration
- `buffer_size_kb` - per-direction buffer size (in KiB) for internal
  client connections. Default 1024.

## Stats / errors
- No counters specific to this extension; listener and filter chain
  stats come from the wrapped listener infrastructure.

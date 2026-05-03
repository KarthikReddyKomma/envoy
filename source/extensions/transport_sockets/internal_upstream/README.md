# Internal Upstream Transport Socket (`envoy.transport_sockets.internal_upstream`)

Upstream-only transport that targets Envoy's user-space "internal listener" IO handles. It wraps any inner upstream transport (typically `raw_buffer`) and, at `setTransportSocketCallbacks` time, pushes passthrough metadata and filter-state objects from the downstream connection into the user-space `PassthroughState` attached to the internal IO handle. This is the mechanism that lets filter chains on an internal listener see metadata/filter-state from the original downstream connection.

Proto: `envoy.extensions.transport_sockets.internal_upstream.v3.InternalUpstreamTransport`.

## Files
- `config.h/cc` — registers `InternalUpstreamConfigFactory`. Parses `passthrough_metadata` entries (host/cluster metadata selectors), builds the stats struct (`internal_upstream.no_metadata`), and wraps the inner transport factory.
- `internal_upstream.h/cc` — defines `InternalSocket`, the `PassthroughSocket` subclass that injects the extracted metadata and filter-state objects into the `IoSocket::UserSpace::IoHandle::passthroughState()`.

## Transport socket role
Extends `PassthroughSocket`. Only overrides `setTransportSocketCallbacks`:
- `internal_upstream.cc:14-22` — after forwarding callbacks to the inner socket, it `dynamic_cast`s the IO handle to `IoSocket::UserSpace::IoHandle`. If non-null and `passthroughState()` is set, it calls `initialize(metadata, filter_state_objects)`, transferring ownership of both to the user-space state. Fields are cleared so they are initialized exactly once.

The `InternalSocketFactory` (`config.h:45`) extends `PassthroughFactory` and in `createTransportSocket` it builds the inner socket, then extracts metadata via `Config::extractMetadata(host)` and filter-state objects from `options->downstreamSharedFilterStateObjects()`, passing both into the `InternalSocket`.

## Lifecycle
- Connect path: no handshake; metadata/filter-state is pushed into `PassthroughState` as soon as callbacks are attached (before any I/O).
- Data path: identical to the inner socket (usually `raw_buffer`).
- Close path: delegated to the inner socket.

## Key decision points
- `config.cc:51-71` — `Config` constructor validates `MetadataKind::Host` or `MetadataKind::Cluster`; other kinds throw `EnvoyException`.
- `config.cc:73-101` — `extractMetadata` merges the selected filter-metadata entries from the upstream host or cluster into a fresh `envoy::config::core::v3::Metadata`. Missing sources are logged and counted in `stats_.no_metadata_`.
- `config.cc:110-124` — `createTransportSocket` short-circuits if the inner factory produces `nullptr`, then bundles extracted metadata and the downstream shared filter-state objects into the `InternalSocket`.
- `internal_upstream.cc:16-21` — the `dynamic_cast` is the mechanism that distinguishes an internal (user-space) IO handle from a real OS socket; if the cast fails, the metadata injection is silently skipped.

## Configuration
- `transport_socket` — required; the inner upstream transport. Typically `raw_buffer`.
- `passthrough_metadata[]` — list of `{kind, name}` selectors. `kind` may be `Host` or `Cluster`; `name` is the filter-metadata namespace to copy from the resolved upstream host/cluster into the internal connection.

## Stats
Prefixed `internal_upstream.`:
- `no_metadata` (counter) — incremented each time a configured metadata source was missing on the host/cluster.

## Errors
Configuration-time `EnvoyException` on unsupported metadata kinds (`config.cc:66`). Runtime errors are deferred to the inner transport.

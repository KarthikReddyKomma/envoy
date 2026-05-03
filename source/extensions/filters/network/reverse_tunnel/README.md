# Reverse Tunnel (`envoy.filters.network.reverse_tunnel`)

Terminal L4 filter that accepts reverse-tunnel handshake requests from downstream agents. It parses a single HTTP/1 request on the connection, extracts the `node_id`, `cluster_id`, and `tenant_id` headers, optionally validates them against formatter-derived expected values, and — on success — duplicates the underlying socket and hands it to the upstream `UpstreamSocketManager` so that Envoy can initiate reverse-direction traffic on it afterwards. The filter replies with 200 OK on acceptance and 4xx on parse / validation failures.

Proto: `envoy.extensions.filters.network.reverse_tunnel.v3.ReverseTunnel`.

A subdirectory `drain_aware_hcm/` ships a companion HCM variant used on the same listener; see that folder's README.

## Files
- `reverse_tunnel_filter.h/cc` — `ReverseTunnelFilterConfig`, the `ReverseTunnelFilter` itself, and its nested `RequestDecoderImpl` that bridges the HTTP/1 codec to the L4 filter.
- `config.h/cc` — factory `ReverseTunnelFilterConfigFactory`. Registers the filter as terminal (`true /* isTerminalFilter */`, config.h:24) and builds the shared config at `createFilterFactoryFromProtoTyped` (config.cc:11).
- `drain_aware_hcm/` — see subfolder README; separate filter name `envoy.filters.network.reverse_tunnel.drain_aware_hcm`.

## Lifecycle
- `onNewConnection()` (reverse_tunnel_filter.cc:240) — only logs; returns `Continue`. The filter does not make decisions until data arrives.
- `onData()` (reverse_tunnel_filter.cc:246) — lazily constructs an HTTP/1 server codec (`Http::Http1::ServerConnectionImpl`, reverse_tunnel_filter.cc:251-254), dispatches the buffer into it, and always returns `StopIteration` so the filter owns the connection. Codec errors close the connection with `FlushWrite` (reverse_tunnel_filter.cc:262).
- `initializeReadFilterCallbacks()` (reverse_tunnel_filter.cc:268) — stores callbacks.
- `newStream()` (reverse_tunnel_filter.cc:272) — `Http::ServerConnectionCallbacks` hook invoked by the codec on each request; creates a `RequestDecoderImpl` (one-shot) that buffers the request and triggers `processIfComplete` when end-stream is seen.

There is no `onWrite()`; this is a pure `ReadFilter`.

## Decision / logic
All top-level routing lives in `RequestDecoderImpl::processIfComplete` (reverse_tunnel_filter.cc:332):
1. Method/path gate — if method differs from configured `request_method_` or path does not match `request_path_`, reply 404 `reverse_tunnel_not_found` and close (reverse_tunnel_filter.cc:344-351). Default path comes from `ReverseConnectionUtility::DEFAULT_REVERSE_TUNNEL_REQUEST_PATH`; default method is `GET` (reverse_tunnel_filter.cc:142-148).
2. Header extraction — reads `reverseTunnelNodeIdHeader`, `reverseTunnelClusterIdHeader`, `reverseTunnelTenantIdHeader` (reverse_tunnel_filter.cc:354-360). Missing any header increments `parse_error`, replies 400, closes (reverse_tunnel_filter.cc:361-370).
3. Tenant-isolation delimiter check — when the bootstrap-level `UpstreamSocketManager` has tenant isolation enabled, reject identifiers containing `ReverseConnectionUtility::TENANT_SCOPE_DELIMITER` (reverse_tunnel_filter.cc:382-403). Also bumps `parse_error`.
4. Required upstream cluster name — if `required_cluster_name_` is non-empty, verify the `reverseTunnelUpstreamClusterNameHeader`: missing header bumps `parse_error`, mismatch bumps `validation_failed`; both reply 400 (reverse_tunnel_filter.cc:406-434).
5. Identifier validation via substitution-format — `ReverseTunnelFilterConfig::validateIdentifiers` (reverse_tunnel_filter.cc:161) formats the configured `node_id_format` / `cluster_id_format` / `tenant_id_format` against the stream info and compares against the header-supplied values; mismatch bumps `validation_failed` and replies 403 (reverse_tunnel_filter.cc:445-454).
6. Optional metadata emission — `emitValidationMetadata` writes `{node_id, cluster_id, tenant_id, validation_result}` to dynamic metadata under `dynamic_metadata_namespace_` (default `envoy.filters.network.reverse_tunnel`, reverse_tunnel_filter.cc:156-158) when configured.
7. Success path — reply 200 (reverse_tunnel_filter.cc:457-459), call `processAcceptedConnection`, bump `accepted`, and close if `auto_close_connections_` (reverse_tunnel_filter.cc:465-469).

`processAcceptedConnection` (reverse_tunnel_filter.cc:472) resolves the thread-local `UpstreamSocketManager` via `getThreadLocalSocketManager()` (reverse_tunnel_filter.cc:31), duplicates the connection's `IoHandle` (reverse_tunnel_filter.cc:500), wraps it in a fresh `ConnectionSocketImpl` preserving local/remote addresses (reverse_tunnel_filter.cc:507-509), resets file events (reverse_tunnel_filter.cc:512), and hands the socket to `addConnectionSocket(node, cluster, socket, ping_seconds, false)` (reverse_tunnel_filter.cc:534). When tenant isolation is on, the node/cluster keys are rewritten through `buildTenantScopedIdentifier` (reverse_tunnel_filter.cc:522-531). Finally, the upstream extension's `reportConnection` is invoked (reverse_tunnel_filter.cc:540-543).

## Configuration
- `ping_interval` — default 2000 ms; passed to the socket manager as seconds (reverse_tunnel_filter.cc:131-134, 515).
- `auto_close_connections` — close the handshake socket after handing off (reverse_tunnel_filter.cc:135-136, 465).
- `request_path` — default `ReverseConnectionUtility::DEFAULT_REVERSE_TUNNEL_REQUEST_PATH` (reverse_tunnel_filter.cc:137-141).
- `request_method` — `RequestMethod` enum; `METHOD_UNSPECIFIED` becomes `GET` (reverse_tunnel_filter.cc:142-148).
- `required_cluster_name` — when non-empty, the `x-envoy-reverse-tunnel-upstream-cluster-name` header must match (reverse_tunnel_filter.cc:159, 406).
- `validation.node_id_format` / `cluster_id_format` / `tenant_id_format` — `SubstitutionFormatString` inline strings compiled into `Formatter` instances (reverse_tunnel_filter.cc:69-118).
- `validation.emit_dynamic_metadata` + `validation.dynamic_metadata_namespace` — controls the metadata sink (reverse_tunnel_filter.cc:152-158).

## Stats
Counters emitted under prefix `reverse_tunnel.handshake.` (reverse_tunnel_filter.cc:238):
- `parse_error` — missing required headers, delimiter violation, or missing cluster-name header.
- `accepted` — successful handshake.
- `rejected` — reserved counter, declared in `ALL_REVERSE_TUNNEL_HANDSHAKE_STATS` (reverse_tunnel_filter.h:117-121) but not incremented by current code paths.
- `validation_failed` — cluster-name mismatch or formatter-based identifier mismatch (reverse_tunnel_filter.cc:424, 446).

## Factory
`ReverseTunnelFilterConfigFactory` (config.h:17) extends `Common::ExceptionFreeFactoryBase`, is registered via `REGISTER_FACTORY` as a `NamedNetworkFilterConfigFactory` (config.cc:33), and is marked terminal in its constructor (config.h:22-24). `createFilterFactoryFromProtoTyped` (config.cc:11) builds `ReverseTunnelFilterConfig::create(...)`, captures the listener `Stats::Scope` and server `OverloadManager` pointers, and returns a lambda that adds a new `ReverseTunnelFilter` as a `ReadFilter` (config.cc:24-27).

# Proxy Protocol Transport Socket (`envoy.transport_sockets.upstream_proxy_protocol`)

Upstream-only transport that prepends a PROXY protocol v1 or v2 header before any application bytes, so the upstream server sees the true source/destination addresses from the downstream connection. Wraps another upstream transport (typically `raw_buffer` or `tls`).

Proto: `envoy.extensions.transport_sockets.proxy_protocol.v3.ProxyProtocolUpstreamTransport` (contains a `ProxyProtocolConfig` plus the inner `transport_socket`).

## Files
- `config.h/cc` — registers `UpstreamProxyProtocolSocketConfigFactory`. Wraps the inner transport factory and builds per-factory stats.
- `proxy_protocol.h/cc` — defines `UpstreamProxyProtocolSocket` (the `PassthroughSocket` subclass) and `UpstreamProxyProtocolSocketFactory`. Builds V1 or V2 headers and manages TLV merging.

## Transport socket role
Extends `PassthroughSocket`. Overrides:
- `setTransportSocketCallbacks` (`proxy_protocol.cc:51`) — records callbacks for later use by header generation.
- `onConnected` (`proxy_protocol.cc:144`) — generates the header into `header_buffer_` immediately, then forwards.
- `doWrite` (`proxy_protocol.cc:57`) — while `header_buffer_` has bytes, writes them first; once drained, forwards to the inner socket. Mirrors bytes_processed across the two writes.

`UpstreamProxyProtocolSocketFactory` extends `PassthroughFactory` and additionally:
- `hashKey` (`proxy_protocol.cc:166`) — includes the case-insensitive hash of `proxy_protocol_options.asStringForHash()` so connections with different source/dest addresses are pooled separately.

## Lifecycle
- Connect path: on `onConnected`, `generateHeader()` runs; V1 formats IPv4/IPv6 lines, V2 writes the binary signature + addr block + TLVs.
- Data path: first `doWrite` call drains the header buffer; subsequent writes are plain passthrough. Reads are always passthrough.
- Close path: delegated to the inner socket.

## Key decision points
- `proxy_protocol.cc:29-49` — constructor parses `pass_through_tlvs` (`INCLUDE` vs `INCLUDE_ALL`) and materializes statically added TLVs.
- `proxy_protocol.cc:78-91` — V1 generation prefers `options_->proxyProtocolOptions()` for source/dest addresses; falls back to the local/remote addresses of the current connection (e.g. for health checks).
- `proxy_protocol.cc:100-116` — V2 generation; when no options are present, emits a LOCAL command. Otherwise calls `Common::ProxyProtocol::generateV2Header` with merged TLVs, incrementing `v2_tlvs_exceed_max_length_` if truncation was required.
- `proxy_protocol.cc:180-249` — `buildCustomTLVs`: merges host-level typed filter metadata (`envoy.transport_sockets.proxy_protocol` → `PerHostConfig`) with the config-level `added_tlvs_`. Host TLVs take precedence. The runtime feature `envoy.reloadable_features.proxy_protocol_allow_duplicate_tlvs` controls whether duplicate TLV types are allowed or the later occurrence is skipped.
- `proxy_protocol.cc:118-142` — `writeHeader` loops on `ioHandle().write(header_buffer_)` until drained, translating `Again` to `KeepOpen` and other errors to `Close`.

## Configuration
From `ProxyProtocolConfig`:
- `version` — `V1` (ASCII) or `V2` (binary).
- `pass_through_tlvs` — `INCLUDE_ALL` or an explicit include-set of TLV types (V2 only).
- `added_tlvs[]` — static `{type, value}` TLVs the filter always emits.

Per-host overrides: metadata filter `envoy.transport_sockets.proxy_protocol` typed as `envoy.config.core.v3.PerHostConfig`, which can contribute additional `added_tlvs` with priority over the config-level list.

Source/destination addresses come from `TransportSocketOptions::proxyProtocolOptions()`; if absent, the current connection's own local/remote addresses are used.

## Stats
Prefixed `upstream.proxyprotocol.`:
- `v2_tlvs_exceed_max_length` (counter) — incremented when combined TLVs overflow the V2 header and are truncated (also logged as a warning by `Common::ProxyProtocol::generateV2Header`).

## Errors
Header write failures close the connection. Malformed host-metadata is logged and the config-level TLVs are still emitted.

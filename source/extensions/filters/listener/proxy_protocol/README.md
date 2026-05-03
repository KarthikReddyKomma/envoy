# PROXY Protocol Listener Filter (`envoy.filters.listener.proxy_protocol`)

Consumes a HAProxy PROXY protocol v1 or v2 preamble from the front of each accepted connection, extracts the real remote/local addresses, and restores them onto the `ConnectionSocket`. Supports TCP4/TCP6/UNKNOWN for v1 and TCP/UDP over IPv4/IPv6 for v2 (AF_UNIX in v2 is rejected). Parses v2 TLV extensions into either dynamic metadata (default) or filter state, and always exposes the raw TLV vector plus the detected version via `Network::ProxyProtocolFilterState`. Supports rejecting a specific version and allowing pass-through when no preamble is present (for trusted downstreams).

Proto: `envoy.extensions.filters.listener.proxy_protocol.v3.ProxyProtocol`.

## Files
- `proxy_protocol.h` / `proxy_protocol.cc` — `Filter`, `Config`, the three stats structs (`LegacyProxyProtocolStats`, `GeneralProxyProtocolStats`, `VersionedProxyProtocolStats`), and `TlvFilterStateObject` (TLV map stored in filter state, `proxy_protocol.cc:61`).
- `proxy_protocol_header.h` — `WireHeader` POD describing a parsed preamble (lengths, addresses, IP version).
- `config.cc` — `ProxyProtocolConfigFactory`: validates the proto, builds the shared `Config`, and registers under `envoy.filters.listener.proxy_protocol` with deprecated alias `envoy.listener.proxy_protocol` (`config.cc:43`, `config.cc:49`).

## Lifecycle
- `onAccept(cb)` — caches callbacks and immediately returns `StopIteration` to wait for data (`proxy_protocol.cc:227`).
- `onData(buffer)` — drives `parseBuffer`, then uses the detected `header_version_` to update per-version stats. Returns `Continue` on `Done`, `StopIteration` on `TryAgainLater`, and on `Error`/`Denied` it increments the legacy error counter, closes the socket, and returns `StopIteration` (`proxy_protocol.cc:234`-`proxy_protocol.cc:264`).
- `maxReadBytes()` — returns `max_proxy_protocol_len_`; starts at `MAX_PROXY_PROTO_LEN_V2` (`PROXY_PROTO_V2_HEADER_LEN + PROXY_PROTO_V2_ADDR_LEN_UNIX`) and grows when v2 TLV extensions push `wholeHeaderLength()` past the current cap (`proxy_protocol.cc:284`).

## Decision / logic
- `readProxyHeader` (`proxy_protocol.cc:709`) first short-circuits when `allow_requests_without_proxy_protocol` is true and the bytes match neither v1 nor v2 signature, returning `Done` without touching the socket. Otherwise it discriminates on the 12-byte v2 signature vs the 5-byte v1 signature `PROXY `.
- v2 path: `parseV2Header` (`proxy_protocol.cc:395`) interprets the `ver_cmd` byte; `LOCAL` commands leave the socket addresses untouched (health checks), `ONBEHALF_OF` with AF_INET/AF_INET6 + STREAM/DGRAM fills `remote_address_`/`local_address_` from the packed `pp_ipv4_addr` / `pp_ipv6_addr` structs.
- v1 path: `parseV1Header` (`proxy_protocol.cc:503`) splits on spaces, requires the `PROXY` token followed by `TCP4`/`TCP6`/`UNKNOWN`, and parses `src dst src_port dst_port`. `UNKNOWN` preserves the real socket addresses.
- Version denial: if `disallowed_versions` includes the detected version, the header is still parsed but `readProxyHeader` returns `Denied`, which closes the socket (`proxy_protocol.cc:740`, `proxy_protocol.cc:794`).
- Address application (`proxy_protocol.cc:335`-`proxy_protocol.cc:363`): after parsing, the filter validates that the address family matches `protocol_version_` and that both addresses are unicast. Then, if the parsed local address differs from the current socket local address, it calls `socket.connectionInfoProvider().restoreLocalAddress(...)`; it always calls `setRemoteAddress(remote_address_)`. Local commands skip all of this.
- Filter state: a `Network::ProxyProtocolFilterState` is written at connection lifespan with the full `ProxyProtocolDataWithVersion` (remote, local, parsed TLVs, version) either from the parsed header or, for local commands, from the current socket addresses (`proxy_protocol.cc:297`-`proxy_protocol.cc:333`).
- TLV handling (`parseTlvs`, `proxy_protocol.cc:575`): for types listed in `rules`, the value is sanitized and either written to typed/untyped dynamic metadata at the configured namespace (default `envoy.filters.listener.proxy_protocol`) or to a `TlvFilterStateObject` at key `envoy.network.proxy_protocol.tlv` when `tlv_location == FILTER_STATE`. TLVs covered by `pass_through_tlvs` are appended to `parsed_tlvs_` so they are carried in the filter-state entry.
- Finally, `buffer.drain(wholeHeaderLength())` removes the preamble so the downstream filters never see it (`proxy_protocol.cc:365`).

## Configuration
- `rules[]` — per-`tlv_type`: `on_tlv_present.key` / `metadata_namespace` drives which TLVs are emitted (`proxy_protocol.cc:171`).
- `pass_through_tlvs` — `INCLUDE_ALL` or a whitelist of TLV types stored in `pass_through_tlvs_` (`proxy_protocol.cc:175`).
- `allow_requests_without_proxy_protocol` — skip the filter for non-PROXY traffic from trusted peers.
- `disallowed_versions[]` — toggles `allow_v1_` / `allow_v2_`; rejecting both is invalid and throws `ProtoValidationException` (`proxy_protocol.cc:195`).
- `tlv_location` — `DYNAMIC_METADATA` (default) or `FILTER_STATE`.
- `stat_prefix` — extra segment inserted between `proxy_proto.` and the per-stat name.

## Stats
Produced by `ProxyProtocolStats::create` (`proxy_protocol.cc:109`):
- Legacy (listener scope): `downstream_cx_proxy_proto_error`.
- General (under `proxy_proto.[stat_prefix.]`): `not_found_allowed`, `not_found_disallowed`.
- Per version (under `proxy_proto.[stat_prefix.]versions.{v1,v2}.`): `found`, `disallowed`, `error`.

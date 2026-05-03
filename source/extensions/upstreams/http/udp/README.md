# HTTP-over-UDP (CONNECT-UDP) Upstream

Router upstream implementation for CONNECT-UDP, which tunnels HTTP/3 Datagram
capsules over a UDP socket. Creates one `Network::SocketImpl` per stream and
translates capsules to UDP packets in both directions.

Proto: `envoy.extensions.upstreams.http.generic.v3.GenericConnectionPoolProto`

## Files
- `upstream_request.h` - `UdpConnPool`, `UdpUpstream` declarations.
- `upstream_request.cc` - socket creation, capsule<->datagram translation,
  downstream handshake.
- `config.h` / `config.cc` - module-local config glue.

## Interface
- `UdpConnPool` implements `Router::GenericConnPool`.
- `UdpUpstream` implements `Router::GenericUpstream`,
  `Network::UdpPacketProcessor`, and `quiche::CapsuleParser::Visitor`.

## Logic
1. `UdpConnPool::newStream(callbacks)` (`upstream_request.cc:28`):
   - `createSocket(host_)` - allocate a datagram socket bound to the host's
     address.
   - Apply upstream local-address options (`PREBIND` socket options) and
     bind to the selector's chosen local address.
   - Construct a `UdpUpstream`, build a new `StreamInfoImpl`, and
     `callbacks->onPoolReady(...)`. `cancelAnyPendingStream` is a no-op
     because the UDP upstream completes synchronously (no handshake).
2. `UdpUpstream` constructor registers a file event on the socket; the read
   handler calls `onSocketReadReady` which decodes UDP datagrams and feeds
   them to the downstream as HTTP/3 DATAGRAM capsules.
3. `encodeHeaders` synthesizes a downstream `:status 200` +
   `capsule-protocol: ?1` response so the CONNECT-UDP handshake completes.
4. `encodeData` fans bytes through `CapsuleParser::IngestCapsuleFragment`;
   the visitor callback `OnCapsule` translates each capsule into a UDP
   datagram and writes it to the socket.
5. `processPacket` (UdpPacketProcessor) encodes received UDP packets back
   into capsules and writes them to the downstream via the shared
   `UpstreamToDownstream`.

## Key decision points
- `upstream_request.cc:28-52` - socket creation + bind, and the fast-path
  `onPoolReady` without a pending stream handle.
- `upstream_request.h:29-58` - `UdpConnPool` semantics (no pending streams,
  no cancel).
- `upstream_request.h:82-102` - `UdpPacketProcessor` hooks; dropped
  datagrams are logged but counted only in `datagrams_dropped_` locally.

## Configuration
Uses the cluster's `UpstreamLocalAddressSelector` for source-address
selection. No UDP-specific user config today beyond what the parent
`GenericConnectionPool` factory exposes.

## Stats
None yet (see TODO referencing
`https://github.com/envoyproxy/envoy/issues/23564`). A local
`datagrams_dropped_` counter is maintained for diagnostics.

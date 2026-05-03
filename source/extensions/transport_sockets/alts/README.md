# ALTS Transport Socket (`envoy.transport_sockets.alts`)

Implements Google's Application Layer Transport Security (ALTS). Performs a gRPC-TSI handshake with a configured ALTS handshaker service to establish mutual authentication and a per-connection frame protector (AES-128-GCM with rekeying), then encrypts/decrypts application bytes in place of TLS. Available as both upstream and downstream transports.

Proto: `envoy.extensions.transport_sockets.alts.v3.Alts` (fields: `handshaker_service` address, `peer_service_accounts[]` allow-list).

## Files
- `config.h/cc` — registers `UpstreamAltsTransportSocketConfigFactory` and `DownstreamAltsTransportSocketConfigFactory`. Holds `AltsSharedState` (a `Singleton::Instance` that owns an `AltsChannelPool` to the handshaker service and, on Google gRPC builds, a `GoogleGrpcContext`). Builds a `HandshakerFactory` closure that returns a `TsiHandshaker` per connection and a `HandshakeValidator` that checks the peer's service account against `peer_service_accounts`.
- `alts_channel_pool.h/cc` — `AltsChannelPool`: round-robin pool of gRPC channels to the ALTS handshaker service, created once per unique `handshaker_service_address` via Envoy's singleton manager.
- `alts_proxy.h/cc` — `AltsProxy`: owns a single bidirectional gRPC stream to `HandshakerService`. Sends `StartClientHandshakeReq`, `StartServerHandshakeReq`, and `NextHandshakeMessageReq` synchronously (blocks the worker thread for the duration of each RPC — noted in header comments).
- `alts_tsi_handshaker.h/cc` — `AltsTsiHandshaker`: state machine that drives the `AltsProxy` across handshake rounds. On success, produces an `AltsHandshakeResult` containing the peer identity, unused bytes, and a `TsiFrameProtector`.
- `tsi_handshaker.h/cc` — `TsiHandshaker`: Event-loop-aware wrapper around `AltsTsiHandshaker`. Serializes calls, posts results to the dispatcher, and implements `DeferredDeletable` so the handshaker can be torn down safely while an RPC is in flight.
- `tsi_frame_protector.h/cc` — thin C++ wrapper around gRPC's C `tsi_frame_protector` API providing `protect` and `unprotect` methods that operate on Envoy `Buffer::Instance` objects.
- `tsi_socket.h/cc` — `TsiSocket` (the `Network::TransportSocket` implementation) and `TsiSocketFactory`. Owns the inner `RawBufferSocket`, drives the handshake, then switches to frame-protected I/O.
- `noop_transport_socket_callbacks.h` — passthrough `TransportSocketCallbacks` used to decouple the inner raw socket's reads from upper-layer buffer signaling during the handshake. `TsiTransportSocketCallbacks` overrides `shouldDrainReadBuffer` to apply the handshake buffer watermark.
- `grpc_tsi.h` — type aliases (e.g. `CFrameProtectorPtr`) bridging gRPC's C types with `CSmartPtr`.

## Transport socket role (`TsiSocket`)
Directly implements `Network::TransportSocket` plus `TsiHandshakerCallbacks`:
- `doRead` (`tsi_socket.cc:236`) — before handshake: reads raw bytes into `raw_read_buffer_`, hands them to `doHandshakeNext`. After handshake: calls `repeatReadAndUnprotect` which calls `frame_protector_->unprotect` until the kernel buffer drains or the downstream watermark is reached.
- `doWrite` (`tsi_socket.cc:312`) — before handshake: queues into `raw_write_buffer_` only indirectly (via `doHandshakeNextDone`) and returns `KeepOpen,0`. After handshake: `repeatProtectAndWrite` protects up to `actual_frame_size_to_use_ - frame_overhead_size_` bytes per frame and flushes through the raw buffer socket. Short writes are resumed from `prev_bytes_to_drain_`.
- `onConnected` (`tsi_socket.cc:341`) — on upstream (`!downstream_`), kicks off `doHandshakeNext` (clients initiate). On downstream, waits for the first read.
- `closeSocket` (`tsi_socket.cc:334`) — `handshaker_.release()->deferredDelete()` to avoid destroying while an async handshake RPC may still fire.
- `onNextDone` (`tsi_socket.cc:349`) — handshake-state callback posted from the dispatcher.
- `canFlushClose` returns `handshake_complete_`. `ssl()` returns `nullptr` (ALTS is not TLS).

## Lifecycle
- Connect path: factory builds `AltsSharedState` via the singleton manager, creates a `TsiHandshaker` wrapping a client or server `AltsTsiHandshaker`. `doHandshakeNext` calls into `AltsTsiHandshaker::next` which issues `StartClient/ServerHandshakeReq` or `NextHandshakeMessageReq` via `AltsProxy`. Multiple round-trips are possible; between them, handshake bytes ride on `raw_write_buffer_` / `raw_read_buffer_`.
- Handshake completion: `doHandshakeNextDone` (`tsi_socket.cc:91`) validates the peer via `handshake_validator_` (allow-list check against `peer_service_accounts_`), stores `peer_identity` in dynamic metadata under `envoy.transport_sockets.peer_information`, grafts any `unused_bytes` back onto `raw_read_buffer_`, updates watermarks to the negotiated frame size, installs the `frame_protector_`, sets `handshake_complete_ = true`, and raises `ConnectionEvent::Connected`.
- Data path: `protect`/`unprotect` per frame. `actual_frame_size_to_use_` comes from the handshake result; overhead is 20 bytes (4-byte type + 16-byte GCM tag — matches gRPC's zero-copy protector).
- Close path: `deferredDelete` for the handshaker (it may have a pending gRPC call); `TsiSocket` itself owns no other async state.

## Key decision points
- `config.cc:93-97` — the `AltsSharedState` singleton is keyed by the registration name only, so all config blocks share the same channel pool regardless of address; the address used is the *first* one that triggered creation. (A subtle constraint worth noting when multiple ALTS configs coexist.)
- `config.cc:98-111` — `HandshakerFactory` closure captures `alts_shared_state` by value to keep the singleton alive for the lifetime of the `TsiSocketFactory`.
- `tsi_socket.cc:108-131` — peer validation: if `handshake_validator_` is set and the peer service account is not in the allow-list, the connection is closed. The peer identity is always written to dynamic metadata on success.
- `tsi_socket.cc:263-310` — `repeatProtectAndWrite` loops to consume the write buffer in frame-sized chunks, handling partial writes by remembering `prev_bytes_to_drain_` so the next invocation resumes without re-protecting.
- `tsi_socket.h:108-116` — `default_max_frame_size_` = 16 KiB initial watermark before negotiation; post-handshake, the watermark becomes `max(actual_frame_size, connection.bufferLimit())`.

## Configuration
- `handshaker_service` — address (host:port) of the ALTS handshaker gRPC service. Typically `metadata.google.internal:8080` inside GCE.
- `peer_service_accounts[]` — optional allow-list; when non-empty, the handshake is rejected for any peer not in the list. When empty, no validation is performed (all authenticated peers are accepted).

## Stats
None directly from this extension. Handshake failures close the connection with reason `tsi_handshake_failed` (`tsi_socket.cc:354`) and `failed_creating_handshaker` (`tsi_socket.cc:73`).

## Errors / warnings
- Handshake RPC errors bubble up as `absl::Status` through `AltsTsiHandshaker::next` → `TsiHandshaker::onNextDone` → `TsiSocket::onNextDone`, which closes the connection.
- `AltsProxy` methods are synchronous and block the worker thread — explicitly flagged in the header (`alts_proxy.h:30-33`). A slow or unavailable handshaker service will stall the worker.

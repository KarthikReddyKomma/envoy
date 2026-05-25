# Overview, Part 2 — Listener, dispatcher, session, and connection

> *Read [`OVERVIEW_PART1`](OVERVIEW_PART1_architecture_and_layering.md) first.*

This part walks **L1 → L2 → L3**:

1. Where does the UDP listener come from?
2. How does the **dispatcher** demux a packet to the right connection?
3. How is a brand‑new **session** created when a `CHLO` arrives?
4. What are the **client‑side** counterparts (which look very different)?

## Server side — the path from socket to session

### Block diagram

```mermaid
flowchart LR
  subgraph Sockets["Per-listener / per-worker UDP socket(s)"]
    SOCK["Network::Socket<br/>(SO_REUSEPORT + BPF)"]
  end

  subgraph Listener["ActiveQuicListener (1 per worker, 1 per listener)"]
    UDP["Network::UdpListenerImpl<br/>(libevent read event)"]
    AQL["ActiveQuicListener<br/>- onDataWorker<br/>- onReadReady<br/>- onWriteReady"]
    CRYP["quic::QuicCryptoServerConfig<br/>(holds ProofSource)"]
    PSRC["EnvoyQuicProofSource"]
    PW["EnvoyQuicPacketWriter<br/>or UdpGsoBatchWriter"]
    HLP["EnvoyQuicConnectionHelper"]
    ALMF["EnvoyQuicAlarmFactory"]
    DISP["EnvoyQuicDispatcher<br/>(quic::QuicDispatcher)"]
    CIDGEN["EnvoyDeterministicConnectionIdGenerator"]
    SIL["Http::SessionIdleList<br/>(optional)"]
  end

  subgraph Per_connection["Per-connection (created on first valid CHLO)"]
    SESS["EnvoyQuicServerSession<br/>= Network::Connection"]
    CONN["EnvoyQuicServerConnection<br/>(quic::QuicConnection)"]
    LFM["QuicListenerFilterManagerImpl"]
  end

  SOCK --> UDP
  UDP --> AQL
  AQL --> DISP
  AQL --> CRYP
  AQL --> PW
  AQL --> HLP
  AQL --> ALMF
  AQL --> CIDGEN
  AQL --> SIL
  CRYP --> PSRC
  DISP -.->|"CreateQuicSession"| SESS
  SESS --> CONN
  CONN --> LFM
```

### Who creates whom, when?

```mermaid
sequenceDiagram
  autonumber
  participant Cfg as ActiveQuicListenerFactory<br/>(per-listener, server thread)
  participant Wk as Worker thread
  participant AQL as ActiveQuicListener
  participant Disp as EnvoyQuicDispatcher
  participant PS as EnvoyQuicProofSource

  Note over Cfg: At server startup
  Cfg->>Cfg: parse QuicProtocolOptions<br/>build crypto stream factory<br/>build proof-source factory<br/>build CID generator factory
  Cfg->>Wk: createActiveUdpListener(per worker)

  Wk->>AQL: ctor(runtime, worker_idx, dispatcher, ...)
  AQL->>AQL: random_seed_ := QuicRandom::RandBytes
  AQL->>PS: proof_source_factory.createQuicProofSource(<br/>  listen_socket_, filter_chain_manager, ...)
  AQL->>AQL: crypto_config_ := QuicCryptoServerConfig(<br/>  random_seed_, PS, KeyExchangeSource::Default)
  AQL->>AQL: connection_helper_ := EnvoyQuicConnectionHelper(dispatcher)
  AQL->>AQL: alarm_factory_ := EnvoyQuicAlarmFactory(dispatcher, clock)
  AQL->>Disp: new EnvoyQuicDispatcher(<br/>  crypto_config_, quic_config, version_manager,<br/>  helper, alarm_factory, ...,<br/>  crypto_server_stream_factory, cid_generator)
  AQL->>AQL: build packet writer via UdpListenerConfig::packetWriterFactory()
  AQL->>Disp: InitializeWithWriter(packet_writer)
```

The factory runs **once** in the listener-manager (main) thread. The listener itself is constructed **once per worker** when `Worker::addListener()` calls `createActiveUdpListener`. The dispatcher is per‑listener‑per‑worker; the proof source and crypto config are too.

### Receiving a packet

`Network::UdpListenerImpl` registers a libevent read event on the UDP socket. When the event fires:

```mermaid
sequenceDiagram
  autonumber
  participant Ev as libevent
  participant UDP as UdpListenerImpl
  participant AQL as ActiveQuicListener
  participant Disp as EnvoyQuicDispatcher
  participant Conn as Existing<br/>EnvoyQuicServerConnection
  participant CHLO as buffered_packets_<br/>(in QuicDispatcher)

  Ev->>UDP: fd readable
  UDP->>UDP: recvmmsg() up to N packets
  loop each packet
    UDP->>AQL: destination(data) -> worker index
    alt destination == this worker
      UDP->>AQL: onDataWorker(UdpRecvData)
      AQL->>Disp: processPacket(self, peer, QuicReceivedPacket)
      alt CID matches existing
        Disp->>Conn: ProcessUdpPacket(...)
      else CID unknown (first packet)
        Disp->>CHLO: BufferEarlyArrivingPackets(...)
        AQL->>UDP: activateRead()<br/>(reschedule for ProcessBufferedChlos)
      end
    else destination != this worker
      UDP->>UDP: forward to correct worker
    end
  end
  AQL->>UDP: onReadReady() at end of event loop
  AQL->>Disp: ProcessBufferedChlos(kNumSessionsToCreatePerLoop)
  Disp->>Disp: CreateQuicSession()<br/>(see "Creating a session")
```

Two things to notice:

- `processPacket()` returns `bool`. False means QUICHE couldn't dispatch (e.g. unknown CID, invalid version, stateless reset issued). `ActiveQuicListener::onDataWorker()` then forwards the raw packet to the optional `NonDispatchedUdpPacketHandler` — used during hot restart so the *new* Envoy can take over inflight packets that miss its own CID table.
- New sessions are created **in batches** (`kNumSessionsToCreatePerLoop = 16`) by `ProcessBufferedChlos()` at the *end* of the event loop, not in the middle of `onDataWorker`. This bounds the time spent per event loop and keeps tail latency predictable under SYN‑flood‑style CHLO bursts.

### Creating a session (the L1 → L3 jump)

```mermaid
sequenceDiagram
  autonumber
  participant Disp as EnvoyQuicDispatcher
  participant PS as EnvoyQuicProofSource
  participant FCM as FilterChainManager
  participant TSF as QuicServerTransportSocketFactory
  participant SCF as EnvoyQuicCryptoServerStreamFactory
  participant Sess as EnvoyQuicServerSession
  participant Conn as EnvoyQuicServerConnection
  participant Hsk as EnvoyTlsServerHandshaker

  Disp->>Disp: ProcessChlo(ParsedClientHello chlo)
  Disp->>Disp: CreateQuicSession(cid, self, peer, alpn, version, chlo, generator)
  Note over Disp: Parses SNI from chlo, hands it<br/>+ self/peer addr to ProofSource

  Disp->>PS: GetCertChain(self, peer, hostname)
  PS->>FCM: findFilterChain(socket, stream_info)
  FCM-->>PS: const Network::FilterChain*
  PS->>TSF: getTlsCertificateAndKey(sni)
  TSF-->>PS: (cert_chain, private_key)
  PS-->>Disp: chain (+ EnvoyQuicProofSourceDetails(filter_chain))

  Disp->>Conn: new EnvoyQuicServerConnection(<br/>  cid, self, peer, helper, alarm_factory,<br/>  packet_writer, version, socket,<br/>  cid_generator, listener_filter_manager)
  Disp->>Sess: new EnvoyQuicServerSession(<br/>  quic_config, versions, connection,<br/>  visitor, helper, crypto_config,<br/>  certs_cache, dispatcher, ...,<br/>  crypto_server_stream_factory, stream_info)

  Sess->>Sess: Initialize()
  Sess->>Hsk: CreateQuicCryptoServerStream(crypto_config, certs_cache)
  Hsk->>Hsk: pin ServerContextImpl in SSL ex_data<br/>install session-ticket-key callback

  Note over Disp,Hsk: From here QUICHE drives TLS 1.3.<br/>See OVERVIEW_PART3.
```

After this dance, the session is fully wired:

- `EnvoyQuicServerConnection` knows its socket, its peer/self address, its CID, and the listener filter manager.
- `EnvoyQuicServerSession` is registered with QUICHE under that CID, so subsequent packets routed by the dispatcher arrive at `ProcessUdpPacket()`.
- The crypto stream is allocated and pinned to the matched filter chain's `ServerContextImpl`.

### Receiving subsequent packets (the steady state)

Once a session exists, packet flow shortcuts the buffered‑CHLO machinery:

```mermaid
flowchart LR
  UDP -->|"onDataWorker"| AQL
  AQL -->|"processPacket"| Disp
  Disp -->|"CID lookup"| Conn
  Conn -->|"ProcessUdpPacket"| QUICHE_decrypt
  QUICHE_decrypt -->|"per-frame"| Sess
  Sess -->|"OnStreamFrame"| Str["EnvoyQuicServerStream"]
```

The dispatcher and QUICHE core do all the framing / decryption / pacing internally. Envoy only sees the per‑frame callbacks (`OnStreamFrame`, `OnHeadersFrame`, `OnTrailersFrame`, `OnRstStream`, `OnStopSending`, ...). See [`OVERVIEW_PART4`](OVERVIEW_PART4_streams_codecs_http3.md).

### Sending packets (the write path)

QUICHE owns pacing and congestion control. Envoy only owns the **packet writer**.

```mermaid
sequenceDiagram
  participant Sess as EnvoyQuicServerSession
  participant Conn as EnvoyQuicServerConnection
  participant Pkt as quic::QuicPacketCreator
  participant PW as EnvoyQuicPacketWriter
  participant UPW as Network::UdpPacketWriter
  participant K as Kernel UDP

  Sess->>Conn: WriteOrBufferBody / SendStreamData
  Conn->>Pkt: assemble + encrypt
  Pkt->>PW: WritePacket(buffer, self, peer, params)
  alt write blocked
    PW-->>Pkt: WriteResult{status=WRITE_STATUS_BLOCKED}
    Pkt-->>Conn: stop, mark blocked
    Note over Conn: When writer becomes writable,<br/>onWriteReady() -> OnCanWrite()
  else ok
    PW->>UPW: writePacket(...)
    UPW->>K: sendmsg() / sendmmsg()
  end
```

For high throughput on Linux, the listener can be configured to use `UdpGsoBatchWriter`, which implements `quic::QuicPacketWriter` *directly* (no `EnvoyQuicPacketWriter` adapter). It coalesces up to ~64 KB of equally‑sized packets into one `sendmsg()` via `UDP_SEGMENT` GSO. See [`alarm_and_packet_io.md`](alarm_and_packet_io.md).

### Connection‑ID routing across workers

For `concurrency > 1`, the kernel BPF program (installed at listener creation) hashes the CID modulo `concurrency` to pick a worker. The CID is generated by `EnvoyDeterministicConnectionIdGenerator`, which **embeds the worker index in the first 4 bytes of the CID** so the BPF program can decode it. The same logic runs in userspace as a fallback when BPF is unavailable (`ActiveQuicListener::destination()`).

```mermaid
flowchart LR
  Pkt["Incoming datagram"] --> BPF["BPF: hash(CID) % concurrency"]
  BPF -->|"== worker 0"| W0["Worker 0 socket"]
  BPF -->|"== worker 1"| W1["Worker 1 socket"]
  W0 --> AQL0["ActiveQuicListener (worker 0)"]
  W1 --> AQL1["ActiveQuicListener (worker 1)"]
  AQL0 -->|"select_connection_id_worker_<br/>(userspace double-check)"| AQL0
  AQL1 -->|"same"| AQL1
```

If the userspace check disagrees with the kernel routing, the packet stays on the wrong worker (logged at error level). This only happens in the brief window during BPF setup.

---

## Client side — the path from `connect()` to session

The client side is symmetric in concept but very different in shape. There's no listener and no dispatcher: each client connection owns its **own** UDP socket and registers its **own** read event.

```mermaid
flowchart LR
  subgraph PCQI["PersistentQuicInfoImpl (per cluster)"]
    HLP["EnvoyQuicConnectionHelper"]
    ALMF["EnvoyQuicAlarmFactory"]
    QC["quic::QuicConfig"]
    CSF["EnvoyQuicCryptoClientStreamFactoryImpl"]
    MIG["QuicConnectionMigrationConfig"]
    WF["QuicClientPacketWriterFactory"]
  end

  subgraph PerConnection["Per upstream connection"]
    CSESS["EnvoyQuicClientSession<br/>= Network::ClientConnection"]
    CCONN["EnvoyQuicClientConnection<br/>= quic::QuicConnection + UdpPacketProcessor"]
    SOCK["Network::ConnectionSocket"]
    PW["EnvoyQuicPacketWriter"]
    PV["EnvoyQuicProofVerifier"]
  end

  CSESS --> CCONN
  CCONN --> SOCK
  CCONN --> PW
  CSESS --> PV
  CSESS -.->|"shares"| HLP
  CSESS -.->|"shares"| ALMF
  CSESS -.->|"shares"| QC
  CSESS -.->|"shares"| CSF
  CCONN -.->|"shares"| WF
```

### Per‑cluster vs per‑connection state

- **Per cluster**: `PersistentQuicInfoImpl` (defined in `client_connection_factory_impl.h`) caches the bits that don't change between connections to the same cluster — alarm factory, connection helper, `QuicConfig`, crypto stream factory, migration config, packet‑writer factory. One instance per cluster, lazily created.
- **Per connection**: everything else. Each `createQuicNetworkConnection()` call builds a fresh `EnvoyQuicClientConnection` + `EnvoyQuicClientSession`.

### Connecting and handshaking

```mermaid
sequenceDiagram
  autonumber
  participant Pool as Http::ConnectionPoolImpl
  participant Fac as createQuicNetworkConnection
  participant Sess as EnvoyQuicClientSession
  participant Conn as EnvoyQuicClientConnection
  participant Sock as Network::ConnectionSocket
  participant PV as EnvoyQuicProofVerifier
  participant CV as ClientContextImpl<br/>(source/common/tls)

  Pool->>Fac: createQuicNetworkConnection(info, crypto, server_id, ...)
  Fac->>Sock: create UDP socket bound to local_addr
  Fac->>Conn: new EnvoyQuicClientConnection(<br/>  server_cid, helper, alarm_factory, writer,<br/>  versions, dispatcher, socket, cid_generator)
  Fac->>Sess: new EnvoyQuicClientSession(<br/>  config, versions, connection, server_id,<br/>  crypto_config, dispatcher, ...,<br/>  crypto_stream_factory, rtt_cache, scope,<br/>  transport_socket_options, factory)
  Pool->>Sess: connect()
  Sess->>Conn: setUpConnectionSocket(socket, delegate)
  Conn->>Conn: register libevent read event<br/>(via Network::UdpPacketProcessor)
  Sess->>Sess: QuicSpdyClientSession::CryptoConnect()
  Note over Sess: TLS 1.3 handshake driven by QUICHE
  Sess->>PV: VerifyCertChain(hostname, port, certs, ocsp, sct, ...)
  PV->>CV: customVerifyCertChainForQuic(...)
  CV-->>PV: result
  PV-->>Sess: ok / error
  Sess-->>Pool: OnTlsHandshakeComplete()
  Pool->>Pool: state -> connected, ready for streams
```

### Receiving packets (no dispatcher)

The client connection implements `Network::UdpPacketProcessor` itself; the file event is registered by the connection, not by a listener:

```mermaid
sequenceDiagram
  participant Ev as libevent
  participant Conn as EnvoyQuicClientConnection
  participant Sess as EnvoyQuicClientSession

  Ev->>Conn: onFileEvent(socket)
  Conn->>Conn: socket.ioHandle().recvmmsg(...)
  loop each packet
    Conn->>Conn: processPacket(local, peer, buffer, time, tos, cmsg)
    Conn->>Sess: ProcessUdpPacket(self, peer, QuicReceivedPacket)
  end
```

This is structurally simpler than the server side because there's no demux: every packet on this socket belongs to this connection (modulo migration). The complication is **connection migration**: the connection can hold multiple sockets in `connection_sockets_`, and only the *last* one is used for writing. See [`envoy_quic_connection.md`](envoy_quic_connection.md) for the full migration story.

---

## The Session ↔ Connection relationship

A common point of confusion: there is **one `Session` and one `Connection` per QUIC connection**, and they have very specific responsibilities.

| Concern | Lives in `Session` | Lives in `Connection` |
|---|---|---|
| `Network::Connection` interface (HCM contract) | ✓ (via `QuicFilterManagerConnectionImpl`) | |
| Stream creation / lookup | ✓ (`QuicSession`) | |
| HTTP/3 framing, QPACK | ✓ (`QuicSpdySession`) | |
| TLS handshake | ✓ (via crypto stream) | |
| Packet encryption / decryption | | ✓ (QUIC packet headers, frame parser) |
| Congestion control, loss recovery | | ✓ (`QuicSentPacketManager`) |
| ACK / RTT tracking | | ✓ |
| Migration / path validation | | ✓ |
| The UDP socket | | ✓ (server: borrowed from listener; client: owned) |
| Filter chain | ✓ (`Network::FilterManager`) | |
| Listener filters (QUIC only) | | ✓ (server: `QuicListenerFilterManagerImpl`) |

The session **owns** the connection (`std::unique_ptr<EnvoyQuicServerConnection> quic_connection_;`). When the session is destroyed, the connection goes with it. The connection never outlives its session.

---

## What's next

- [`OVERVIEW_PART3_crypto_and_tls.md`](OVERVIEW_PART3_crypto_and_tls.md) — Where the certs come from. How `EnvoyQuicProofSource` finds the right filter chain, how `EnvoyTlsServerHandshaker` plumbs session tickets, and how `EnvoyQuicProofVerifier` reuses `source/common/tls`.
- [`OVERVIEW_PART4_streams_codecs_http3.md`](OVERVIEW_PART4_streams_codecs_http3.md) — From `OnStreamFrame` to `decodeHeaders` and back. The stream + codec layer.
- [`active_quic_listener.md`](active_quic_listener.md) — Deep dive into the listener and dispatcher code.
- [`envoy_quic_session.md`](envoy_quic_session.md) — Deep dive into both sessions.
- [`envoy_quic_connection.md`](envoy_quic_connection.md) — Deep dive into both connections, including migration.

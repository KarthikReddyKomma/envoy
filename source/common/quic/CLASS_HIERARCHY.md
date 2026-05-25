# Class hierarchy — UML view

> *A single place to pin the class graph in your head. Diagrams here use Mermaid `classDiagram` with the common UML conventions: solid arrow with hollow head = inheritance / `is a`, dashed arrow = composition / `has a`, `<<QUICHE>>` and `<<Envoy>>` markers indicate which world a base class belongs to.*

This file is dense on purpose. Read it together with [`OVERVIEW_PART1`](OVERVIEW_PART1_architecture_and_layering.md) (architecture) — that doc tells you *why* the inheritance is shaped this way; this one shows you the exact graph.

## Sections

1. [Listener & dispatcher](#1-listener--dispatcher)
2. [Sessions](#2-sessions)
3. [Connections](#3-connections)
4. [Streams](#4-streams)
5. [HTTP/3 codec adapters](#5-http3-codec-adapters)
6. [Crypto, ProofSource, ProofVerifier](#6-crypto-proofsource-proofverifier)
7. [Transport socket factories](#7-transport-socket-factories)
8. [Glue (clock, alarm, packet writer)](#8-glue-clock-alarm-packet-writer)
9. [Connection ID and packet routing](#9-connection-id-and-packet-routing)
10. [Watermark, stats, monitors](#10-watermark-stats-monitors)
11. [Everything‑on‑one‑canvas (collapsed)](#11-everything-on-one-canvas-collapsed)

---

## 1. Listener & dispatcher

```mermaid
classDiagram
    class Network_ActiveUdpListenerFactory {
        <<Envoy>>
        +createActiveUdpListener()
    }
    class Server_ActiveUdpListenerBase {
        <<Envoy>>
        #udp_listener_
        #listen_socket_
        +pauseListening()*
        +shutdownListener()*
    }
    class quic_QuicDispatcher {
        <<QUICHE>>
        +ProcessChlo()*
        +CreateQuicSession()*
        +OnFailedToDispatchPacket()
        +OnCanWrite()
        +Shutdown()
    }
    class quic_QuicTimeWaitListManager {
        <<QUICHE>>
        +SendPublicReset()*
    }

    class ActiveQuicListener {
        +onDataWorker(UdpRecvData)
        +onReadReady()
        +onWriteReady()
        +destination(data)
        +numPacketsExpectedPerEventLoop()
        +pauseListening()
        +resumeListening()
        +shutdownListener()
        +updateListenerConfig()
        +onFilterChainDraining()
        +onCloseIdleHttpConnections()
        -quic_dispatcher_
        -crypto_config_
        -udp_packet_writer_
        -connection_id_generator_
        -select_connection_id_worker_
    }
    class ActiveQuicListenerFactory {
        +createActiveUdpListener()
        +isTransportConnectionless() false
        -quic_config_
        -crypto_server_stream_factory_
        -proof_source_factory_
        -quic_cid_generator_factory_
    }
    class EnvoyQuicDispatcher {
        +OnConnectionClosed()
        +CreateQuicSession()
        +CreateQuicTimeWaitListManager()
        +processPacket(self, peer, packet) bool
        +closeConnectionsWithFilterChain()
        +closeIdleQuicConnections()
        -connections_by_filter_chain_
        -session_idle_list_
    }
    class EnvoyQuicTimeWaitListManager {
        +SendPublicReset()
    }
    class EnvoyQuicCryptoServerStreamHelper {
        +CanAcceptClientHello() false
    }

    Server_ActiveUdpListenerBase <|-- ActiveQuicListener
    Network_ActiveUdpListenerFactory <|-- ActiveQuicListenerFactory
    quic_QuicDispatcher <|-- EnvoyQuicDispatcher
    quic_QuicTimeWaitListManager <|-- EnvoyQuicTimeWaitListManager

    ActiveQuicListener "1" *-- "1" EnvoyQuicDispatcher : owns
    ActiveQuicListenerFactory ..> ActiveQuicListener : creates
    EnvoyQuicDispatcher ..> EnvoyQuicTimeWaitListManager : creates
    EnvoyQuicDispatcher ..> EnvoyQuicCryptoServerStreamHelper : creates
```

---

## 2. Sessions

The two session classes share `QuicFilterManagerConnectionImpl` but each adds its own QUICHE base on the other side.

```mermaid
classDiagram
    class Network_ConnectionImplBase {
        <<Envoy>>
        +addConnectionCallbacks()
        +setConnectionStats()
        #connection_callbacks_
        #stats_
    }
    class SendBufferMonitor {
        +updateBytesBuffered()*
    }
    class QuicWriteEventCallback {
        +onWriteEventDone()*
    }
    class QuicFilterManagerConnectionImpl {
        +addReadFilter()
        +addWriteFilter()
        +initializeReadFilters()
        +addAccessLogHandler()
        +close(type)
        +state() State
        +connecting() bool
        +ssl() ConnectionInfoConstSharedPtr
        +streamInfo() StreamInfo&
        +setBufferLimits()
        +rawWrite()
        +updateBytesBuffered(old, new)
        +onWriteEventDone()
        #network_connection_
        #quic_ssl_info_
        -filter_manager_
        -stream_info_
        -write_buffer_watermark_simulation_
    }
    class quic_QuicSession {
        <<QUICHE>>
        +ProcessUdpPacket()
        +OnCanWrite()
        +OnStreamFrame()
        +CreateIncomingStream()*
        +CreateOutgoingBidirectionalStream()*
        +Initialize()*
        +GetCryptoStream()*
    }
    class quic_QuicSpdySession {
        <<QUICHE>>
    }
    class quic_QuicServerSessionBase {
        <<QUICHE>>
        +CreateQuicCryptoServerStream()*
        +GetSSLConfig()
        +SelectAlpn()
    }
    class quic_QuicSpdyClientSession {
        <<QUICHE>>
        +CreateQuicCryptoStream()*
        +OnHttp3GoAway()
    }
    class Http_IdleSessionInterface {
        <<Envoy>>
        +TerminateIdleSession()*
    }
    class Network_ClientConnection {
        <<Envoy>>
        +connect()*
    }
    class PacketsToReadDelegate {
        +numPacketsExpectedPerEventLoop()*
    }

    class EnvoyQuicServerSession {
        +requestedServerName() string_view
        +setHttpConnectionCallbacks()
        +setH3GoAwayLoadShedPoints()
        +Initialize()
        +OnTlsHandshakeComplete()
        +SelectAlpn()
        +CreateQuicCryptoServerStream()
        +CreateIncomingStream()
        +TerminateIdleSession()
        +setHttp3Options()
        +OnStreamClosed()
        -quic_connection_
        -http_connection_callbacks_
        -crypto_server_stream_factory_
        -session_idle_list_
    }

    class EnvoyQuicClientSession {
        +connect()
        +Initialize()
        +CreateClientStream()
        +CreateIncomingStream()
        +CreateQuicCryptoStream()
        +OnHttp3GoAway()
        +OnTlsHandshakeComplete()
        +OnNewEncryptionKeyAvailable()
        +OnConfigNegotiated()
        +OnProofVerifyDetailsAvailable()
        +OnServerPreferredAddressAvailable()
        +registerNetworkObserver()
        -crypto_config_
        -crypto_stream_factory_
        -transport_socket_factory_
        -network_connectivity_observer_
    }

    Network_ConnectionImplBase <|-- QuicFilterManagerConnectionImpl
    SendBufferMonitor <|-- QuicFilterManagerConnectionImpl
    QuicWriteEventCallback <|-- QuicFilterManagerConnectionImpl

    quic_QuicSession <|-- quic_QuicSpdySession
    quic_QuicSpdySession <|-- quic_QuicServerSessionBase
    quic_QuicSpdySession <|-- quic_QuicSpdyClientSession

    quic_QuicServerSessionBase <|-- EnvoyQuicServerSession
    QuicFilterManagerConnectionImpl <|-- EnvoyQuicServerSession
    Http_IdleSessionInterface <|-- EnvoyQuicServerSession

    QuicFilterManagerConnectionImpl <|-- EnvoyQuicClientSession
    quic_QuicSpdyClientSession <|-- EnvoyQuicClientSession
    Network_ClientConnection <|-- EnvoyQuicClientSession
    PacketsToReadDelegate <|-- EnvoyQuicClientSession
```

---

## 3. Connections

Both connection classes inherit from `quic::QuicConnection`, but the client also implements `Network::UdpPacketProcessor` because it owns its socket directly.

```mermaid
classDiagram
    class quic_QuicConnection {
        <<QUICHE>>
        +OnPacketHeader()
        +ProcessUdpPacket()
        +OnCanWrite()
        +OnPathDegradingDetected()
        +OnEffectivePeerMigrationValidated()
        +WritePacket()
    }
    class QuicNetworkConnection {
        +setConnectionStats()
        +setEnvoyConnection()
        +connectionSocket() ptr&
        +id() uint64_t
        #setConnectionSocket()
        #onWriteEventDone()
        #networkConnection()
        -connection_sockets_
        -envoy_connection_
        -write_callback_
    }
    class Network_UdpPacketProcessor {
        <<Envoy>>
        +processPacket(local, peer, buf, time, tos, cmsg)*
        +maxDatagramSize()*
        +onDatagramsDropped()*
        +numPacketsExpectedPerEventLoop()*
        +saveCmsgConfig()*
    }

    class EnvoyQuicServerConnection {
        +OnPacketHeader()
        +OnCanWrite()
        +ProcessUdpPacket()
        +actuallyDeferSend()
        +OnEffectivePeerMigrationValidated()
        -listener_filter_manager_
        -first_packet_received_
    }
    class EnvoyQuicSelfIssuedConnectionIdManager {
        +(uses quic CID manager)
    }
    class EnvoyQuicClientConnection {
        +processPacket()
        +maxDatagramSize()
        +setUpConnectionSocket(socket, delegate)
        +switchConnectionSocket()
        +OnPathDegradingDetected()
        +OnCanWrite()
        +onPathValidationSuccess()
        +onPathValidationFailure()
        +probeAndMigrateToServerPreferredAddress()
        +getOrCreateMigrationHelper()
        -dispatcher_
        -migration_helper_
        -writer_factory_
    }
    class EnvoyQuicPathValidationContext {
        +WriterToUse()
        +releaseWriter()
        +probingSocket()
        +releaseSocket()
    }
    class EnvoyQuicMigrationHelper {
        +FindAlternateNetwork()
        +CreateQuicPathContextFactory()
        +OnMigrationToPathDone()
        +GetDefaultNetwork()
        +GetCurrentNetwork()
    }
    class EnvoyPathValidationResultDelegate {
        +OnPathValidationSuccess()
        +OnPathValidationFailure()
    }
    class EnvoyQuicClinetPathContextFactory {
        +CreatePathValidationContext()
    }
    class quic_QuicPathValidationContext {
        <<QUICHE>>
    }
    class quic_QuicMigrationHelper {
        <<QUICHE>>
    }
    class quic_QuicSelfIssuedConnectionIdManager {
        <<QUICHE>>
    }
    class quic_QuicPathValidator_ResultDelegate {
        <<QUICHE>>
    }
    class quic_QuicPathContextFactory {
        <<QUICHE>>
    }

    quic_QuicConnection <|-- EnvoyQuicServerConnection
    QuicNetworkConnection <|-- EnvoyQuicServerConnection
    quic_QuicConnection <|-- EnvoyQuicClientConnection
    QuicNetworkConnection <|-- EnvoyQuicClientConnection
    Network_UdpPacketProcessor <|-- EnvoyQuicClientConnection
    quic_QuicSelfIssuedConnectionIdManager <|-- EnvoyQuicSelfIssuedConnectionIdManager

    EnvoyQuicClientConnection +-- EnvoyQuicPathValidationContext : nested
    EnvoyQuicClientConnection +-- EnvoyQuicMigrationHelper : nested
    EnvoyQuicClientConnection +-- EnvoyPathValidationResultDelegate : nested
    EnvoyQuicClientConnection +-- EnvoyQuicClinetPathContextFactory : nested

    quic_QuicPathValidationContext <|-- EnvoyQuicPathValidationContext
    quic_QuicMigrationHelper <|-- EnvoyQuicMigrationHelper
    quic_QuicPathValidator_ResultDelegate <|-- EnvoyPathValidationResultDelegate
    quic_QuicPathContextFactory <|-- EnvoyQuicClinetPathContextFactory
```

---

## 4. Streams

The stream layer is the densest cross‑inheritance.

```mermaid
classDiagram
    class quic_QuicSpdyStream {
        <<QUICHE>>
        +OnBodyAvailable()*
        +OnInitialHeadersComplete()*
        +OnTrailingHeadersComplete()*
        +OnClose()*
        +OnCanWrite()*
        +WriteHeaders()
        +WriteOrBufferBody()
    }
    class quic_QuicSpdyStream_MetadataVisitor {
        <<QUICHE>>
        +OnMetadataComplete()*
    }
    class quic_QuicSpdyServerStreamBase {
        <<QUICHE>>
        +OnConnectionClosed()*
        +CloseWriteSide()*
    }
    class quic_QuicSpdyClientStream {
        <<QUICHE>>
    }
    class Http_StreamEncoder {
        <<Envoy>>
        +encodeData()
        +encodeMetadata()
        +getStream()*
    }
    class Http_RequestEncoder {
        <<Envoy>>
        +encodeHeaders(req, fin) Status
        +encodeTrailers(req)
        +enableTcpTunneling()
    }
    class Http_ResponseEncoder {
        <<Envoy>>
        +encode1xxHeaders()
        +encodeHeaders(resp, fin)
        +encodeTrailers(resp)
        +streamErrorOnInvalidHttpMessage()*
    }
    class Http_MultiplexedStreamImplBase {
        <<Envoy>>
        +readDisable()
        +addCallbacks()
        +bufferLimit()
        +onPendingFlushTimer()*
        +hasPendingData()*
    }
    class HeaderValidator {
        +validateHeader()*
        +startHeaderBlock()*
        +finishHeaderBlock()*
    }
    class SendBufferMonitor

    class EnvoyQuicStream {
        +encodeData(buf, fin)
        +readDisable(bool)
        +validateHeader()
        +startHeaderBlock()
        +finishHeaderBlock()
        +bufferLimit()
        +addCallbacks()
        +removeCallbacks()
        +updateBytesBuffered(old, new)
        +bytesMeter()
        +statsGatherer()
        #switchStreamBlockState()*
        #streamId()*
        #connection()*
        #onStreamError()*
        #send_buffer_simulation_
        #header_validator_
        #stats_gatherer_
        #http_datagram_handler_
        #async_stream_blockage_change_
    }
    class EnvoyQuicServerStream {
        +setRequestDecoder(decoder)
        +encode1xxHeaders()
        +encodeHeaders(resp, fin)
        +encodeTrailers(resp)
        +setDeferredLoggingHeadersAndTrailers()
        +resetStream(reason)
        +codecStreamId() id
        +OnStreamFrame()
        +OnBodyAvailable()
        +OnStopSending()
        +OnStreamReset()
        +OnClose()
        +OnCanWrite()
        +OnConnectionClosed()
        +OnMetadataComplete()
        -request_decoder_
        -saw_path_
    }
    class EnvoyQuicClientStream {
        +setResponseDecoder(decoder)
        +encodeHeaders(req, fin)
        +encodeTrailers(req)
        +resetStream(reason)
        +OnStreamFrame()
        +OnStopSending()
        +OnBodyAvailable()
        +OnStreamReset()
        +OnClose()
        +OnCanWrite()
        +OnConnectionClosed()
        +OnMetadataComplete()
        -response_decoder_
        -response_decoder_handle_
        -upgrade_protocol_
    }
    class IncrementalBytesSentTracker {
        +ctor(stream, meter, header_bytes)
        +dtor()
    }

    Http_StreamEncoder <|-- Http_RequestEncoder
    Http_StreamEncoder <|-- Http_ResponseEncoder
    Http_StreamEncoder <|-- EnvoyQuicStream
    Http_MultiplexedStreamImplBase <|-- EnvoyQuicStream
    SendBufferMonitor <|-- EnvoyQuicStream
    HeaderValidator <|-- EnvoyQuicStream

    quic_QuicSpdyStream <|-- quic_QuicSpdyServerStreamBase
    quic_QuicSpdyStream <|-- quic_QuicSpdyClientStream

    EnvoyQuicStream <|-- EnvoyQuicServerStream
    quic_QuicSpdyServerStreamBase <|-- EnvoyQuicServerStream
    Http_ResponseEncoder <|-- EnvoyQuicServerStream
    quic_QuicSpdyStream_MetadataVisitor <|-- EnvoyQuicServerStream

    EnvoyQuicStream <|-- EnvoyQuicClientStream
    quic_QuicSpdyClientStream <|-- EnvoyQuicClientStream
    Http_RequestEncoder <|-- EnvoyQuicClientStream
    quic_QuicSpdyStream_MetadataVisitor <|-- EnvoyQuicClientStream
```

---

## 5. HTTP/3 codec adapters

```mermaid
classDiagram
    class Http_Connection {
        <<Envoy>>
        +dispatch(buf) Status
        +protocol() Protocol
        +wantsToWrite() bool
        +goAway()
        +onUnderlyingConnectionAboveWriteBufferHighWatermark()
        +onUnderlyingConnectionBelowWriteBufferLowWatermark()
    }
    class Http_ServerConnection {
        <<Envoy>>
        +shutdownNotice()
    }
    class Http_ClientConnection {
        <<Envoy>>
        +newStream(decoder) RequestEncoder&
    }
    class QuicHttpServerConnectionFactory {
        <<Envoy>>
        +createQuicHttpServerConnectionImpl()*
    }

    class QuicHttpConnectionImplBase {
        +dispatch() panic
        +protocol() Http3
        +wantsToWrite()
        #quic_session_
        #stats_
    }
    class QuicHttpServerConnectionImpl {
        +goAway()
        +shutdownNotice()
        +onUnderlyingConnectionAboveWriteBufferHighWatermark()
        +onUnderlyingConnectionBelowWriteBufferLowWatermark()
        +quicServerSession()
        -quic_server_session_
    }
    class QuicHttpClientConnectionImpl {
        +newStream(decoder)
        +goAway()
        +shutdownNotice() noop
        +onUnderlyingConnectionAboveWriteBufferHighWatermark()
        +onUnderlyingConnectionBelowWriteBufferLowWatermark()
        -quic_client_session_
    }
    class QuicHttpServerConnectionFactoryImpl {
        +createQuicHttpServerConnectionImpl()
    }

    Http_Connection <|-- Http_ServerConnection
    Http_Connection <|-- Http_ClientConnection
    Http_Connection <|-- QuicHttpConnectionImplBase
    QuicHttpConnectionImplBase <|-- QuicHttpServerConnectionImpl
    Http_ServerConnection <|-- QuicHttpServerConnectionImpl
    QuicHttpConnectionImplBase <|-- QuicHttpClientConnectionImpl
    Http_ClientConnection <|-- QuicHttpClientConnectionImpl
    QuicHttpServerConnectionFactory <|-- QuicHttpServerConnectionFactoryImpl
```

---

## 6. Crypto, ProofSource, ProofVerifier

```mermaid
classDiagram
    class quic_ProofSource {
        <<QUICHE>>
        +GetProof()*
        +GetCertChain()*
        +ComputeTlsSignature()*
        +OnNewSslCtx()
        +GetTicketCrypter()
        +SupportedTlsSignatureAlgorithms()*
    }
    class quic_ProofSource_Details {
        <<QUICHE>>
    }
    class quic_TlsServerHandshaker {
        <<QUICHE>>
    }
    class quic_QuicCryptoServerStreamBase {
        <<QUICHE>>
    }
    class quic_ProofVerifier {
        <<QUICHE>>
        +VerifyProof()
        +VerifyCertChain()*
    }
    class quic_ProofVerifyContext {
        <<QUICHE>>
    }
    class quic_ProofVerifyDetails {
        <<QUICHE>>
        +Clone()*
    }

    class EnvoyQuicProofSourceDetails {
        +filterChain() FilterChain&
        -filter_chain_
    }
    class EnvoyQuicProofSourceBase {
        +GetProof()
        +ComputeTlsSignature()
        +SupportedTlsSignatureAlgorithms()
        +GetTicketCrypter() nullptr
        #signPayload()*
    }
    class EnvoyQuicProofSource {
        +OnNewSslCtx()
        +GetCertChain()
        +signPayload()
        +updateFilterChainManager()
        -listen_socket_
        -filter_chain_manager_
        -listener_stats_
        -time_source_
    }
    class EnvoyQuicProofSourceFactoryInterface {
        +createQuicProofSource()*
    }

    class EnvoyTlsServerHandshaker {
        +ticketKeyCallback() static
        +handshakerExDataIndex() static
        -pinnedServerContext()
        -pinned_ssl_ctx_
    }
    class EnvoyQuicCryptoServerStreamFactoryInterface {
        +createEnvoyQuicCryptoServerStream()*
    }

    class EnvoyQuicProofVerifyContext {
        +dispatcher()*
        +isServer()*
        +transportSocketOptions()*
        +extraValidationContext()*
    }
    class EnvoyQuicProofVerifierBase {
        +VerifyProof()
    }
    class EnvoyQuicProofVerifier {
        +VerifyCertChain()
        -context_
        -accept_untrusted_
    }
    class CertVerifyResult {
        +isValid()
        +Clone()
        -is_valid_
    }
    class EnvoyQuicCryptoClientStreamFactoryInterface {
        +createEnvoyQuicCryptoClientStream()*
    }

    quic_ProofSource <|-- EnvoyQuicProofSourceBase
    EnvoyQuicProofSourceBase <|-- EnvoyQuicProofSource
    quic_ProofSource_Details <|-- EnvoyQuicProofSourceDetails
    quic_TlsServerHandshaker <|-- EnvoyTlsServerHandshaker
    quic_ProofVerifier <|-- EnvoyQuicProofVerifierBase
    EnvoyQuicProofVerifierBase <|-- EnvoyQuicProofVerifier
    quic_ProofVerifyContext <|-- EnvoyQuicProofVerifyContext
    quic_ProofVerifyDetails <|-- CertVerifyResult
```

---

## 7. Transport socket factories

QUIC transport socket factories never make a real `TransportSocket`. They exist only to carry TLS context config.

```mermaid
classDiagram
    class Network_DownstreamTransportSocketFactory {
        <<Envoy>>
        +createDownstreamTransportSocket()*
        +implementsSecureTransport()*
    }
    class Network_UpstreamTransportSocketFactory {
        <<Envoy>>
        +createTransportSocket()*
    }
    class Server_DownstreamTransportSocketConfigFactory {
        <<Envoy>>
        +createTransportSocketFactory()*
    }
    class Server_UpstreamTransportSocketConfigFactory {
        <<Envoy>>
        +createTransportSocketFactory()*
    }
    class QuicTransportSocketFactoryBase {
        +initialize()*
        +supportedAlpnProtocols()
        #onSecretUpdated()*
        #supported_alpns_
        #stats_
    }
    class QuicTransportSocketConfigFactory {
        +name() "envoy.transport_sockets.quic"
    }
    class QuicServerTransportSocketFactory {
        +create() static
        +createDownstreamTransportSocket() panic
        +implementsSecureTransport() true
        +initialize()
        +getTlsCertificateAndKey()
        +earlyDataEnabled()
        +getSessionTicketConfig()
        +sslCtx()
        +onSecretUpdated()
        -manager_
        -config_
        -ssl_ctx_
    }
    class QuicClientTransportSocketFactory {
        +createTransportSocket() panic
        +implementsSecureTransport() true
        +clientContext()
    }
    class QuicServerTransportSocketConfigFactory {
        +createTransportSocketFactory()
    }
    class QuicClientTransportSocketConfigFactory {
        +createTransportSocketFactory()
    }

    Network_DownstreamTransportSocketFactory <|-- QuicServerTransportSocketFactory
    QuicTransportSocketFactoryBase <|-- QuicServerTransportSocketFactory
    Network_UpstreamTransportSocketFactory <|-- QuicClientTransportSocketFactory
    QuicTransportSocketFactoryBase <|-- QuicClientTransportSocketFactory
    QuicTransportSocketConfigFactory <|-- QuicServerTransportSocketConfigFactory
    Server_DownstreamTransportSocketConfigFactory <|-- QuicServerTransportSocketConfigFactory
    QuicTransportSocketConfigFactory <|-- QuicClientTransportSocketConfigFactory
    Server_UpstreamTransportSocketConfigFactory <|-- QuicClientTransportSocketConfigFactory
```

---

## 8. Glue (clock, alarm, packet writer)

```mermaid
classDiagram
    class quic_QuicClock {
        <<QUICHE>>
        +ApproximateNow()*
        +Now()*
        +WallNow()*
    }
    class quic_QuicAlarm {
        <<QUICHE>>
        +SetImpl()*
        +CancelImpl()*
        +UpdateImpl()
    }
    class quic_QuicAlarmFactory {
        <<QUICHE>>
        +CreateAlarm(delegate)*
        +CreateAlarm(arena_delegate, arena)*
    }
    class quic_QuicConnectionHelperInterface {
        <<QUICHE>>
        +GetClock()*
        +GetRandomGenerator()*
        +GetStreamSendBufferAllocator()*
    }
    class quic_QuicPacketWriter {
        <<QUICHE>>
        +WritePacket()*
        +IsWriteBlocked()*
        +SetWritable()*
        +IsBatchMode()*
        +Flush()
    }
    class quic_QuicGsoBatchWriter {
        <<QUICHE>>
    }
    class Network_UdpPacketWriter {
        <<Envoy>>
        +writePacket()*
        +isWriteBlocked()*
        +setWritable()*
        +flush()*
    }

    class EnvoyQuicClock {
        +ApproximateNow()
        +Now()
        +WallNow()
        -dispatcher_
    }
    class EnvoyQuicAlarm {
        +SetImpl()
        +CancelImpl()
        +UpdateImpl()
        -timer_
        -clock_
    }
    class EnvoyQuicAlarmFactory {
        +CreateAlarm(delegate)
        +CreateAlarm(arena_delegate, arena)
        -dispatcher_
        -clock_
    }
    class EnvoyQuicConnectionHelper {
        +GetClock()
        +GetRandomGenerator()
        +GetStreamSendBufferAllocator()
        -clock_
        -random_generator_
        -buffer_allocator_
    }
    class EnvoyQuicPacketWriter {
        +WritePacket()
        +IsWriteBlocked()
        +SetWritable()
        +IsBatchMode()
        +SupportsReleaseTime() false
        +SupportsEcn() false
        +GetMaxPacketSize()
        +GetNextWriteLocation()
        +Flush()
        -envoy_udp_packet_writer_
    }
    class UdpGsoBatchWriter {
        +writePacket() Envoy facing
        +WritePacket() QUIC facing
        +flush()
        +isWriteBlocked()
        +setWritable()
        +isBatchMode()
        +getMaxPacketSize()
        -stats_
        -gso_size_
    }

    quic_QuicClock <|-- EnvoyQuicClock
    quic_QuicAlarm <|-- EnvoyQuicAlarm
    quic_QuicAlarmFactory <|-- EnvoyQuicAlarmFactory
    quic_QuicConnectionHelperInterface <|-- EnvoyQuicConnectionHelper
    quic_QuicPacketWriter <|-- EnvoyQuicPacketWriter
    quic_QuicGsoBatchWriter <|-- UdpGsoBatchWriter
    Network_UdpPacketWriter <|-- UdpGsoBatchWriter

    EnvoyQuicConnectionHelper *-- EnvoyQuicClock
    EnvoyQuicAlarmFactory ..> EnvoyQuicAlarm : creates
```

---

## 9. Connection ID and packet routing

```mermaid
classDiagram
    class quic_ConnectionIdGeneratorInterface {
        <<QUICHE>>
        +GenerateNextConnectionId()*
        +MaybeReplaceConnectionId()*
    }
    class quic_DeterministicConnectionIdGenerator {
        <<QUICHE>>
    }
    class EnvoyQuicConnectionIdGeneratorFactory {
        <<interface>>
        +createQuicConnectionIdGenerator()*
        +createCompatibleLinuxBpfSocketOption()*
        +getCompatibleConnectionIdWorkerSelector()*
    }
    class EnvoyDeterministicConnectionIdGenerator {
        +GenerateNextConnectionId()
        +MaybeReplaceConnectionId()
    }
    class EnvoyDeterministicConnectionIdGeneratorFactory {
        +createQuicConnectionIdGenerator()
        +createCompatibleLinuxBpfSocketOption()
        +getCompatibleConnectionIdWorkerSelector()
    }

    quic_ConnectionIdGeneratorInterface <|-- quic_DeterministicConnectionIdGenerator
    quic_DeterministicConnectionIdGenerator <|-- EnvoyDeterministicConnectionIdGenerator
    EnvoyQuicConnectionIdGeneratorFactory <|-- EnvoyDeterministicConnectionIdGeneratorFactory
    EnvoyDeterministicConnectionIdGeneratorFactory ..> EnvoyDeterministicConnectionIdGenerator : creates
```

---

## 10. Watermark, stats, monitors

```mermaid
classDiagram
    class quic_QuicAckListenerInterface {
        <<QUICHE>>
        +OnPacketAcked()*
        +OnPacketRetransmitted()*
    }
    class SendBufferMonitor {
        +updateBytesBuffered()*
        +isDoingWatermarkAccounting()
    }
    class ScopedWatermarkBufferUpdater {
        +ctor(quic_stream, monitor)
        +dtor() computes delta
    }
    class EnvoyQuicSimulatedWatermarkBuffer {
        +checkHighWatermark()
        +checkLowWatermark()
        +highWatermark()
        +lowWatermark()
        -high_
        -low_
    }
    class QuicStatsGatherer {
        +OnPacketAcked()
        +OnPacketRetransmitted()
        +addBytesSent()
        +setAccessLogHandlers()
        +setDeferredLoggingHeadersAndTrailers()
        +maybeDoDeferredLog()
        +loggingDone()
        +bytesOutstanding()
        -bytes_outstanding_
        -access_log_handlers_
        -stream_info_
        -time_source_
    }
    class QuicWriteEventCallback {
        +onWriteEventDone()*
    }
    class QuicNetworkConnectivityObserver {
        <<interface>>
        +onNetworkChanged()*
        +onNetworkConnected()*
        +onNetworkDisconnected()*
        +onNetworkSoonToDisconnect()*
        +onNetworkMadeDefault()*
    }
    class QuicNetworkConnectivityObserverImpl

    quic_QuicAckListenerInterface <|-- QuicStatsGatherer
    SendBufferMonitor +-- ScopedWatermarkBufferUpdater : nested
    QuicNetworkConnectivityObserver <|-- QuicNetworkConnectivityObserverImpl
```

---

## 11. Everything‑on‑one‑canvas (collapsed)

A bird's‑eye composition view. Shows ownership (`*--`), not inheritance — for inheritance, use the sections above.

```mermaid
classDiagram
    class ActiveQuicListener
    class EnvoyQuicDispatcher
    class EnvoyQuicProofSource
    class quic_QuicCryptoServerConfig {
        <<QUICHE>>
    }
    class EnvoyQuicCryptoServerStreamFactory {
        <<interface>>
    }
    class EnvoyQuicConnectionHelper
    class EnvoyQuicAlarmFactory
    class EnvoyQuicPacketWriter
    class UdpGsoBatchWriter
    class EnvoyDeterministicConnectionIdGenerator
    class Http_SessionIdleList {
        <<Envoy>>
    }

    class EnvoyQuicServerSession
    class EnvoyQuicServerConnection
    class QuicListenerFilterManagerImpl
    class EnvoyTlsServerHandshaker
    class ServerContextImpl {
        <<source/common/tls>>
    }
    class EnvoyQuicServerStream
    class QuicHttpServerConnectionImpl
    class QuicStatsGatherer

    class EnvoyQuicClientSession
    class EnvoyQuicClientConnection
    class EnvoyQuicProofVerifier
    class ClientContextImpl {
        <<source/common/tls>>
    }
    class EnvoyQuicClientStream
    class QuicHttpClientConnectionImpl
    class PersistentQuicInfoImpl

    ActiveQuicListener *-- EnvoyQuicDispatcher
    ActiveQuicListener *-- quic_QuicCryptoServerConfig
    ActiveQuicListener *-- EnvoyQuicConnectionHelper
    ActiveQuicListener *-- EnvoyQuicAlarmFactory
    ActiveQuicListener *-- EnvoyDeterministicConnectionIdGenerator
    ActiveQuicListener o-- EnvoyQuicPacketWriter : owns one of
    ActiveQuicListener o-- UdpGsoBatchWriter : or this
    ActiveQuicListener o-- Http_SessionIdleList : optional

    quic_QuicCryptoServerConfig *-- EnvoyQuicProofSource

    EnvoyQuicDispatcher ..> EnvoyQuicServerSession : creates
    EnvoyQuicServerSession *-- EnvoyQuicServerConnection
    EnvoyQuicServerConnection *-- QuicListenerFilterManagerImpl
    EnvoyQuicServerSession ..> EnvoyTlsServerHandshaker : creates
    EnvoyTlsServerHandshaker o-- ServerContextImpl : pins shared_ptr

    EnvoyQuicServerSession ..> EnvoyQuicServerStream : creates
    EnvoyQuicServerStream o-- QuicStatsGatherer : ack listener
    QuicHttpServerConnectionImpl --> EnvoyQuicServerSession : holds ref

    PersistentQuicInfoImpl *-- EnvoyQuicConnectionHelper
    PersistentQuicInfoImpl *-- EnvoyQuicAlarmFactory

    EnvoyQuicClientSession *-- EnvoyQuicClientConnection
    EnvoyQuicClientSession o-- EnvoyQuicProofVerifier : referenced via crypto
    EnvoyQuicProofVerifier o-- ClientContextImpl

    EnvoyQuicClientSession ..> EnvoyQuicClientStream : creates
    QuicHttpClientConnectionImpl --> EnvoyQuicClientSession : holds ref
```

That's the whole folder. If you ever feel lost, come back to this picture — every concrete class you'll touch is in it.

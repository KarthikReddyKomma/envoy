# TLS Transport Socket (`envoy.transport_sockets.tls`)

Envoy's built-in TLS implementation, backed by BoringSSL (or AWS-LC — see `source/common/tls/aws_lc_compat.h`). Registers two transport socket config factories — upstream (client) and downstream (server) — under the well-known name `tls`. This folder is intentionally a thin registration shim; the heavy lifting lives in `source/common/tls/` and in the `cert_mappers/`, `cert_selectors/`, and `cert_validator/` sibling folders.

Proto:
- Downstream: `envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext`.
- Upstream: `envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext`.

## Files (this folder)
- `config.h` — `SslSocketConfigFactory` base class: returns the name `envoy.transport_sockets.tls`.
- `upstream_config.h/cc` — `UpstreamSslSocketFactory`. `createTransportSocketFactory` builds a `ClientContextConfigImpl` from the proto and then a `ClientSslSocketFactory` from `source/common/tls/client_ssl_socket.{h,cc}`. `LEGACY_REGISTER_FACTORY(..., "tls")`.
- `downstream_config.h/cc` — `DownstreamSslSocketFactory`. Builds a `ServerContextConfigImpl` and then a `ServerSslSocketFactory`. Takes `server_names` from the listener so the factory can match SNI against filter-chain server names.
- `BUILD` — splits the code into `base_config` (header only), `downstream_config`, `upstream_config`, and an aggregate `config` extension target.

## Core classes (in `source/common/tls/`, not this folder)
Because the implementation is large, the major classes are summarized here:
- `ContextImpl`, `ClientContextImpl`, `ServerContextImpl` — `SSL_CTX` lifetime and cert/cipher/ALPN/session-ticket configuration. `ServerContextImpl` drives SNI and certificate selection via pluggable selectors (see `cert_selectors/`).
- `ContextConfigImpl`, `ClientContextConfigImpl`, `ServerContextConfigImpl` — proto-to-C++ translation, including extracting cert validation, private-key providers, session ticket keys, OCSP policy, ALPN, and cipher suites.
- `ContextManagerImpl` — tracks live TLS contexts for hot-reload and for daily-expiration stats rollup.
- `SslSocket`, `ClientSslSocket`, `ServerSslSocket` — the `Network::TransportSocket` implementations. They own a BoringSSL `SSL*` and an `IoHandleBio` that bridges BoringSSL's BIO API to Envoy's `IoHandle`.
- `SslHandshakerImpl` — implements the handshaker interface used by `SslSocket` to run the BoringSSL state machine (`SSL_do_handshake`), dispatch callbacks for async certificate selection, verify peer certs (via a `CertValidator`), and raise `Connected` / `RemoteClose` events.
- `DefaultTlsCertificateSelector` — the built-in certificate selector that matches on SNI; overridden by the pluggable selectors in `cert_selectors/on_demand/`.
- `CertValidator` (in `source/common/tls/cert_validator/`) — abstract base for peer-certificate validation. Default implementation validates against a PEM trust bundle; alternative implementations live in `cert_validator/spiffe/` and `cert_validator/dynamic_modules/`.

## Transport socket role
`SslSocket` implements `Network::TransportSocket`:
- `doRead` / `doWrite` — drive `SSL_read` / `SSL_write` through the handshaker; deliver `SSL_ERROR_WANT_READ/WRITE` back as `PostIoAction::KeepOpen`; surface fatal errors as `Close`.
- `onConnected` — kicks off the handshake (`SSL_do_handshake`) on the upstream side; downstream defers until the first read.
- `closeSocket` — calls `SSL_shutdown` where possible to emit a close_notify.
- `ssl()` — returns a `ConnectionInfo` (peer cert fingerprint, SAN, cipher, version, ALPN, SNI, etc.).
- `failureReason()` — formatted string including the BoringSSL error stack.

`ClientSslSocketFactory` / `ServerSslSocketFactory` (from `common/tls/`) are the factories exposed to listeners/clusters; they expose `implementsSecureTransport() == true`, `supportsAlpn()`, `defaultServerNameIndication()`, `hashKey()` for upstream cluster pooling, `sslCtx()`, and `getCryptoConfig()` (for QUIC).

## Lifecycle
- Connect path: handshake runs either eagerly (client, on `onConnected`) or lazily (server, on first `doRead`). Certificate selection on server side hooks into `cert_selectors/` via `ServerContextImpl::selectTlsContext`. If a selector returns `Pending`, the handshake is suspended and resumed when the selector notifies its callback on the worker dispatcher.
- Data path: post-handshake, BoringSSL transparently encrypts/decrypts records; TLS 1.3 and TLS 1.2 are both supported.
- Close path: `SSL_shutdown` is attempted best-effort; pending writes can be drained via `canFlushClose()`.

## Key decision points
- `upstream_config.cc:15-29` — creates the client context config from the proto, then constructs a `ClientSslSocketFactory` via `ClientSslSocketFactory::create(...)`. The factory is registered through the `sslContextManager()` owned by the server factory context so hot restarts can re-home the contexts.
- `downstream_config.cc:15-29` — same pattern for server side; also passes `server_names` so the factory can reject configurations whose SNI set is inconsistent with the listener's filter chain match.
- `config.h:18` — the factory name is the single well-known string `envoy.transport_sockets.tls`; both up/downstream factories inherit from `SslSocketConfigFactory` to share that identity.

## Configuration
Both proto messages embed a `CommonTlsContext` which carries:
- `tls_params` — min/max protocol version, cipher suites, curves, ECDH.
- `tls_certificates[]` and `tls_certificate_sds_secret_configs[]` — inline or SDS-delivered certs/keys; can reference `PrivateKeyProvider` for HSM/async key operations.
- `validation_context` (inline, SDS, or combined) — trust bundle, verification mode, SAN matchers, CRL, `custom_validator_config` (see `cert_validator/`).
- `alpn_protocols`, `key_log`, `custom_handshaker`, `custom_tls_certificate_selector` (see `cert_selectors/`), `tls_certificate_provider_instance`, `custom_tls_certificate_mappers` (see `cert_mappers/`).

Downstream-only additions: `require_client_certificate`, `session_ticket_keys*`, `disable_stateless_session_resumption`, `ocsp_staple_policy`, `session_timeout`, `full_scan_certs_on_sni_mismatch`.

## Stats
Emitted by `source/common/tls/stats.{h,cc}` under `<stats_prefix>.ssl.`:
- Handshake counters: `handshake`, `connection_error`, `no_certificate`, `fail_verify_no_cert`, `fail_verify_error`, `fail_verify_san`, `fail_verify_cert_hash`, `ocsp_staple_failed`, `ocsp_staple_omitted`, `ocsp_staple_requests`, `ocsp_staple_responses`, `session_reused`, `was_key_usage_invalid`.
- Per-cipher / per-version / per-curve / per-sigalg tagged counters.
- Gauges: `days_until_first_cert_expiring` (per cert name).

## Errors
- Config-time: proto validation errors surface as `absl::InvalidArgumentError` through `ContextConfigImpl::create`. Missing CA certs, unreadable files, invalid PEMs, and PrivateKeyProvider initialization failures all abort factory creation.
- Handshake errors: captured in `SslSocket::failureReason()`; stats under `fail_verify_*` and `connection_error` depending on root cause.

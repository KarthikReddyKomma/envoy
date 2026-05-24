# `source/common/tls` — cases and scenarios

Quick‑reference, bullet‑point answers to "what happens when X". Use it to sanity‑check intuition against the actual code paths.

For each case: **trigger → flow → who's involved → notes**.

### How to use this doc

This is the "Stack Overflow" half of the documentation — answers without a lot of theory. Each section groups cases by lifecycle stage (config, handshake, data path, reload, debug, edge cases). The deep dives in `README.md`, `socket_layer.md`, `handshaker.md`, etc. explain the *why*. This doc just tells you the *path* — which files the call goes through, which counters move, where to set a breakpoint.

### Convention

- **Trigger**: what the user / xDS / network did to cause the case.
- **Flow**: the call chain through this codebase.
- **Notes**: gotchas, related counters, related configuration.

---

## A. Bring‑up cases

Config‑time scenarios — listener/cluster construction, SDS readiness, certificate parsing. These cases all run on the **main thread** and happen at most a few times per Envoy lifetime (per listener or per SDS update). Failures here generally surface as "listener failed to warm" or "secret not ready" rather than handshake errors.

### A1. Listener starts with an inline `DownstreamTlsContext`

- Listener manager hands the proto to `ServerSslSocketFactory::create` via the registration shim in `source/extensions/transport_sockets/tls/`.
- `ServerContextConfigImpl::create` parses certs, ciphers, ALPN, OCSP, session ticket keys.
- `ContextManagerImpl::createSslServerContext` takes the global mutex, builds a `ServerContextImpl`, inserts it into `contexts_`, returns a `shared_ptr`.
- `ServerContextImpl` builds one `SSL_CTX` per cert; installs `SSL_CTX_set_select_certificate_cb` on `tls_contexts_[0].ssl_ctx_`; computes the session context ID hash.
- Factory is ready; listener stops being "warming".

### A2. Listener starts but cert is from SDS and hasn't arrived

- `ServerContextConfigImpl::isReady()` returns false (`tls_certificate_providers_` non‑empty, `tls_certificate_configs_` empty).
- `ServerSslSocketFactory` does **not** build a `ServerContextImpl`.
- Incoming connections receive a `NotReadySslSocket` whose `failureReason()` says "secret not ready".
- `downstream_context_secrets_not_ready` counter increments per connection.
- When SDS eventually delivers, `onAddOrUpdateSecret` fires, the factory builds the context, and new connections get real `SslSocket`s.

### A3. Cluster starts with an `UpstreamTlsContext` that uses `auto_host_sni`

- `ClientContextConfigImpl::auto_host_sni_ = true`, `server_name_indication_` is the static default (typically empty).
- For each outgoing connection, `ClientContextImpl::newSsl` looks at the upstream host's DNS name and calls `SSL_set_tlsext_host_name(ssl, host_dns)` before handshake.
- ALPN is set via `SSL_set_alpn_protos(ssl, parsed_alpn_protocols_)`.

### A4. Listener with `custom_tls_certificate_selector = on_demand_secret`

- `ContextConfigImpl` builds the custom selector factory from the proto.
- `ServerContextImpl::tls_certificate_selector_` is the on‑demand selector instead of `DefaultTlsCertificateSelector`.
- `tls_contexts_` is still populated with statically configured certs (if any), but the on‑demand selector ignores them and asks SDS instead.
- Restriction enforced at config time: stateful and stateless session resumption must be disabled.

### A5. Server context build fails (bad cert, unreadable file)

- `ServerContextConfigImpl::create` returns `absl::InvalidArgumentError` with details.
- `ServerSslSocketFactory::create` propagates the error.
- Factory creation aborts; the listener fails to start (or fails the LDS update if reloaded).

---

## B. Handshake cases — server side

Per‑connection scenarios on the **downstream** side, running on a worker thread. These cases all start from `SslSocket::doRead` being called with the first bytes of a ClientHello and end either at `HandshakeComplete` or at a fatal error closing the connection. The interesting variations come from SNI, mTLS, OCSP, and the three async escape hatches.

### B1. Classic ClientHello, SNI matches a configured cert

- `SslSocket::doRead` fires on first bytes.
- `SslHandshakerImpl::doHandshake` → `SSL_do_handshake` → BIO reads via `IoHandleBio`.
- BoringSSL invokes `select_certificate_cb` (set on `tls_contexts_[0]`).
- `ServerContextImpl::selectTlsContext(client_hello)` runs `DefaultTlsCertificateSelector::selectTlsContext`.
- Selector looks up SNI in `server_names_map_`, picks the right key‑type variant (RSA vs ECDSA) based on client's `signature_algorithms`.
- `SSL_set_SSL_CTX(ssl, chosen.ssl_ctx_)` swaps in the right cert.
- If `ocspStapleAction == Staple` and a fresh OCSP response is available, `SSL_set_tlsext_status_ocsp_resp` adds the staple.
- BoringSSL finishes the handshake; `onSuccess(ssl)` increments `handshake`, logs per‑cipher/version/curve/sigalg tagged counters, transitions state to `HandshakeComplete`.

### B2. ClientHello has no SNI, `full_scan_certs_on_sni_mismatch = false`

- Selector falls through to `tls_contexts_[0]` (the first configured cert).
- Same handshake outcome but driven by the first cert regardless of any names it covers.

### B3. ClientHello has no SNI, `full_scan_certs_on_sni_mismatch = true`

- Selector scans all `tls_contexts_`, picking the first whose key type and OCSP capability match the client.
- Slower but more permissive — useful when many domains share a multi‑SAN cert.

### B4. SNI matches a wildcard cert (`*.example.com`)

- The cert was indexed under `.example.com` in `server_names_map_` (wildcards are prefixed with `.`).
- Selector looks up exact SNI first (miss), then wildcard form.
- `Utility::dnsNameMatch` follows RFC 6125 for the actual match.

### B5. mTLS — server requires client cert

- Validator's `addClientValidationContext` is called at config time with `require_client_cert = true`, sets `SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT`, populates the client CA name list (so the client knows which cert to send).
- During handshake, BoringSSL invokes `SSL_CTX_set_custom_verify` callback → `ContextImpl::customVerifyCallback` → `cert_validator_->doVerifyCertChain`.
- `DefaultCertValidator` walks the chain, runs SAN matchers / hash pinning / SPKI pinning.
- On success returns `ValidationResults{Successful, [chain]}`.
- The chain is stashed in `SslHandshakerImpl::validated_chain_` so `validatedPeerIssuer()` returns the direct issuer.

### B6. mTLS — client cert missing

- BoringSSL calls custom verify with an empty chain.
- `DefaultCertValidator::doVerifyCertChain` returns `Failed, ClientValidationStatus::NoClientCertificate`.
- `fail_verify_no_cert` counter increments.
- Handshake aborts with TLS alert `bad_certificate`.

### B7. mTLS — chain valid but SAN mismatch

- Chain verification succeeds.
- `verifyCertAndUpdateStatus` runs SAN matchers, none match.
- Returns `Failed`, `ClientValidationStatus::Failed`.
- `fail_verify_san` counter increments.
- TLS alert is `unsupported_certificate` (or whatever the matcher specifies via `out_alert`).

### B8. mTLS with dynamic_modules validator — async validation

- `cert_validator_` is `DynamicModuleCertValidator`.
- `doVerifyCertChain` calls into the module, which returns `Pending` after starting an external RPC.
- BoringSSL returns `SSL_ERROR_WANT_CERTIFICATE_VERIFY`; `SslHandshakerImpl::state_` stays `HandshakeInProgress`; `PostIoAction::KeepOpen` returned to socket layer.
- Module eventually invokes `ValidateResultCallbackImpl::onCertValidationResult` on the worker dispatcher.
- `SslExtendedSocketInfoImpl::onCertificateValidationCompleted` is called, which triggers `SslSocket::onAsynchronousCertValidationComplete`, which calls `resumeHandshake`.
- `SSL_do_handshake` is invoked again; this time the validator returns the stored result synchronously and the handshake completes.

### B9. On‑demand cert selector — cache miss

- `selectTlsContext` returns `SelectionResult::Pending`; `ServerContextImpl::selectTlsContext` returns `ssl_select_cert_retry`.
- BoringSSL pauses with `SSL_ERROR_PENDING_CERTIFICATE`.
- Selector kicks off SDS subscription for the SNI.
- When SDS delivers, `CertificateSelectionCallbackImpl::onCertificateSelectionResult` is posted to the worker dispatcher.
- `SslExtendedSocketInfoImpl::onCertificateSelectionCompleted` runs → `SslSocket::onAsynchronousCertificateSelectionComplete` → `resumeHandshake`.
- Handshake resumes; cert is cached in `ThreadLocalCerts` for subsequent connections.

### B10. Async private‑key operation (HSM signing)

- After cert selection, BoringSSL needs to sign the handshake transcript with the cert's private key.
- Cert's `private_key_method_provider_` is set → BoringSSL calls the provider's `sign` callback.
- Provider returns `ssl_private_key_retry`; BoringSSL returns `SSL_ERROR_WANT_PRIVATE_KEY_OPERATION`.
- `SslHandshakerImpl` keeps state at `HandshakeInProgress`.
- HSM finishes; provider calls `Ssl::PrivateKeyConnectionCallbacks::onPrivateKeyMethodComplete` → `SslSocket::onPrivateKeyMethodComplete` → `resumeHandshake`.
- BoringSSL re‑enters `method->complete`, gets the signature, finishes the handshake.

### B11. TLS 1.3 cert compression on the wire

- Both `registerBrotli` and `registerZlib` were called during `ContextImpl` build (unconditional).
- Client advertises `compress_certificate` extension with one or both algorithms.
- BoringSSL invokes `compressBrotli` / `compressZlib` from `cert_compression.cc` to compress the leaf+chain before sending.
- Client decompresses on receipt; no change in semantics, just fewer bytes on the wire.

### B12. Session resumption (TLS 1.3 ticket)

- Client sends a `pre_shared_key` extension with a valid ticket.
- BoringSSL invokes the ticket key callback (`ServerContextImpl::sessionTicketProcess`) to decrypt.
- Session matches; full handshake skipped; `session_reused` counter increments.
- ALPN / SNI / cert all carry over from the resumed session.

### B13. Session resumption fails because validator config changed

- Validator's `updateDigestForSessionId` mixed the (now‑old) config into the session context ID hash.
- New context has a different hash; the resumed session's `sid_ctx` doesn't match.
- BoringSSL falls back to a full handshake.
- No counter for "resumption attempted but skipped" — only `session_reused` (incremented on success).

### B14. OCSP staple expired, policy = `LENIENT_STAPLING`

- Selector calls `ocspStapleAction(ctx, client_ocsp_capable=true, policy=LENIENT_STAPLING)`.
- Action returned: `NoStaple`.
- `ocsp_staple_omitted` counter increments.
- Handshake proceeds normally without a staple.

### B15. OCSP staple expired, policy = `STRICT_STAPLING`

- Same setup, but action returned: `Fail`.
- Cert is not used; selector iterates to find another suitable cert.
- If no cert can be used, handshake aborts with `ocsp_staple_failed` counter incremented.

### B16. Cert is `must_staple` and client didn't advertise `status_request`

- `is_must_staple_` was set from the `1.3.6.1.5.5.7.1.24` extension on the cert during build.
- Selector returns `Fail` because the client can't receive the staple.
- Handshake aborts; cert can't be used for this client.

---

## C. Handshake cases — client side

Per‑connection scenarios on the **upstream** side. Mirror image of section B with one key difference: the client *initiates* the handshake immediately on connect, so the handshake kicks off in `onConnected()` rather than the first `doRead`. The client also has its own session‑ticket cache and may present a client cert if mTLS is in play.

### C1. Upstream connection — fresh handshake

- `ClientSslSocketFactory::createTransportSocket` returns an `SslSocket` initialised as `InitialState::Client`.
- `Network::Connection` calls `onConnected()` → `SslSocket::onConnected()` → `doHandshake()` (client kicks off the handshake immediately, no waiting for bytes).
- ALPN list set via `SSL_set_alpn_protos`; SNI set per `server_name_indication_` (or `auto_host_sni`); session ticket cache consulted.
- BoringSSL drives `BIO_write` (ClientHello) and `BIO_read` (ServerHello, Certificate, etc.) via `IoHandleBio`.
- Upstream cert validation runs through `cert_validator_->doVerifyCertChain` (`is_server = false`, `host_name` populated for SAN matching).
- On success, `ClientContextImpl::newSessionKey` may stash an `SSL_SESSION` for future resumption.

### C2. Upstream connection — session ticket reused

- Before `SSL_do_handshake`, `ClientContextImpl::newSsl` checked `session_keys_` under `session_keys_mu_`, found a ticket, called `SSL_set_session(ssl, session)`.
- BoringSSL skips full handshake; uses abbreviated handshake.
- `session_reused` counter increments on success.
- If `session_keys_single_use_` is true, the ticket is dropped from the deque.

### C3. Upstream mTLS — present a client cert

- Configured client cert in `tls_certificates[0]` of `UpstreamTlsContext`.
- Server requests a cert in `CertificateRequest`.
- BoringSSL pulls cert from `tls_contexts_[0]`; signs with the configured private key (or via provider for async signing).
- `enforce_rsa_key_usage_` flag (if set) checks the server's cert has the right `keyUsage` extension before completing.

### C4. Upstream with on‑demand client cert (filter_state_override mapper)

- A downstream filter wrote the desired secret name into filter state under key `envoy.tls.certificate_mappers.on_demand_secret`.
- `ClientContextImpl`'s upstream cert selector calls the `filter_state_override` mapper, which pulls the name from `options->downstreamSharedFilterStateObjects()`.
- Same on‑demand SDS subscription flow as B9, but for client cert.

### C5. Upstream SAN matcher with `auto_sni_san_match`

- `UpstreamTlsContext.auto_sni_san_match` set, and no explicit `match_typed_subject_alt_names`.
- `DefaultCertValidator` validates the server's cert SAN against the actual SNI sent.
- This is a guard against "I requested foo.example.com but got a cert valid for bar.example.com".

---

## D. Data‑path cases

Post‑handshake traffic — the hot path that fires for every byte going through the connection. The interesting code here is short (`SSL_read`/`SSL_write` loops) but the corner cases (partial writes, close_notify, mid‑stream errors) are where bugs hide.

### D1. `doRead` after handshake complete

- `info_.state() == HandshakeComplete`, so `doHandshake` is skipped.
- `sslReadIntoSlice` loop: `SSL_read` decrypts into a `RawSlice`; loop until `WANT_READ` or buffer full.
- Returns `IoResult{KeepOpen, bytes_read, false}`.

### D2. `doWrite` with partial write

- `SSL_write` returns < length because the underlying `IoHandle` couldn't take it all.
- `bytes_to_retry_` set to the *original* length (BoringSSL contract).
- Next `doWrite` must re‑pass the same pointer/length; `SSL_write` resumes.

### D3. Remote `close_notify`

- `SSL_read` returns 0; `SSL_get_error` returns `SSL_ERROR_ZERO_RETURN`.
- `SslSocket::doRead` returns `IoResult{Close, 0, false}` — clean shutdown.
- Local sends a `close_notify` via `closeSocket` if not already shut down.

### D4. Fatal SSL error mid‑stream

- `SSL_read` / `SSL_write` returns < 0; `SSL_get_error` returns something other than `WANT_READ` / `WANT_WRITE` / `ZERO_RETURN`.
- `drainErrorQueue` collects the BoringSSL error stack via `Utility::getLastCryptoError`, stores in `failure_reason_`.
- `connection_error` counter increments.
- Returns `IoResult{Close, ...}`.

---

## E. Reload / lifecycle cases

Configuration churn — what happens when SDS rotates a secret, LDS pushes a new listener, or a connection dies in the middle of an async step. The recurring theme is **lifetime separation**: new connections get the new config, in‑flight connections keep the old one via `shared_ptr`, and async callbacks are cancellation‑aware.

### E1. SDS pushes a new cert mid‑traffic

- SDS subscription fires `onAddOrUpdateSecret` on the factory.
- Factory locks `ssl_ctx_mu_`, rebuilds the `ContextImpl` via the manager, swaps `ssl_ctx_`, increments `ssl_context_update_by_sds`.
- New connections after the swap get the new context; in‑flight connections keep the old context until they close (held alive by `shared_ptr` on the `SslSocket`).
- Manager's `removeContext(old)` erases the old context from `contexts_` so stats stop including it.

### E2. SDS removes a secret

- Provider notifies the config; `ContextConfigImpl::isReady()` may flip to false.
- Factory rebuilds with no cert; future connections see `NotReadySslSocket`.
- In‑flight connections finish their current request and close.

### E3. Listener config update with new cipher list

- LDS push triggers full factory rebuild.
- New `ServerContextConfigImpl` built; new `ServerContextImpl` with different `SSL_CTX`s.
- All `SSL_CTX`s have new cipher list, new ALPN, possibly new validator.
- In‑flight connections use the old context; session resumption across the boundary fails because the session context ID hash changed.

### E4. Connection drops while waiting for on‑demand cert

- `SslSocket` destructor runs while selector still has a pending `Handle`.
- `~SslExtendedSocketInfoImpl` calls `CertificateSelectionCallbackImpl::onSslHandshakeCancelled`.
- The pending callback flips a flag so when SDS eventually delivers, the callback no‑ops instead of touching freed memory.

### E5. Connection drops while waiting for async cert validation

- Same idea, different callback (`ValidateResultCallbackImpl::onSslHandshakeCancelled`).
- The dynamic module's later `onCertValidationResult` post is harmless.

---

## F. Debug and admin cases

What's exposed to operators for visibility — `/certs` admin endpoint, expiration gauges, TLS keylog for packet decryption, transport failure reason in access logs. These cases all touch read‑only accessors and don't affect the data path.

### F1. `/certs` admin endpoint queried

- Admin handler iterates `ContextManagerImpl::iterateContexts`.
- Each context returns its `getCertChainInformation()` and `getCaCertInformation()`.
- Both use `Utility::certificateDetails(cert, path, time_source)` to format SANs, expiration, fingerprints.

### F2. `days_until_first_cert_expiring` stat refresh

- `ContextManagerImpl::daysUntilFirstCertExpires` iterates all contexts.
- Each context iterates `tls_contexts_` and the validator's CA cert.
- Minimum days wins.

### F3. TLS keylog enabled

- Listener config has `key_log.path = "/tmp/keys"` and address ranges.
- `ContextImpl` sets `SSL_CTX_set_keylog_callback(ssl_ctx, &keylogCallback)`.
- Per connection, callback checks `tls_keylog_local_` / `tls_keylog_remote_` ranges, writes one line per `CLIENT_RANDOM <client_random_hex> <secret_hex>` to `tls_keylog_file_`.
- Wireshark / tshark can decrypt the capture using this file.

### F4. Failure reason exposed to access log

- `SslSocket::failureReason()` returns `failure_reason_` populated in `drainErrorQueue`.
- Format: includes BoringSSL error stack (one or more lines) and the `SSL_get_error` code description.
- Available via `%CONNECTION_TERMINATION_DETAILS%` formatter and `connection.transportFailureReason()` from filters.

---

## G. Edge cases

The weird stuff — alternate crypto libraries (AWS‑LC), renegotiation, QUIC TLS integration, PKCS12 ingest, FIPS compliance toggle. None of these are common in default deployments, but every one of them has corner cases worth knowing about.

### G1. AWS‑LC build on ppc64le

- `OPENSSL_IS_AWSLC` defined; `aws_lc_compat.h` macros activate.
- `sk_X509_NAME_find` → `_awslc` variant.
- `SSL_CTX_set_compliance_policy` returns 0; `ContextConfigImpl` reports "compliance policy not supported" if `tls_params.compliance_policy` was set.
- `X509_NAME_dup` const wrapper applied.
- Everything else compiles unchanged.

### G2. Renegotiation requested by upstream

- BoringSSL processes the renegotiation request.
- If `ClientContextImpl::allow_renegotiation_ == false` (default), BoringSSL rejects.
- If true, BoringSSL re‑enters handshake state machine; same `SslHandshakerImpl::doHandshake` path.

### G3. QUIC TLS context

- `ENVOY_ENABLE_QUIC` define enables `TlsContext::quic_cert_` and `quic_private_key_`.
- QUIC proof source pulls these via `customVerifyCertChainForQuic`, bypassing the standard handshake path.
- Restriction: the on‑demand cert selector explicitly rejects QUIC listeners.

### G4. Cert is loaded from PKCS12 instead of PEM

- `TlsContext::loadPkcs12(data, path, password, fips_mode)` invoked from `ContextConfigImpl`.
- Decrypts the PKCS12 blob using the password, extracts cert + key.
- Same downstream path from there.

### G5. FIPS mode enabled

- `ClientContextConfigImpl::DEFAULT_CIPHER_SUITES_FIPS` / `DEFAULT_CURVES_FIPS` used as defaults.
- `loadPrivateKey(data, path, password, fips_mode=true)` runs additional sanity checks on the key.
- `was_key_usage_invalid` may fire if the cert lacks `digitalSignature` and enforcement is on.

### G6. PrivateKeyProvider isn't available (e.g. QAT card not plugged in)

- Provider's `isAvailable()` returns false at build time.
- `ContextConfigImpl::create` aborts with an error; the factory fails to build.
- Listener / cluster fails to come up cleanly rather than silently falling back to PEM.

---

## H. Stat trigger map

For quick reference: which counter fires from which path. Pair this table with the `SslStats` struct in `stats.h` and the tagged‑counter logic in `ContextImpl::logHandshake` to chase down any TLS‑related alert.

| Stat | Trigger |
|---|---|
| `handshake` | `ContextImpl::logHandshake` on success |
| `session_reused` | same path if `SSL_session_reused(ssl)` returns 1 |
| `no_certificate` | `DefaultCertValidator` saw chain but verification skipped (`ACCEPT_UNTRUSTED`) |
| `fail_verify_no_cert` | server required client cert, didn't get one |
| `fail_verify_error` | chain build / CA mismatch / CRL revoked |
| `fail_verify_san` | SAN matchers all returned false |
| `fail_verify_cert_hash` | hash pinning or SPKI pinning mismatch |
| `ocsp_staple_requests` | client advertised `status_request` |
| `ocsp_staple_responses` | Envoy actually stapled |
| `ocsp_staple_omitted` | `LENIENT_STAPLING` and staple expired/missing |
| `ocsp_staple_failed` | policy required staple and none usable |
| `connection_error` | fatal SSL error mid‑connection (`drainErrorQueue`) |
| `was_key_usage_invalid` | RSA cert without `digitalSignature`, with enforcement on |
| `ssl_context_update_by_sds` | factory rebuilt context after SDS push |
| `upstream_context_secrets_not_ready` | client TLS asked for socket before SDS delivered |
| `downstream_context_secrets_not_ready` | server TLS same as above |

Plus tagged counters for cipher / version / curve / sigalg (per‑handshake) via `ContextImpl::incCounter`.

---

## I. Where to look first when…

| Symptom | Start here |
|---|---|
| Connections fail with TLS error on a listener | `SslSocket::failureReason()` in access log; `connection_error` stat |
| "Secret not ready" failures | SDS subscription health; `*_context_secrets_not_ready` stat |
| Wrong cert presented for SNI | `DefaultTlsCertificateSelector::findTlsContext`; `server_names_map_` indexing |
| mTLS rejects valid client | `cert_validator_->doVerifyCertChain` path; `fail_verify_*` stat |
| Handshake hangs | Look for `SSL_ERROR_PENDING_CERTIFICATE` / `WANT_CERTIFICATE_VERIFY` / `WANT_PRIVATE_KEY_OPERATION` and the corresponding async callback never firing |
| Wrong cert chosen on cipher‑heavy ClientHello | `DefaultTlsCertificateSelector` key‑type filter; `tls_contexts_[i].ec_group_curve_name_` |
| Cert chain too big | `cert_compression.{h,cc}` should already compress; check Brotli/Zlib registration |
| OCSP staple not present | `ocspStapleAction` in `default_tls_certificate_selector.cc`; OCSP wrapper expiry; client `status_request` extension |
| Slow handshake | Inspect cert chain size, private key op latency (HSM round‑trip), validator async work |
| FIPS compliance issue | `tls_params.compliance_policy` + `SSL_CTX_set_compliance_policy` (no‑op on AWS‑LC) |

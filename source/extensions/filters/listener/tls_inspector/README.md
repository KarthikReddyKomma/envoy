# TLS Inspector Listener Filter (`envoy.filters.listener.tls_inspector`)

Peeks at the TLS ClientHello on newly accepted connections and, without terminating TLS, extracts the SNI, ALPN, and optionally computes JA3 / JA4 fingerprints. Extracted values are written to the `ConnectionSocket` so later filter-chain matching can route by `server_names` or `application_protocols`. The inspector drives a BoringSSL handshake against an in-memory BIO and aborts from the early `select_certificate` callback (`ssl_select_cert_error`) so no real handshake is performed.

Proto: `envoy.extensions.filters.listener.tls_inspector.v3.TlsInspector`.

## Files
- `tls_inspector.h` / `tls_inspector.cc` — `Config` (holds the shared `SSL_CTX` configured with `TLS_with_buffers_method`, min/max TLS versions, and fingerprint flags, `tls_inspector.cc:51`) and `Filter` (per-connection listener filter).
- `ja4_fingerprint.h` / `ja4_fingerprint.cc` — `JA4Fingerprinter::create()` produces the `tXXdYYZZ_CIPHERHASH_EXTENSIONHASH` JA4 string from a `SSL_CLIENT_HELLO`.
- `config.cc` — `TlsInspectorConfigFactory`: constructs the shared `Config` and installs the filter via `filter_manager.addAcceptFilter`; registers `envoy.filters.listener.tls_inspector` plus deprecated alias `envoy.listener.tls_inspector` (`config.cc:43`, `config.cc:49`).

## Lifecycle
- `onAccept(cb)` — caches callbacks and returns `StopIteration` to wait for peeked bytes (`tls_inspector.cc:106`).
- `onData(buffer)` — because peeking returns the same bytes repeatedly, only bytes past `read_` are fed into BoringSSL (`tls_inspector.cc:151`). Calls `parseClientHello(...)`. On `Error` closes the IO handle and returns `StopIteration`; on `Done` returns `Continue`; on `Continue` keeps waiting (`tls_inspector.cc:157`-`tls_inspector.cc:167`).
- `maxReadBytes()` — returns `requested_read_bytes_`, which starts at `initial_read_buffer_size_` and doubles up to `maxClientHelloSize()` when BoringSSL signals `SSL_ERROR_WANT_READ` (`tls_inspector.cc:200`).

## Decision / logic
- `Config` constructor installs `SSL_CTX_set_select_certificate_cb` (`tls_inspector.cc:78`) that fires as soon as the ClientHello is parsed. From inside that callback the filter calls `createJA3Hash`, `createJA4Hash`, `onALPN`, `onServername`, and returns `ssl_select_cert_error` to halt the handshake.
- `onServername(name)` — `socket.setRequestedServerName(name)` when SNI is present and increments `sni_found`; otherwise increments `sni_not_found`. Always marks `clienthello_success_ = true` (`tls_inspector.cc:134`).
- `onALPN(data,len)` — parses the ALPN extension and calls `socket.setRequestedApplicationProtocols(protocols)`, setting `alpn_found_ = true` (`tls_inspector.cc:113`-`tls_inspector.cc:131`).
- `createJA3Hash` — emits `version,ciphers,extensions,curves,point_formats` into a fingerprint string, MD5s it, hex-encodes, and calls `socket.setJA3Hash(...)` (`tls_inspector.cc:370`).
- `createJA4Hash` — invokes `JA4Fingerprinter::create(...)` and calls `socket.setJA4Hash(...)` (`tls_inspector.cc:394`).
- `parseClientHello` — feeds the new bytes into a mem-BIO (`BIO_new_mem_buf`, `BIO_set_mem_eof_return(-1)`) and calls `SSL_do_handshake` (`tls_inspector.cc:249`). The expected result is always `<= 0` because the select-cert callback errors out (`tls_inspector.cc:264`). `getParserState` maps `SSL_get_error` to `Continue`/`Done`/`Error`:
  - `SSL_ERROR_WANT_READ` with `read_ >= maxConfigReadBytes()` → increments `client_hello_too_large`, writes dynamic metadata `failure_reason = ClientHelloTooLarge`, returns `Error` (`tls_inspector.cc:192`-`tls_inspector.cc:199`).
  - `SSL_ERROR_SSL` with `clienthello_success_` → increments `tls_found`, `alpn_found`/`alpn_not_found`, calls `socket.setDetectedTransportProtocol("tls")` and returns `Done` (`tls_inspector.cc:214`-`tls_inspector.cc:221`).
  - `SSL_ERROR_SSL` without success → increments `tls_not_found`, sets transport failure reason via `streamInfo().setDownstreamTransportFailureReason(...)`, optionally closes the connection when `close_connection_on_client_hello_parsing_errors` is set (`tls_inspector.cc:226`-`tls_inspector.cc:242`).
- At the end of every parse attempt, `bytes_processed` histogram records the number of bytes BoringSSL consumed (`tls_inspector.cc:269`).

## Configuration
- `enable_ja3_fingerprinting` / `enable_ja4_fingerprinting` — toggle the hashers.
- `max_client_hello_size` — upper bound (default and max `SSL3_RT_MAX_PLAIN_LENGTH`, validated in constructor, `tls_inspector.cc:69`).
- `initial_read_buffer_size` — starting peek size; capped to `max_client_hello_size`.
- `close_connection_on_client_hello_parsing_errors` — if true, non-TLS traffic is dropped instead of being passed through.

## Stats
Rooted under `tls_inspector.` in the listener scope (`tls_inspector.cc:54`), declared at `tls_inspector.h:25`:
- `tls_inspector.client_hello_too_large` — BoringSSL requested more than `max_client_hello_size` bytes.
- `tls_inspector.tls_found` / `tls_inspector.tls_not_found` — ClientHello detected or not.
- `tls_inspector.alpn_found` / `tls_inspector.alpn_not_found` — ALPN extension presence (only counted when TLS was detected).
- `tls_inspector.sni_found` / `tls_inspector.sni_not_found` — SNI extension presence.
- `tls_inspector.bytes_processed` — histogram (bytes) for each parse completion.

Dynamic metadata at namespace `envoy.filters.listener.tls_inspector` may carry `failure_reason = ClientHelloTooLarge | ClientHelloNotDetected` (`tls_inspector.cc:404`-`tls_inspector.cc:417`).

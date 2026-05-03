# TLS Certificate Selectors

A *certificate selector* plugs into `ServerContextImpl` / `ClientContextImpl` at handshake time and decides which `Ssl::TlsContext` (certificate + key + chain + OCSP response) to use for the connection. Selectors can return synchronously (the context is already resident) or asynchronously (the selector must fetch the secret first).

Two interfaces are defined in `envoy/ssl/handshaker.h`:
- `Ssl::TlsCertificateSelector` — downstream. `selectTlsContext(const SSL_CLIENT_HELLO&, CertificateSelectionCallbackPtr)` returns a `SelectionResult` with status `Success` / `Pending` / `Failed`.
- `Ssl::UpstreamTlsCertificateSelector` — upstream. `selectTlsContext(const SSL&, TransportSocketOptionsConstSharedPtr, CertificateSelectionCallbackPtr)`.

`SelectionResult::Pending` suspends the BoringSSL handshake; the selector then invokes the callback on the worker dispatcher when the certificate becomes available.

The default selector (`source/common/tls/default_tls_certificate_selector.{h,cc}`) walks the statically configured `tls_certificates[]` and picks by SNI / signature algorithm / curve. This folder holds *alternative* selectors that plug in via `custom_tls_certificate_selector` in the `CommonTlsContext`.

## Subfolders
- `on_demand/` — asynchronously loads certificates from SDS on the first handshake that references them, keyed by a name produced by a `../cert_mappers/` plug-in. Caches per-worker via a thread-local slot. Supports both downstream and upstream.

## Stats / errors
Selector-level stats live in each subfolder. Configuration errors surface at `createTlsCertificateSelectorFactory` time (e.g. the on-demand selector rejects QUIC listeners and configurations that leave session resumption enabled).

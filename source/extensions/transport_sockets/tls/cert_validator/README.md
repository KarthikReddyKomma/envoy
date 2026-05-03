# TLS Certificate Validators

Peer certificate validation plug-ins. Consumed by `SslHandshakerImpl` when it verifies the presented certificate chain during a TLS handshake. The `CertValidator` interface (`source/common/tls/cert_validator/cert_validator.h`) defines the contract:

- `addClientValidationContext(SSL_CTX*, bool require_client_cert)` — register CA names / verify callbacks on the `SSL_CTX`.
- `doVerifyCertChain(...)` — invoked when BoringSSL produces a chain; returns `ValidationResults` (Success / Failed, detailed `ClientValidationStatus`, optional TLS alert, error string).
- `initializeSslContexts(...)` — returns the `SSL_VERIFY_*` flags to apply.
- `updateDigestForSessionId(...)` — ensures per-validator state is hashed into the session ID so resumption doesn't cross validator boundaries.
- `daysUntilFirstCertExpires`, `getCaFileName`, `getCaCertInformation` — admin-plane surfaces.

The default validator lives in `source/common/tls/cert_validator/` and validates against a PEM trust bundle plus optional CRL/SAN matchers. This folder holds *alternative* validators selectable via `CertificateValidationContext.custom_validator_config` in the TLS context proto.

## Subfolders
- `spiffe/` — SPIFFE / SVID validator. Validates URI SANs of the form `spiffe://<trust-domain>/<path>` against a trust-domain-indexed bundle store. Supports both static trust domains and a dynamic SPIFFE bundle map via `DataSourceProvider`.
- `dynamic_modules/` — delegates validation to a dynamically loaded native module implementing the `envoy_dynamic_module_on_cert_validator_*` ABI. Used for embedding custom validation logic (e.g. CA signed by internal infrastructure, third-party attestation services).

## Stats
Common TLS stats (`source/common/tls/stats.{h,cc}`) are incremented by the validators themselves:
- `fail_verify_error` — any chain verification failure.
- `fail_verify_san` — SAN match failed.
- `fail_verify_no_cert`, `fail_verify_cert_hash` — handled elsewhere.

Per-validator stats are documented in each subfolder's README.

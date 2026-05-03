# SPIFFE Certificate Validator (`envoy.tls.cert_validator.spiffe`)

Certificate validator for SPIFFE X.509-SVIDs. Extracts the trust domain from the leaf certificate's single URI SAN of the form `spiffe://<trust-domain>/<path>`, then verifies the chain against the matching trust-bundle `X509_STORE` for that (peer-trust-domain, local-trust-domain) pair. Supports both static trust domains and a dynamic SPIFFE trust-bundle map.

Proto: `envoy.extensions.transport_sockets.tls.v3.SPIFFECertValidatorConfig` (fields: `trust_domains[]` with inline CA PEMs, or `trust_bundles` data source referencing a JWKS-style JSON bundle map). Referenced via `CertificateValidationContext.custom_validator_config`.

## Files
- `spiffe_validator.h` — declares:
  - `SpiffeData` (`spiffe_validator.h:36`) — the core state: `trust_bundle_stores_` is a nested map `peer_trust_domain -> local_trust_domain -> X509_STORE` plus a flat list of all CA certs (`ca_certs_`) for admin / session-ID hashing.
  - `SPIFFEValidator` — the `CertValidator` implementation.
- `spiffe_validator.cc` — implements the validator plus the `SPIFFEValidatorFactory` and an anonymous `parseTrustBundles` helper that turns a JSON bundle map into a shared `SpiffeData`.

## Interface / implementation

### Construction (`spiffe_validator.cc:124-204`)
Two configuration modes:
1. **`trust_bundles` data source**: registered as a `DataSource::ProviderSingleton<SpiffeData>` under `spiffe_trust_bundles`. A shared provider polls the file / SDS source; on each update, `parseTrustBundles` (`spiffe_validator.cc:40-118`) re-parses JSON into a new `SpiffeData`. The validator reads the current bundle via `bundle_provider_->data()` at verify time (`spiffe_validator.h:67-70`).
2. **Static `trust_domains[]`**: each entry's inline CA PEM is loaded into an `X509_STORE`; duplicates of `(name, workload_trust_domain)` are rejected. CRLs embedded in the PEM are added and the `CRL_CHECK` / `CRL_CHECK_ALL` flags are set. Collected `ca_certs_` are used for session-ID hashing and admin introspection.

### `doVerifyCertChain` (`spiffe_validator.cc:294-323`)
1. Rejects an empty chain with `NotValidated`.
2. Extracts the *workload* trust domain from the connection's filter state under key `envoy.tls.cert_validator.spiffe.workload_trust_domain`:
   - Server-side: reads `Router::StringAccessor` from the connection's `streamInfo().filterState()`.
   - Client-side: reads from `transport_socket_options->downstreamSharedFilterStateObjects()`.
3. Calls `verifyCertChainUsingTrustBundleStore` (`spiffe_validator.cc:243`), which:
   - Runs `certificatePrecheck` — rejects CA certs (`EXFLAG_CA`) and certs with `keyCertSign` / `cRLSign` usage (per SPIFFE X.509-SVID §5.2).
   - Extracts the peer trust domain from the leaf's URI SAN (`getTrustBundleStore`, `spiffe_validator.cc:325`).
   - Looks up `trust_bundle_stores_[peer_trust_domain][workload_trust_domain]`; missing entries → `Failed`.
   - Allocates a fresh `X509_STORE_CTX`, copies verify params, optionally sets `X509_V_FLAG_NO_CHECK_TIME` if `allow_expired_certificate_` is set, calls `X509_verify_cert`.
   - On success, captures the verified chain via `X509_STORE_CTX_get0_chain` (returned in `ValidationResults`).
   - Applies any configured `subject_alt_name_matchers_` (URI-only — non-URI matchers are ignored with a TODO comment at `spiffe_validator.cc:136`).
4. Returns `Successful` / `Failed` with appropriate `ClientValidationStatus` and the captured validated chain.

### Other `CertValidator` methods
- `addClientValidationContext` (`spiffe_validator.cc:206`) — pushes deduped `X509_NAME` entries from every CA into `SSL_CTX_set_client_CA_list` so the server's CertificateRequest advertises them.
- `initializeSslContexts` (`spiffe_validator.cc:241`) — always returns `SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT`; SPIFFE always wants a client cert.
- `updateDigestForSessionId` (`spiffe_validator.cc:229`) — hashes every CA's SHA-256 into the session digest so different bundles don't share session IDs.
- `daysUntilFirstCertExpires` (`spiffe_validator.cc:421`) — minimum remaining days across `ca_certs_`; returns `UINT32_MAX` when empty.
- `getCaFileName` / `getCaCertInformation` — returns the first CA's name / details (TODO comments note the single-cert limitation).

### `extractTrustDomain` (`spiffe_validator.cc:388`)
Parses `spiffe://domain/rest` into `domain`. Returns `""` on any prefix or format mismatch; the validator then refuses to find a trust bundle.

## Lifecycle
- Config load: `SPIFFEValidator` is constructed; if `trust_bundles` is set, a `SpiffeTrustBundles` singleton is created or reused and a `DataSourceProvider` subscription started. Initial bundle parse happens in the provider before returning.
- Runtime: on bundle update (inotify or SDS), the provider swaps `data()`; subsequent verifications see the new bundle without holding any lock.
- Shutdown: the singleton is reference-counted via the factory; once no validators hold it, the SDS subscription is torn down.

## Key decision points
- `spiffe_validator.cc:131-141` — only URI SAN matchers are honored; other types are silently dropped.
- `spiffe_validator.cc:292` — workload trust domain lookup is `key = "envoy.tls.cert_validator.spiffe.workload_trust_domain"`. Not supplying it means the nested lookup uses the empty string, which matches entries configured with an empty `workload_trust_domain`.
- `spiffe_validator.cc:264-266` — `allow_expired_certificate_` sets `X509_V_FLAG_NO_CHECK_TIME` on the per-verify context (not the bundle's `X509_STORE`).
- `spiffe_validator.cc:274-281` — the validated chain is extracted from `X509_STORE_CTX_get0_chain` and uprefed into `validated_chain`; this is what downstream consumers (like the `peer_certificate_chain` connection info getter) see.
- `spiffe_validator.cc:401-419` — per-CA expiration gauges are initialized with an index suffix so `<ca_cert_name>_0`, `<ca_cert_name>_1`, … each track their own expiry.

## Configuration
Two mutually exclusive modes:
- `trust_domains[]` — each `{name, trust_bundle, workload_trust_domain?}` where `trust_bundle` is a PEM data source. Workload trust domain is an inner-keyed dimension for multi-tenant routing.
- `trust_bundles` — a data source containing a JSON SPIFFE bundle map `{"trust_domains": {"<domain>": {"keys": [{"use": "x509-svid", "x5c": ["<base64-DER>", ...]}]}}}`.

Only `use: x509-svid` is currently supported; `jwt` keys are rejected with an `InvalidArgumentError`.

## Stats / errors
- `fail_verify_error` (common TLS counter) — chain verification failure, missing trust bundle, precheck failure.
- `fail_verify_san` — SAN matcher list present and no match.
- Per-cert expiration gauges under `<scope>.ssl.certificate.<cert_name>_<idx>.expiration_unix_time_seconds`.
- Config errors surface through `creation_status`: missing trust domains, duplicate entries, malformed JSON, invalid x5c, unreadable PEM.

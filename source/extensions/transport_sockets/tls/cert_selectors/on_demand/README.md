# On-Demand Certificate Selector (`envoy.tls.certificate_selectors.on_demand_secret`)

Certificate selector that fetches TLS certificates via SDS at handshake time rather than eagerly at config load. For each incoming connection, the configured `../cert_mappers/` plug-in derives a secret name from the ClientHello (or upstream ServerHello / filter state). If the named context is already cached on this worker, the handshake continues synchronously; otherwise the selector returns `Pending`, starts an SDS subscription, and resumes the handshake once the secret arrives. Supports both downstream and upstream sides.

Proto: `envoy.extensions.transport_sockets.tls.cert_selectors.on_demand_secret.v3.Config` (fields: `config_source`, `certificate_mapper`, `prefetch_secret_names[]`).

## Files
- `config.h` — declares the main types (see below).
- `config.cc` — implements them.

## Key classes

### `SecretManager` (`config.h:163`)
Thread-safe, main-thread-authoritative cache of SDS subscriptions and derived TLS contexts. Each cache entry (`CacheEntry`, `config.h:221`) owns:
- `AsyncContextConfigConstPtr cert_config_` — the active SDS subscription + last received `TlsCertificateConfig`.
- `AsyncContextConstSharedPtr cert_context_` — the fully built TLS context (`ServerContextImpl` / `ClientContextImpl` subclass).
- `std::vector<std::weak_ptr<Handle>> callbacks_` — pending handshakes waiting for this secret.

Also maintains `ThreadLocalCerts` (`config.h:229`) — a lock-free worker-local map of `name -> AsyncContextConstSharedPtr` for synchronous lookups by the selector on worker threads. Main-thread `setContext` (`config.cc:204`) pushes updates into each worker's slot via `runOnAllThreads`.

### `AsyncContextConfig` (`config.h:43`)
Holds a live SDS subscription. On `loadCert` (`config.cc:28`), when the SDS secret has arrived, builds a `TlsCertificateConfigImpl` and calls the `update_cb` (which maps to `SecretManager::updateCertificate`). Calls `remove_cb` on secret deletion.

### `ServerAsyncContext` / `ClientAsyncContext` (`config.h:90`, `config.h:109`)
Specialize `ServerContextImpl` / `ClientContextImpl` for a *single* certificate fetched on demand. They plug into the rest of the TLS machinery so that once the context is populated, handshakes proceed as if the cert were statically configured.

### `Handle` (`config.h:130`)
Represents either a synchronous (already-cached) or asynchronous pending handshake. `notify(cert_ctx)` (`config.cc:46`) posts `CertificateSelectionCallback::onCertificateSelectionResult` to the worker's dispatcher.

### `AsyncSelector` / `UpstreamAsyncSelector` (`config.h:249`, `config.h:269`)
Implement the selector interfaces. Both wrap a `cert_mappers` mapper and share `BaseAsyncSelector::doSelectTlsContext` (`config.cc:229`) which:
1. Calls `secret_manager_->getContext(name)` on the thread-local cache.
2. On hit: returns `SelectionResult{Success, selected_ctx, staple, handle}` with the cached `TlsContext*`.
3. On miss: returns `Pending` with `handle = secret_manager_->fetchCertificate(name, cb, client_ocsp_capable)`, which posts to the main thread to start the SDS subscription.

`AsyncSelector::selectTlsContext` (`config.cc:253`) also computes OCSP capability from the ClientHello. `AsyncSelector::findTlsContext` panics (QUIC path not supported — see `config.cc:313`).

### `OnDemandTlsCertificateSelectorFactory` / `OnDemandTlsCertificateSelectorConfigFactory`
Downstream selector factory (`config.h:293`, `config.h:330`). `createTlsCertificateSelectorFactory` (`config.cc:308`) enforces two invariants:
- `for_quic` must be false (`config.cc:313`).
- Both stateless and stateful session resumption must be disabled (`config.cc:320`) — session IDs are keyed by the parent TLS context's certs, which would not match an on-demand cert.

### `UpstreamOnDemandTlsCertificateSelectorConfigFactory`
Upstream counterpart (`config.h:348`, `config.cc:336`). No QUIC / session-resumption restrictions, but otherwise structurally identical.

## Lifecycle
- Config load: `SecretManager` is created; any `prefetch_secret_names[]` kick off SDS subscriptions eagerly so the first handshake sees a cache hit.
- Handshake miss: selector returns `Pending`, posts to main to create an `AsyncContextConfig`. When SDS delivers the secret, `AsyncContextConfig::loadCert` calls `SecretManager::updateCertificate`, which constructs the `AsyncContext`, pushes it to all workers, and notifies each pending `Handle`. Each `Handle` then posts `onCertificateSelectionResult` to the originating worker, and BoringSSL resumes the paused handshake.
- Config push (parent TLS context updated): `BaseCertificateSelectorFactory::onConfigUpdate` → `SecretManager::updateAll` (`config.cc:135`) rebuilds all cached contexts with the new parent config.
- Secret removal: `SecretManager::removeCertificateConfig` (`config.cc:151`) posts `doRemoveCertificateConfig` to main, which notifies pending handles with `nullptr` (→ handshake aborts) and clears worker caches.
- Handle destruction: if the connection is reset before the cert arrives, `Handle` is dropped; the `weak_ptr` in `CacheEntry::callbacks_` expires and is skipped on notify.

## Key decision points
- `config.cc:50-62` — `Handle::notify` computes OCSP staple action *before* posting to the dispatcher so the final callback has everything it needs without touching main-thread state.
- `config.cc:189-200` — `fetchCertificate` uses `weak_ptr<SecretManager>` to survive the case where the filter chain is removed while a post is in flight.
- `config.cc:204-215` — `setContext` pushes updates using `TypedSlot::runOnAllThreads`; the completion callback increments `cert_updated` once all workers have applied the change.

## Configuration
- `config_source` — SDS config source for fetching certificates.
- `certificate_mapper` — a `../cert_mappers/` plug-in extension config.
- `prefetch_secret_names[]` — optional list of secret names to begin subscribing to at config load.

Parent `DownstreamTlsContext` must have both stateful and stateless session resumption disabled to use this selector downstream.

## Stats
Prefixed `<scope>.on_demand_secret.`:
- `cert_requested` (counter) — new subscription started.
- `cert_updated` (counter) — worker caches updated for a certificate.
- `cert_active` (gauge) — number of cached cert subscriptions currently alive.

## Errors
Configuration-time errors (returned from `createTlsCertificateSelectorFactory`):
- `"Does not support QUIC listeners."`
- `"On demand certificates are not integrated with session resumption support."`
Handshake-time: certificate build errors surface via the `creation_status` flow in `ServerAsyncContext` / `ClientAsyncContext` constructors; failed handles notify callbacks with `nullptr`, which BoringSSL treats as a failed handshake.

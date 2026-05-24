# Config translation & context management

> Files: `context_config_impl.{h,cc}`, `context_manager_impl.{h,cc}`

These two together form the **bridge between protobuf and `SSL_CTX`**. `ContextConfigImpl` translates `CommonTlsContext` (and its server / client wrappers) into the strongly‑typed internal config that `ContextImpl` consumes. `ContextManagerImpl` is the per‑process owner of every live `ContextImpl`.

### Why the split?

Translation is **expensive and synchronous** (PEM parsing, SDS fetching, validator factory instantiation). It happens once per listener/cluster at config load (or on SDS update) on the main thread. Building the actual `SSL_CTX` is *also* expensive (cipher list compilation, CA store population, OCSP response parsing). Splitting "translate proto" from "build context" buys two things: (1) the translated config can be **inspected and tested** without touching BoringSSL, and (2) the manager can dedupe identical contexts by hashing the translated config, so two listeners with the same TLS settings share one `SSL_CTX`.

### Lifecycle in one sentence

xDS pushes proto → `ContextConfigImpl::create` translates it → manager builds a `Client`/`ServerContextImpl` → factory caches a `shared_ptr` → workers see it on next `createTransportSocket`.

---

## Layered view

```mermaid
flowchart TB
    subgraph Proto["Protobuf inputs"]
      DSP["DownstreamTlsContext"]
      USP["UpstreamTlsContext"]
      CTC["CommonTlsContext<br/>(shared)"]
    end

    subgraph CfgImpl["Config translation (context_config_impl)"]
      CCI["ContextConfigImpl<br/>(base)"]
      CCFI["ClientContextConfigImpl"]
      SCFI["ServerContextConfigImpl"]
    end

    subgraph Mgr["Manager (context_manager_impl)"]
      CMI["ContextManagerImpl"]
      PKMM["PrivateKeyMethodManagerImpl"]
    end

    subgraph CtxImpl["TLS contexts (context_impl)"]
      CCI2["ClientContextImpl"]
      SCI2["ServerContextImpl"]
    end

    DSP --> CTC
    USP --> CTC
    DSP --> SCFI
    USP --> CCFI
    CTC --> CCI
    CCI --> CCFI
    CCI --> SCFI

    CCFI --> CMI
    SCFI --> CMI
    CMI --> CCI2
    CMI --> SCI2

    CMI -. "owns" .-> PKMM
```

Translation is one‑shot at config time; the manager keeps a `flat_hash_set` of contexts for the life of the process.

The five layers are deliberately kept *unidirectional* — proto flows into config, config flows into context, context flows into manager. Nothing flows back the other way. That makes the wiring easy to reason about: a misconfigured cipher in proto only ever affects the matching `SSL_CTX`, never the manager or the wider Envoy state.

---

## `ContextConfigImpl` — proto → internal config

`context_config_impl.h:33‑139`. Implements `Ssl::ContextConfig`. The exposed interface (what `ContextImpl` reads):

```mermaid
classDiagram
    class ContextConfig {
      <<interface>>
      +alpnProtocols() : string
      +cipherSuites() : string
      +ecdhCurves() : string
      +signatureAlgorithms() : string
      +compliancePolicy() : CompliancePolicy?
      +tlsCertificates() : TlsCertificateConfig[]
      +certificateValidationContext() : CertificateValidationContextConfig*
      +minProtocolVersion() / maxProtocolVersion()
      +tlsKeyLogLocal / tlsKeyLogRemote / tlsKeyLogPath()
      +isReady()
      +setSecretUpdateCallback(cb)
      +createHandshaker() : HandshakerFactoryCb
      +capabilities() / sslctxCb()
    }

    class ContextConfigImpl {
      #api_, options_, singleton_manager_, lifecycle_notifier_
      #auto_sni_san_match_
      -alpn_protocols_ / cipher_suites_ / ecdh_curves_ / signature_algorithms_
      -tls_certificate_configs_ : vector~TlsCertificateConfigImpl~
      -validation_context_config_ : CertificateValidationContextConfigPtr
      -default_cvc_ (for combined CVC)
      -tls_certificate_providers_ : provider+name[]
      -tc_update_callback_handles_ : CallbackHandle[]
      -certificate_validation_context_provider_
      -cvc_update_callback_handle_
      -min_protocol_version_ / max_protocol_version_
      -handshaker_factory_cb_ / capabilities_ / sslctx_cb_
      -tls_keylog_*
      -compliance_policy_
      +getCombinedValidationContextConfig(dynamic_cvc, name)
    }

    class ClientContextConfigImpl {
      +DEFAULT_CIPHER_SUITES / _FIPS
      +DEFAULT_CURVES / _FIPS
      -server_name_indication_
      -auto_host_sni_ / allow_renegotiation_ / enforce_rsa_key_usage_
      -max_session_keys_
      -tls_certificate_selector_factory_
    }

    class ServerContextConfigImpl {
      -require_client_certificate_
      -session_ticket_keys_ / session_ticket_keys_provider_
      -ocsp_staple_policy_ / ocsp_*
      -full_scan_certs_on_sni_mismatch_
      -prefer_client_ciphers_
      -tls_certificate_selector_factory_
    }

    ContextConfigImpl --|> ContextConfig
    ClientContextConfigImpl --|> ContextConfigImpl
    ServerContextConfigImpl --|> ContextConfigImpl
```

(`ServerContextConfigImpl` is declared in `server_context_config_impl.h` — same layout pattern, omitted here for brevity.)

### What `ContextConfigImpl` does at construction (high level)

```mermaid
flowchart TB
    A["ContextConfigImpl(CommonTlsContext config, ...)"] --> B["tlsVersionFromProto(min, max, defaults)"]
    B --> C["alpn_protocols_ = config.alpn_protocols (joined)"]
    C --> D["cipher_suites_ = config.tls_params.cipher_suites or default"]
    D --> E["ecdh_curves_ = config.tls_params.ecdh_curves or default"]
    E --> F{"tls_certificates set inline?"}
    F -- yes --> G["build TlsCertificateConfigImpl per cert"]
    F -- no --> H{"tls_certificate_sds_secret_configs set?"}
    H -- yes --> I["start SDS providers,<br/>register update callbacks -> tc_update_callback_handles_"]
    H -- no --> J{"tls_certificate_provider_instance set?"}
    J -- yes --> K["lookup named provider, register"]

    G --> L["build validation_context_config_ or combined_validation_context"]
    I --> L
    K --> L
    L --> M["handshaker_factory_cb_ = build handshaker (default or custom)"]
    M --> N["compliance_policy_ from tls_params (FIPS_202205 etc.)"]
    N --> O["tls_keylog_* from key_log"]
```

### `isReady()`

The critical predicate used by the socket factory. Three sub‑conditions:

```cpp
bool tls_is_ready =
    (tls_certificate_providers_.empty() || !tls_certificate_configs_.empty());
bool combined_cvc_is_ready =
    (default_cvc_ == nullptr || validation_context_config_ != nullptr);
bool cvc_is_ready = (certificate_validation_context_provider_.provider_ == nullptr ||
                     default_cvc_ != nullptr || validation_context_config_ != nullptr);
return tls_is_ready && combined_cvc_is_ready && cvc_is_ready;
```

i.e. **"all the SDS providers I'm waiting on have delivered at least one secret."** If false, `ServerSslSocketFactory::createDownstreamTransportSocket()` hands out a `NotReadySslSocket` instead.

This is one of the most user‑visible parts of TLS configuration: a listener that boots before its SDS server has responded will accept TCP connections but immediately fail TLS with `"secret not ready"`. The fix is almost always to make SDS reach a deliverable state — wait for the control plane, fix a typo in the secret name, or pre‑load a fallback inline cert. The counter `downstream_context_secrets_not_ready` lights up exactly this scenario.

### `setSecretUpdateCallback`

The factory registers itself; on any SDS secret update, the registered callback fires and rebuilds the context. The handles in `tc_update_callback_handles_` and `cvc_update_callback_handle_` keep the subscriptions alive.

### `getCombinedValidationContextConfig`

Used when `CommonTlsContext.combined_validation_context` is set: dynamic SDS‑delivered CVC is *merged* (`Message::MergeFrom`) into `default_cvc_` before being turned into a config. This is what lets the static config say "trust this CA" and SDS push "also trust this SAN matcher".

The merge model is "**SDS adds, never replaces**" for repeated fields and "**SDS overrides**" for scalars. Practically: a static config carrying CA roots can be augmented with dynamic SAN matchers from SDS without a config push, which is a key pattern for mTLS with rotating identity authorities (SPIFFE, IAM workload identities, etc.).

### `ClientContextConfigImpl` specifics

Lines 141‑182. Adds:

- **`DEFAULT_CIPHER_SUITES` / `DEFAULT_CIPHER_SUITES_FIPS`** — static defaults defined in the `.cc`. Used when `tls_params.cipher_suites` is empty.
- **`DEFAULT_CURVES` / `DEFAULT_CURVES_FIPS`** — same for ECDH curves.
- **`DEFAULT_MIN_VERSION` / `DEFAULT_MAX_VERSION`** — TLS 1.2 .. 1.2 for clients (intentionally narrower than the server default).
- **`tls_certificate_selector_factory_`** — built from `CommonTlsContext.custom_tls_certificate_selector` if present; otherwise null and the default selector kicks in.
- Bit‑field bools to keep the struct small: `auto_host_sni_ : 1`, etc.

`ServerContextConfigImpl` adds session ticket key handling (inline or via SDS), `require_client_certificate`, OCSP staple policy, `prefer_client_ciphers`, etc.

---

## `ContextManagerImpl` — per‑process owner

`context_manager_impl.h:26‑50`. Implements `Ssl::ContextManager`.

```mermaid
classDiagram
    class ContextManager {
      <<interface>>
      +createSslClientContext(scope, config)
      +createSslServerContext(scope, config, additional_init)
      +daysUntilFirstCertExpires()
      +secondsUntilFirstOcspResponseExpires()
      +iterateContexts(cb)
      +privateKeyMethodManager()
      +removeContext(ctx)
    }

    class ContextManagerImpl {
      -factory_context_
      -contexts_ : flat_hash_set~ContextSharedPtr~
      -private_key_method_manager_
      +createSslClientContext()
      +createSslServerContext()
      +daysUntilFirstCertExpires()
      +secondsUntilFirstOcspResponseExpires()
      +iterateContexts(cb)
      +removeContext(ctx)
    }
    ContextManagerImpl --|> ContextManager
```

### Threading model (verbatim from the header)

> Contexts can be allocated on the main thread. They can be released from any thread (and in practice are since cluster information can be released from any thread). Context allocation/free is a very uncommon thing so we just do a global lock to protect it all.

The "any thread can release" caveat is what motivates the `flat_hash_set<ContextSharedPtr>` design. A worker that drops the last `shared_ptr` to a context (because its connection finally closed long after an SDS rotation) will execute the deleter on the worker thread; that deleter touches `contexts_` and must take the manager's mutex. The set membership is `shared_ptr`‑keyed precisely so a context can be removed atomically from any thread without coordinating with main.

```mermaid
sequenceDiagram
    autonumber
    participant Main as Main thread (listener / cluster build)
    participant Mgr as ContextManagerImpl
    participant Worker as Worker thread (cluster release)

    Main->>Mgr: createSslClientContext(scope, config)
    Mgr->>Mgr: lock global mutex
    Mgr->>Mgr: ClientContextImpl::create(...)
    Mgr->>Mgr: contexts_.insert(ctx)
    Mgr->>Mgr: unlock
    Mgr-->>Main: shared_ptr<ClientContextImpl>

    Note over Main,Worker: ... time passes, cluster torn down ...

    Worker->>Mgr: removeContext(ctx)
    Mgr->>Mgr: lock global mutex
    Mgr->>Mgr: contexts_.erase(ctx)
    Mgr->>Mgr: unlock
    Note over Worker: ctx may still live via in-flight shared_ptrs
```

### What it owns

- **`contexts_`** — every `ContextImpl` ever created and still referenced. Lets stats rollups (`daysUntilFirstCertExpires`, `secondsUntilFirstOcspResponseExpires`) iterate.
- **`private_key_method_manager_`** — singleton-per-process owner of `PrivateKeyMethodProvider` factories. Created lazily.
- **`factory_context_`** — passed to every context build, gives access to `serverFactoryContext().accessLogManager()`, dispatchers, etc.

### `iterateContexts` and stats rollup

```mermaid
flowchart LR
    A["Admin handler or stats refresh"] --> B["ContextManagerImpl::daysUntilFirstCertExpires"]
    B --> C["iterate contexts_"]
    C --> D["ctx.daysUntilFirstCertExpires()"]
    D --> E["min over all contexts"]
    E --> F["return absl::optional&lt;uint32_t&gt;"]
```

Same shape for `secondsUntilFirstOcspResponseExpires`. This is what populates the per‑cert `days_until_first_cert_expiring` gauge.

---

## Hot reload — end‑to‑end

```mermaid
sequenceDiagram
    autonumber
    participant CP as Control plane (xDS / SDS)
    participant SP as SDS provider
    participant CFG as ContextConfigImpl
    participant FAC as Client/ServerSslSocketFactory
    participant MGR as ContextManagerImpl
    participant CTX_OLD as old ContextImpl
    participant CTX_NEW as new ContextImpl

    CP->>SP: push new Secret
    SP->>CFG: secret update callback fires
    CFG->>FAC: onAddOrUpdateSecret()
    FAC->>MGR: createSsl*Context(scope, config)
    MGR->>MGR: lock mutex
    MGR->>CTX_NEW: build
    MGR->>MGR: contexts_.insert(CTX_NEW)
    MGR->>MGR: unlock
    MGR-->>FAC: shared_ptr<CTX_NEW>
    FAC->>FAC: swap ssl_ctx_ = CTX_NEW
    FAC->>MGR: removeContext(CTX_OLD)
    MGR->>MGR: lock, erase, unlock
    Note over CTX_OLD: still alive on in-flight connections via shared_ptr, freed when last one closes
```

The "build" step inside `MGR->CTX_NEW` is non‑trivial — it reads cert/key bytes, sets ciphers and ALPN on every `SSL_CTX`, computes the session context ID hash, registers OCSP staples, etc. All under the manager's mutex. The mutex is acceptable because reloads are rare (seconds, not microseconds).

---

## Cheat sheet

| Question | Answer |
|---|---|
| Where does proto → C++ happen? | `ContextConfigImpl` constructor |
| How do I know if SDS has delivered yet? | `ContextConfigImpl::isReady()` |
| Who builds the actual `ContextImpl`? | `ContextManagerImpl::createSsl{Client,Server}Context` |
| Where do I add a new field to the TLS config? | (1) proto, (2) `ContextConfig` interface, (3) `ContextConfigImpl` (parse + store + accessor), (4) `Client`/`ServerContextConfigImpl` if it's side‑specific, (5) `ContextImpl` consumes via the accessor |
| What protects `contexts_` from concurrent access? | A single global mutex in `ContextManagerImpl` |
| What lifetime guarantees does a context have? | Lives as long as any `shared_ptr` holds it — typically the factory plus all in‑flight connections |
| Where are the default cipher suites? | Static members of `ClientContextConfigImpl` (FIPS and non‑FIPS variants) |
| Where do SDS update handles live? | `tc_update_callback_handles_` and `cvc_update_callback_handle_` on `ContextConfigImpl` |

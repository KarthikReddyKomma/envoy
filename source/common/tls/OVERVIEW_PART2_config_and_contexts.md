# TLS Overview — Part 2: Configuration & Contexts

> **Previous:** [Part 1 — Architecture & Layering](OVERVIEW_PART1_architecture_and_layering.md)

## From Protobuf to Live `SSL_CTX`

This part covers the **build pipeline**: how a `Downstream/UpstreamTlsContext` proto is translated into an internal config, owned by a per-process manager, and turned into a bundle of `SSL_CTX` objects ready to serve handshakes.

```mermaid
flowchart TB
    subgraph Proto["Protobuf inputs"]
      DSP["DownstreamTlsContext"]
      USP["UpstreamTlsContext"]
      CTC["CommonTlsContext"]
    end

    subgraph CfgImpl["Config translation (context_config_impl)"]
      CCI["ContextConfigImpl<br/>(base)"]
      CCFI["ClientContextConfigImpl"]
      SCFI["ServerContextConfigImpl"]
      TCC["TlsCertificateConfigImpl"]
      CVC["CertificateValidationContextConfig"]
    end

    subgraph SDS["SDS / providers"]
      TCP["TlsCertificateProvider(s)"]
      CVCP["CertificateValidationContextProvider"]
      STKP["SessionTicketKeysProvider"]
    end

    subgraph Mgr["Manager (context_manager_impl)"]
      CMI["ContextManagerImpl<br/>contexts_: flat_hash_set<br/>(global mutex)"]
      PKMM["PrivateKeyMethodManagerImpl"]
    end

    subgraph CtxImpl["TLS contexts (context_impl)"]
      CCI2["ClientContextImpl"]
      SCI2["ServerContextImpl"]
      TC["TlsContext[] (one per cert)"]
      DEFSEL["DefaultTlsCertificateSelector"]
    end

    DSP --> CTC
    USP --> CTC
    DSP --> SCFI
    USP --> CCFI
    CTC --> CCI
    CCI --> CCFI
    CCI --> SCFI

    CCFI -.->|secret refs| TCP
    SCFI -.->|secret refs| TCP
    SCFI -.->|stk| STKP
    CCFI -.->|cvc| CVCP
    SCFI -.->|cvc| CVCP

    CCFI --> CMI
    SCFI --> CMI
    CMI --> CCI2
    CMI --> SCI2
    CCI2 --> TC
    SCI2 --> TC
    SCI2 --> DEFSEL
    CMI -. owns .-> PKMM
```

The five layers are deliberately kept **unidirectional** — proto flows into config, config flows into context, context flows into manager. Nothing flows back the other way. A misconfigured cipher in proto only ever affects the matching `SSL_CTX`, never the manager or the wider Envoy state.

## File Map for Part 2

| File | Role | Key Classes |
|------|------|-------------|
| `context_config_impl.{h,cc}` | proto -> internal config | `ContextConfigImpl`, `ClientContextConfigImpl`, `TlsCertificateConfigImpl` |
| `server_context_config_impl.{h,cc}` | downstream-only config additions | `ServerContextConfigImpl` |
| `context_manager_impl.{h,cc}` | per-process context registry | `ContextManagerImpl` |
| `context_impl.{h,cc}` | base `SSL_CTX` build, ALPN, key log, custom verify | `ContextImpl`, `TlsContext` |
| `client_context_impl.{h,cc}` | upstream context (session cache, SNI policy) | `ClientContextImpl` |
| `server_context_impl.{h,cc}` | downstream context (cert selector, session ticket keys) | `ServerContextImpl` |
| `default_tls_certificate_selector.{h,cc}` | built-in SNI -> cert selector | `DefaultTlsCertificateSelector` |

## `ContextConfigImpl` — Proto to Internal Config

`ContextConfigImpl` translates `CommonTlsContext` (and its `Downstream`/`Upstream` wrappers) into the strongly-typed internal config that `ContextImpl` consumes.

### Why split translation from build?

Translation is **expensive and synchronous** (PEM parsing, SDS fetching, validator factory instantiation). Build is **also expensive** (cipher list compilation, CA store population, OCSP response parsing). Splitting them buys two things:

1. The translated config can be **inspected and tested** without touching BoringSSL.
2. The manager can dedupe identical contexts by hashing the translated config, so two listeners with the same TLS settings share one `SSL_CTX`.

### Construction Pipeline

```mermaid
flowchart TB
    A["ContextConfigImpl(CommonTlsContext, ...)"] --> B["tlsVersionFromProto(min, max, defaults)"]
    B --> C["alpn_protocols_ = config.alpn_protocols (joined)"]
    C --> D["cipher_suites_ = config.tls_params.cipher_suites or default"]
    D --> E["ecdh_curves_ = config.tls_params.ecdh_curves or default"]
    E --> F{"tls_certificates set inline?"}
    F -- yes --> G["build TlsCertificateConfigImpl per cert"]
    F -- no --> H{"tls_certificate_sds_secret_configs?"}
    H -- yes --> I["start SDS providers,<br/>register update callbacks"]
    H -- no --> J{"tls_certificate_provider_instance?"}
    J -- yes --> K["lookup named provider, register"]
    G --> L["build validation_context_config_<br/>(default / combined / SDS)"]
    I --> L
    K --> L
    L --> M["handshaker_factory_cb_ = build handshaker"]
    M --> N["compliance_policy_ from tls_params"]
    N --> O["tls_keylog_* from key_log"]
```

### `isReady()` — The Critical Predicate

```mermaid
flowchart TB
    A["ContextConfigImpl::isReady()"]
    A --> B{"tls_certificate_providers_<br/>delivered at least one cert?"}
    A --> C{"combined CVC has both<br/>default and dynamic?"}
    A --> D{"CVC provider delivered<br/>or no provider needed?"}
    B & C & D --> E{All true?}
    E -- yes --> F["isReady = true<br/>factory builds ContextImpl"]
    E -- no --> G["isReady = false<br/>factory hands out NotReadySslSocket"]
    G --> H["downstream_context_secrets_not_ready++<br/>(or upstream_*)"]
    G --> I["wait for SDS update -><br/>setSecretUpdateCallback fires"]
    I --> A
```

If `isReady()` returns false, `ServerSslSocketFactory::createDownstreamTransportSocket()` hands out a `NotReadySslSocket` instead — the connection accepts TCP but immediately fails TLS with `"secret not ready"`.

### `getCombinedValidationContextConfig`

The merge model is "**SDS adds, never replaces**" for repeated fields and "**SDS overrides**" for scalars. Practically: a static config carrying CA roots can be augmented with dynamic SAN matchers from SDS without a config push — a key pattern for mTLS with rotating identity authorities (SPIFFE, IAM workload identities, etc.).

### Client vs. Server Subclass Differences

```mermaid
classDiagram
    class ContextConfigImpl {
        <<base>>
        +alpnProtocols, cipherSuites, ecdhCurves, sigAlgs
        +tlsCertificates, validationContext
        +tls_certificate_providers_
        +min/max protocol version
        +tls_keylog
        +compliancePolicy
        +createHandshaker, capabilities, sslctxCb
    }

    class ClientContextConfigImpl {
        +DEFAULT_CIPHER_SUITES / _FIPS
        +DEFAULT_CURVES / _FIPS
        +DEFAULT_MIN_VERSION = TLS 1.2
        +DEFAULT_MAX_VERSION = TLS 1.2
        +server_name_indication_
        +auto_host_sni_
        +allow_renegotiation_
        +enforce_rsa_key_usage_
        +max_session_keys_
        +tls_certificate_selector_factory_
    }

    class ServerContextConfigImpl {
        +require_client_certificate_
        +session_ticket_keys_
        +session_ticket_keys_provider_
        +ocsp_staple_policy_
        +full_scan_certs_on_sni_mismatch_
        +prefer_client_ciphers_
        +tls_certificate_selector_factory_
    }

    ContextConfigImpl <|-- ClientContextConfigImpl
    ContextConfigImpl <|-- ServerContextConfigImpl
```

## `ContextManagerImpl` — Per-Process Owner

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
        -global mutex
    }

    ContextManagerImpl --|> ContextManager
```

### Threading Contract

> Contexts can be allocated on the **main thread**. They can be released from **any thread** (and in practice are, since cluster information can be released from any thread). Context allocation/free is uncommon, so a global lock protects all of it.

```mermaid
sequenceDiagram
    autonumber
    participant Main as Main thread
    participant Mgr as ContextManagerImpl
    participant W1 as Worker 1
    participant W2 as Worker 2

    Main->>Mgr: createSslServerContext(config)
    Mgr->>Mgr: lock global mutex
    Mgr->>Mgr: build ServerContextImpl
    Mgr->>Mgr: contexts_.insert(ctx_sp)
    Mgr->>Mgr: unlock
    Mgr-->>Main: shared_ptr<ServerContextImpl>

    Main->>W1: hand context to factory
    Main->>W2: hand context to factory

    Note over W1,W2: connections accept context

    Note over W1: Last connection closes
    W1->>Mgr: ~ContextSharedPtr (deleter on worker)
    Mgr->>Mgr: lock global mutex
    Mgr->>Mgr: contexts_.erase(ctx_sp)
    Mgr->>Mgr: unlock
```

### Aggregate Stats

```mermaid
flowchart LR
    A["daysUntilFirstCertExpires()"] --> B["iterate contexts_"]
    B --> C["each ctx: iterate tls_contexts_"]
    C --> D["each tls_ctx: getDaysUntilExpiration(cert)"]
    D --> E["min across all"]

    F["iterateContexts(cb) (admin /certs)"] --> G["lock, copy snapshot, unlock"]
    G --> H["call cb on each"]
    H --> I["each ctx returns getCertChainInformation,<br/>getCaCertInformation"]
```

## `TlsContext` — One Per Certificate

```mermaid
classDiagram
    class TlsContext {
        +ssl_ctx_ : SSL_CTX
        +cert_chain_ : X509
        +cert_chain_file_path_ : string
        +ocsp_response_ : OcspResponseWrapper
        +ec_group_curve_name_ : CurveNID
        +is_must_staple_ : bool
        +private_key_method_provider_
        +loadCertificateChain()
        +loadPrivateKey()
        +loadPkcs12()
        +checkPrivateKey()
        +isCipherEnabled()
    }
```

| Field | Why |
|-------|-----|
| `ssl_ctx_` | The actual BoringSSL handle |
| `cert_chain_` + `cert_chain_file_path_` | Parsed leaf cert and source path (for /certs admin) |
| `ocsp_response_` | Cached OCSP response, if any |
| `ec_group_curve_name_` | ECDSA curve identifier; lets selector quickly match client's signature_algorithms (`EC_CURVE_INVALID_NID = -1` means "not ECDSA") |
| `is_must_staple_` | Derived from the `1.3.6.1.5.5.7.1.24` extension; controls whether OCSP staple is mandatory |
| `private_key_method_provider_` | HSM / async signer (see Part 4) |

## `ContextImpl` — Abstract Base

```mermaid
classDiagram
    class Context {
      <<interface>>
      +daysUntilFirstCertExpires()
      +getCaCertInformation()
      +getCertChainInformation()
      +secondsUntilFirstOcspResponseExpires()
    }

    class ContextImpl {
      <<abstract>>
      #tls_contexts_ : vector~TlsContext~
      #cert_validator_ : CertValidatorPtr
      #stats_ : SslStats
      #parsed_alpn_protocols_
      #tls_max_version_
      #tls_keylog_*
      +newSsl(options, host) : SSL
      +logHandshake(ssl)
      +customVerifyCallback() (static)
      +keylogCallback() (static)
      +customVerifyCertChain()
      +customVerifyCertChainForQuic()
    }

    class ClientContextImpl {
      -server_name_indication_
      -auto_host_sni_
      -allow_renegotiation_
      -max_session_keys_
      -session_keys_ : deque~SSL_SESSION~
      -tls_certificate_selector_ : UpstreamTlsCertificateSelectorPtr
      +newSsl(options, host) : SSL
      +selectTlsContext(ssl) : int
      -newSessionKey(session)
    }

    class ServerContextImpl {
      -tls_certificate_selector_ : TlsCertificateSelectorPtr
      -session_ticket_keys_
      -ocsp_staple_policy_
      +selectTlsContext(client_hello) : ssl_select_cert_result_t
      +findTlsContext(sni, ecdsa_caps, ocsp, *matched)
      +getClientEcdsaCapabilities(client_hello)
      -alpnSelectCallback()
      -sessionTicketProcess()
      -generateHashForSessionContextId()
    }

    Context <|.. ContextImpl
    ContextImpl <|-- ClientContextImpl
    ContextImpl <|-- ServerContextImpl
    ContextImpl *-- TlsContext : owns vector
```

### What `ContextImpl` Does

| Function | Purpose |
|----------|---------|
| `newSsl(options, host)` | Allocate a new `SSL*` from `tls_contexts_[0].ssl_ctx_`. The `SSL_CTX` may be swapped later via `selectTlsContext`. Sets transport socket options (verify_san list, SNI override). |
| `logHandshake(ssl)` | After `SSL_do_handshake` succeeds, increments per-cipher / per-version / per-curve / per-sigalg counters via `incCounter`. Called from `SslHandshakerImpl::onSuccess`. |
| `customVerifyCallback` (static) | Wired into every `SSL_CTX` via `SSL_CTX_set_custom_verify`. Looks up `ContextImpl*` from `SSL_CTX_get_app_data`, fetches the validator, delegates to `customVerifyCertChain`. This is the path async cert validation flows through. |
| `customVerifyCertChainForQuic` | Same idea but used by QUIC's proof source path, where the chain is presented as `STACK_OF(X509)*` directly. |
| `keylogCallback` | Wired via `SSL_CTX_set_keylog_callback` when `key_log` is configured. Writes one line per session, filtered by `tls_keylog_local_` / `tls_keylog_remote_`. |

### Ex-Data Indices

```mermaid
flowchart LR
    A["SSL_CTX"] -- "app_data slot" --> CTX["ContextImpl* (sslContextIndex())"]
    B["SSL"] -- "ex_data slot 1" --> CTX2["ContextImpl* (sslContextIndex())"]
    B -- "ex_data slot 2" --> EXT["SslExtendedSocketInfoImpl*<br/>(sslExtendedSocketInfoIndex())"]
    B -- "ex_data slot 3" --> SOCK["SslSocket*<br/>(sslSocketIndex())"]
```

All three indices are allocated once per process via `SSL_get_ex_new_index`. They give static callback functions a way to recover the C++ objects from a raw `SSL*`.

## `ServerContextImpl` — Downstream Side

```mermaid
flowchart TB
    SCI["ServerContextImpl"]

    SCI --> SC1["1. Cert selection hook<br/>SSL_CTX_set_select_certificate_cb<br/>(installed on tls_contexts_[0].ssl_ctx_)"]
    SCI --> SC2["2. Session-ticket processing<br/>SSL_CTX_set_tlsext_ticket_key_cb<br/>(name + HMAC + AES keys)"]
    SCI --> SC3["3. Session context ID hash<br/>(certs + validator state + server_names)"]
    SCI --> SC4["4. ALPN select callback<br/>(server-side preference)"]
    SCI --> SC5["5. OCSP staple policy<br/>LENIENT / STRICT / MUST_STAPLE"]
```

### Cert Selection Hook

```mermaid
flowchart TB
    A["SSL_CTX_set_select_certificate_cb<br/>(only on tls_contexts_[0].ssl_ctx_)"]
    B["lambda -> ServerContextImpl::selectTlsContext(client_hello)"]
    C["tls_certificate_selector_->selectTlsContext(ClientHello, cb)"]
    D{"SelectionResult"}
    E1["Success: SSL_set_SSL_CTX(chosen)<br/>return ssl_select_cert_success"]
    E2["Pending: return ssl_select_cert_retry"]
    E3["Failed: return ssl_select_cert_error"]

    A --> B --> C --> D
    D -- Success --> E1
    D -- Pending --> E2
    D -- Failed --> E3
```

The factory hooks the callback only on the **first** `tls_contexts_[0].ssl_ctx_`. When BoringSSL processes the ClientHello, it invokes the callback there; the callback resolves the per-process `ServerContextImpl*` via `SSL_CTX_get_app_data` and dispatches to the selector.

### Session Context ID — Subtle Security Boundary

`generateHashForSessionContextId(server_names)` produces a stable 32-byte digest that BoringSSL uses to scope session resumption. It includes:

- Every cert's SHA-256.
- The validator's `updateDigestForSessionId` contribution (so changes in CA / SAN matchers break old sessions).
- The list of `server_names` from the listener filter chain.

If two listeners shared a session-ticket key but had different cert/CA configurations, a client that handshook against one could resume against the other and skip validation entirely. The digest closes that hole: BoringSSL rejects resumption when the digest doesn't match.

## `ClientContextImpl` — Upstream Side

```mermaid
flowchart TB
    CCI["ClientContextImpl"]
    CCI --> CC1["1. Per-host session-ticket cache<br/>(huge perf win for upstream conn pools)"]
    CCI --> CC2["2. SNI policy<br/>(static / auto_host_sni / TSO override)"]
    CCI --> CC3["3. Renegotiation toggle<br/>(default off)"]
    CCI --> CC4["4. enforce_rsa_key_usage<br/>(strict server cert validation)"]
    CCI --> CC5["5. Custom upstream cert selector<br/>(on_demand_secret etc., for mTLS)"]
```

### Session Ticket Cache

```mermaid
flowchart LR
    A["TLS handshake to upstream"] --> B["new SSL_SESSION received"]
    B --> C{"max_session_keys_ > 0?"}
    C -- yes --> D["newSessionKey(session) -><br/>push_front into session_keys_<br/>(under session_keys_mu_)"]
    C -- no --> E["discard"]
    F["new outgoing connection<br/>to same host"] --> G["pop SSL_SESSION from session_keys_"]
    G --> H["SSL_set_session(ssl, session)"]
    H --> I["abbreviated handshake"]
```

Without resumption, every new upstream connection pays a full handshake — RSA signing or ECDSA + DH which is the single most expensive thing per connection. With resumption, the handshake is a single round trip with no asymmetric crypto.

## `DefaultTlsCertificateSelector` — SNI Matcher

```mermaid
classDiagram
    class DefaultTlsCertificateSelector {
        -server_ctx_ : ServerContextImpl&
        -tls_contexts_ : vector~TlsContext~&
        -server_names_map_ : ServerNamesMap
        -has_rsa_ : bool
        -ocsp_staple_policy_
        -full_scan_certs_on_sni_mismatch_
        +selectTlsContext(client_hello, cb)
        +findTlsContext(sni, ecdsa, ocsp, *matched)
        -populateServerNamesMap()
    }

    class ServerNamesMap {
        <<typedef>>
        string -> PkeyTypesMap
    }

    class PkeyTypesMap {
        <<typedef>>
        EVP_PKEY_id -> ref~TlsContext~
    }

    DefaultTlsCertificateSelector --> ServerNamesMap
    ServerNamesMap --> PkeyTypesMap
```

The map keys both **exact** server names and **wildcard** patterns. Wildcards are prefixed with `.` (so `*.example.com` is stored as `.example.com`) to distinguish them from exact entries. Each entry is then sub-keyed by key type (`EVP_PKEY_RSA` vs `EVP_PKEY_EC`) so RSA + ECDSA certs for the same SNI can coexist.

### Selection Algorithm

```mermaid
flowchart TB
    A["selectTlsContext(client_hello, cb)"] --> B["sni = SSL_get_servername"]
    B --> C["ecdsa_caps = getClientEcdsaCapabilities"]
    C --> D["ocsp_capable = isClientOcspCapable"]
    D --> E["findTlsContext(sni, ecdsa_caps, ocsp_capable)"]

    E --> F{"exact SNI match?"}
    F -- yes --> G["pick best key type for client<br/>(ECDSA if supported, else RSA)"]
    F -- no --> H{"wildcard match (.example.com)?"}
    H -- yes --> G
    H -- no --> I{"full_scan_certs_on_sni_mismatch?"}
    I -- yes --> J["scan all tls_contexts_<br/>for key-type / OCSP fit"]
    I -- no --> K["fall back to tls_contexts_[0]"]
    J --> G
    K --> G
    G --> L["compute OCSP staple action"]
    L --> M["SelectionResult{Success, ctx, staple}"]
```

The "best key type for client" decision is **performance and compatibility** rolled together. Modern clients almost always advertise ECDSA support, in which case ECDSA wins — the signatures are smaller and faster. Old clients (and some embedded devices) only support RSA; the selector falls back transparently. The fact that this happens inside the selector rather than via separate listeners is what lets one Envoy listener serve both populations on the same port without configuration gymnastics.

## OCSP Staple Action — Decision Table

| Client `status_request` | Configured `ocsp_response` | Response fresh? | Cert `must_staple`? | Policy | Action |
|------------------------|--------------------------|-----------------|--------------------|----|--------|
| No | — | — | No | any | NoStaple |
| No | — | — | Yes | any | Fail (cert can't be used) |
| Yes | None | — | No | LENIENT_STAPLING | NoStaple |
| Yes | None | — | No | STRICT_STAPLING | Fail |
| Yes | Yes | Yes | any | any | Staple |
| Yes | Yes | No (expired) | any | LENIENT_STAPLING | NoStaple (`ocsp_staple_omitted++`) |
| Yes | Yes | No (expired) | any | STRICT_STAPLING | Fail (`ocsp_staple_failed++`) |

## Server-Side Handshake Start — Putting It Together

```mermaid
sequenceDiagram
    autonumber
    participant BSSL as BoringSSL
    participant SCtx as ServerContextImpl
    participant Sel as DefaultTlsCertificateSelector
    participant Map as server_names_map_
    participant TC as TlsContext (chosen)

    Note over BSSL: ClientHello received
    BSSL->>SCtx: select_certificate_cb(client_hello)
    SCtx->>Sel: selectTlsContext(client_hello, cb)
    Sel->>Sel: extract sni, ecdsa_caps, ocsp_capable
    Sel->>Map: lookup sni (exact, then .sni)
    Map-->>Sel: PkeyTypesMap entry
    Sel->>Sel: pick best key type for client
    Sel-->>SCtx: SelectionResult{Success, ctx=TC, staple=Yes}
    SCtx->>BSSL: SSL_set_SSL_CTX(ssl, TC.ssl_ctx_)
    SCtx->>BSSL: SSL_set_tlsext_status_ocsp_resp(...)<br/>(if staple)
    SCtx-->>BSSL: ssl_select_cert_success
    Note over BSSL: continues with chosen cert
```

## Lookup & Decision Cheat Sheet

| Question | Answer |
|----------|--------|
| Why is `tls_contexts_` a vector? | One `SSL_CTX` per configured cert; selector picks per handshake |
| Where does cert selection actually swap the cert? | `selectTlsContext` -> `SSL_set_SSL_CTX(ssl, chosen.ssl_ctx_)` |
| What about non-SNI clients? | `findTlsContext` falls back to `tls_contexts_[0]` (or full scan if enabled) |
| Where is the cert-chain validator called? | `ContextImpl::customVerifyCallback` (static), wired in by `SSL_CTX_set_custom_verify` |
| Where do session tickets get encrypted? | `ServerContextImpl::sessionTicketProcess` |
| Where is ALPN selected? | `ServerContextImpl::alpnSelectCallback` (downstream); `SSL_set_alpn_protos` on client (upstream) |
| Where does the cipher / version stats counter increment? | `ContextImpl::logHandshake` -> `incCounter`, called from `SslHandshakerImpl::onSuccess` |
| What's the upstream equivalent of the cert selector? | `ClientContextImpl::tls_certificate_selector_` — only used by upstream `on_demand_secret` etc. |
| When is `isReady()` false? | Any SDS provider hasn't delivered yet -> `NotReadySslSocket` for new connections |
| How does session resumption stay safe across config reloads? | `generateHashForSessionContextId` mixes cert + validator state into the digest |

---

Continue with **[Part 3: Sockets, Handshaker & Data Path](OVERVIEW_PART3_sockets_handshaker_and_data_path.md)** for what happens once a context is built and a connection arrives.

# Cert validator — `source/common/tls/cert_validator/`

> Files: `cert_validator.h`, `default_validator.{h,cc}`, `factory.{h,cc}`, `san_matcher.{h,cc}`

The peer certificate validation layer. Sits between BoringSSL's `SSL_CTX_set_custom_verify` callback and the actual policy decisions ("is this cert trusted, does its SAN match, is the hash on my pinning list, is the CRL OK"). Pluggable via `CommonTlsContext.validation_context.custom_validator_config`.

### Why bypass BoringSSL's built‑in verifier?

BoringSSL's default verifier (`X509_verify_cert` via `X509_STORE_CTX`) is fine for "trust this CA bundle", but Envoy needs more: SAN matchers (allow only specific peer identities), hash pinning, SPKI pinning, CRL checks, and — critically — **asynchronous** validation (handing the chain off to an external policy engine). The interface here is what makes those extensions possible without surgery on BoringSSL.

This folder has the abstract interface (`CertValidator`), the **default** implementation (`DefaultCertValidator`), the **factory base** (`CertValidatorFactory`), and the **SAN matcher** type system (`san_matcher.{h,cc}`).

Other validators live elsewhere — they all implement `CertValidator` from here:
- `source/extensions/transport_sockets/tls/cert_validator/spiffe/` — SPIFFE trust bundle validator.
- `source/extensions/transport_sockets/tls/cert_validator/dynamic_modules/` — delegates validation to a dynamic shared library (see [`/api/.../cert_validator/dynamic_modules/`](../../../../api/envoy/extensions/transport_sockets/tls/cert_validator/dynamic_modules/v3/dynamic_modules.proto)).

---

## Architecture overview

```mermaid
flowchart TB
    subgraph Interface["This folder"]
      CV["CertValidator (interface)"]
      Fac["CertValidatorFactory (base)"]
      DV["DefaultCertValidator"]
      SAN["SanMatcher / StringSanMatcher / DnsExactStringSanMatcher"]
    end

    subgraph Plugins["Other folders"]
      SPIFFE["spiffe validator<br/>(extensions/.../spiffe/)"]
      DYN["dynamic_modules validator<br/>(extensions/.../dynamic_modules/)"]
    end

    subgraph Caller["Caller surface"]
      CTX["ContextImpl<br/>+ ServerContextImpl<br/>+ ClientContextImpl"]
      BSSL["BoringSSL"]
    end

    DV --|> CV
    SPIFFE --|> CV
    DYN --|> CV
    Fac -- "creates" --> DV
    Fac -- "creates" --> SPIFFE
    Fac -- "creates" --> DYN

    CTX --> CV : via cert_validator_
    BSSL --> CTX : SSL_CTX_set_custom_verify -><br/>ContextImpl::customVerifyCallback
    CTX --> CV : doVerifyCertChain
    DV --> SAN : matchSubjectAltName
```

---

## `CertValidator` — the interface

`cert_validator.h:50‑114`.

```mermaid
classDiagram
    class CertValidator {
      <<interface>>
      +addClientValidationContext(SSL_CTX*, require_client_cert) : Status
      +doVerifyCertChain(chain, callback, options, ssl_ctx, ctx, is_server, host_name) : ValidationResults
      +initializeSslContexts(contexts, handshaker_provides_certs, scope) : StatusOr~int~
      +updateDigestForSessionId(md, buf, len)
      +daysUntilFirstCertExpires() : optional~uint32_t~
      +getCaFileName() : string
      +getCaCertInformation() : CertificateDetailsPtr
    }

    class ValidationResults {
      +status : Pending / Successful / Failed
      +detailed_status : ClientValidationStatus
      +tls_alert : optional~uint8_t~
      +error_details : optional~string~
      +validated_chain : vector~X509~
    }

    class ExtraValidationContext {
      +callbacks : TransportSocketCallbacks*
    }

    CertValidator ..> ValidationResults : returns
    CertValidator ..> ExtraValidationContext : takes
```

### Method roles

| Method | Called when | Returns |
|---|---|---|
| `initializeSslContexts(contexts, ...)` | Once per context at build time | The `SSL_VERIFY_*` mode to install (`SSL_VERIFY_NONE` / `_PEER` / `_FAIL_IF_NO_PEER_CERT`) |
| `addClientValidationContext(ssl_ctx, require_client_cert)` | Server‑side config of one `SSL_CTX` | Status. Default impl also populates client CA names |
| `doVerifyCertChain(chain, callback, ...)` | Per handshake from BoringSSL custom_verify | `ValidationResults` — sync or async |
| `updateDigestForSessionId(md, buf, len)` | Server‑side, computing session context ID | Updates `md` to bind sessions to validator config |
| `daysUntilFirstCertExpires` / `getCaFileName` / `getCaCertInformation` | Admin / stats rollup | CA‑cert introspection |

### `doVerifyCertChain` async return path

```mermaid
sequenceDiagram
    autonumber
    participant BSSL as BoringSSL
    participant CTX as ContextImpl<br/>(customVerifyCallback)
    participant CV as CertValidator
    participant CB as ValidateResultCallback
    participant Disp as Worker dispatcher

    BSSL->>CTX: customVerifyCallback(ssl, *alert)
    CTX->>CV: doVerifyCertChain(chain, cb, opts, ssl_ctx, ctx, is_server, host)
    alt sync success
      CV-->>CTX: {Successful, [chain], no_alert}
      CTX-->>BSSL: ssl_verify_ok
    else sync failure
      CV-->>CTX: {Failed, alert, error_details}
      CTX-->>BSSL: ssl_verify_invalid (sets *alert)
    else async
      CV-->>CTX: {Pending}
      CTX-->>BSSL: ssl_verify_retry
      Note over BSSL: SSL_ERROR_WANT_CERTIFICATE_VERIFY
      CV->>CB: later, onCertValidationResult(...)
      CB->>Disp: dispatcher.post(...)
      Disp-->>BSSL: resume handshake
    end
```

The handshaker side of this is in [`handshaker.md`](../handshaker.md). The validator side is just: return `Pending` and remember the callback for later.

The "return `Pending`" contract is the most important thing about this interface. It's how Envoy plugs **arbitrary** async work into a synchronous BoringSSL primitive without ever blocking a worker thread. A SPIFFE federated trust check could spend 50ms on an RPC; a dynamic‑module validator could do PKI policy evaluation. Both look identical to BoringSSL: return `Pending`, callback fires later, handshake resumes.

---

## `DefaultCertValidator` — the default implementation

`default_validator.h:35‑127`. This is what's used when `custom_validator_config` is **not** set on `CertificateValidationContext`. It supports the standard X.509 + SAN + CRL + pinning verification.

### Inputs (config)

From `CertificateValidationContextConfig`:

| Field | Used for |
|---|---|
| `trusted_ca` | Built into an `X509_STORE` for chain validation |
| `crl` | Added to the store as `X509_CRL`s |
| `match_typed_subject_alt_names` | Matchers in `subject_alt_name_matchers_` |
| `verify_certificate_hash` | `verify_certificate_hash_list_` (SHA‑256 of DER cert) |
| `verify_certificate_spki` | `verify_certificate_spki_list_` (SHA‑256 of SPKI) |
| `allow_expired_certificate` | Toggles whether `X509_V_ERR_CERT_HAS_EXPIRED` is treated as a soft pass |
| `trust_chain_verification` | `VERIFY_TRUST_CHAIN` vs `ACCEPT_UNTRUSTED` |
| `auto_sni_san_match` | If true and no SAN matchers configured, validates SNI against peer SANs |

### `doVerifyCertChain` flow

```mermaid
flowchart TB
    A["doVerifyCertChain(chain, ...)"] --> B{"chain empty?"}
    B -- yes --> C["ClientValidationStatus::NoClientCertificate -> Failed"]
    B -- no --> D{"verify_trusted_ca_?"}
    D -- yes --> E["X509_STORE chain verification<br/>(via configured store)"]
    D -- no --> F["skip chain verify (ACCEPT_UNTRUSTED)"]
    E --> G{"chain verify passed?"}
    G -- no, allow_expired_certificate? --> H["soft pass with status=Failed but ok"]
    G -- no --> I["return Failed, fail_verify_error++"]
    G -- yes --> J["leaf cert verification"]
    F --> J
    J --> K["verifyCertAndUpdateStatus(leaf)"]
    K --> K1["SAN matchers (matchSubjectAltName)"]
    K --> K2["hash pinning (verifyCertificateHashList)"]
    K --> K3["SPKI pinning (verifyCertificateSpkiList)"]
    K1 & K2 & K3 --> L{"all pass?"}
    L -- yes --> M["return Successful, populate validated_chain"]
    L -- no --> N["return Failed with alert + error details"]
```

### Helper functions

`default_validator.h:62‑106` exposes some helpers that are also reused elsewhere:

- `verifyCertificate(cert, verify_san_list, matchers, stream_info, *error, *alert)` — single‑cert verification, used by SPIFFE validator too.
- `verifyCertificateHashList(cert, expected_hashes)` — static. SHA‑256 of the DER cert (binary‑encoded pin).
- `verifyCertificateSpkiList(cert, expected_hashes)` — static. SHA‑256 of the SPKI (key‑only pin, survives cert reissuance).
- `verifySubjectAltName(cert, names)` — static. Legacy string‑equality SAN matching.
- `matchSubjectAltName(cert, stream_info, matchers)` — static. Full `SanMatcher` based matching.

### `addClientValidationContext`

Server‑side only. Installs the CA name list that's sent to the client in the `CertificateRequest` message (so the client knows which CA's cert it should present). Also sets `SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT` if `require_client_cert` is true.

### `updateDigestForSessionId`

Hashes every part of the config into the session context ID. **Critical**: if you reconfigure `trusted_ca` or `match_typed_subject_alt_names`, old TLS sessions cannot resume — they'd be validated under stale rules.

---

## `CertValidatorFactory` — the plug‑in registry

`factory.h:13‑33`. Registered factories all live under category `envoy.tls.cert_validator`. `getCertValidatorName(config)` reads the validator name from the config (defaults to the built‑in `envoy.tls.cert_validator.default` when not set).

```mermaid
flowchart LR
    A["Config has custom_validator_config?"] --> B{"yes / no"}
    B -- yes --> C["look up factory by name"]
    B -- no --> D["use DefaultCertValidatorFactory"]
    C --> E["factory.createCertValidator(config, stats, ctx, scope)"]
    D --> E
    E --> F["CertValidatorPtr -> stored on ContextImpl"]
```

The default factory (`DefaultCertValidatorFactory`) is declared via `DECLARE_FACTORY` in `default_validator.h:129` and constructs `DefaultCertValidator`.

---

## `SanMatcher` — SAN matching machinery

`san_matcher.h:23‑89`. Two concrete types:

```mermaid
classDiagram
    class SanMatcher {
      <<interface, Ssl::>>
      +match(GENERAL_NAME*) : bool
      +match(GENERAL_NAME*, StreamInfo&) : bool
    }

    class StringSanMatcher {
      -general_name_type_ : int
      -matcher_ : StringMatcherImpl
      -general_name_oid_ : ASN1_OBJECT (for othername)
      +match(GENERAL_NAME*)
      +match(GENERAL_NAME*, StreamInfo&)
    }

    class DnsExactStringSanMatcher {
      -dns_exact_match_ : string
      +match(GENERAL_NAME*)
    }

    SanMatcher <|-- StringSanMatcher
    SanMatcher <|-- DnsExactStringSanMatcher
```

### Why two types?

Because **DNS SAN equality is not regular string equality** — it's RFC 6125 (case‑insensitive, label‑aware). Other SAN types (URI, email, IP, othername, otherName) use the regular `StringMatcher` (exact/prefix/suffix/regex/contains).

`StringSanMatcher` has a runtime assertion that `general_name_type != GEN_DNS || match_pattern_case != Exact` — exact DNS matchers must use `DnsExactStringSanMatcher` instead, which calls `Utility::dnsNameMatch` for RFC 6125 semantics.

This is a "**defensive type system**" choice. If exact DNS matching used `StringSanMatcher` with case‑sensitive `==`, a peer presenting `EXAMPLE.com` would fail when `example.com` was configured, and a peer presenting `foo.example.com` would also fail when `*.example.com` was configured. Both are real interoperability problems. By making the type system enforce "DNS exact must use the RFC 6125 matcher", the bug becomes a compile/assert error instead of a silent connectivity issue.

### `createStringSanMatcher`

`san_matcher.h:87` — factory entry point. Translates a `SubjectAltNameMatcher` proto into the right concrete matcher. Used by `ContextConfigImpl` when parsing `match_typed_subject_alt_names`.

### The two‑arg `match`

The variant that takes `StreamInfo&` is used when the matcher's `StringMatcher` references **stream context** (e.g. compare against a downstream header). Default impl ignores `stream_info` and delegates to the single‑arg version.

---

## How `DefaultCertValidator` is wired in

```mermaid
sequenceDiagram
    autonumber
    participant CFG as ContextConfigImpl
    participant FAC as CertValidatorFactory<br/>(Default or custom)
    participant CV as CertValidator (instance)
    participant CTX as ContextImpl
    participant BSSL as BoringSSL

    CFG->>CFG: read certificateValidationContext().customValidatorConfig()
    CFG->>FAC: lookup by name (or default)
    FAC->>CV: createCertValidator(config, stats, ctx, scope)
    CV-->>CFG: CertValidatorPtr
    Note over CFG: passed into ContextImpl ctor
    CFG->>CTX: build ContextImpl with cert_validator_=CV
    CTX->>CV: initializeSslContexts(ssl_ctxs, ...)
    CV-->>CTX: SSL_VERIFY_PEER | FAIL_IF_NO_PEER_CERT
    loop per SSL_CTX
      CTX->>BSSL: SSL_CTX_set_custom_verify(ssl_ctx, verify_mode, customVerifyCallback)
      CTX->>BSSL: SSL_CTX_set_app_data(ssl_ctx, this)
    end

    Note over BSSL: ... later, handshake ...
    BSSL->>CTX: customVerifyCallback(ssl, *alert)
    CTX->>CV: doVerifyCertChain(chain, cb, ...)
    CV-->>CTX: ValidationResults
    CTX-->>BSSL: ssl_verify_ok / retry / invalid
```

---

## Plug‑in landscape

The validator is the **biggest extension surface** in `common/tls/`. Three production validators exist today and each one represents a fundamentally different trust model: built‑in PKI (X.509 + SAN), SPIFFE federated workload identity, and bring‑your‑own policy via dynamic modules. New validators just implement `CertValidator` from this folder; nothing else changes.

```mermaid
flowchart TB
    A["CertValidator (interface)<br/>cert_validator.h"]
    A --> D["DefaultCertValidator<br/>(in this folder)"]
    A --> S["SpiffeValidator<br/>(extensions/.../spiffe/)"]
    A --> M["DynamicModuleCertValidator<br/>(extensions/.../dynamic_modules/)"]

    D --> D1["X509 + SAN + CRL + pinning"]
    S --> S1["SPIFFE trust bundle (JWT-style trust domain bundle)"]
    M --> M1["Delegate to a dynamic shared library"]
```

Which validator runs at handshake time is decided **once at config time** in `ContextConfigImpl`, based on `CertificateValidationContextConfig::customValidatorConfig()` (proto field `custom_validator_config` on `CertificateValidationContext`).

---

## Cheat sheet

| Question | Answer |
|---|---|
| Where is peer cert validation called? | `ContextImpl::customVerifyCallback` → `cert_validator_->doVerifyCertChain` |
| What's the default behaviour? | `DefaultCertValidator` — full X.509 chain validation + SAN match + pinning |
| How do I add an async validator? | Implement `CertValidator`, return `ValidationStatus::Pending` and remember the callback |
| Where do SAN matchers live? | `san_matcher.{h,cc}` — `StringSanMatcher` for non‑DNS, `DnsExactStringSanMatcher` for exact DNS |
| Where is RFC 6125 DNS matching? | `Utility::dnsNameMatch`, called from `DnsExactStringSanMatcher` |
| Where is hash pinning? | `DefaultCertValidator::verifyCertificateHashList` (cert DER hash) and `verifyCertificateSpkiList` (SPKI hash) |
| Why hash session context ID against validator config? | So config changes invalidate cached sessions |
| Can I disable chain verification? | Yes — set `trust_chain_verification = ACCEPT_UNTRUSTED` on the CVC |

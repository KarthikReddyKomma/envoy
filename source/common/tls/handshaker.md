# Handshaker — `SslHandshakerImpl`

> Files: `ssl_handshaker.{h,cc}`

`SslHandshakerImpl` is the bridge between BoringSSL's blocking‑looking handshake API (`SSL_do_handshake`) and Envoy's non‑blocking, single‑threaded‑per‑worker dispatcher. It also doubles as the `Ssl::ConnectionInfo` impl exposed to filters (inherits `ConnectionInfoImplBase`), so the handshaker and the post‑handshake info object are literally the same C++ object.

### Why one object for both handshake driving and connection info?

Filter chains routinely call `connection.ssl()->peerCertificatePresented()`, `connection.ssl()->ciphersuiteString()`, etc. — and they do this **during** request processing, possibly long after the handshake finished. Allocating a separate "info" object after the handshake would mean copying everything out of the `SSL*`, which is wasteful when the `SSL*` already has it. So Envoy makes the handshaker live for the entire connection lifetime and exposes its accessor surface directly. This also gives filters a stable identity to compare against (same `Ssl::ConnectionInfo*` across the whole request).

The header also defines the **async escape hatches**:
- `ValidateResultCallbackImpl` — resumes after async cert validation.
- `CertificateSelectionCallbackImpl` — resumes after async cert selection.
- `SslExtendedSocketInfoImpl` — per‑connection scratch space that owns those callbacks.

Plus the **factory plumbing**: `HandshakerFactoryImpl` registers the default handshaker under the name `envoy.default_tls_handshaker`. Other handshakers can be registered via `CommonTlsContext.custom_handshaker`.

---

## Class layout

```mermaid
classDiagram
    class ConnectionInfoImplBase {
      <<abstract>>
      +ssl() : SSL*
      +sha256PeerCertificateDigest()
      +peerCertificateValidated()
      +alpn() / sni() / cipher / version
      ... 30+ accessors
    }

    class Handshaker {
      <<interface>>
      +doHandshake() : PostIoAction
    }

    class SslHandshakerImpl {
      +ssl_ : bssl::UniquePtr~SSL~
      -state_ : SocketState
      -extended_socket_info_ : SslExtendedSocketInfoImpl
      -validated_chain_ : vector~X509~
      -handshake_callbacks_ : HandshakeCallbacks*
      +doHandshake() : PostIoAction
      +state() / setState()
      +setValidatedCertChain(chain)
      +validatedPeerIssuer() : X509*
    }

    SslHandshakerImpl --|> ConnectionInfoImplBase
    SslHandshakerImpl --|> Handshaker

    class SslExtendedSocketInfoImpl {
      -ssl_handshaker_ : SslHandshakerImpl&
      -cert_validate_result_callback_
      -cert_selection_callback_
      -cert_validation_status_
      -cert_validation_result_
      -cert_selection_result_
      +createValidateResultCallback()
      +createCertificateSelectionCallback()
      +onCertificateValidationCompleted()
      +onCertificateSelectionCompleted()
    }

    class ValidateResultCallbackImpl {
      -dispatcher_
      -extended_socket_info_
      +onCertValidationResult(success, status, error, alert)
      +onSslHandshakeCancelled()
    }

    class CertificateSelectionCallbackImpl {
      -dispatcher_
      -extended_socket_info_
      +onCertificateSelectionResult(ctx, staple)
      +onSslHandshakeCancelled()
    }

    SslHandshakerImpl o-- SslExtendedSocketInfoImpl : owns
    SslExtendedSocketInfoImpl o-- ValidateResultCallbackImpl : weak ref
    SslExtendedSocketInfoImpl o-- CertificateSelectionCallbackImpl : weak ref

    class HandshakerFactoryImpl {
      +name() = "envoy.default_tls_handshaker"
      +createHandshakerCb() : returns shared_ptr~SslHandshakerImpl~
      +capabilities() : empty (Envoy handles everything)
      +sslctxCb() : nullptr
    }
```

Notice that `SslHandshakerImpl` extends **both** `ConnectionInfoImplBase` (for filter‑facing accessors) and `Ssl::Handshaker` (for the state machine). That's why filters that ask `connection.ssl()` get back the same object that's running the handshake.

`SslExtendedSocketInfoImpl` is the per‑connection **scratch space** that holds whichever async callback is currently in flight. Its key job is making cancellation safe: when the connection dies, its destructor calls `onSslHandshakeCancelled()` on the live callback, which flips a flag inside that callback so the eventual external completion (from SDS, the dynamic module, or the HSM) becomes a no‑op instead of trying to call back into a freed handshaker. Without this contract Envoy would crash on any connection that closes during an async TLS step — which is a fairly common case during traffic spikes.

---

## State machine

```mermaid
stateDiagram-v2
    [*] --> PreHandshake : SslHandshakerImpl constructed
    PreHandshake --> HandshakeInProgress : first doHandshake()<br/>SSL_do_handshake returns < 1
    HandshakeInProgress --> HandshakeInProgress : I/O pump (BIO_read/BIO_write)
    HandshakeInProgress --> HandshakeComplete : SSL_do_handshake returns 1<br/>handshake_callbacks_.onSuccess(ssl)
    HandshakeInProgress --> [*] : fatal error<br/>handshake_callbacks_.onFailure()

    state HandshakeInProgress {
        [*] --> SyncIo
        SyncIo --> AsyncCertSelect : SSL_ERROR_PENDING_CERTIFICATE
        SyncIo --> AsyncCertVerify : SSL_ERROR_WANT_CERTIFICATE_VERIFY
        SyncIo --> AsyncPrivateKey : SSL_ERROR_WANT_PRIVATE_KEY_OPERATION
        AsyncCertSelect --> SyncIo : onCertificateSelectionResult()<br/>posted to dispatcher
        AsyncCertVerify --> SyncIo : onCertValidationResult()<br/>posted to dispatcher
        AsyncPrivateKey --> SyncIo : onPrivateKeyMethodComplete()<br/>posted to dispatcher
    }

    HandshakeComplete --> ShutdownSent : SSL_shutdown()
    ShutdownSent --> [*]
```

`state_` is an `Ssl::SocketState` enum (defined in `envoy/ssl/ssl_socket_state.h`). The set of states above is what's reachable from `SslHandshakerImpl`.

---

## `doHandshake()` — the main loop

From `ssl_handshaker.cc:116‑140` (paraphrased):

```mermaid
flowchart TB
    A["doHandshake()"] --> B["rc = SSL_do_handshake(ssl)"]
    B --> C{"rc == 1?"}
    C -- yes --> D["state = HandshakeComplete<br/>handshake_callbacks.onSuccess(ssl)"]
    D --> E["return PostIoAction::KeepOpen"]
    C -- no --> F["err = SSL_get_error(ssl, rc)"]
    F --> G{"err"}
    G -- WANT_READ / WANT_WRITE --> H1["return PostIoAction::KeepOpen<br/>(stay in current state)"]
    G -- PENDING_CERTIFICATE<br/>WANT_CERTIFICATE_VERIFY<br/>WANT_PRIVATE_KEY_OPERATION<br/>WANT_X509_LOOKUP --> H2["state = HandshakeInProgress<br/>return PostIoAction::KeepOpen"]
    G -- other --> H3["handshake_callbacks.onFailure()<br/>return PostIoAction::Close"]
```

The "stay in current state" branch is the trick: when BoringSSL says `WANT_READ`/`WANT_WRITE`, the worker's event loop will fire `doRead`/`doWrite` again later (when more bytes arrive or the write buffer drains), and that re‑enters `doHandshake` to try again.

The "async" branches return the same `PostIoAction::KeepOpen` but transition state to `HandshakeInProgress` *if not already there*. The actual resume signal comes later, from one of the three callbacks.

The signature is `PostIoAction` and not `bool`/`StatusOr` because the handshaker has to tell the **transport socket layer** (its caller) what to do with the underlying connection after each step — either keep it open and wait for more I/O, or close it. `KeepOpen` is returned for both "still working" and "done successfully" — the actual state change is communicated by `state()` returning `HandshakeComplete`, which `SslSocket::doHandshake` checks immediately after the call returns.

---

## Async cert selection — full flow

This is the path taken by the `on_demand_secret` cert selector (and any future async selector).

```mermaid
sequenceDiagram
    autonumber
    participant BSSL as BoringSSL
    participant SCtx as ServerContextImpl
    participant Sel as Selector (on-demand etc.)
    participant Ext as SslExtendedSocketInfoImpl
    participant Cb as CertificateSelectionCallbackImpl
    participant Disp as Worker dispatcher
    participant Sock as SslSocket

    BSSL->>SCtx: select_certificate_cb(client_hello)
    SCtx->>Sel: selectTlsContext(ClientHello, ext.createCertificateSelectionCallback())
    Sel->>Ext: createCertificateSelectionCallback()
    Ext-->>Sel: CertificateSelectionCallbackImpl (Cb)
    Sel-->>SCtx: SelectionResult{Pending, handle}
    SCtx-->>BSSL: ssl_select_cert_retry
    Note over BSSL: handshake suspended,<br/>SSL_ERROR_PENDING_CERTIFICATE
    BSSL-->>Sock: SSL_do_handshake returns rc<1
    Sock-->>Sock: state = HandshakeInProgress, return KeepOpen

    Note over Sel: ... later, secret arrives via SDS ...

    Sel->>Cb: onCertificateSelectionResult(ctx, staple)
    Cb->>Disp: dispatcher.post(...)
    Disp->>Ext: onCertificateSelectionCompleted(ctx, staple, async=true)
    Ext->>Sock: handshake_callbacks_.onAsynchronousCertificateSelectionComplete()
    Sock->>Sock: resumeHandshake()
    Sock->>BSSL: SSL_do_handshake() (retry)
    BSSL->>BSSL: SSL_set_SSL_CTX(chosen)
    BSSL-->>Sock: rc == 1, HandshakeComplete
```

Key detail: **the callback is posted to the worker's dispatcher**, not invoked directly. That's `CertificateSelectionCallbackImpl::onCertificateSelectionResult` in `ssl_handshaker.h:56` — it stores the result on `extended_socket_info_` and then posts to `dispatcher_`. This is what makes the selector safe to invoke from the main thread (the on‑demand secret manager lives on main).

The diagram has two distinct horizontal sections separated by the `Note over Sel: ... later, secret arrives via SDS ...` divider. Everything above the divider happens **synchronously** inside one `SSL_do_handshake` call — BoringSSL hits the `select_certificate_cb`, that callback dispatches to the selector, the selector returns `Pending`, BoringSSL bubbles `SSL_ERROR_PENDING_CERTIFICATE` back up to `SslHandshakerImpl`, which returns `KeepOpen` and lets the worker get on with other connections. Everything below the divider may happen **milliseconds, seconds, or never** — that's the point of decoupling. The result of "never" is handled by the cancellation path: when the connection idles out or the client gives up, `SslExtendedSocketInfoImpl`'s destructor cancels the in‑flight callback before it can fire.

`SslExtendedSocketInfoImpl::~SslExtendedSocketInfoImpl()` calls `onSslHandshakeCancelled` on any pending callback — if the connection is dropped while waiting for a cert, the in‑flight callback is told to do nothing instead of touching freed memory.

---

## Async cert verification — full flow

Same idea, different callback. Used when a `CertValidator` returns `ValidationStatus::Pending` — typically the dynamic‑modules validator calling out to an external policy engine.

```mermaid
sequenceDiagram
    autonumber
    participant BSSL as BoringSSL
    participant CV as CertValidator
    participant Ext as SslExtendedSocketInfoImpl
    participant Cb as ValidateResultCallbackImpl
    participant Disp as Worker dispatcher
    participant Sock as SslSocket

    BSSL->>BSSL: SSL_CTX_set_custom_verify cb fires
    BSSL->>CV: doVerifyCertChain(chain, callback)
    CV->>Ext: createValidateResultCallback()
    Ext-->>CV: ValidateResultCallbackImpl (Cb)
    CV-->>BSSL: ValidationResults{Pending}
    BSSL-->>Sock: SSL_ERROR_WANT_CERTIFICATE_VERIFY

    Note over CV: ... async work (RPC, HSM, policy engine) ...

    CV->>Cb: onCertValidationResult(success, status, error, alert)
    Cb->>Disp: dispatcher.post(...)
    Disp->>Ext: onCertificateValidationCompleted(success, async=true)
    Ext->>Sock: handshake_callbacks_.onAsynchronousCertValidationComplete()
    Sock->>Sock: resumeHandshake()
    Sock->>BSSL: SSL_do_handshake() (retry)
```

The validated chain (if the validator returned one) is also handed back through `setValidatedCertChain`. `SslHandshakerImpl::validatedPeerIssuer()` then returns `chain[1]` — the direct issuer — which is used in some access‑log formatters.

The reason the validator returns the chain and Envoy stores it is **trust transitivity**. After validation, downstream filters often want to introspect not just the leaf cert (the peer) but also the issuer (e.g. for spiffe trust domain checks). BoringSSL's default `X509_STORE_CTX` would have this info, but Envoy bypasses it with `SSL_CTX_set_custom_verify`, so the validator becomes the authoritative source of "what cert chain was actually validated".

---

## `SslExtendedSocketInfoImpl` — per‑connection scratch space

This is the object BoringSSL can reach via `SSL_get_ex_data(ssl, ContextImpl::sslExtendedSocketInfoIndex())`. It carries:

| Field | What it holds |
|---|---|
| `certificate_validation_status_` | Validation status reported to filters |
| `cert_validation_result_` | `NotStarted` / `Pending` / `Completed` / `Failed` |
| `cert_validation_alert_` | TLS alert to emit on validation failure |
| `cert_validate_result_callback_` | Latched ref to the live async callback, nullopt otherwise |
| `cert_selection_result_` | `NotStarted` / `Pending` / `Successful` / `Failed` |
| `cert_selection_callback_` | Latched ref to the live async callback |
| `cert_selection_handle_` | Opaque handle the selector uses to identify this connection |
| `cert_validation_error_` | Detailed error string (surfaces in access logs / failure reason) |

```mermaid
flowchart LR
    SSL["BoringSSL SSL*<br/>(ex_data slot)"]
    Ext["SslExtendedSocketInfoImpl"]
    HS["SslHandshakerImpl"]
    Sock["SslSocket"]

    SSL -- "SSL_get_ex_data<br/>(sslExtendedSocketInfoIndex)" --> Ext
    HS -- "owns by value (mutable)" --> Ext
    Sock -- "shared_ptr" --> HS
```

The `ContextImpl::sslExtendedSocketInfoIndex()` index is allocated once per process via `SSL_get_ex_new_index` at static‑init time, so cross‑library plug‑ins can pull the same pointer back out of any `SSL*`.

This ex‑data trick is the **glue** between BoringSSL callbacks (which take only the `SSL*`) and Envoy's object graph (which lives outside BoringSSL). Whenever Envoy installs a BoringSSL callback — cert selection, ALPN selection, custom verify, session‑ticket key cb, key‑log cb — the callback's first job is `SSL_get_ex_data(ssl, index)` to pull back its own `SslHandshakerImpl` or `ContextImpl` pointer. Without it the callbacks would have to be C‑style globals with no per‑connection state.

---

## `HandshakerFactoryImpl` — the default registration

`ssl_handshaker.h:169‑194` registers the **default** handshaker under name `envoy.default_tls_handshaker`. Three things to note about the default:

1. **`createHandshakerCb`** returns a callable that just constructs `SslHandshakerImpl` with the `SSL*` and the ex‑data index.
2. **`capabilities()`** returns an empty `Ssl::HandshakerCapabilities{}` — meaning "Envoy itself handles certificate selection, validation, ALPN, etc.". Custom handshakers can declare more capabilities to take over some of those steps.
3. **`sslctxCb()`** returns `nullptr` — the default doesn't post‑process `SSL_CTX`. A custom handshaker can return a callback to e.g. wire BoringSSL extensions before any handshake runs.

A `custom_handshaker` (configured via `CommonTlsContext.custom_handshaker`) would replace this factory with one that produces its own `Ssl::Handshaker`. The `HandshakerFactoryContextImpl` (lines 149‑167) hands the custom factory the things it needs to bootstrap: API, lifecycle notifier, ALPN string, singleton manager.

---

## Common pitfalls

- **Lifetime of `SslHandshakerImpl`.** It's owned via `shared_ptr` by both `SslSocket` and BoringSSL's `ex_data` slot. If you nuke `SslSocket` while a handshake is in flight, the callbacks need to detect that — that's exactly what `onSslHandshakeCancelled` (lines 43, 58) is for.
- **Don't invoke the callback from main thread directly.** Always go through `dispatcher_.post(...)`. Otherwise you'd be touching `SslHandshakerImpl` from main while a worker is running on it.
- **`SSL_ERROR_WANT_X509_LOOKUP` also pauses.** It's in the async branch alongside the cert‑select / cert‑verify errors. This was added for the future where a sync verifier might decide it actually needs an async lookup mid‑validation.
- **Async cert verification skips the "store context" path.** Default BoringSSL validation uses `X509_STORE_CTX`. Envoy installs `SSL_CTX_set_custom_verify` and routes through `CertValidator`, which is why the `X509_STORE` configured on the `SSL_CTX` is largely ignored for the verify step.
- **Validation status vs. selection status.** Two completely separate state machines (`cert_validation_result_` and `cert_selection_result_`). A connection can be `Pending` on selection but `NotStarted` on validation if mTLS isn't enabled.

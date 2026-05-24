# OCSP — `source/common/tls/ocsp/`

> Files: `ocsp.{h,cc}`, `asn1_utility.{h,cc}`

OCSP (Online Certificate Status Protocol) support for **TLS stapling**. Envoy doesn't run OCSP requests itself — operators provide a **pre‑fetched, DER‑encoded OCSP response** alongside each cert, and Envoy staples it into the ServerHello to save the client a round‑trip to the CA's OCSP responder.

This folder parses those DER blobs (`ocsp.cc`), keeps them per `TlsContext`, and computes whether to staple at handshake time.

> ⚠️ This module **does not validate signatures on OCSP responses**. The header comment is explicit: it assumes responses come from a trusted source (operator's config / SDS). It only checks they are well‑formed and not expired.

### Why "pre‑fetched"?

A live OCSP fetch from inside the handshake would add a synchronous network round‑trip — completely negating the latency benefit of stapling. The standard pattern (and what Envoy assumes) is a separate background job: a sidecar or cron that calls the CA's OCSP responder every few hours, drops the DER bytes into a file or pushes them through SDS, and Envoy hot‑reloads. This is also why signature validation is delegated: the responder authentication happened in the sidecar, the operator vouches for the bytes by configuring them, and Envoy treats the blob as authoritative.

### What stapling actually buys you

Without stapling, every client that checks revocation does an extra DNS lookup + TLS handshake + HTTP request to the CA's responder, **per connection**. For a public endpoint with millions of clients this is enormous. With stapling, the server includes a current OCSP response in the handshake; the client validates it inline. Some clients (Firefox, recent Chrome) refuse to handshake at all without a staple for must‑staple certs — so getting this right is sometimes a hard availability requirement, not just a perf optimisation.

---

## Where stapling fits in the handshake

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Server as Envoy (ServerContextImpl)
    participant Selector as TlsCertificateSelector
    participant TC as TlsContext (chosen)

    Client->>Server: ClientHello (with status_request extension)
    Note over Server: isClientOcspCapable(client_hello) -> true
    Server->>Selector: selectTlsContext(client_hello)
    Selector-->>Server: SelectionResult{TC, OcspStapleAction}
    alt staple
      Server->>Server: SSL_set_tlsext_status_ocsp_resp(TC.ocsp_response_)
      Server->>Client: ServerHello + Certificate + OCSP staple
    else no staple
      Server->>Client: ServerHello + Certificate (no staple)
    end
```

`ocspStapleAction` (declared in `default_tls_certificate_selector.h:19‑21`, implemented in `default_tls_certificate_selector.cc`) decides one of: `Staple`, `NoStaple`, `Fail` based on:

- The cert's `must_staple` extension (`TlsContext::is_must_staple_`).
- The configured `ocsp_staple_policy` on the server config.
- Whether the client signalled OCSP capability in the ClientHello.
- Whether the OCSP response is present and not expired.

---

## Data model (`ocsp.h`)

```mermaid
classDiagram
    class OcspResponse {
      +status_ : OcspResponseStatus
      +response_ : ResponsePtr
    }

    class OcspResponseStatus {
      <<enum>>
      Successful = 0
      MalformedRequest = 1
      InternalError = 2
      TryLater = 3
      SigRequired = 5
      Unauthorized = 6
    }

    class Response {
      <<abstract>>
      +getNumCerts() : size_t
      +getCertSerialNumber() : string
      +getThisUpdate() : SystemTime
      +getNextUpdate() : optional~SystemTime~
    }

    class BasicOcspResponse {
      +OID = "1.3.6.1.5.5.7.48.1.1"
      -data_ : ResponseData
      +getNumCerts() / getCertSerialNumber()
      +getThisUpdate() / getNextUpdate()
    }

    class ResponseData {
      +single_responses_ : SingleResponse[]
    }

    class SingleResponse {
      +cert_id_ : CertId
      +this_update_ : SystemTime
      +next_update_ : optional~SystemTime~
    }

    class CertId {
      +serial_number_ : string
    }

    class OcspResponseWrapperImpl {
      -raw_bytes_ : vector~uint8_t~
      -response_ : unique_ptr~OcspResponse~
      -time_source_ : TimeSource&
      +create(der_response, time_source) : StatusOr
      +getResponseStatus() : OcspResponseStatus
      +matchesCertificate(X509&) : bool
      +secondsUntilExpiration() : uint64_t
      +getThisUpdate() / getNextUpdate()
      +isExpired() : bool
      +rawBytes() : vector~uint8_t~
    }

    OcspResponse o-- OcspResponseStatus
    OcspResponse o-- Response
    Response <|-- BasicOcspResponse
    BasicOcspResponse *-- ResponseData
    ResponseData *-- SingleResponse
    SingleResponse *-- CertId
    OcspResponseWrapperImpl o-- OcspResponse
```

### Important constraint

`BasicOcspResponse::getNumCerts` is used as a sanity check: Envoy enforces that **each OCSP response covers exactly one certificate**. The cert this is for is identified by serial number (`CertId.serial_number_`).

The "one cert per response" rule is stricter than OCSP itself allows (the spec permits multiple `SingleResponse` entries for batch queries), but in practice CAs always return one response per cert and Envoy's per‑cert model lines up with that. If a response carrying multiple entries ever arrived, the constraint check would reject it at config‑load time rather than fail mysteriously at handshake time.

### `OcspResponseWrapperImpl`

The lifetime object owned by `TlsContext.ocsp_response_`. Holds:

- The raw DER bytes (`raw_bytes_`) — sent verbatim on the wire.
- The parsed `OcspResponse` (`response_`).
- A `TimeSource&` so `isExpired` and `secondsUntilExpiration` are computable.

```mermaid
flowchart LR
    A["config: ocsp_staple bytes"] --> B["OcspResponseWrapperImpl::create(bytes, time_source)"]
    B --> C["Asn1OcspUtility::parseOcspResponse(cbs)"]
    C --> D{"parse ok?"}
    D -- no --> E["return InvalidArgumentError"]
    D -- yes --> F["build OcspResponseWrapperImpl"]
    F --> G["store on TlsContext.ocsp_response_"]

    H["at handshake: TlsContext chosen"] --> I["wrapper.isExpired()?"]
    I -- no --> J["wrapper.rawBytes() sent on wire"]
    I -- yes --> K["NoStaple (or Fail if must_staple)"]
```

`matchesCertificate(cert)` verifies that the OCSP response's serial number matches the leaf cert's serial — protects against accidentally configuring an OCSP response for the wrong cert.

This serial‑number check has caught real misconfigurations during cert rotation. When the cert rotates but the operator forgets to also rotate the OCSP response, the new cert's serial won't match the stale staple and Envoy will refuse to staple it (or fail closed for must‑staple certs). The alternative — sending a mismatched staple — would cause clients to fail the handshake instead, which is much worse for debuggability.

---

## DER parsing pipeline

```mermaid
flowchart TB
    A["DER bytes (operator-provided / SDS)"]
    A --> B["Asn1OcspUtility::parseOcspResponse(cbs)"]
    B --> C["parseResponseStatus -> OcspResponseStatus"]
    B --> D{"status == Successful?"}
    D -- no --> E["wrap in OcspResponse with null body"]
    D -- yes --> F["parseResponseBytes -> ResponsePtr"]
    F --> F1["check OID == BasicOcspResponse::OID"]
    F1 --> G["parseBasicOcspResponse"]
    G --> H["parseResponseData"]
    H --> I["parseSequenceOf~SingleResponse~"]
    I --> J["parseSingleResponse"]
    J --> K["parseCertId(serial)<br/>parseGeneralizedTime(this_update)<br/>parseOptional(GeneralizedTime, next_update)"]
```

Every parsing function takes a `CBS&` (BoringSSL "crypto byte string" cursor) and advances it past the element it consumed. This matches BoringSSL's recommended ASN.1 parsing pattern (the alternative is `d2i_*` functions, which have a different memory ownership model).

---

## `Asn1Utility` — the ASN.1 toolkit

`asn1_utility.h:51‑158`. Generic ASN.1 DER parsing helpers, factored out of OCSP because they're useful for parsing any ASN.1.

| Function | Purpose |
|---|---|
| `cbsToString(cbs)` | Get the remaining bytes as a `string_view` |
| `parseSequenceOf<T>(cbs, parse_elem)` | Generic `SEQUENCE OF` parser (template) |
| `parseOptional<T>(cbs, parse, tag)` | Explicitly tagged optional element |
| `getOptional(cbs, tag)` | "Is this tag present?" with optional CBS payload |
| `parseOid(cbs)` | OBJECT IDENTIFIER → dotted‑number string |
| `parseGeneralizedTime(cbs)` | GENERALIZEDTIME → `SystemTime` |
| `parseInteger(cbs)` | Arbitrary‑precision INTEGER → hex string |
| `parseOctetString(cbs)` | OCTET STRING → `vector<uint8_t>` |
| `skipOptional(cbs, tag)` / `skip(cbs, tag)` | Advance past a value without parsing it |

The two template functions (`parseSequenceOf`, `parseOptional`) are defined inline in the header so callers can instantiate them for any element type.

### `parseSequenceOf` invariant

```mermaid
flowchart TB
    A["CBS_get_asn1(cbs, &seq_elem, CBS_ASN1_SEQUENCE)"] --> B["seq_elem points to inner bytes"]
    B --> C{"CBS_data(seq_elem) < CBS_data(cbs)?"}
    C -- yes --> D["parse_element(seq_elem)<br/>MUST advance seq_elem"]
    D --> E["push T into vec"]
    E --> C
    C -- no --> F["RELEASE_ASSERT(CBS_data(cbs) == CBS_data(seq_elem))"]
    F --> G["return vec"]
```

The assertion (`asn1_utility.h:180`) catches the case where `parse_element` misbehaves and leaves bytes unconsumed — that would indicate a malformed input or a bug in the element parser.

---

## Staple decision

```mermaid
flowchart TB
    A["OcspStapleAction ocspStapleAction(ctx, client_ocsp_capable, policy)"]
    A --> B{"client_ocsp_capable?"}
    B -- no --> C{"is_must_staple_?"}
    C -- yes --> D["Fail (handshake aborts)"]
    C -- no --> E["NoStaple"]
    B -- yes --> F{"ctx.ocsp_response_ present?"}
    F -- no --> G{"is_must_staple_ or policy==MUST_STAPLE?"}
    G -- yes --> H["Fail"]
    G -- no --> I["NoStaple"]
    F -- yes --> J{"response expired?"}
    J -- yes --> K{"policy"}
    K -- LENIENT_STAPLING --> L["NoStaple (proceed)"]
    K -- STRICT_STAPLING --> M["Fail (cert not used)"]
    K -- MUST_STAPLE --> M
    J -- no --> N["Staple"]
```

(Exact logic lives in `default_tls_certificate_selector.cc`; this is the policy summary.)

The three OCSP stats from `SslStats` fire at:

- `ocsp_staple_requests` — client sent `status_request`.
- `ocsp_staple_responses` — Envoy actually stapled.
- `ocsp_staple_omitted` — Envoy could have stapled but chose not to (`LENIENT_STAPLING` + expired).
- `ocsp_staple_failed` — `Fail` outcome above.

---

## Lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant Cfg as ContextConfigImpl
    participant Wrap as OcspResponseWrapperImpl
    participant TC as TlsContext
    participant Sel as Selector
    participant BSSL

    Cfg->>Wrap: create(der_bytes, time_source)
    Wrap->>Wrap: parse via Asn1OcspUtility
    Wrap-->>Cfg: unique_ptr<OcspResponseWrapperImpl> or InvalidArgument
    Cfg->>TC: store in tls_contexts_[i].ocsp_response_

    Note over TC: per handshake
    Sel->>TC: check ocsp_response_ + isExpired()
    Sel->>BSSL: SSL_set_tlsext_status_ocsp_resp(rawBytes())
```

---

## What's *not* in this folder

- **Live OCSP fetching.** Envoy doesn't run an OCSP client. If you want fresh responses, an external process must refresh them and push them in via SDS or config reload.
- **OCSP request building.** Envoy never builds an OCSP request — it only ingests responses.
- **OCSP signature validation.** As called out in the header, this module assumes the responses come from a trusted source.

These omissions are by design — fetching OCSP responses at handshake time would add latency and reliability issues; pre‑fetched stapling is the modern best practice.

---

## Cheat sheet

| Question | Answer |
|---|---|
| Does Envoy run OCSP requests? | No — operator pre‑fetches and configures responses |
| Where is the staple decision made? | `ocspStapleAction` in `default_tls_certificate_selector.cc` |
| How is the response stored? | `TlsContext::ocsp_response_` (per cert) |
| How is it sent on the wire? | `SSL_set_tlsext_status_ocsp_resp(ssl, raw_bytes, len)` |
| What happens if expired? | Depends on `ocsp_staple_policy` (LENIENT/STRICT/MUST_STAPLE) and the cert's `must_staple` extension |
| Are signatures on OCSP responses validated? | No — module trusts the source |
| Where is the ASN.1 parser? | `asn1_utility.{h,cc}` — generic helpers, OCSP‑specific uses in `ocsp.cc` |
| Where are the OCSP stats? | `SslStats::ocsp_staple_{requests,responses,omitted,failed}` |

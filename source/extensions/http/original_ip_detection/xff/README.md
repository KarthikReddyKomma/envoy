# XFF Original IP Detection

An `OriginalIPDetection` extension consumed by the HTTP Connection Manager.
It inspects the `x-forwarded-for` request header and walks back either
a fixed number of hops or a list of trusted proxy CIDRs to derive the
downstream client's real address. This replaces the legacy
`xff_num_trusted_hops` / `use_remote_address` HCM fields by moving them
into a pluggable extension.

Proto: `envoy.extensions.http.original_ip_detection.xff.v3.XffConfig`.

## Files
- `xff.h/cc` - `XffIPDetection`.
- `config.h/cc` - `XffIPDetectionFactory`.

## Interface
- Implements `Envoy::Http::OriginalIPDetection`; called by the HCM's
  IP-detection pipeline.
- Factory implements `Envoy::Http::OriginalIPDetectionFactory`.

## Logic
- Configuration is validated in `XffIPDetection::create`
  (`xff.cc:12`): `xff_trusted_cidrs` and `xff_num_trusted_hops` are
  mutually exclusive; setting both returns an error status.
- On construction, trusted CIDRs are parsed via
  `Network::Address::CidrRange::create`. Invalid entries are silently
  dropped (the error path is logged by `CidrRange::create` itself).
- `detect(params)`:
  - CIDR mode (list non-empty): first check that
    `downstream_remote_address` is itself inside the trusted CIDR set
    (`Http::Utility::remoteAddressIsTrustedProxy`). If not, returns
    `{nullptr, false, ..., skip_xff_append=false}` - the HCM will fall
    through to the next extension or use the socket address.
    Otherwise, walk the XFF list from the right and return the last
    address that is *not* inside the trusted CIDRs via
    `Http::Utility::getLastNonTrustedAddressFromXFF`.
  - Hops mode: return the address `xff_num_trusted_hops` positions from
    the right via `Http::Utility::getLastAddressFromXFF`.
- `skip_xff_append` is propagated from config (default true, preserves
  legacy HCM behavior of not re-appending this hop to XFF).
- `allow_trusted_address_checks_` in the returned result is set by the
  utility helpers to true only when the XFF walk arrived at the first
  element (i.e. the leftmost client address).

## Key decision points
- `xff.cc:15` - mutual exclusion between hops and CIDRs.
- `xff.cc:47` - CIDR mode requires the immediate socket peer to be a
  trusted proxy; untrusted peers disable IP detection entirely.
- `xff.cc:53` - CIDR mode walks until the first non-trusted address.
- `xff.cc:58` - hops mode uses a simple index from the rightmost entry.
- `xff.h:37` - `skip_xff_append_` defaults to true
  (`PROTOBUF_GET_WRAPPED_OR_DEFAULT`).

## Configuration
- `xff_num_trusted_hops` - integer, count of trusted proxy hops from the
  right of the XFF list.
- `xff_trusted_cidrs` - list of CIDR ranges considered trusted proxies
  (mutually exclusive with the above).
- `skip_xff_append` - if true, HCM will not append this hop's address to
  XFF (default true).

## Stats / errors
No stats. Construction-time errors surface as an `InvalidArgumentError`
from `create`; malformed CIDRs are dropped (with a trace from the CIDR
helper).

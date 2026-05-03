# Custom Header Original IP Detection

An `OriginalIPDetection` extension consumed by the HTTP Connection Manager
(HCM). When `original_ip_detection_extensions` are configured on the HCM,
the HCM iterates them to compute the "real" downstream address for
remote-address-aware behaviors (connection tracking, access logs, RBAC,
etc.). This extension trusts a configurable request header as the source
of truth (e.g. `x-real-ip`, `cf-connecting-ip`) with optional configurable
rejection on missing/invalid values.

Proto: `envoy.extensions.http.original_ip_detection.custom_header.v3.CustomHeaderConfig`.

## Files
- `custom_header.h/cc` - `CustomHeaderIPDetection`.
- `config.h/cc` - `CustomHeaderIPDetectionFactory`.

## Interface
- Implements `Envoy::Http::OriginalIPDetection`: single method
  `detect(params)` returning an `OriginalIPDetectionResult` with the
  parsed address, a "trusted address" flag, an optional
  `RejectRequestOptions`, and a `skip_xff_append` flag.
- Factory implements `Envoy::Http::OriginalIPDetectionFactory`, registered
  via `Registry::RegisterFactory`.

## Logic
- Config carries a `header_name` (required, lower-cased in the ctor),
  `allow_extension_to_set_address_as_trusted` (whether the detected
  address should be marked as a trusted proxy chain endpoint for
  downstream decisions), and an optional `reject_with_status` HTTP code.
- `detect`:
  1. Reads the configured header. If absent, returns `{nullptr, false,
     reject_options_, skip_xff_append=true}` - the HCM will reject (if
     configured) or fall through to the next extension.
  2. If present, parses the value with
     `Network::Utility::parseInternetAddressNoThrow`. On success, returns
     the address with the configured trusted-flag. On parse failure,
     returns the same null+reject result as the missing case.
- `skip_xff_append` is hard-coded to true
  (`custom_header.cc:31`) so the detected address is not appended to
  `x-forwarded-for` - callers owning XFF should use the `xff` extension
  instead (this preserves the pre-#31831 behavior).

## Key decision points
- `custom_header.cc:29-31` - `skip_xff_append` is always true for this
  extension.
- `custom_header.h:29` - `toErrorCode` clamps the configured reject status
  to a safe `4xx/5xx` band, defaulting to `403` if out of range.
- `custom_header.cc:33` - missing header falls through without logging
  (to avoid per-request noise).
- `custom_header.cc:40` - failed parse does not allow the request to
  proceed with an unknown address; returns reject options.

## Configuration
- `header_name` (required).
- `allow_extension_to_set_address_as_trusted` - if true, the parsed
  address is flagged as a trusted proxy for e.g. XFF trust decisions.
- `reject_with_status` - optional HTTP status to apply when the header is
  missing or unparsable.

## Stats / errors
No dedicated stats. Rejections surface through the HCM as a local reply
with the configured status (or 403 fallback).

# Envoy Default Header Validator

The default Universal Header Validator (UHV). Each codec in the HCM asks
its configured `HeaderValidatorFactory` for a validator; this extension is
the built-in implementation that applies Envoy's interpretation of
RFC 9110 / 9112 / 9113 rules to request and response headers/trailers
across HTTP/1, HTTP/2, and HTTP/3. It also owns path normalization -
percent-decoding, slash merging, dot-segment collapsing, backslash-to-slash
translation, and encoded-slash handling.

Proto: `envoy.extensions.http.header_validators.envoy_default.v3.HeaderValidatorConfig`.

## Files
- `header_validator.h/cc` - protocol-agnostic base class with the shared
  methods (`validateMethodHeader`, `validateStatusHeader`,
  `validateGenericHeaderName`, `validateGenericHeaderValue`,
  `validateContentLengthHeader`, `validateHostHeader`, path/host helpers,
  trailer validation, URL path transform).
- `http1_header_validator.h/cc` - HTTP/1 server + client validators; adds
  `Transfer-Encoding` / `Content-Length` coexistence rules and HTTP/1
  specific header name validation.
- `http2_header_validator.h/cc` - HTTP/2 (and HTTP/3) server + client
  validators; enforces lowercase header names, pseudo-header ordering, and
  connection-specific header bans.
- `path_normalizer.h/cc` - `PathNormalizer` multi-pass path rewrite
  (decode, merge slashes, collapse dot-segments, split query).
- `character_tables.h` - pre-built `uint32_t[8]` bitmaps for allowed
  header/path character sets.
- `config_overrides.h` - wrapper bundling reloadable-feature flags that
  change validator behavior at runtime.
- `config.h/cc`, `header_validator_factory.h/cc` - factory registering
  the `HeaderValidatorFactory`.

## Interface
- Factory implements `Envoy::Http::HeaderValidatorFactory`. For each
  stream, the HCM asks for a `ServerHeaderValidator` (downstream side) or
  `ClientHeaderValidator` (upstream side). Each validator implements
  `validateRequestHeaders`, `validateResponseHeaders`,
  `validateRequestTrailers`, `validateResponseTrailers`, and the matching
  `transform*Headers` / `transform*Trailers` methods.
- The validators produce a `ValidationResult` with an action
  (Accept/Reject/Redirect) and error details; the codec uses this to
  decide whether to send 400/stream-reset/redirect.

## Logic
- `HeaderValidator` is the shared base. Each derived validator keeps its
  own `HeaderValidatorMap` of header-name -> per-header validator function
  (e.g. `:method`, `:status`, `:path`, `host`, `content-length`,
  `transfer-encoding`, plus pseudo-headers). Generic headers fall back to
  `validateGenericHeaderName` / `validateGenericHeaderValue` character-set
  checks built from `character_tables.h`.
- Path handling: `validateRequestHeaders` calls
  `transformUrlPath`, which runs `PathNormalizer::normalizePathUri`. Its
  passes (decode, merge slashes, collapse dot segments) each return
  accept/reject/redirect. Encoded slashes are handled via
  `sanitizeEncodedSlashes` (which may return a redirect action). `%00`
  detection, fragment stripping, and non-compliant character encoding are
  all gated by `ConfigOverrides` flags (e.g.
  `envoy.uhv.allow_non_compliant_characters_in_path`).
- HTTP/1 specifics (`http1_header_validator.cc`): request is rejected if
  both `Transfer-Encoding` and `Content-Length` are present unless
  `allow_chunked_length` is set, in which case `Content-Length` is
  stripped during `transformRequestHeaders`. Server CONNECT with
  `Content-Length: 0` has the header removed
  (`ServerHttp1HeaderValidator::sanitizeContentLength`).
- HTTP/2 specifics (`http2_header_validator.cc`): rejects uppercase header
  names, connection-specific headers, and any pseudo-header after the
  first regular header.
- `headers_with_underscores_action` (in config) is enforced via
  `sanitizeHeadersWithUnderscores` for DROP or at validation time for
  REJECT.

## Key decision points
- `header_validator.h:88` - `HostHeaderValidationResult` tuple carries the
  validated address and port back to the caller.
- `path_normalizer.h:32` - `PercentDecodeResult` drives the decode pass's
  accept/reject/redirect outcome.
- `http1_header_validator.h:44` - `allow_chunked_length` toggle between
  RFC compliance and legacy behavior.
- `http1_header_validator.h:76` - alternate "permissive" path validator
  used when `envoy.uhv.allow_non_compliant_characters_in_path` is true.
- `http1_header_validator.h:130` - CONNECT special-case to strip zero
  `Content-Length`.

## Configuration
- `http1_protocol_options.allow_chunked_length`.
- `uri_path_normalization_options` (normalize path, merge slashes,
  encoded-slashes action).
- `restrict_http_methods`.
- `headers_with_underscores_action` (ALLOW / REJECT_REQUEST / DROP).
- Runtime guards on
  `envoy.uhv.*` (exposed via `ConfigOverrides`).

## Stats / errors
Stats are provided by the HCM-side `HeaderValidatorStats` (`messaging_error_`,
`requests_rejected_with_underscores_in_headers_`, etc.). Rejection paths
produce detail strings that surface in access logs.

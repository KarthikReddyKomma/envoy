# AWS Request Signing (`envoy.filters.http.aws_request_signing`)

Decoder-only filter that applies AWS SigV4 or SigV4A signing to the outgoing request. Unlike `aws_lambda`, this filter does not rewrite the target URL; it just signs whatever request the stream is already carrying, so it can be dropped in front of any AWS service endpoint (S3, DynamoDB, Bedrock, etc.). Per-route overrides allow different services/regions/credentials on a per-route basis.

Proto: `envoy.extensions.filters.http.aws_request_signing.v3.AwsRequestSigning` (plus `AwsRequestSigningPerRoute`).

## Files
- `aws_request_signing_filter.h/cc` — `Filter`, `FilterConfig` interface, `FilterConfigImpl`, stats.
- `config.h/cc` — `AwsRequestSigningFilterFactory` (downstream + upstream), signer/credentials/region resolution.

## Lifecycle
`Filter` extends `Http::PassThroughDecoderFilter` (`aws_request_signing_filter.h:88`). Only decode-side methods are overridden.

- `decodeHeaders(headers, end_stream)` (`aws_request_signing_filter.cc:34`):
  - Resolves the effective config (per-route or listener) via `getConfig()` (`aws_request_signing_filter.cc:152-159`).
  - Applies optional `host_rewrite` by calling `headers.setHost(host_rewrite)` (`aws_request_signing_filter.cc:40-42`).
  - Stashes `request_headers_` for later.
  - When `use_unsigned_payload=false` and body will follow (`end_stream=false`), returns `StopIteration` so signing is deferred until the buffered body hash is known in `decodeData` (`aws_request_signing_filter.cc:46-48`).
  - Otherwise arms `cancel_callback_` with a `continueDecodeHeaders + continueDecoding` closure and asks the signer `addCallbackIfCredentialsPending`. If creds are ready, signs immediately and `Continue`; if pending, `StopIteration` and the dispatcher-posted callback resumes decoding when creds arrive (`aws_request_signing_filter.cc:53-71`).
- `continueDecodeHeaders(config)` (`aws_request_signing_filter.cc:74`):
  - `use_unsigned_payload=true` -> `signer.signUnsignedPayload(headers)` (adds `x-amz-content-sha256: UNSIGNED-PAYLOAD`).
  - Else `signer.signEmptyPayload(headers)` (empty body hash).
  - Result feeds `addSigningStats` which bumps `signing_added` or `signing_failed`.
- `decodeData(data, end_stream)` (`aws_request_signing_filter.cc:94`):
  - `use_unsigned_payload=true` -> immediate `Continue`; body has already been signed as unsigned (`aws_request_signing_filter.cc:97-99`).
  - Buffers until end-of-stream (`StopIterationAndBuffer`).
  - On end-of-stream, appends `data` to the decoding buffer, computes SHA-256 over the full buffer, and runs the same async-credentials dance around `continueDecodeData(config, hash)` (`aws_request_signing_filter.cc:105-133`).
- `continueDecodeData(config, hash)` (`aws_request_signing_filter.cc:136`): calls `signer.sign(headers, hash)` and records both `addSigningStats` (`signing_added`/`signing_failed`) and `addSigningPayloadStats` (`payload_signing_added`/`payload_signing_failed`).
- Destructor calls `cancel_callback_()` to unregister pending callbacks (`aws_request_signing_filter.h:91`).

## Decision / logic
- Per-route override via `resolveMostSpecificPerFilterConfig<FilterConfig>` — `FilterConfig` is itself a `Router::RouteSpecificFilterConfig` (`aws_request_signing_filter.h:38`, `aws_request_signing_filter.cc:152-159`).
- The split between `signEmptyPayload` / `signUnsignedPayload` / full-body `sign(hash)` is the core signing decision (`aws_request_signing_filter.cc:74-83`, `aws_request_signing_filter.cc:136-141`).
- The factory enforces algorithm/region consistency: SigV4 rejects region strings containing `*` or `,` (region sets), which are only legal for SigV4A (`config.cc:20-27`, `config.cc:162-176`).
- Region resolution falls back through `RegionProviderChain` when proto `region` is empty — `getRegionSet()` for SigV4A, `getRegion()` for SigV4. Failure -> `InvalidArgumentError` (`config.cc:101-116`).
- Credentials resolution (`config.cc:122-142`):
  - `credential_provider.inline_credential` -> single `InlineCredentialProvider`.
  - `credential_provider` (other cases) -> `customCredentialsProviderChain`.
  - None configured -> `defaultCredentialsProviderChain`.
- SigV4 vs SigV4A is selected from `signing_algorithm` and instantiates `SigV4SignerImpl` or `SigV4ASignerImpl` with include/exclude header matchers, optional query-string signing, and an expiration (default `SignatureQueryParameterValues::DefaultExpiration`) (`config.cc:148-176`).

## Configuration
- `service_name`, `region`, `signing_algorithm` (SIGV4 / SIGV4A).
- `host_rewrite` — optional `:authority` rewrite before signing.
- `use_unsigned_payload` — sign headers only (no body hash). When true, `decodeData` is a pass-through.
- `match_included_headers` / `match_excluded_headers` — header matchers fed to the signer.
- `query_string.expiration_time` — when signing via presigned URL parameters.
- `credential_provider` — inline credentials, custom chain, or file provider; otherwise default SDK chain.
- Per-route (`AwsRequestSigningPerRoute`) carries a full `AwsRequestSigning` plus a `stat_prefix` and produces a new `FilterConfigImpl` via `createRouteSpecificFilterConfigTyped` (`config.cc:66-81`).

## Stats
Prefix `<stats_prefix>aws_request_signing.` (`aws_request_signing_filter.cc:29-32`):

- `signing_added` — header signing succeeded.
- `signing_failed` — header signing failed.
- `payload_signing_added` — full-body (hash) signing succeeded.
- `payload_signing_failed` — full-body signing failed.

## Factory
`AwsRequestSigningFilterFactory` (downstream) and `UpstreamAwsRequestSigningFilterFactory` (upstream) are both registered (`config.cc:182-185`). The shared helper `createFilterFactoryFromProtoHelper` (`config.cc:29-45`) builds a signer via `createSigner(...)` (`config.cc:83-177`), wraps it in a `FilterConfigImpl`, and returns a callback that calls `addStreamDecoderFilter(std::make_shared<Filter>(filter_config))`. Route-specific configs are built by `createRouteSpecificFilterConfigTyped` (`config.cc:66-81`) which creates an independent signer and a const `FilterConfigImpl` used via the per-route resolution at request time.

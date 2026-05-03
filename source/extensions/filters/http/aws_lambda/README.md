# AWS Lambda (`envoy.filters.http.aws_lambda`)

HTTP filter that invokes an AWS Lambda function as the upstream. It rewrites the request URL/method/headers to the Lambda invoke API, applies AWS SigV4 signing, and optionally JSON-wraps the request body and JSON-unwraps the response. Supports synchronous and asynchronous invocation, payload pass-through (raw body), and per-route ARN overrides.

Proto: `envoy.extensions.filters.http.aws_lambda.v3.Config` / `PerRouteConfig`.

## Files
- `aws_lambda_filter.h/cc` — `Filter`, `Arn`, `FilterSettings`/`FilterSettingsImpl`, stats, ARN parser.
- `request_response.proto` — internal protos used to (de)serialize the Lambda request/response JSON envelopes.
- `config.h/cc` — `AwsLambdaFilterFactory` (downstream + upstream), SigV4 signer wiring, credentials provider selection.

## Lifecycle
`Filter` extends `Http::PassThroughFilter` (`aws_lambda_filter.h:116`) and overrides both sides.

- `decodeHeaders(headers, end_stream)` (`aws_lambda_filter.cc:131`):
  - If not an upstream filter and the routed cluster does not have `com.amazonaws.lambda.egress_gateway=true` in its `filter_metadata`, sets `skip_=true` and `Continue` — the filter is a no-op for non-Lambda clusters (`aws_lambda_filter.cc:132-139`, `aws_lambda_filter.cc:61-79`).
  - Stashes `request_headers_`. When `end_stream` is false (body will follow), returns `StopIteration` and defers signing to `decodeData` (`aws_lambda_filter.cc:143-147`).
  - When headers-only, asks the signer `addCallbackIfCredentialsPending(cb)`. If no credentials fetch is pending, signs immediately and `Continue` (`aws_lambda_filter.cc:148-162`). If pending, stores `cancel_callback_` and returns `StopIteration`; the dispatcher-posted callback calls `continueDecodeHeaders` + `continueDecoding()` when creds arrive.
- `continueDecodeHeaders(settings)` (`aws_lambda_filter.cc:169`):
  - `payload_passthrough=true` path: rewrite headers via `setLambdaHeaders` and call `signer.signEmptyPayload` (`aws_lambda_filter.cc:170-178`).
  - Else JSON-envelope the headerless request (`jsonizeRequest`), set Content-Length and `content-type: application/json`, sign with SHA-256 of the JSON, and push the JSON into the decode buffer via `addDecodedData(..., false)` (`aws_lambda_filter.cc:180-196`).
- `decodeData(data, end_stream)` (`aws_lambda_filter.cc:222`):
  - No-op when `skip_`.
  - Buffers until end-of-stream (`StopIterationAndBuffer`) (`aws_lambda_filter.cc:227-229`).
  - On end-of-stream, same async-credentials dance as `decodeHeaders` around `continueDecodeData` (`aws_lambda_filter.cc:231-253`).
- `continueDecodeData(settings)` (`aws_lambda_filter.cc:256`):
  - For non-passthrough, swaps the decoding buffer contents with the JSON-envelope (built by `jsonizeRequest` with the buffered body) and sets JSON content-type/length (`aws_lambda_filter.cc:260-270`).
  - Always rewrites headers with `setLambdaHeaders`, signs with SHA-256 of the (now-JSON) buffer, and records the `upstream_rq_payload_size` histogram (`aws_lambda_filter.cc:272-282`).
- `encodeHeaders(headers, end_stream)` (`aws_lambda_filter.cc:198`):
  - Short-circuits `Continue` if already `skip_` or headers-only response (`aws_lambda_filter.cc:199-201`).
  - Sets `skip_=true` when HTTP status `>= 300` or when `x-amz-function-error` is present, so errors are passed through unmodified (`aws_lambda_filter.cc:206-216`).
  - Otherwise stashes `response_headers_` and returns `StopIteration` to wait for the body (`aws_lambda_filter.cc:218-219`).
- `encodeData(data, end_stream)` (`aws_lambda_filter.cc:285`):
  - Skip-through when `skip_`, `payload_passthrough=true`, or invocation mode is `Asynchronous` (no body to unwrap) (`aws_lambda_filter.cc:286-290`).
  - Buffers until end-of-stream. Then `dejsonizeResponse` rebuilds the response headers/body from the Lambda JSON envelope (status, cookies, base64 body decoding if `is_base64_encoded`) (`aws_lambda_filter.cc:292-307`, `aws_lambda_filter.cc:363-408`).
- Destructor calls `cancel_callback_()` to drop any pending credentials-callback (`aws_lambda_filter.h:120`).

## Decision / logic
- Per-route override resolved via `resolveMostSpecificPerFilterConfig<FilterSettings>` in `getSettings()` (`aws_lambda_filter.cc:122-129`); `FilterSettings` is itself a `Router::RouteSpecificFilterConfig` (`aws_lambda_filter.h:82`).
- `setLambdaHeaders`: always POST, path `/2015-03-31/functions/<arn>/invocations`, `x-amz-invocation-type: RequestResponse` for Synchronous or `Event` for Asynchronous, optional `host_rewrite` (`aws_lambda_filter.cc:44-56`).
- `isContentTypeTextual` decides body base64-encoding: chunked/gzip/deflate transfer-encoding forces base64; `application/json`, `application/javascript`, `application/xml`, and `text/*` are passed as raw strings (`aws_lambda_filter.cc:81-114`).
- ARN parser splits on `:` and requires at least 7 parts starting with `arn`; a trailing version/alias is folded into `function_name` (`aws_lambda_filter.cc:410-439`).
- Invalid JSON in `dejsonizeResponse` flips the response to `500` and bumps `server_error` (`aws_lambda_filter.cc:371-380`).

## Configuration
- `arn` (required) — invoked Lambda function ARN; region extracted from it for signing.
- `invocation_mode` — `SYNCHRONOUS` or `ASYNCHRONOUS` (`config.cc:10-21`).
- `payload_passthrough` — skip JSON wrapping/unwrapping.
- `host_rewrite` — optional `:authority` override.
- `credentials` / `credentials_profile` — explicit static credentials or profile; otherwise the default SDK provider chain is used (`config.cc:28-57`).
- Per-route `PerRouteConfig.invoke_config` creates a `FilterSettingsImpl` with its own signer (`config.cc:99-127`).

## Stats
Prefix `<stats_prefix>aws_lambda.` (`aws_lambda_filter.cc:441-445`):

- counter `server_error` — JSON parsing of Lambda response failed.
- histogram `upstream_rq_payload_size` (bytes) — size of the signed request body.

## Factory
`AwsLambdaFilterFactory` and `UpstreamAwsLambdaFilterFactory` are both registered (`config.cc:143-145`). The helper in `config.cc:59-89`:

1. Parses the ARN; invalid -> `InvalidArgumentError`.
2. Builds a `CredentialsProviderChain` (config creds, profile, or SDK default).
3. Builds a `SigV4SignerImpl` for service `lambda` in the ARN's region.
4. Wraps everything in `FilterSettingsImpl` and returns a callback that `addStreamFilter`s a new `Filter(settings, stats, is_upstream)` per stream.

`createRouteSpecificFilterConfigTyped` builds the same `FilterSettingsImpl` from `PerRouteConfig.invoke_config` (`config.cc:99-127`).

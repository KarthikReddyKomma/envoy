# Save Processing Response

An `OnProcessingResponse` plugin consumed by the `ext_proc` HTTP filter
(`source/extensions/filters/http/ext_proc`). After the filter receives each
`ProcessingResponse` from the external processor, registered
`OnProcessingResponse` extensions are notified. This extension snapshots
selected responses into stream `FilterState` so that downstream filters,
access logs, or CEL attributes can see what the external processor
returned.

Proto: `envoy.extensions.http.ext_proc.response_processors.save_processing_response.v3.SaveProcessingResponse`.

## Files
- `save_processing_response.h/cc` - callback implementation and
  `SaveProcessingResponseFilterState` object.
- `save_processing_response_factory.h/cc` - factory registration.

## Interface
- Implements
  `Envoy::Extensions::HttpFilters::ExternalProcessing::OnProcessingResponse`
  (one hook per response type: request/response headers, trailers,
  immediate response, streamed immediate response). Request/response body
  hooks are explicit no-ops (`save_processing_response.h:46`).
- `SaveProcessingResponseFilterState` is a `FilterState::Object`
  published under the name
  `envoy.http.ext_proc.response_processors.save_processing_response` (or
  that name + a user-configured suffix).
- Factory implements `OnProcessingResponseFactory`, registered via
  `REGISTER_FACTORY`.

## Logic
- Each hook gets its own `SaveOptions` (from `config.save_request_headers`
  etc.) with two bits: `save_response` (should we save at all) and
  `save_on_error` (also save when the processing call returned
  non-OK). `shouldSaveResponse` combines these with the runtime status.
- When saving, the hook constructs a `ProcessingResponse` containing only
  the relevant oneof field and calls `addToFilterState`.
- `addToFilterState` looks up the filter-state object by
  `filter_state_name_` (base name plus optional user suffix), creating
  it as a mutable object if missing, and writes the
  `Response{processing_status, processing_response}` into it. Subsequent
  saves overwrite the previous `response`.
- Suffix support lets multiple side-by-side ext_proc filters each expose
  their captured data under distinct filter-state keys.

## Key decision points
- `save_processing_response.h:76` - `shouldSaveResponse`: early out when
  `save_response` is false, then gate on processing status unless
  `save_on_error` is set.
- `save_processing_response.cc:11` - filter-state key is the constant
  `kFilterStateName` optionally joined with the configured suffix.
- `save_processing_response.cc:27` - creates a new object on first write
  and registers it as `Mutable` so later hooks can update it.
- `save_processing_response.h:46-50` - body responses are intentionally
  not saved; use the ext_proc body callbacks if you need them.

## Configuration
- `save_request_headers`, `save_response_headers`, `save_request_trailers`,
  `save_response_trailers`, `save_immediate_response` - each a
  `SaveOptions` message carrying `save_response` and `save_on_error`.
- `filter_state_name_suffix` - optional suffix to disambiguate multiple
  instances (appended with a `.`).

## Stats / errors
No stats. Bad processing calls are recorded (when `save_on_error` is set)
via the saved `processing_status`; callers read `Response.processing_status`
from filter state.

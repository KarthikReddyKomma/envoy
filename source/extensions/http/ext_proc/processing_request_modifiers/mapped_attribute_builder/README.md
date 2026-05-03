# Mapped Attribute Builder

A `ProcessingRequestModifier` consumed by the `ext_proc` HTTP filter
(`source/extensions/filters/http/ext_proc`). Before the filter sends a
`ProcessingRequest` to the external processor, registered modifiers get a
chance to add/change fields on that request. This extension populates the
request's CEL-evaluated `attributes` map under the `ext_proc` namespace
using a configured key -> CEL-expression mapping, allowing custom attribute
keys (unlike the filter's built-in fixed attribute set).

Proto: `envoy.extensions.http.ext_proc.processing_request_modifiers.mapped_attribute_builder.v3.MappedAttributeBuilder`.

## Files
- `mapped_attribute_builder.h/cc` - modifier implementation.
- `mapped_attribute_builder_factory.h/cc` - factory registering the
  extension.

## Interface
- Implements
  `Envoy::Extensions::HttpFilters::ExternalProcessing::ProcessingRequestModifier`
  (`modifyRequest` is called by the ext_proc filter before each
  `ProcessingRequest` is sent).
- Factory implements
  `ProcessingRequestModifierFactory`, registered via `REGISTER_FACTORY`.

## Logic
- Configuration carries two `map<string,string>`:
  `mapped_request_attributes` and `mapped_response_attributes`. The key is
  the attribute name that will appear on the wire; the value is a CEL
  expression.
- At construction, the unique set of CEL expression values is extracted and
  fed into an `ExpressionManager` (one for request, one for response)
  through `protoMapValuesToUniqueVector`, so each expression is compiled
  once even if reused across keys.
- `modifyRequest` picks the right map based on
  `params.traffic_direction`. Each direction fires at most once per
  modifier lifetime (tracked by `sent_request_attributes_` /
  `sent_response_attributes_`), matching the built-in attribute builder's
  semantics.
- CEL activation is built from local info, stream info, request headers,
  and (for the response direction) response headers/trailers. Attributes
  are evaluated in bulk, then reshuffled: for each configured
  `output_key -> cel_expr`, the result of `cel_expr` is copied into the
  outgoing `attributes["envoy.filters.http.ext_proc"].fields[output_key]`.
- Returns `true` when it wrote attributes, `false` when nothing to send
  (empty map or already sent).

## Key decision points
- `mapped_attribute_builder.cc:39` - `traffic_direction == INBOUND`
  selects request vs response map.
- `mapped_attribute_builder.cc:46` / `:53` - one-shot behavior per
  direction.
- `mapped_attribute_builder.cc:57` - activation uses
  `dynamic_cast` on `params.response_headers`/`response_trailers` so the
  request-phase call (where these are null) degrades to headers-only
  evaluation.
- `mapped_attribute_builder.cc:68` - output map is cleared and fully
  rewritten each call so stale entries never leak.

## Configuration
- `mapped_request_attributes` - `map<string, string>` of output name to
  CEL expression, evaluated once per request.
- `mapped_response_attributes` - same, evaluated once per response.

## Stats / errors
No dedicated stats. CEL evaluation errors follow the standard
`ExpressionManager` path (failed expressions omit their value from the
output map).

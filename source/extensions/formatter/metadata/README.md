# Metadata Formatter

Adds the `%METADATA(TYPE:namespace:path...)%` substitution command.
Generalised metadata access across request/response lifecycle:
cluster, route, upstream host, listener, listener filter chain, virtual
host, and dynamic metadata.

Proto: `envoy.extensions.formatter.metadata.v3.Metadata` (empty).
Factory names: `envoy.formatter.metadata`,
`envoy.built_in_formatters.metadata`.

## Files
- `metadata.h/cc` - `MetadataFormatterCommandParser` plus four
  metadata-type-specific adapters
  (`RouteMetadataFormatter`, `ListenerMetadataFormatter`,
  `ListenerFilterChainMetadataFormatter`, `VirtualHostMetadataFormatter`)
  that extend `::Envoy::Formatter::MetadataFormatter`.
- `config.h/cc` - `MetadataFormatterFactory` (typed) and
  `BuiltInMetadataFormatterFactory` (built-in) registrations.

## Interface
- Factory bases: `Envoy::Formatter::CommandParserFactory` and
  `BuiltInCommandParserFactory`.
- Formatter base: `Envoy::Formatter::MetadataFormatter` per type.

## Logic
- `parse` recognises the literal command `METADATA`, then uses
  `SubstitutionFormatUtils::parseSubcommand(':', ...)` to split the
  subcommand into `type`, `filter_namespace`, and a `path` vector.
- A static `formatterProviderFuncTable` maps each type name to a
  constructor lambda:
  - `DYNAMIC` -> `::Envoy::Formatter::DynamicMetadataFormatter`.
  - `CLUSTER` -> `::Envoy::Formatter::ClusterMetadataFormatter`.
  - `UPSTREAM_HOST` -> `::Envoy::Formatter::UpstreamHostMetadataFormatter`.
  - `ROUTE` / `LISTENER` / `LISTENER_FILTER_CHAIN` / `VIRTUAL_HOST`
    use the locally defined adapters that know where to pull the
    `envoy::config::core::v3::Metadata` pointer from the `StreamInfo`.
- Each adapter is a trivial `MetadataFormatter` subclass passing a
  lambda `(stream_info) -> const Metadata*` that returns the right
  metadata slot (see `metadata.cc:17` for the four variants). Missing
  slots return `nullptr` which the base formatter turns into
  `absl::nullopt`.

## Key decision points
- `metadata.cc:92` - the lookup table is built with `CONSTRUCT_ON_FIRST_USE`
  so the static hash map is initialized once per process.
- `metadata.cc:150` - unknown metadata type strings throw
  `EnvoyException` at parse time, guaranteeing misconfigurations are
  surfaced immediately.
- `config.cc:13` - the typed factory logs a warning noting the formatter
  is built-in and doesn't need explicit configuration (kept for
  backwards compat with older configs).

## Configuration
- Proto is empty; all inputs come from the subcommand syntax
  `TYPE:namespace:path...`.

## Stats / errors
- None. Missing metadata slots produce empty placeholders in the
  rendered output.

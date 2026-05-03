# Tracer common helpers

Shared helpers used by concrete tracer extensions. This directory does not
register a tracer by itself. It exists so every tracer factory in
`source/extensions/tracers/*` can inherit a small amount of boilerplate
instead of re-implementing the `Server::Configuration::TracerFactory`
plumbing. All tracer extensions depend on it via `envoy_cc_library` in
`BUILD`.

Proto: none. The tracer factories that derive from `FactoryBase<T>` plug in
their own `T = envoy.config.trace.v3.<Message>` or
`envoy.extensions.tracers.<name>.v3.<Message>`.

## Files
- `factory_base.h` - `FactoryBase<ConfigProto>` template that partially
  implements `Server::Configuration::TracerFactory`. Subclasses only provide
  `createTracerDriverTyped(const ConfigProto&, TracerFactoryContext&)`. The
  base handles `createTracerDriver` (downcast + validate the generic
  `Protobuf::Message`), `createEmptyConfigProto` (returns a default instance
  of the typed proto), and `name()` (set via the constructor).
- `BUILD` - declares `tracer_extension_factory_base` and publishes it to the
  tracer extensions packages.

## Tracer role
No tracer is produced here directly. The helper lives one step above the
`Tracing::Driver::startSpan(...)`, `Span::injectContext`, and
`Span::finishSpan` surface: it is the glue that turns a registered factory
into an object that the server configuration machinery can instantiate and
then hands the result to the caller as a `Tracing::DriverSharedPtr`.

## Flow
- Extension's `config.cc` declares a subclass of `FactoryBase<ConfigProto>`
  and registers it via `REGISTER_FACTORY(..., TracerFactory)`.
- Server bootstrap resolves the factory by `name()`, calls
  `createEmptyConfigProto()` to build an empty message, parses the user
  configuration into it, and then calls `createTracerDriver(proto, ctx)`.
- `FactoryBase::createTracerDriver` validates/downcasts the message and
  delegates to the extension's `createTracerDriverTyped`, which constructs
  and returns the concrete `Tracing::Driver`.

## Key decision points
- `factory_base.h:20` - `MessageUtil::downcastAndValidate<const ConfigProto&>`
  runs proto validation before the concrete factory ever sees the config.
- `factory_base.h:37` - pure virtual hook each tracer implements.

## Configuration
No user-visible configuration. Each tracer extension's README documents the
message type that is plugged in as `ConfigProto`.

## Stats / errors
No stats. Validation failures in `downcastAndValidate` surface as
`ProtoValidationException` and abort configuration load.

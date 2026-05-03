# Common Compressor Factory Base

Template base class that every named compressor library (brotli, gzip, zstd,
etc.) derives from. Handles proto down-casting and validation so each
extension only has to implement the typed factory creation.

## Files
- `factory_base.h` - `CompressorLibraryFactoryBase<ConfigProto>`.

## Interface
- Base: `Envoy::Compression::Compressor::NamedCompressorLibraryConfigFactory`.

## Logic
- `createCompressorFactoryFromProto` calls
  `MessageUtil::downcastAndValidate<const ConfigProto&>` with the factory
  context's validation visitor, then forwards the typed message to the
  subclass's `createCompressorFactoryFromProtoTyped` virtual.
- `createEmptyConfigProto` returns a default-constructed `ConfigProto`.
- `name()` returns the string supplied at construction time.

## Key decision points
- `factory_base.h:20` - the downcast-and-validate pattern gives every
  concrete extension uniform proto validation without boilerplate.

## Configuration
- None directly; each subclass defines its own proto.

## Stats / errors
- Validation failures throw `ProtoValidationException` from
  `downcastAndValidate`.

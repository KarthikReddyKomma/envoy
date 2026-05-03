# Common Decompressor Factory Base

Template base class shared by named decompressor library extensions. Mirrors
the compressor factory base: it handles proto down-casting and validation so
concrete extensions only implement the typed factory hook.

## Files
- `factory_base.h` - `DecompressorLibraryFactoryBase<ConfigProto>`.

## Interface
- Base:
  `Envoy::Compression::Decompressor::NamedDecompressorLibraryConfigFactory`.

## Logic
- `createDecompressorFactoryFromProto` calls
  `MessageUtil::downcastAndValidate<const ConfigProto&>` with the context
  validation visitor, then forwards to the subclass's
  `createDecompressorFactoryFromProtoTyped`.
- `createEmptyConfigProto` returns a default-constructed `ConfigProto`.
- `name()` returns the string supplied at construction.

## Key decision points
- `factory_base.h:17` - uniform proto-validation pipeline for all
  decompressor library extensions (brotli, gzip, zstd).

## Configuration
- None directly; each subclass defines its own proto.

## Stats / errors
- Validation failures throw `ProtoValidationException`.

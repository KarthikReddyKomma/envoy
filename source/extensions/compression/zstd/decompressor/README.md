# Zstd Decompressor

Envoy decompressor implementation backed by `libzstd`. Supports multiple
dictionaries concurrently (the manager runs in `replace_mode = false`) so
streams compressed against older dictionaries continue to decompress after
rotations.

Proto: `envoy.extensions.compression.zstd.decompressor.v3.Zstd`.
Factory name: (registered via `DecompressorLibraryFactoryBase`; the
extension name lives in `config.cc`).

## Files
- `zstd_decompressor_impl.h/cc` - `ZstdDecompressorImpl` built on
  `Envoy::Compression::Zstd::Common::Base`, plus the
  `ALL_ZSTD_DECOMPRESSOR_STATS` counters.
- `config.h/cc` - `ZstdDecompressorFactory` /
  `ZstdDecompressorLibraryFactory`.

## Interface
- Base: `Envoy::Compression::Decompressor::Decompressor`.
- Registered via `DecompressorLibraryFactoryBase`.

## Logic
- `decompress` computes `limit = MaxInflateRatio * input_buffer.length()`
  (`MaxInflateRatio = 100`) and iterates the input slices.
- On the first non-empty slice, if a dictionary manager was configured it
  calls `ZSTD_getDictID_fromFrame`. A non-zero id triggers a lookup via
  `ZstdDDictManager::getDictionaryById`; missing dictionaries bump
  `zstd_dictionary_error` and abort. Matching dictionaries are attached to
  the context with `ZSTD_DCtx_refDDict`.
- `process` loops `ZSTD_decompressStream` until the slice is drained,
  flushing to the output buffer each iteration. Bomb protection checks
  `output_buffer.length() > limit` after each processed slice.
- `isError` maps `ZSTD_getErrorCode` values to counters
  (`zstd_memory_error`, `zstd_dictionary_error`,
  `zstd_checksum_wrong_error`, or catch-all `zstd_generic_error`).

## Key decision points
- `zstd_decompressor_impl.cc:55` - bomb protection is still gated by
  `envoy.reloadable_features.enable_compression_bomb_protection`, matching
  the gzip and brotli decoders.
- `zstd_decompressor_impl.cc:33` - dictionary is bound once (first
  non-empty slice) by frame id; subsequent slices reuse the reference.
- `config.cc:14` - factory creates the `ZstdDDictManager` with
  `replace_mode = false` so multiple dictionaries coexist.

## Configuration
- `chunk_size` - output chunk size; defaults to `ZSTD_DStreamOutSize()`.
- `dictionaries` - repeated `DataSource`s; each dictionary is indexed by
  its zstd id.

## Stats / errors
Counters (prefix matches the library stats prefix):
`zstd_generic_error`, `zstd_dictionary_error`, `zstd_checksum_wrong_error`,
`zstd_memory_error`.

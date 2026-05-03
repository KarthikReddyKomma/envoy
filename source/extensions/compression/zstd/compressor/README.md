# Zstd Compressor

Envoy compressor implementation backed by `libzstd`. Thin subclass of the
shared `ZstdCompressorImplBase`; most streaming work happens there. This
directory wires the Envoy factory plumbing and optionally plugs in a
compression dictionary via the shared dictionary manager.

Proto: `envoy.extensions.compression.zstd.compressor.v3.Zstd`.
Factory name: (registered via `CompressorLibraryFactoryBase`; look up the
Zstd extension name in the common factory base).

## Files
- `zstd_compressor_impl.h/cc` - `ZstdCompressorImpl` and the
  `ZstdCDictManager` / `ZstdCDictManagerPtr` typedefs around the
  templated `Common::DictionaryManager`.
- `config.h/cc` - `ZstdCompressorFactory` and
  `ZstdCompressorLibraryFactory`.

## Interface
- Base: `Envoy::Compression::Zstd::Compressor::ZstdCompressorImplBase`
  (which itself implements `Envoy::Compression::Compressor::Compressor`).
- Registered via `CompressorLibraryFactoryBase` as a
  `NamedCompressorLibraryConfigFactory`.

## Logic
- Construction: if a dictionary is configured, the dict manager exposes one
  pre-built `ZSTD_CDict` and the implementation calls `ZSTD_CCtx_refCDict`.
  Otherwise, `ZSTD_c_compressionLevel` is set directly on the context.
  Result codes are checked with `ZSTD_isError`.
- `compressProcess` hands each input slice to the base class via
  `setInput` + `process(ZSTD_e_continue)`; `compressPreprocess` and
  `compressPostprocess` are no-ops because Zstd framing needs no extra
  pre/post work.
- Factory eagerly constructs the `ZstdCDictManager` (with
  `replace_mode = true`) if `zstd.has_dictionary()`. The dict manager runs
  on the server's main thread dispatcher and publishes to all workers.

## Key decision points
- `zstd_compressor_impl.cc:15` - dictionary and compression level are
  mutually exclusive paths; a dictionary already encodes the level at
  build time so the explicit `ZSTD_c_compressionLevel` setter is skipped.
- `config.cc:19` - `replace_mode = true` means rotating a dictionary file
  removes the old id from the worker map; compression always uses the
  latest key.

## Configuration
- `compression_level` - defaults to `ZSTD_CLEVEL_DEFAULT`.
- `enable_checksum` - enables Zstd's content checksum trailer.
- `strategy` - passed straight through to Zstd's strategy parameter.
- `chunk_size` - output chunk size; defaults to `ZSTD_CStreamOutSize()`.
- `dictionary` - optional single `DataSource`.

## Stats / errors
- No stats emitted from the compressor itself. Library errors surface as
  `ZSTD_isError`-triggered `RELEASE_ASSERT`s.

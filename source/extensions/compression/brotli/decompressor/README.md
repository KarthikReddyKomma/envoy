# Brotli Decompressor

Envoy decompressor implementation backed by the `brotli` decode library. Used
by the HTTP decompressor filter.

Proto: `envoy.extensions.compression.brotli.decompressor.v3.Brotli`.
Factory name: `envoy.compression.brotli.decompressor`.

## Files
- `brotli_decompressor_impl.h/cc` - `BrotliDecompressorImpl` owning a
  `BrotliDecoderState` and emitting counters through `BrotliDecompressorStats`.
- `config.h/cc` - `BrotliDecompressorFactory` /
  `BrotliDecompressorLibraryFactory` registration.

## Interface
- Base: `Envoy::Compression::Decompressor::Decompressor`.
- Registered via `DecompressorLibraryFactoryBase` as a
  `NamedDecompressorLibraryConfigFactory`.

## Logic
- Constructor creates `BrotliDecoderState` and sets
  `BROTLI_DECODER_PARAM_DISABLE_RING_BUFFER_REALLOCATION` from config.
- `decompress()` builds a `Common::BrotliContext` where
  `max_output_size = MaxInflateRatio * input_buffer.length()`
  (`MaxInflateRatio = 100`), iterates input slices feeding
  `BrotliDecoderDecompressStream`, and drains residual output after the input
  is exhausted.
- `process()` switches on `BrotliDecoderResult`:
  - `SUCCESS` with leftover input bumps `brotli_error` /
    `brotli_redundant_input` and bails out to avoid endless loops.
  - `NEEDS_MORE_INPUT` / `NEEDS_MORE_OUTPUT` check the bomb-protection limit
    before flushing the chunk.
  - `ERROR` bumps `brotli_error` and aborts.

## Key decision points
- `brotli_decompressor_impl.cc:19` fixes the compression-bomb inflate ratio
  at 100.
- `brotli_decompressor_impl.cc:86` gates bomb protection behind runtime
  feature `envoy.reloadable_features.enable_compression_bomb_protection`.
- `brotli_decompressor_impl.cc:71` treats trailing input after `SUCCESS` as
  an error (prevents infinite reprocessing of already-finished streams).

## Configuration
- `chunk_size` - output chunk buffer size (default `4096`).
- `disable_ring_buffer_reallocation` - skips Brotli's adaptive ring buffer
  sizing (allocates to the full window regardless of content length).

## Stats / errors
Counters (prefix `brotli.`):
- `brotli_error` - any decompression failure.
- `brotli_output_overflow` - output exceeded inflate-ratio limit.
- `brotli_redundant_input` - input left over after stream `SUCCESS`.

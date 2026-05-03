# Gzip Compressor

Gzip-format compressor implementation backed by zlib `deflate`. Wraps
`ZlibCompressorImpl` (deriving from `Common::Base`) and registers a named
compressor library.

Proto: `envoy.extensions.compression.gzip.compressor.v3.Gzip`.
Factory name: `envoy.compression.gzip.compressor`.

## Files
- `zlib_compressor_impl.h/cc` - `ZlibCompressorImpl` with `CompressionLevel`,
  `CompressionStrategy` enums and `init`/`compress` entrypoints.
- `config.h/cc` - `GzipCompressorFactory` and
  `GzipCompressorLibraryFactory`. Defaults live in `config.h` /
  `config.cc` (`DefaultWindowBits = 12`, `DefaultMemoryLevel = 5`,
  `DefaultChunkSize = 4096`, `GzipHeaderValue = 16`).

## Interface
- Base: `Envoy::Compression::Compressor::Compressor`.
- Registered via `CompressorLibraryFactoryBase` as a
  `NamedCompressorLibraryConfigFactory`. `contentEncoding()` returns `gzip`.

## Logic
- Constructor initialises the `z_stream` (`zalloc/zfree/opaque = Z_NULL`)
  and points `next_out`/`avail_out` at the shared chunk.
- `init()` runs `deflateInit2` with the mapped level, strategy, window bits
  (the factory ORs `GzipHeaderValue = 16` to select gzip framing rather than
  raw deflate) and memory level.
- `compress()` walks the input's raw slices: feeds bytes to `deflate` with
  `Z_NO_FLUSH`, flushes any filled chunks into the buffer, then drains the
  slice. After all slices are done it calls `deflate` with `Z_FINISH` (on
  `State::Finish`) or `Z_SYNC_FLUSH` to emit remaining bytes.
- `deflateNext` returns false on `Z_STREAM_END` and on `Z_BUF_ERROR` with no
  pending input (zlib's signal for "needs more input").

## Key decision points
- `zlib_compressor_impl.cc:55` - `process(buffer, Z_NO_FLUSH)` writes to the
  tail of the same buffer that is being drained; the subsequent
  `buffer.drain(input_slice.len_)` trims the consumed input so read/write
  cursors don't overlap.
- `config.cc:14` - `window_bits` is always ORed with `GzipHeaderValue (16)`
  so the output carries the gzip header/trailer.

## Configuration
- `compression_level` - mapped to zlib levels 1-9 / best / speed /
  `Z_DEFAULT_COMPRESSION` (`config.cc:18`).
- `compression_strategy` - mapped to `Z_FILTERED`, `Z_FIXED`,
  `Z_HUFFMAN_ONLY`, `Z_RLE`, or `Z_DEFAULT_STRATEGY` (`config.cc:45`).
- `memory_level`, `window_bits`, `chunk_size` with defaults above.

## Stats / errors
- No stats. Invalid zlib results trip `RELEASE_ASSERT` in `deflateNext`.

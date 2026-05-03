# Gzip Decompressor

Gzip-format decompressor implementation backed by zlib `inflate`. Wraps
`ZlibDecompressorImpl` (deriving from `Common::Base`) and registers a named
decompressor library.

Proto: `envoy.extensions.compression.gzip.decompressor.v3.Gzip`.
Factory name: `envoy.compression.gzip.decompressor`.

## Files
- `zlib_decompressor_impl.h/cc` - `ZlibDecompressorImpl` with
  `ALL_ZLIB_DECOMPRESSOR_STATS` counters and a `decompression_error_`
  field for callers that want the raw zlib result code.
- `config.h/cc` - `GzipDecompressorFactory` /
  `GzipDecompressorLibraryFactory` with defaults
  (`DefaultWindowBits = 15`, `DefaultChunkSize = 4096`,
  `DefaultMaxInflateRatio = 100`) and the `GzipHeaderValue = 16` flag
  that tells zlib to decode the gzip frame.

## Interface
- Base: `Envoy::Compression::Decompressor::Decompressor`.
- Registered via `DecompressorLibraryFactoryBase` as a
  `NamedDecompressorLibraryConfigFactory`.

## Logic
- `init(window_bits)` runs `inflateInit2` (`window_bits | 16` selects
  gzip).
- `decompress()` computes `limit = max_inflate_ratio * input_buffer.length()`
  and walks input slices. For each slice, `inflateNext` runs `inflate`
  with `Z_NO_FLUSH` in a loop; when `avail_out == 0` the chunk is flushed,
  and the bomb-protection runtime guard checks `output_buffer.length() >
  limit` after each iteration.
- On `Z_STREAM_END` the code issues a second `inflate(Z_FINISH)` so zlib
  releases its sliding window (memory optimisation documented in
  `zlib.net/manual.html`).

## Key decision points
- `zlib_decompressor_impl.cc:59` - bomb protection is gated by
  `envoy.reloadable_features.enable_compression_bomb_protection`; when
  tripped, the decoder bumps `zlib_data_error` and returns without
  processing the rest of the input.
- `zlib_decompressor_impl.cc:86` - `Z_BUF_ERROR` with `avail_in == 0` is
  zlib's "needs more input" signal and is treated as a normal exit.
- `zlib_decompressor_impl.cc:104` - `chargeErrorStats` maps each negative
  result to a distinct counter for observability.

## Configuration
- `window_bits` (default 15, ORed with 16 for gzip framing).
- `chunk_size` - output chunk buffer size (default 4096).
- `max_inflate_ratio` - output/input size cap for bomb protection
  (default 100).

## Stats / errors
Counters (prefix `gzip.`):
`zlib_errno`, `zlib_stream_error`, `zlib_data_error`, `zlib_mem_error`,
`zlib_buf_error`, `zlib_version_error`.

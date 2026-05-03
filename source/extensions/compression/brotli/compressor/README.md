# Brotli Compressor

Envoy compressor implementation backed by the `brotli` encode library. Used by
the HTTP compressor filter and anywhere a named compressor library is
requested.

Proto: `envoy.extensions.compression.brotli.compressor.v3.Brotli`.
Factory name: `envoy.compression.brotli.compressor`.

## Files
- `brotli_compressor_impl.h/cc` - `BrotliCompressorImpl` owning a
  `BrotliEncoderState` and a shared `Common::BrotliContext`.
- `config.h/cc` - `BrotliCompressorFactory` /
  `BrotliCompressorLibraryFactory` registration and default constants.

## Interface
- Base: `Envoy::Compression::Compressor::Compressor`.
- Registered via `CompressorLibraryFactoryBase` as a
  `NamedCompressorLibraryConfigFactory`.
- Reports `contentEncoding()` as `br` via
  `Http::CustomHeaders::ContentEncodingValues.Brotli`.

## Logic
- Constructor sets five encoder parameters on the Brotli state: `QUALITY`,
  `LGWIN` (window bits), `LGBLOCK` (input block bits),
  `DISABLE_LITERAL_CONTEXT_MODELING`, and `MODE`. All results are checked
  with `RELEASE_ASSERT`.
- `compress()` walks the input `Buffer::Instance` slice by slice, feeding
  each slice into `BrotliEncoderCompressStream` with
  `BROTLI_OPERATION_PROCESS`. Encoded chunks are appended to an
  accumulation buffer.
- After the input is drained, the code re-enters the encoder with
  `BROTLI_OPERATION_FINISH` (on `State::Finish`) or
  `BROTLI_OPERATION_FLUSH` otherwise, looping while
  `BrotliEncoderHasMoreOutput` is true, then calls
  `ctx.finalizeOutput` to flush the partial chunk.

## Key decision points
- `brotli_compressor_impl.cc:49` drives the encoder one chunk at a time to
  keep output memory bounded.
- `brotli_compressor_impl.cc:64` handles the finish/flush tail; the comment
  explains why the loop is required (`ISLAST` / `ISLASTEMPTY` headers or
  internal buffering can produce output after input is exhausted).
- `config.cc:25` maps the proto `EncoderMode` enum to the internal
  `BrotliCompressorImpl::EncoderMode` (`Generic`, `Text`, `Font`,
  `Default`).

## Configuration
Defaults (from `config.h`): `quality = 3`, `window_bits = 18`,
`input_block_bits = 24`, `chunk_size = 4096`.
`disable_literal_context_modeling` is a decoding-speed vs. ratio trade-off.
`encoder_mode` tunes the encoder for the expected input shape.

## Stats / errors
- No stats. Library-level failures trigger `RELEASE_ASSERT` ("unable to
  compress") since Brotli encode errors are not recoverable.

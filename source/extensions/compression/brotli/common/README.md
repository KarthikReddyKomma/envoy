# Brotli Common

Shared streaming context used by the Brotli compressor and decompressor
implementations. It owns the output chunk buffer and tracks the encoder/decoder
stream cursors so producers only need to drive the Brotli library and flush
full chunks into an Envoy `Buffer::Instance`.

## Files
- `base.h/cc` - defines `BrotliContext` with `chunk_size_`, `next_in_`,
  `next_out_`, `avail_in_`, `avail_out_`, and helpers `updateOutput` /
  `finalizeOutput`.

## Interface
- Internal helper (not a registered extension). Consumed by
  `BrotliCompressorImpl` and `BrotliDecompressorImpl`.

## Logic
- Constructor allocates a single `chunk_size_` byte buffer
  (`std::unique_ptr<uint8_t[]>`) and points `next_out_` at the start with
  `avail_out_ = chunk_size`.
- `updateOutput` copies the full chunk into the supplied `Buffer::Instance`
  when `avail_out_ == 0`, then calls `resetOut` to reuse the buffer for the
  next block.
- `finalizeOutput` flushes any residual bytes (`chunk_size_ - avail_out_`)
  once the stream is done.

## Key decision points
- `base.cc:14` only emits to `Buffer::Instance` when the chunk is exactly
  full; partial writes are deferred to `finalizeOutput` to avoid
  per-call buffer churn.
- `base.cc:28` resets the cursor back to the chunk start for reuse across
  consecutive stream operations.
- The optional `max_output_size_` on the context is used by the
  decompressor to detect compression bombs (enforced in
  `brotli_decompressor_impl.cc`, not here).

## Configuration
- None. The chunk size is passed through by the owning compressor or
  decompressor factory.

## Stats / errors
- None emitted directly from the context; errors surface in the caller
  (e.g., brotli decompressor bumps `brotli_error` / `brotli_output_overflow`).

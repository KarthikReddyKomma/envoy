# Gzip Common

Shared state and helpers used by the gzip compressor (deflate) and
decompressor (inflate). Wraps the zlib `z_stream` so both directions share
a single chunk allocator and output-flush helper.

## Files
- `base.h/cc` - defines `Common::Base`:
  - owns the output chunk (`unsigned char[]`) and the `z_stream`.
  - `updateOutput()` copies ready bytes into an Envoy
    `Buffer::Instance` and rearms `next_out`/`avail_out`.
  - `checksum()` exposes `zstream_ptr_->adler` for tests/integration.

## Interface
- Internal helper (not a registered extension). Used as a base class by
  `ZlibCompressorImpl` and `ZlibDecompressorImpl`.

## Logic
- Constructor receives a `chunk_size` and a custom `z_stream` deleter; the
  subclass supplies the right `deflateEnd` / `inflateEnd` call, then
  `delete z`.
- `updateOutput()` only emits bytes when the zstream actually produced output
  (`n_output > 0`), avoiding empty writes that would confuse the surrounding
  buffer accounting.
- `initialized_` tracks whether `deflateInit2` / `inflateInit2` has run so
  subclasses can guard against double-init.

## Key decision points
- `base.cc:15` - skip the `output_buffer.add` when no bytes were produced
  during the zlib call (avoids allocator churn).
- `base.cc:21` - reset `avail_out` / `next_out` back to the chunk start
  after every flush so the same buffer is reused across calls.

## Configuration
- None directly; chunk size is passed in by the owning compressor or
  decompressor factory.

## Stats / errors
- None emitted here; error counters live in the decompressor subclass.

# Envoy Buffer Logic — Case-by-Case Explanation

This document explains the **logic** of the buffer subsystem in `source/common/buffer/` as a series of bullet-driven cases. It complements `BUFFER_ARCHITECTURE.md` (which covers structure / diagrams) by walking through *what the code actually decides at each branch*.

Files covered:

- `buffer_impl.h` / `buffer_impl.cc` — `Slice`, `SliceDeque`, `OwnedImpl`, fragments
- `watermark_buffer.h` / `watermark_buffer.cc` — `WatermarkBuffer`, `BufferMemoryAccountImpl`, `WatermarkBufferFactory`
- `buffer_util.h` — `Util::serializeDouble`
- `zero_copy_input_stream_impl.h` / `.cc` — Protobuf zero-copy adapter

---

## 1. `Slice` — the contiguous memory block

A `Slice` is the unit of storage. It splits its memory into 3 sections via two offsets `data_` and `reservable_`:

```
| 0 .. data_ | data_ .. reservable_ | reservable_ .. capacity_ |
|  Drained   |        Data          |       Reservable         |
```

### 1.1 Constructors — three creation modes

- **Empty / default** (`Slice()`): `capacity_ = 0`, `base_ = nullptr`. Used as a "moved-from" placeholder.
- **Owned mutable** (`Slice(min_capacity, account)`):
  - `capacity_ = sliceSize(min_capacity)` (rounded up to the next 4 KB page).
  - Allocates `new uint8_t[capacity_]` via `storage_`.
  - If `account` is non-null, calls `account->charge(capacity_)` to track memory.
- **Pre-allocated owned** (`Slice(SizedStorage, used_size, account)`):
  - Reuses an externally allocated buffer (e.g., from the thread-local free list).
  - `reservable_ = used_size` so the front `used_size` bytes are already "data".
- **Immutable fragment** (`Slice(BufferFragment&)`):
  - `storage_ = nullptr`, `base_` points to caller-owned memory.
  - `reservable_ = fragment.size()` so the entire range is "data" with **zero reservable space**.
  - Stores a `releasor_` lambda that calls `fragment.done()` on destruction.

### 1.2 `isMutable()` vs `canCoalesce()`

- Both return `storage_ != nullptr`.
- An immutable slice (fragment) is rejected from coalescing because:
  - Its destructor invokes a user `releasor_` (arbitrary side effect).
  - Copying its bytes into another slice would defer or skip the releasor.

### 1.3 `drain(size)` — pop from the front

- `data_ += size` (no memory move; pointer math).
- **Special case**: if `data_ == reservable_` (slice now empty), reset both to 0 → reuses the slice's full capacity for new writes.

### 1.4 `reserve(size)` — get a write window

- Returns `{nullptr, 0}` if `size == 0` or `available_size == 0`.
- Caps the reservation at `available_size = capacity_ - reservable_`.
- Returns a pointer to `base_ + reservable_`. **Does not** advance `reservable_` — the caller must `commit()`.

### 1.5 `commit<SafeCommit>(reservation)` — finalize a reservation

- Two modes via the `SafeCommit` template parameter:
  - `SafeCommit = true` (default): validates that `reservation.mem_` matches the expected position and that bytes fit within `capacity_`. Returns `false` on mismatch.
  - `SafeCommit = false`: only `ASSERT`s — used in performance-critical paths where the caller guarantees correctness (e.g., `addFragments`).
- Advances `reservable_` by `reservation.len_`.

### 1.6 `append(data, size)` — copy and commit in one call

- Computes `copy_size = min(size, reservableSize())`.
- Skips if `copy_size == 0` (slice is full).
- Otherwise `memcpy` and bump `reservable_`.

### 1.7 `prepend(data, size)` — write in front of existing data

Two cases inside `Slice::prepend`:

- **Slice is empty** (`dataSize() == 0`):
  - Place data at the *very back* of the slice (`reservable_ = capacity_`, `data_ = capacity_ - copy_size`).
  - This leaves maximum space at the front for future prepends.
- **Slice has data**:
  - If `data_ == 0` (no front room), return 0 — caller must allocate a new slice.
  - Else copy at most `data_` bytes into `[base_, base_+data_)` and decrement `data_`.

### 1.8 Drain trackers and accounts

- `addDrainTracker(fn)` queues a function to run when the slice is destroyed or coalesced away.
- `callAndClearDrainTrackersAndCharges()`:
  - Invokes every queued tracker in FIFO order.
  - If an account is attached, calls `account_->credit(capacity_)` and resets it.
- `transferDrainTrackersTo(dest)` moves trackers without invoking them — used during slice coalescing.
- `maybeChargeAccount(account)`: only charges if **all** of: not already charged, slice owns memory, and account is non-null.

---

## 2. `SliceDeque` — custom ring buffer of `Slice`

Replaces `std::deque<Slice>` because benchmarks showed std::deque was too slow.

- **Inline storage** holds the first 8 slices (`InlineRingCapacity = 8`) without any heap allocation.
- **External growth**: when `size_ == capacity_`, `growRing()` doubles capacity by allocating `new Slice[2 * capacity_]` and copying entries from `start_` linearly to `[0..size_)`.
- `start_` + `size_` form a circular range; `internalIndex(i) = (start_ + i) % capacity_`.
- `pop_front()` does not shrink the ring; it just default-constructs the slot and advances `start_`.
- A custom move constructor is required because `ring_` may point to either `inline_ring_` or `external_ring_`, and that pointer must be re-derived after moving.

---

## 3. `OwnedImpl` — the main `Buffer::Instance`

Holds a `SliceDeque slices_`, a precomputed `length_` (an `OverflowDetectingUInt64`), and an optional account.

### 3.1 `add(data, size)` / `addImpl`

Logic loop:

1. Start with `new_slice_needed = slices_.empty()`.
2. While `size != 0`:
   - If `new_slice_needed`, push a new `Slice(size, account_)` to the back.
   - Try `slices_.back().append(src, size)` — copies up to `reservableSize()` bytes.
   - Subtract `copy_size`, advance `src`, accumulate into `length_`.
   - Set `new_slice_needed = true` for the next iteration (forces a fresh slice if append couldn't take all bytes).

**Cases**:
- Empty buffer + small write → 1 new slice, fits fully.
- Existing back slice has room → fill the tail, no new slice.
- Existing back slice full + remaining bytes → 1 extra slice, repeat until consumed.

### 3.2 `addBufferFragment(fragment)` — true zero-copy

- `length_ += fragment.size()` and `slices_.emplace_back(fragment)`.
- No `memcpy`. The slice is immutable and runs `fragment.done()` on destruction.

### 3.3 `addFragments(fragments)` — multiple `string_view`s in one call

Two distinct strategies based on whether one reservation can hold *all* the fragments:

- **Single reservation fits all** (`reservation.len_ == total_size_to_copy`):
  - One `memcpy` per fragment into the same back slice.
  - Single `commit<false>(reservation)` with the unsafe (assert-only) variant.
- **Reservation only partial**:
  1. Greedily copy as many *complete* fragments as fit into the back slice.
  2. Commit only the bytes actually written.
  3. Sum remaining fragment sizes, allocate ONE new slice exactly that big, copy the rest there.
  - This avoids creating one slice per fragment.

### 3.4 `prepend(string_view)` — write at the front

Symmetric to `add()` but using `slices_.emplace_front()` and `slices_.front().prepend(...)`.

### 3.5 `prepend(Instance& other)` — splice another buffer at the front

- Iterates `other.slices_` from back to front (so original order is preserved at the front of `this`).
- `emplace_front(std::move(other.slices_.back()))`.
- Calls `maybeChargeAccount(account_)` on each moved slice (re-tags ownership).
- Calls `other.postProcess()` at the end (lets watermark check fire).

### 3.6 `move(rhs)` — drain rhs into this (zero-copy)

For each slice in `rhs.slices_` (front-to-back):

- Calls `coalesceOrAddSlice(std::move(slice))`.
- Decrements `rhs.length_`, pops from rhs.

Finally invokes `rhs.postProcess()`.

#### `coalesceOrAddSlice` decision tree

Coalesces (copies into existing back slice) **only if all four** are true:

1. `other_slice.canCoalesce()` — the source is mutable (not a fragment).
2. `!slices_.empty()` — there is a back slice to copy into.
3. `slice_size < CopyThreshold` (= 512 bytes) — small enough that copying is faster than fragmentation.
4. `slices_.back().reservableSize() >= slice_size` — there is room without spilling.

Otherwise → take ownership: `maybeChargeAccount`, `slices_.emplace_back(std::move(other_slice))`, `length_ += slice_size`.

### 3.7 `move(rhs, length)` — partial move

For each front slice of `rhs`, three cases:

- `copy_size == 0` (zero-byte slice) → just pop it.
- `copy_size < slice_size` (only part of the front slice fits) → call `add()` to copy bytes, then `slices_.front().drain(copy_size)`. **Always copies; partial moves cannot share storage today.**
- `copy_size == slice_size` (whole slice transfers) → optionally call `callAndClearDrainTrackersAndCharges()` if `reset_drain_trackers_and_accounting` is true (used when the source is a user-space IOHandle whose connection might die), then `coalesceOrAddSlice`.

### 3.8 `drain(size)` / `drainImpl`

- Loop while `size != 0` and `!slices_.empty()`:
  - If front slice fits inside `size` → `pop_front()`, subtract its full data size.
  - Else → `slices_.front().drain(size)`, set `size = 0`.
- After the loop, drop any zero-byte slices that were left at the front (sentinels for flushed data).

### 3.9 `linearize(size)` — force the first `size` bytes contiguous

Two cases:

- `slices_[0].dataSize() >= size` → already contiguous, return `slices_.front().data()`.
- Otherwise:
  1. Allocate a fresh `Slice(size, account_)`, reserve its full capacity, `copyOut(0, size, …)` into it, commit.
  2. `drainImpl(size)` to remove the original head bytes (uses `drainImpl` to bypass watermark-low callbacks).
  3. `slices_.emplace_front(std::move(new_slice))`, restore `length_ += size`.

### 3.10 `reserveForRead()` / `reserveWithMaxLength(max_length)` — multi-slice read window

Algorithm:

- Trim trailing empty slices.
- **Tail reuse**: if the back slice's `reservableSize >= max_length` *or* `>= default_slice_size_/8` (= 2 KB), reserve from it first.
- Then loop allocating fresh `default_slice_size_` (16 KB) slices via `slices_owner->newStorage()` (which may pop from the thread-local free list `free_list_`).
- Stop when one of:
  - `bytes_remaining == 0`.
  - `reservation_slices.size() == MAX_SLICES_`.
  - "Overshoot guard": next slice would exceed `bytes_remaining` AND we already reserved at least one full slice.
- Returns a `Reservation` that owns the underlying storages until `commit()` or destruction.

### 3.11 `reserveSingleSlice(length, separate_slice)` — guaranteed single contiguous range

Two cases:

- If `!separate_slice` and the back slice has `reservableSize >= length` → reserve from the back slice.
- Else → allocate a brand-new slice (rounded up to a 4 KB page) owned by the reservation; this slice is *not* in `slices_` yet — it joins on `commit()`.

### 3.12 `commit(length, slices, owner)` — finalize a multi-slice reservation

For each `(raw_slice, owned_storage)` pair:

- Cap `slices[i].len_` at `bytes_remaining` (in case the caller wrote less than reserved).
- If `owned_storage.mem_ != nullptr` → this was a *new* slice; build a `Slice(std::move(owned_storage), len, account_)` and `emplace_back`.
- Else → this was the *tail-reuse* case; `slices_.back().commit<false>(slices[i])` (unsafe variant — assertions only).
- `length_ += slices[i].len_`.

### 3.13 `OverflowDetectingUInt64`

- Wraps a `uint64_t` for `length_`.
- `+=` `RELEASE_ASSERT`s no overflow occurred.
- `-=` `RELEASE_ASSERT`s no underflow occurred.
- Catches accounting bugs immediately rather than silently corrupting buffer state.

### 3.14 `search(needle, size, start, length)` — naive O(M·N) substring search

- Special case: `size == 0` returns `start` if `start <= length_`, else -1 (matches evbuffer semantics).
- Skips slices that lie entirely before `start`, accumulating `offset`.
- Within each slice uses `memchr` for the **first byte** of the needle.
- On a first-byte hit, walks the remaining `size-1` needle bytes, possibly *crossing slice boundaries*. If mismatch, restores `left_to_search` and continues from the next byte.
- Returns the absolute byte offset on full match, or -1.

### 3.15 `startsWith(prefix)` — fast path for known prefix

- Returns `false` immediately if `length() < prefix.length()`.
- Returns `true` for empty prefix.
- Walks slices: if `slice_size >= remaining size`, do one `memcmp` for the rest. Else `memcmp` the slice fully and continue with `prefix += slice_size`.
- Falls through to `IS_ENVOY_BUG` ("unexpected data in slices") only if `length()` lied about the data — defensive only.

### 3.16 `extractMutableFrontSlice()` — hand the front slice to the caller

- Drops empty front slices (so the result is guaranteed non-empty).
- If the front slice is **immutable** (fragment) → allocate a mutable copy and append the data; the original's destructor will invoke its releasor.
- If **mutable** → call `callAndClearDrainTrackersAndCharges()` first (so any side effects fire while we still own context), then move the slice out.
- Returns a `SliceDataImpl` that exposes `getMutableData()`.

### 3.17 Reservation-slice owner free lists

`OwnedImplReservationSlicesOwnerMultiple` keeps a **thread-local** free list (`free_list_`, capped at `MAX_SLICES_` = ~16) of default-sized 16 KB storage chunks:

- `newStorage()` → pop from free list if available, else `new uint8_t[default_slice_size_]`.
- Destructor → push back into the free list, capped to avoid unbounded growth.

This is why in steady-state, multi-slice reads barely allocate. The single-slice owner does **not** use the free list (the comment cites thread-local resolving overhead).

---

## 4. `WatermarkBuffer` — flow control wrapper

Subclass of `OwnedImpl` that fires callbacks when buffer size crosses thresholds.

### 4.1 Three callbacks

- `above_high_watermark_` — backpressure begins.
- `below_low_watermark_` — backpressure ends.
- `above_overflow_watermark_` — emergency / disconnect.

### 4.2 `setWatermarks(high, overflow_multiplier)`

- **Overflow guard**: if `high * overflow_multiplier` would overflow `uint64_t`, set `overflow_multiplier = 0` to disable overflow.
- `low_watermark_ = high / 2` (hardcoded hysteresis to avoid oscillation).
- Recheck both watermarks immediately so a re-config below the current size fires callbacks now.

### 4.3 `checkHighAndOverflowWatermarks()`

Conditions for `above_high_watermark_()`:
- `high_watermark_ != 0`.
- `length() > high_watermark_`.
- `!above_high_watermark_called_` (latch — fires **once per crossing**).

Then conditions for `above_overflow_watermark_()`:
- `overflow_watermark_ != 0`.
- `!above_overflow_watermark_called_` (latched **forever** until destruction).
- `length() > overflow_watermark_`.

### 4.4 `checkLowWatermark()`

- Returns early unless `above_high_watermark_called_ == true`. (No point firing low if we never went high.)
- Fires `below_low_watermark_()` only when `length() <= low_watermark_` *or* `high_watermark_ == 0` (watermarks disabled).
- Resets `above_high_watermark_called_ = false`. Note: `above_overflow_watermark_called_` is **not** reset — overflow is one-shot.

### 4.5 Override list — every size-changing op re-checks watermarks

- Growth ops: `add`, `prepend`, `commit`, `move`, `appendSliceForTest`, `addFragments` → call `checkHighAndOverflowWatermarks()` after parent.
- Shrink ops: `drain`, `extractMutableFrontSlice` → call `checkLowWatermark()`.
- `postProcess()` (called from `move()` on the source side) → calls `checkLowWatermark()` so the source can drop below low watermark.

### 4.6 `reserveForRead()` — adaptive reservation size

Avoids reserving way past the high watermark:

- If `current_length >= high_watermark_` → cap reservation at one `default_slice_size_` (16 KB) to still satisfy "at least some data".
- Else → `available_length = high - current`, round up to a multiple of 16 KB, then cap at `default_read_reservation_size_`.

This lets reads naturally honor the watermark without overshoot.

---

## 5. `BufferMemoryAccountImpl` + `WatermarkBufferFactory` — overload tracking

### 5.1 Account creation

- Private constructor → forced through `BufferMemoryAccountImpl::createAccount(factory, reset_handler)`.
- Stores its own `shared_ptr` in `shared_this_` (chosen over `enable_shared_from_this` to skip atomics on hot path).
- `WatermarkBufferFactory::createAccount` returns `nullptr` if tracking is effectively disabled (`bitshift_ == 63`).

### 5.2 `charge(amount)` / `credit(amount)`

- `charge` ASSERTs no overflow (using `numeric_limits<uint64_t>::max() - balance >= amount`).
- `credit` ASSERTs no underflow.
- After each, calls `updateAccountClass()` to potentially re-bucket.

### 5.3 `balanceToClassIndex()` — bucket selection

- `shifted_balance = balance >> bitshift_` (configurable via `minimum_account_to_track_power_of_two`).
- If `shifted_balance == 0` → not worth tracking → return `nullopt`.
- Else `class_idx = bit_width(shifted_balance) - 1`, clipped to `[0, NUM_MEMORY_CLASSES_ - 1]` (= 8 buckets).
- Buckets are power-of-2 ranges, e.g., with default 1 MB threshold: bucket 0 = [1MB, 2MB), bucket 1 = [2MB, 4MB), …, bucket 7 = [128MB, ∞).

### 5.4 `updateAccountClass()` — move between buckets

- Compares new class to `current_bucket_idx_`. If unchanged, do nothing.
- Calls `factory_->updateAccountClass(shared_this_, current_bucket_idx_, new_class)`.

### 5.5 `WatermarkBufferFactory::updateAccountClass` — the three transitions

- `current = nullopt, new = X` → start tracking: `insert` into bucket X.
- `current = X, new = nullopt` → stop tracking: `erase` from bucket X.
- `current = X, new = Y` → move: `extract` from X, `insert` into Y. Uses `extract`+`insert` to avoid double-allocation.

### 5.6 `clearDownstream()`

- Idempotent.
- Resets `reset_handler_`, unregisters from factory, releases `shared_this_` (so destruction can finally run).

### 5.7 `resetAccountsGivenPressure(pressure)` — overload response

- ASSERT pressure ∈ [0, 1].
- `buckets_to_clear = floor(pressure * 8) + 1`, capped at 8.
- Walks buckets from **highest** index downward (largest streams first).
- Per bucket, while items remain and `num_streams_reset < kMaxNumberOfStreamsToResetPerInvocation` (= 50):
  - Calls `(*it)->resetDownstream()` which triggers `Http::StreamResetHandler::resetStream(OverloadManager)` — the iterator is invalidated by the resulting `unregisterAccount()`, hence the `next = std::next(it)` dance before the call.
- Reset cap (50) prevents this single invocation from triggering the watchdog.
- Logs at `warn` only if at least one bucket had work.

---

## 6. `ZeroCopyInputStreamImpl` — Protobuf adapter

Adapts a `Buffer::Instance` to `google::protobuf::io::ZeroCopyInputStream`.

### 6.1 `Next(data, size)`

- Returns `true` with valid `(data, size)` for the front non-empty slice.
- Internally calls `frontSlice()` and increments `byte_count_` / `position_`.
- **Unfinished stream warning**: if `finished_ == false` and there is no data, returns `true` but with `size = 0` indefinitely. Callers must wrap with `LimitingInputStream` or call `finish()` to avoid spin loops.

### 6.2 `BackUp(count)`

- Pushes the last `count` bytes back so `Next()` re-returns them.
- Adjusts `position_` and `byte_count_`.
- Limited to the current slice's size.

### 6.3 `Skip(count)`

- Drains `count` bytes from the buffer; returns `true` if `count` bytes were available.

### 6.4 `move(instance)` — append more buffered data

- Disallowed if `finished_` (would corrupt parser state).
- Calls `buffer_->move(instance)` which uses zero-copy semantics under the hood.

---

## 7. `Util::serializeDouble` — double → text in the buffer

Conditional implementation based on platform/libc:

- **libc++ ≥ 14000 and not Apple**: uses `std::to_chars` (fastest, ~16 ms in benchmark) into a `char buf[100]`, then `buffer.add(string_view)`.
  - Comment notes there is room to optimize further by serializing directly into the buffer, but the current API doesn't expose a simple "100 chars of raw buffer" primitive.
- **Otherwise** (older compilers, Apple): falls back to `fmt::to_string(number)` (~19 ms), which is correct everywhere but slightly slower.

The benchmark commentary lists the alternatives considered and rejected (`absl::StrCat` loses precision, `snprintf` is too slow, etc.).

---

## 8. Cross-cutting design rules

- **`OverflowDetectingUInt64` for length** → catches accounting drift instantly.
- **All slices owned by a buffer have the buffer's account** → memory pressure attribution.
- **Coalesce only when copy is cheap** (< 512 bytes) → balance between fragmentation and copy cost.
- **Free lists are thread-local and capped** → no global lock, bounded memory.
- **Watermark callbacks are latched** → exactly one `above_high` per crossing, exactly one `above_overflow` ever.
- **Drain trackers are FIFO** → callers can rely on registration order.
- **Static cast to `OwnedImpl` on `move()`** → comment acknowledges the tight coupling and justifies it as a pragmatic perf compromise; documented as a hard requirement that all participating buffers be `OwnedImpl`.

---

## 9. Quick "which method should I call" table

| Goal | Use |
|------|-----|
| Append bytes (copying) | `add(data, size)` / `add(string_view)` |
| Append bytes (zero-copy, external buffer) | `addBufferFragment(fragment)` |
| Append several `string_view`s in one shot | `addFragments({...})` |
| Take bytes from another buffer | `move(rhs)` (full) or `move(rhs, length)` (partial) |
| Insert at the front | `prepend(data)` / `prepend(other_buffer)` |
| Read N bytes contiguously | `linearize(N)` |
| Look at front bytes without copying | `frontSlice()` / `getRawSlices()` |
| Discard N bytes from the front | `drain(N)` |
| Reserve space for socket read | `reserveForRead()` (multi-slice) or `reserveSingleSlice(N)` |
| Search a substring | `search(needle, ...)` / `startsWith(prefix)` |
| Hand off a writable slice | `extractMutableFrontSlice()` |
| Track memory for overload manager | Bind a `BufferMemoryAccountSharedPtr` via `bindAccount()` (only valid on empty buffer) |
| Add backpressure callbacks | Use `WatermarkBuffer` + `setWatermarks(high, overflow_mult)` |


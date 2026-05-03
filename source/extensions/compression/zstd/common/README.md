# Zstd Common

Dictionary manager shared by the Zstd compressor and decompressor
extensions. Loads dictionaries from `DataSource` entries, stores them in a
thread-local map keyed by dictionary id, and watches filesystem sources so
rotations are picked up without restart.

## Files
- `dictionary_manager.h` - `DictionaryManager<T, deleter, getDictId>`
  template, plus inner `DictionarySharedPtr` (custom-deleter
  `shared_ptr<T>`) and `DictionaryThreadLocalMap`
  (`absl::flat_hash_map<unsigned, DictionarySharedPtr>` that is also a
  `ThreadLocal::ThreadLocalObject`).

## Interface
- Internal helper, not a registered factory. Instantiated by the compressor
  and decompressor factories with:
  - `T = ZSTD_CDict`, `deleter = ZSTD_freeCDict`,
    `getDictId = ZSTD_getDictID_fromCDict` for compression.
  - `T = ZSTD_DDict`, `deleter = ZSTD_freeDDict`,
    `getDictId = ZSTD_getDictID_fromDDict` for decompression.

## Logic
- Constructor reads every `DataSource` via `Config::DataSource::read`, calls
  the supplied `DictionaryBuilder` lambda to wrap the bytes in a
  `ZSTD_CDict` / `ZSTD_DDict`, and inserts them into a map keyed by
  `getDictId`. Empty/invalid ids (`id == 0`) trip a `RELEASE_ASSERT`
  because Zstd requires a non-zero dictionary id.
- For each `DataSource` that points at a filename, the manager registers a
  filesystem watch on `Modified | MovedTo`; on change it rereads the file,
  rebuilds the dictionary, and publishes the new entry to all workers via
  `tls_slot_->runOnAllThreads`.
- `replace_mode_` controls whether the previous id is removed when a new
  dictionary arrives (compressor uses `true`, decompressor uses `false` so
  older frames can still be decoded).
- `getDictionary(first_only, id)` underlies both
  `getDictionaryById(id)` (decompression path) and
  `getFirstDictionary()` (compression path where there is only one).

## Key decision points
- `dictionary_manager.h:41` - the `id != 0` assertion rejects malformed
  dictionaries at load time so compression never starts with a broken dict.
- `dictionary_manager.h:95` - the `onDictionaryUpdate` handler keeps the
  previous dictionary if the new one fails to load (`id == 0`) so rotations
  can be partially rolled back without downtime.
- `dictionary_manager.h:56` - thread-local publication clones the map per
  worker to keep updates lock-free on the hot path.

## Configuration
- None directly; consumers pass
  `Protobuf::RepeatedPtrField<envoy::config::core::v3::DataSource>` plus
  the `replace_mode` flag and a `DictionaryBuilder` lambda.

## Stats / errors
- None emitted here. Caller-visible errors come from
  `Config::DataSource::read` (throws) or from the assertion on invalid ids.

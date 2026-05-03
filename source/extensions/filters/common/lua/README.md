# Lua Runtime (shared filter infrastructure)

C++ binding layer around `luajit` that the HTTP Lua filter (`source/extensions/filters/http/lua`) and the Lua string matcher (`source/extensions/string_matcher/lua`) use to run user Lua scripts safely, per worker, with stream attributes exposed as Lua userdata. The library hides the Lua/C stack machinery behind macros, provides a per-worker `ThreadLocalState`, a `Coroutine` wrapper for yielding between HTTP events, and reusable wrappers (headers, buffers, dynamic metadata, SSL info, parsed X509 subjects) that any Lua-hosting filter can plug into its own per-filter wrappers.

## Files
- `lua.h` - Core runtime: `DECLARE_LUA_FUNCTION*` macros, `BaseLuaObject<T>`, `LuaRef`/`LuaDeathRef`, `Coroutine`, `ThreadLocalState`, `LuaException`.
- `lua.cc` - `Coroutine::start/resume`, `ThreadLocalState` construction, script log dispatch, TLS state creation.
- `wrappers.h/cc` - `BufferWrapper`, `MetadataMapWrapper`, `MetadataMapIterator`, `MetadataMapHelper`, `ParsedX509NameWrapper`, `SslConnectionWrapper`, and the `MetadataMapHelper` converter between Lua tables and `Protobuf::Value`.
- `protobuf_converter.h/cc` - `ProtobufConverterUtils` for materializing Lua tables from arbitrary protobuf messages via reflection (used for dynamic typed metadata).

## Public interface
- `template<class T> class BaseLuaObject` (`lua.h:136`) - base for every Lua/C wrapper. Subclasses:
  1. inherit `BaseLuaObject<Derived>`;
  2. declare methods with `DECLARE_LUA_FUNCTION` / `DECLARE_LUA_CLOSURE` (`lua.h:45`, `:57`, `:62`);
  3. list them via `static ExportedFunctions exportedFunctions()`;
  4. are registered once with `ThreadLocalState::registerType<Derived>()`.
  Objects are created Lua-side via `BaseLuaObject<T>::create(state, ...)` (`lua.h:151`) which does placement-new inside `lua_newuserdata`. `markDead()` / `checkDead()` let C++ revoke access across coroutine yields (`lua.h:219`).
- `LuaRef<T>` (`lua.h:316`) - RAII `luaL_ref` holder preventing GC. `LuaDeathRef<T>` (`lua.h:394`) additionally calls `markDead()` on destruction - the preferred container for any wrapper that becomes invalid once a `pcall`/`resume` returns.
- `Coroutine` (`lua.h:430`) - wraps `lua_newthread`. States: `NotStarted -> Yielded -> Finished`. `start(function_ref, num_args, yield_cb)` rearranges the stack and calls `resume` (`lua.cc:48`); `resume` dispatches on `lua_resume` return code: `0` -> Finished, `LUA_YIELD` -> Yielded + yield_cb, anything else throws `LuaException` with the Lua error string (`lua.cc:61`).
- `ThreadLocalState(code, SlotAllocator&)` (`lua.h:470`) - verifies parsing on the main thread, then allocates a fresh `luaL_newstate` on every worker and re-executes the script (`lua.cc:82`). Exposes `createCoroutine()`, `registerGlobal(name, initializers)` / `getGlobalRef(slot)`, `registerType<T>()`, `runtimeBytesUsed()`, `runtimeGC()`.
- Logger: `LuaLoggable::scriptLog(level, message)` (`lua.cc:17`) - `BaseLuaObject` auto-exports `logTrace/logDebug/logInfo/logWarn/logErr/logCritical` on every registered type (`lua.h:264`).
- Wrappers:
  - `BufferWrapper` (`wrappers.h:18`) - `length`, `getBytes(start, len)`, `setBytes(string)`.
  - `MetadataMapWrapper` / `MetadataMapIterator` (`wrappers.h:65`, `:81`) - `get(filter_name)` plus `__pairs` iteration; iterator is reset on `onMarkDead` because iterators cannot survive yields.
  - `ParsedX509NameWrapper` (`wrappers.h:116`) - `commonName`, `organizationName`.
  - `SslConnectionWrapper` (`wrappers.h:141`) - 23 accessors around `Ssl::ConnectionInfo` (SANs, subject, issuer, SHA digests, ciphersuite, TLS version, validFrom/expiration, PEM-encoded peer cert).
- `ProtobufConverterUtils` (`protobuf_converter.h:36`) - `pushLuaTableFromMessage`, `pushLuaValueFromField`, `pushLuaTableFromMapField`, `pushLuaArrayFromRepeatedField`, and `processDynamicTypedMetadataFromLuaCall(state, typed_metadata_map)` which unpacks an `Any` from `typed_filter_metadata` and converts it into a Lua table, returning 1 (table) or 1 (nil) to Lua.

## Implementation logic
- `BaseLuaObject::registerType` creates a metatable keyed by `typeid(T).name()`, sets `__index` to itself, registers all exported functions plus a `__gc` that invokes the C++ destructor (`lua.h:190`). Memory is raw `lua_newuserdata`, so placement new is required - handled by `allocateLuaUserData<T>`/`alignAndCast<T>` (`lua.h:100`, `:108`).
- The `DECLARE_LUA_FUNCTION_EX` macro emits a static thunk that validates userdata via `luaL_checkudata`, calls `checkDead(state)` (which raises a Lua error if the object was revoked), then dispatches to the instance method (`lua.h:45`). `DECLARE_LUA_CLOSURE` is the upvalue variant used for iterators.
- `ThreadLocalState` pre-parses the script on the calling thread and only after success schedules `tls_slot_->set(...)` to materialize one `lua_State` per worker with `luaL_openlibs` + `luaL_dostring` (`lua.cc:82`). Failures throw `LuaException`.
- `registerGlobal` runs on all threads (main + workers) and stores the `luaL_ref` index per thread at position `current_global_slot_++`; if the named global is not a function the slot holds `LUA_REFNIL` so consumers can no-op without errors (`lua.cc:104`).
- `Coroutine::resume` surfaces Lua errors via `LuaException` so the filter can turn them into an HTTP 500 (HTTP filter) or a matcher error.
- `getStringViewFromLuaString` is used throughout wrappers so strings are not materialized - Lua strings are length-prefixed, so `absl::string_view` from `luaL_checklstring` is zero-copy (`lua.h:79`).

## Consumers
- `source/extensions/filters/http/lua` - instantiates `ThreadLocalState` per plugin, hosts its own `StreamHandleWrapper` that owns instances of `BufferWrapper`, `MetadataMapWrapper`, `SslConnectionWrapper`, plus HTTP-specific header/stream wrappers.
- `source/extensions/string_matcher/lua` - uses `ThreadLocalState` without coroutines to call a Lua function per string match.

## Stats / errors / failure modes
- No stats. Script runtime errors propagate as `LuaException` (`lua.h:536`) from `Coroutine::resume` / `ThreadLocalState` constructor.
- `BaseLuaObject::checkDead` is the main safety net: any use of a wrapper after its scope ends raises `luaL_error(state, "object used outside of proper scope")` (`lua.h:221`), which becomes a `LuaException` at the `pcall` boundary.
- `registerGlobal` emits `debug` log when a script does not define the expected global; it does not fail - callers must check for `LUA_REFNIL`.
- Memory accounting is observational only: `runtimeBytesUsed()` / `runtimeGC()` let filters report GC health but there is no automatic eviction.
- Because every worker owns an independent `lua_State`, truly-global state is unavailable; filters requiring shared state (counters, caches) must build that in C++ and expose it through their own wrappers.

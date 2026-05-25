# `utility.{h,cc}` + `yaml_utility.cc` — `MessageUtil` and friends

`utility.h` packs **eight** stateless utility classes and a couple of free-function families. Together they
form the bulk of what "doing protobuf" in Envoy looks like.

| Class                    | Role                                                                |
|--------------------------|---------------------------------------------------------------------|
| `MessageUtil`            | The big one: parse, validate, hash, render, redact, anyConvert, … |
| `ValueUtil`              | Helpers around `google::protobuf::Value`                            |
| `HashedValue`            | A `Value` with its hash cached at ctor (for use in hash maps)       |
| `DurationUtil`           | Validated `Duration` → ms/seconds conversions                       |
| `TimestampUtil`          | `SystemTime` → `Timestamp` serialization                            |
| `StructUtil`             | Recursive `Struct` merge                                            |
| `RepeatedPtrUtil`        | Join, debugString, hash, conversion on repeated fields              |
| `TypeUtil`               | type_url ↔ descriptor full name                                     |
| `ProtoExceptionUtil`     | Centralised `EnvoyException` throwers                               |

YAML/JSON code paths live in `yaml_utility.cc` and are compiled only when `ENVOY_ENABLE_YAML` is on. Their
declarations are gated by the same macro in `utility.h`.

---

## `MessageUtil` — by purpose

### Functor surface

```cpp
class MessageUtil {
public:
  // Use as Hasher in flat_hash_map<MessageT, V, MessageUtil, MessageUtil>:
  std::size_t operator()(const Message& m) const { return hash(m); }
  bool operator()(const Message& l, const Message& r) const {
    return Protobuf::util::MessageDifferencer::Equals(l, r);
  }
};
```

This is how callers turn `MessageUtil` into the hash + equality functors for hash containers keyed by protos.

### Loading from external formats

```cpp
// YAML (full-proto / ENVOY_ENABLE_YAML only)
void loadFromYaml(const std::string& yaml, Message& m, ValidationVisitor& v);
absl::Status loadFromYamlNoThrow(const std::string& yaml, Message& m, ValidationVisitor& v);

// JSON
void loadFromJson(absl::string_view json, Message& m, ValidationVisitor& v);
absl::Status loadFromJsonNoThrow(absl::string_view json, Message& m, bool& has_unknown_field);
absl::Status loadFromJsonNoThrow(absl::string_view json, Struct& m);
void loadFromJson(absl::string_view json, Struct& m);

// Filesystem (picks by extension: .pb / .pb_text / .json / .yaml / .yml)
absl::Status loadFromFile(const std::string& path, Message& m,
                          ValidationVisitor& v, Api::Api& api);
```

`loadFromJsonNoThrow` is the **trust-but-verify** variant. It tries strict-JSON parsing first; if that fails
with an unknown-field error, it retries with `ignore_unknown_fields=true` and reports the outcome via the
`has_unknown_field` out-parameter so the caller can decide whether to warn vs. fail.

`loadFromFile` is what bootstrap parsing uses. The extension table (`MessageUtil::FileExtensions`) declares
the recognised filename suffixes:

```cpp
class FileExtensionValues {
public:
  const std::string ProtoBinary = ".pb";
  const std::string ProtoBinaryLengthDelimited = ".pb_length_delimited";
  const std::string ProtoText = ".pb_text";
  const std::string Json = ".json";
  const std::string Yaml = ".yaml";
  const std::string Yml = ".yml";
};
using FileExtensions = ConstSingleton<FileExtensionValues>;
```

### Validation

```cpp
template <class MessageType>
static void validate(const MessageType& message,
                     ValidationVisitor& v,
                     bool recurse_into_any = false) {
  if (!v.skipValidation()) {
    checkForUnexpectedFields(message, v, recurse_into_any);
  }
  validateDurationFields(message, recurse_into_any);
  if (recurse_into_any) {
    return recursivePgvCheck(message);   // PGV all the way down through Anys
  }
  std::string err;
  if (!Validate(message, &err)) {
    ProtoExceptionUtil::throwProtoValidationException(err, message);
  }
}
```

Four steps in this order:

1. **Unknown-field check** via `traverseMessage(visitor=…, recurse_into_any)`. The visitor delegates to the
   `ValidationVisitor` (`onUnknownField` / `onDeprecatedField` / `onWorkInProgress`).
2. **Duration-field validation** — every `google.protobuf.Duration` in the tree is enforced to be ≥ 0 and ≤
   `MaxDurationSeconds` (~10000 years; the upstream max).
3. **PGV check** — runs the generated `Validate()` from `*.pb.validate.h` for the top-level message. If
   `recurse_into_any` is on, uses `recursivePgvCheck` instead, which walks `Any` payloads and validates them
   too.
4. On any failure, throws via `ProtoExceptionUtil::throwProtoValidationException(err, message)` so the caller
   sees an `EnvoyException` with both error text and the failing message.

`downcastAndValidate<T>(generic_msg, v)` is the convenience for "I just got an `Any` unpacked to
`Protobuf::Message`; cast it to `T` and validate in one step":

```cpp
template <class MessageType>
static const MessageType& downcastAndValidate(const Message& cfg, ValidationVisitor& v) {
  const auto& typed = dynamic_cast<MessageType>(cfg);
  validate(typed, v);
  return typed;
}
```

The `dynamic_cast` will throw `std::bad_cast` if `cfg` isn't actually a `MessageType`.

### Any pack/unpack

```cpp
static void packFrom(Any& any, const Message& msg);                   // throws on serialize failure
static absl::Status unpackTo(const Any& any, Message& msg);           // returns InvalidArg on type/serialize mismatch
template <class T> static void anyConvert(const Any& any, T& typed);  // throws via THROW_IF_NOT_OK on unpackTo failure
template <class T> static T anyConvert(const Any& any);
template <class T> static void anyConvertAndValidate(const Any& any, T& typed, ValidationVisitor& v);
template <class T> static T anyConvertAndValidate(const Any& any, ValidationVisitor& v);

static absl::StatusOr<std::string> anyToBytes(const Any& any);          // StringValue/BytesValue/raw
static absl::StatusOr<std::string> knownAnyToBytes(const Any& any);     // adds Struct → JSON support
```

`anyConvertAndValidate<T>` is the workhorse for xDS: every `DiscoveryResource` arrives as
`google.protobuf.Any` and is unpacked + validated in one line per resource type.

`knownAnyToBytes` exists because some places (cache filter, ext-authz body) want to pass arbitrary opaque
bytes through an `Any` field — supporting `Struct` as JSON adds a useful path for configs that wrap structured
data.

### Hashing

```cpp
static std::size_t hash(const Message& m);   // TextFormat-based, deterministic, slow
```

Uses `Protobuf::TextFormat::Printer` with:

- `SetExpandAny(true)` — recurse into Anys (with descriptor pool lookup).
- `SetUseFieldNumber(true)` — emit field numbers (immune to field renames).
- `SetSingleLineMode(true)` — compact.
- `SetHideUnknownFields(true)` — skip unknown wire data.

…then xxHash64 of the resulting text. The result is stable across runs, across processes, and across rebuilds
where field numbers don't change. Slow on big configs because the text serialization is O(n) and allocates.

For new code, prefer `DeterministicProtoHash::hash` (in [`deterministic_hash.md`](deterministic_hash.md)).

### Rendering

```cpp
static absl::StatusOr<std::string>
getJsonStringFromMessage(const Message& m, bool pretty_print=false, bool always_print_primitive_fields=false);
static std::string getJsonStringFromMessageOrError(const Message& m, bool pretty_print=false, bool always_print_primitive_fields=false);
static std::string getYamlStringFromMessage(const Message& m, bool block_print=true, bool always_print_primitive_fields=false);
static std::string toTextProto(const Message& m);
static std::string convertToStringForLogs(const Message& m, bool pretty_print=false, bool always_print_primitive_fields=false);
```

`getJsonStringFromMessageOrError` returns "error: <reason>" on failure (used in log messages where you want a
best-effort rendering rather than an exception). `convertToStringForLogs` is the macro-friendly variant —
chooses JSON or text-format based on `ENVOY_ENABLE_YAML`.

`always_print_primitive_fields`: when false (the default), JSON skips fields equal to their default (0, false,
empty string). When true, every primitive is emitted. The latter is what `/config_dump` uses so operators can
see "this field is *explicitly* the default" vs "this field isn't set".

### Redaction

```cpp
static void redact(Message& m);
```

Walks the message tree (full reflection required, so calls `createReflectableMessage` first) and:

- `string` field with `[(udpa.annotations.sensitive) = true]` → `"[redacted]"`.
- `bytes` field with the same annotation → `5B72656461637465645D` (UTF-8 of `[redacted]`).
- Primitive (incl. enum) with annotation → cleared.
- `Message` with annotation → recurses to redact its contents.

**Limitations** (in the source comment):

- Doesn't redact `Struct`-encoded data (no annotations).
- Doesn't redact `Any` with type_url not registered in the binary (no descriptor available).

Used by `/config_dump` so secrets / private keys / passwords don't leak.

### Struct & key/value helpers

```cpp
static Struct keyValueStruct(const std::string& key, const std::string& value);
static Struct keyValueStruct(const std::map<std::string, std::string>& fields);
static std::string getStringField(const Message& m, const std::string& field_name);
```

Used by metadata-construction code in filters that need to plant `Struct` fields into
`stream_info.dynamicMetadata()`.

### Misc

```cpp
static const std::string& bytesToString(const std::string& bytes);  // noop, exists as a future migration handle
static std::string codeEnumToString(absl::StatusCode code);
static std::string sanitizeUtf8String(absl::string_view str);       // replaces invalid UTF-8
```

`sanitizeUtf8String` matters for the formatter / access log path: header values coming off the wire can be
arbitrary bytes, but `Proxy-Status` headers and JSON logs must be valid UTF-8.

---

## `ValueUtil`

```cpp
class ValueUtil {
public:
  static std::size_t hash(const Value& v) { return MessageUtil::hash(v); }
  static Value loadFromYaml(const std::string& yaml);                // ENVOY_ENABLE_YAML only
  static bool equal(const Value& a, const Value& b);

  static const Value& nullValue();
  static Value stringValue(absl::string_view s);
  static Value optionalStringValue(const absl::optional<std::string>& s);
  static Value boolValue(bool b);
  static Value structValue(const Struct& obj);
  template <typename T> static Value numberValue(T n);
  static Value listValue(const std::vector<Value>& values);
};
```

`equal` does a recursive structural compare with proper `bool/number/string/struct/list` handling — needed
because `Protobuf::util::MessageDifferencer::Equals` does **bit-level** comparison and so reports
`NumberValue(1)` ≠ `NumberValue(1.0)` (which structurally they aren't, but semantically they are).

### `HashedValue`

```cpp
class HashedValue {
public:
  HashedValue(const Value& v) : value_(v), hash_(ValueUtil::hash(v)) {}
  std::size_t hash() const { return hash_; }
  const Value& value() const { return value_; }
  bool operator==(const HashedValue& r) const { return hash_ == r.hash_ && ValueUtil::equal(value_, r.value_); }
};

// And the std::hash specialization at the bottom of utility.h:
template <> struct std::hash<Envoy::HashedValue> {
  std::size_t operator()(Envoy::HashedValue const& v) const { return v.hash(); }
};
```

Use this when you put `Value`s in `flat_hash_map` keys — caching the hash at construction avoids
re-stringifying the value on every probe.

---

## `DurationUtil`

```cpp
class DurationUtil {
public:
  static uint64_t durationToMilliseconds(const Duration& d);             // throws if negative / overflow
  static absl::StatusOr<uint64_t> durationToMillisecondsNoThrow(const Duration& d);
  static uint64_t durationToSeconds(const Duration& d);                  // throws if negative / overflow
};
```

These are stricter than the upstream `TimeUtil::DurationToMilliseconds`: negative durations are rejected
(Envoy semantics: most fields are "timeouts" and negative is nonsense), and saturation overflow is rejected.

Used by the `PROTOBUF_GET_MS_*` and `PROTOBUF_GET_SECONDS_*` macros.

---

## `TimestampUtil`

```cpp
static void systemClockToTimestamp(SystemTime t, Timestamp& out);
```

Goes via `time_t` to avoid undefined-behaviour edge cases at the nanosecond boundary. Used by
`AccessLogCommon` building, by stats sinks that report wall-clock timestamps, etc.

---

## `StructUtil`

```cpp
static void update(Struct& obj, const Struct& with);   // recursive merge
```

Merge semantics:

- Key missing in `obj` → copied from `with`.
- Key present but with a different kind → replaced.
- Both scalar (null/string/number/bool) → replaced.
- Both list → `with`'s values appended to `obj`'s.
- Both struct → recurse with the same merge rules.

Used when bootstrap config layers (CLI overlay, file overlay) need to be combined into a single `Struct`.

---

## `RepeatedPtrUtil`

```cpp
static std::string join(const RepeatedPtrField<std::string>&, const std::string& delim);
template <class T> static std::string debugString(const RepeatedPtrField<T>&);
template <class T> static std::size_t hash(const RepeatedPtrField<T>&);   // same TextFormat trick as MessageUtil::hash
template <typename T, typename R> static R convertToConstMessagePtrContainer(const RepeatedPtrField<T>&);
```

`convertToConstMessagePtrContainer` is the standard way to turn a repeated typed proto field into a
`std::vector<std::unique_ptr<const Message>>` (or similar container) — used widely by filter factories that
want to keep parsed sub-configs alive without holding the parent proto.

---

## `TypeUtil`

```cpp
static absl::string_view typeUrlToDescriptorFullName(absl::string_view type_url);
static std::string descriptorFullNameToTypeUrl(absl::string_view type);
```

`type_url` always looks like `type.googleapis.com/envoy.config.cluster.v3.Cluster`. The two helpers strip /
add the prefix. Used by xDS subscription matching and factory lookup.

---

## `ProtoExceptionUtil`

```cpp
class ProtoExceptionUtil {
public:
  static void throwMissingFieldException(const std::string& name, const Message& m);
  static void throwProtoValidationException(const std::string& err, const Message& m);
};
```

Both throw `EnvoyException` (subclass of `std::exception`). Centralised so that error formatting
("Field '%s' is required at type %s") is consistent across every `PROTOBUF_GET_*_REQUIRED` macro site.

---

## Macros — cheat sheet

```cpp
PROTOBUF_GET_WRAPPED_OR_DEFAULT(cfg, max_conns, 1024)             // UInt32Value etc → uint32_t (default)
PROTOBUF_GET_OPTIONAL_WRAPPED(cfg, max_conns)                     // → absl::optional<uint32_t>
PROTOBUF_GET_WRAPPED_REQUIRED(cfg, max_conns)                     // → uint32_t (throws if unset)

PROTOBUF_GET_MS_OR_DEFAULT(cfg, idle_timeout, 60'000)             // Duration → ms (default)
PROTOBUF_GET_OPTIONAL_MS(cfg, idle_timeout)                       // → absl::optional<chrono::milliseconds>
PROTOBUF_GET_MS_REQUIRED(cfg, idle_timeout)                       // throws if unset
PROTOBUF_GET_SECONDS_OR_DEFAULT(cfg, idle_timeout, 60)
PROTOBUF_GET_SECONDS_REQUIRED(cfg, idle_timeout)

PROTOBUF_GET_STRING_OR_DEFAULT(cfg, name, "default")              // string field → string (default if empty)

PROTOBUF_PERCENT_TO_DOUBLE_OR_DEFAULT(cfg, pct, 50.0)             // envoy::type::v3::Percent → double
PROTOBUF_PERCENT_TO_ROUNDED_INTEGER_OR_DEFAULT(cfg, pct, 10'000, 5'000)
```

All percent macros throw `EnvoyException` on NaN values; the integer one also clamps to `max_value` via
`ProtobufPercentHelper::convertPercent`.

### `ProtobufPercentHelper` (internal)

```cpp
namespace ProtobufPercentHelper {
  uint64_t checkAndReturnDefault(uint64_t default_value, uint64_t max_value);
  uint64_t convertPercent(double percent, uint64_t max_value);
  bool evaluateFractionalPercent(envoy::type::v3::FractionalPercent percent, uint64_t random_value);
  uint64_t fractionalPercentDenominatorToInt(const FractionalPercent::DenominatorType&);
}
```

`evaluateFractionalPercent` is the canonical "roll the dice" used by sampling, rate limit, fault filter, etc.
Returns `true` with probability `numerator / denominatorToInt(denominator)`.

---

## Gotchas

1. **`MessageUtil::hash` is slow for big configs.** Prefer `DeterministicProtoHash::hash` in new code.
2. **`MessageUtil::loadFromJson` throws** on parse errors; `loadFromJsonNoThrow` returns a `Status`. Use the
   latter when caller wants to recover.
3. **`MessageUtil::validate` is templated** to keep the `Validate(...)` PGV function visible. Include the
   relevant `*.pb.validate.h` in your source file — the header doesn't pull it transitively.
4. **`DurationUtil::durationToMilliseconds`** throws on negative durations. If you want to allow them, use the
   `NoThrow` variant.
5. **`MessageUtil::redact` requires reflection.** In lite-proto mode it still works because it goes through
   `createReflectableMessage`, but adds an allocation per message.
6. **`HashedValue` doesn't own the underlying `Value` by value** in the sense of a deep cache — it stores a
   value-typed copy, but `Value` itself is a proto and copies are O(n) in the size of the value. Prefer
   `move`-constructing where possible.

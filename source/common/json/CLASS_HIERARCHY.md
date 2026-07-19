# JSON — class hierarchy (UML)

UML-style Mermaid for the JSON object model, factories, and streamer. See [`OVERVIEW.md`](OVERVIEW.md) for
behavior.

---

## Object model & factories

```mermaid
classDiagram
    class Object {
        <<interface>>
        +getValue(name)* StatusOr~ValueType~
        +getObject(name)* StatusOr~ObjectSharedPtr~
        +getString/getInteger/getBoolean/getDouble(name)* StatusOr
        +asObjectArray()* StatusOr~vector~
        +iterate(ObjectCallback)*
    }
    class Factory {
        <<public>>
        +loadFromString(json)$ StatusOr~ObjectSharedPtr~
        +loadFromProtobufStruct(s)$ ObjectSharedPtr
        +jsonToMsgpack(json)$ vector~uint8_t~
        +listAsJsonString(items)$ string
    }
    class Nlohmann_Factory {
        <<backend>>
        +loadFromString(json)$ / loadFromProtobufStruct(s)$
        +serialize(str)$ / jsonToMsgpack(json)$
    }

    Factory ..> Nlohmann_Factory : delegates
    Nlohmann_Factory ..> Object : creates
    Factory ..> Object : returns

    note for Nlohmann_Factory "wraps nlohmann/json;\nswappable behind Factory"
```

---

## The streamer

```mermaid
classDiagram
    class StreamerBase~OutputBufferType~ {
        -response_ : OutputBufferType
        +makeRootMap() MapPtr
        +makeRootArray() ArrayPtr
        -addNumber/addString/addBool/addNull()
        -topLevel() Level*  (debug only)
    }
    class Level {
        -closer_ : string_view
        +addMap() MapPtr / addArray() ArrayPtr
        +addNumber() / addString() / addBool() / addNull()
        +addSerializedJsonFragment(frag)
        #nextField()
        +~Level() writes closer_
    }
    class Map {
        +addKey(key)
        +addEntries(...)
    }
    class Array

    class BufferOutput {
        +add(string_view...)
        -buffer_ : Buffer::Instance&
    }
    class StringOutput {
        +add(string_view...)
        -buffer_ : std::string&
    }

    StreamerBase *-- Level : level stack
    Level <|-- Map
    Level <|-- Array
    StreamerBase ..> BufferOutput : OutputBufferType
    StreamerBase ..> StringOutput : OutputBufferType

    note for Level "RAII: opener in ctor, closer in dtor\nASSERT_THIS_IS_TOP_LEVEL (debug)"
```

---

## Sanitizer & utility (free functions)

```mermaid
classDiagram
    class json_sanitizer {
        <<namespace>>
        +sanitize(buffer, str)$ string_view
        +stripDoubleQuotes(str)$ string_view
    }
    class Utility {
        +appendValueToString(Value, dest)$
        +appendStructToString(Struct, dest)$
    }
    Utility ..> StreamerBase : uses
    Level ..> json_sanitizer : addString escapes via sanitize

    note for json_sanitizer "zero-copy when no escaping needed"
```

---

## Relationship summary

| Relationship | Type | Meaning |
|---|---|---|
| `Factory` → `Nlohmann_Factory` | delegation | Public API forwards to the nlohmann backend. |
| `Nlohmann_Factory` → `Object` | factory | Produces the navigable tree. |
| `StreamerBase` → `Level` | composition | Maintains the open map/array stack. |
| `Map`/`Array` → `Level` | inheritance | Concrete level kinds (RAII open/close). |
| `StreamerBase` → `BufferOutput`/`StringOutput` | template param | Pluggable output sink. |
| `Level::addString` → `sanitize()` | uses | Escapes strings on the way out. |
| `Utility` → `StreamerBase` | uses | protobuf → JSON via the streamer. |

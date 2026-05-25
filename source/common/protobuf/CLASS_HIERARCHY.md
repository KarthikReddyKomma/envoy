# `protobuf/` — Class hierarchy (UML)

Interfaces from `envoy/protobuf/message_validator.h` are in italics. Concrete classes in this folder are bold.
This folder is dominated by **static utility classes** (no inheritance) so the diagram has more of those than
inheritance trees.

```mermaid
classDiagram
    direction LR

    %% ===== Validation hierarchy =====
    class ValidationVisitor {
        <<interface>>
        +onUnknownField(desc) Status
        +onDeprecatedField(desc, soft) Status
        +skipValidation() bool
        +onWorkInProgress(desc)
        +runtime() OptRef~Loader~
    }
    class ValidationVisitorBase {
        -runtime_ OptRef~Loader~
        -skip_deprecated_logs_ bool
        +setRuntime(loader)
        +clearRuntime()
        +setSkipDeprecatedLogs(b)
        +isSkipDeprecatedLogs() bool
    }
    class WipCounterBase {
        -wip_counter_ Counter*
        -prestats_wip_count_ uint64
        +setWipCounter(counter)
        +onWorkInProgressCommon(desc)
    }
    class NullValidationVisitorImpl {
        +onUnknownField(_) OkStatus
        +onDeprecatedField(_,_) OkStatus
        +skipValidation() true
        +onWorkInProgress(_) no-op
    }
    class WarningValidationVisitorImpl {
        -descriptions_ flat_hash_set~uint64~
        -unknown_counter_ Counter*
        -prestats_unknown_count_ uint64
        +setCounters(unk, wip)
        +onUnknownField(desc)
        +onDeprecatedField(desc, soft)
        +skipValidation() false
        +onWorkInProgress(desc)
    }
    class StrictValidationVisitorImpl {
        +setCounters(wip)
        +onUnknownField(desc) Error
        +onDeprecatedField(desc, soft)
        +skipValidation() false
        +onWorkInProgress(desc)
    }
    ValidationVisitor <|.. ValidationVisitorBase
    ValidationVisitorBase <|-- NullValidationVisitorImpl
    ValidationVisitorBase <|-- WarningValidationVisitorImpl
    ValidationVisitorBase <|-- StrictValidationVisitorImpl
    WipCounterBase <|-- WarningValidationVisitorImpl
    WipCounterBase <|-- StrictValidationVisitorImpl

    class ValidationContext {
        <<interface>>
        +staticValidationVisitor() ValidationVisitor&
        +dynamicValidationVisitor() ValidationVisitor&
    }
    class ValidationContextImpl {
        -static_validation_visitor_ Visitor&
        -dynamic_validation_visitor_ Visitor&
        +staticValidationVisitor()
        +dynamicValidationVisitor()
    }
    class ProdValidationContextImpl {
        -strict_validation_visitor_
        -static_warning_validation_visitor_
        -dynamic_warning_validation_visitor_
        +setCounters(static_unk, dyn_unk, wip)
        +setRuntime(loader)
    }
    ValidationContext <|.. ValidationContextImpl
    ValidationContextImpl <|-- ProdValidationContextImpl
    ProdValidationContextImpl o--> StrictValidationVisitorImpl
    ProdValidationContextImpl o--> WarningValidationVisitorImpl
    ProdValidationContextImpl o--> WarningValidationVisitorImpl
    ProdValidationContextImpl ..> NullValidationVisitorImpl : may reference (ignore_unknown_dynamic)

    %% ===== Visitor =====
    class ConstProtoVisitor {
        <<interface>>
        +onField(msg, FieldDescriptor)
        +onMessage(msg, parents, was_any_or_top_level) Status
    }
    class traverseMessage {
        <<free function>>
        +(visitor, message, recurse_into_any) Status
    }
    class ScopedMessageParents {
        -parents_ vector~Message*~&
        +ctor(parents, message)
        +dtor()
    }
    class Helper {
        <<utility namespace>>
        +typeUrlToMessage(type_url) unique_ptr~Message~
        +convertTypedStruct~T~(msg) StatusOr~pair~
    }
    traverseMessage ..> ConstProtoVisitor : drives
    traverseMessage ..> Helper : Any unpacking
    Helper ..> MessageUtil : jsonConvert

    %% ===== MessageUtil & friends (utility namespace) =====
    class MessageUtil {
        <<utility>>
        +operator() hash op
        +operator() equals op
        +hash(msg) size_t
        +loadFromJson / Yaml / File (string, msg, visitor)
        +loadFromJsonNoThrow / loadFromYamlNoThrow
        +loadFromYamlAndValidate~T~
        +validate~T~ / downcastAndValidate~T~ / recursivePgvCheck
        +checkForUnexpectedFields / validateDurationFields
        +packFrom / unpackTo
        +anyConvert~T~ / anyConvertAndValidate~T~
        +anyToBytes / knownAnyToBytes
        +jsonConvert / jsonConvertValue
        +getJsonStringFromMessage / getYamlStringFromMessage / getJsonStringFromMessageOrError
        +toTextProto / convertToStringForLogs
        +redact / sanitizeUtf8String
        +keyValueStruct / getStringField / codeEnumToString / bytesToString
    }
    class ValueUtil {
        <<utility>>
        +hash(Value) size_t
        +loadFromYaml(string) Value
        +equal(v1, v2) bool
        +nullValue / stringValue / optionalStringValue / boolValue / structValue / numberValue / listValue
    }
    class HashedValue {
        -value_ Value
        -hash_ size_t
        +value() Value&
        +hash() size_t
        +operator==
    }
    class DurationUtil {
        <<utility>>
        +durationToMilliseconds(Duration) uint64
        +durationToMillisecondsNoThrow(Duration) StatusOr~uint64~
        +durationToSeconds(Duration) uint64
    }
    class TimestampUtil {
        <<utility>>
        +systemClockToTimestamp(SystemTime, Timestamp&)
    }
    class StructUtil {
        <<utility>>
        +update(Struct&, const Struct&)
    }
    class RepeatedPtrUtil {
        <<utility>>
        +join(repeated, delim) string
        +debugString~T~(repeated) string
        +hash~T~(repeated) size_t
        +convertToConstMessagePtrContainer~T,R~(repeated) R
    }
    class TypeUtil {
        <<utility>>
        +typeUrlToDescriptorFullName(type_url) string_view
        +descriptorFullNameToTypeUrl(type) string
    }
    class ProtoExceptionUtil {
        <<utility>>
        +throwMissingFieldException(name, msg)
        +throwProtoValidationException(err, msg)
    }
    ValueUtil ..> MessageUtil : hash
    HashedValue o--> ValueUtil : hash on ctor
    MessageUtil ..> traverseMessage : checkForUnexpectedFields / redact / validateDurationFields
    MessageUtil ..> Helper : anyConvert+TypedStruct
    MessageUtil ..> ProtoExceptionUtil : throw on err

    %% ===== Hashing =====
    class DeterministicProtoHash {
        <<utility namespace>>
        +hash(Message) uint64
    }

    %% ===== Lite-proto bridge =====
    class MessageLiteDifferencer {
        <<lite-proto stand-in>>
        +Equals(m1, m2) bool
        +Equivalent(m1, m2) bool
    }
    class createReflectableMessage {
        <<free function>>
        +createReflectableMessage(Message) ReflectableMessage
    }
    class TextFormatTranscoder {
        <<3PP cc_proto_descriptor_library>>
        +loadFileDescriptors(info)
    }
    createReflectableMessage ..> TextFormatTranscoder : lite-proto path
```

### What's not on the diagram (because they're free functions / aliases)

- The macros: `PROTOBUF_GET_*`, `PROTOBUF_PERCENT_TO_*`. Live in `utility.h`.
- The `Envoy::Protobuf` namespace itself: 30+ `using ::google::protobuf::X` declarations under
  `#if !ENVOY_ENABLE_FULL_PROTOS`. Live in `protobuf.h`.
- `Envoy::ProtobufTypes::MessagePtr`, `Envoy::ProtobufUtil`. Aliases in `protobuf.h`.

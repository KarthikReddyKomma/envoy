# Matching engine — class hierarchy (UML)

UML for `envoy/matcher/matcher.h` (interfaces) and `source/common/matcher/` (implementations). Split into
sections for readability.

---

## 1. Core interfaces

```mermaid
classDiagram
    direction TB

    class MatchTree~DataType~ {
        <<interface>>
        +match(data, skipped_cb) ActionMatchResult
        #handleRecursionAndSkips(onMatch, data, cb)$ ActionMatchResult
    }
    class DataInput~DataType~ {
        <<interface>>
        +get(data) DataInputGetResult
        +dataInputType() string_view
    }
    class InputMatcher {
        <<interface>>
        +match(DataInputGetResult) MatchResult
        +supportsDataInputType(type) bool
    }
    class Action {
        <<interface>>
        +typeUrl() string_view
        +getTyped~T~() T&
    }
    class CommonProtocolInput {
        <<interface>>
        +get() DataInputGetResult
    }

    class OnMatch~DataType~ {
        +ActionConstSharedPtr action_
        +MatchTreeSharedPtr matcher_
        +bool keep_matching_
    }
    class ActionMatchResult {
        +isMatch() bool
        +isNoMatch() bool
        +isInsufficientData() bool
        +action() ActionConstSharedPtr
    }
    class DataInputGetResult {
        +DataAvailability availability()
        +stringData() optional~string_view~
        +customData~T~() OptRef~T~
    }

    MatchTree~DataType~ ..> OnMatch~DataType~ : yields
    MatchTree~DataType~ ..> ActionMatchResult : returns
    OnMatch~DataType~ ..> Action : action_
    OnMatch~DataType~ ..> MatchTree~DataType~ : nested matcher_
    DataInput~DataType~ ..> DataInputGetResult : returns
    InputMatcher ..> DataInputGetResult : consumes
```

---

## 2. MatchTree implementations (the nodes)

```mermaid
classDiagram
    direction TB

    class MatchTree~DataType~ {
        <<interface>>
    }
    class AnyMatcher~DataType~ {
        -optional~OnMatch~ on_no_match_
        +match() ActionMatchResult
    }
    class MapMatcher~DataType~ {
        <<abstract>>
        #DataInputPtr data_input_
        #optional~OnMatch~ on_no_match_
        +match() ActionMatchResult
        +addChild(value, onMatch)*
        #doMatch(data, key, cb)* ActionMatchResult
        #doNoMatch(data, cb) ActionMatchResult
    }
    class ExactMapMatcher~DataType~ {
        -flat_hash_map~string,OnMatch~ children_
        +create(input, on_no_match)$
        +addChild()
        #doMatch()
    }
    class PrefixMapMatcher~DataType~ {
        -RadixTree~shared_ptr~OnMatch~~ children_
        +create(input, on_no_match)$
        +addChild()
        #doMatch()
    }
    class ListMatcher~DataType~ {
        -optional~OnMatch~ on_no_match_
        -vector~pair~FieldMatcherPtr,OnMatch~~ matchers_
        +match()
        +addMatcher(matcher, action)
    }

    MatchTree~DataType~ <|.. AnyMatcher~DataType~
    MatchTree~DataType~ <|.. MapMatcher~DataType~
    MatchTree~DataType~ <|.. ListMatcher~DataType~
    MapMatcher~DataType~ <|-- ExactMapMatcher~DataType~
    MapMatcher~DataType~ <|-- PrefixMapMatcher~DataType~
```

---

## 3. FieldMatcher implementations (the predicates)

```mermaid
classDiagram
    direction TB

    class FieldMatcher~DataType~ {
        <<interface>>
        +match(data) MatchResult
    }
    class SingleFieldMatcher~DataType~ {
        -DataInputPtr data_input_
        -InputMatcherPtr input_matcher_
        +create(input, matcher)$ StatusOr
        +match() MatchResult
    }
    class AllFieldMatcher~DataType~ {
        -vector~FieldMatcherPtr~ matchers_
        +match() MatchResult
    }
    class AnyFieldMatcher~DataType~ {
        -vector~FieldMatcherPtr~ matchers_
        +match() MatchResult
    }
    class NotFieldMatcher~DataType~ {
        -FieldMatcherPtr matcher_
        +match() MatchResult
    }

    FieldMatcher~DataType~ <|.. SingleFieldMatcher~DataType~
    FieldMatcher~DataType~ <|.. AllFieldMatcher~DataType~
    FieldMatcher~DataType~ <|.. AnyFieldMatcher~DataType~
    FieldMatcher~DataType~ <|.. NotFieldMatcher~DataType~

    AllFieldMatcher~DataType~ o-- "*" FieldMatcher~DataType~
    AnyFieldMatcher~DataType~ o-- "*" FieldMatcher~DataType~
    NotFieldMatcher~DataType~ o-- "1" FieldMatcher~DataType~
```

---

## 4. InputMatcher implementations

```mermaid
classDiagram
    direction TB
    class InputMatcher {
        <<interface>>
    }
    class StringInputMatcher {
        -StringMatcherImpl matcher_
        +match(DataInputGetResult) MatchResult
    }
    InputMatcher <|.. StringInputMatcher
    StringInputMatcher ..> StringMatcherImpl : delegates to common/common/matchers
```

---

## 5. Factories & glue

```mermaid
classDiagram
    direction TB

    class OnMatchFactory~DataType~ {
        <<interface>>
        +createOnMatch(proto) optional~cb~
    }
    class MatchTreeFactory~DataType_Ctx~ {
        -ActionFactoryContext& action_factory_context_
        -ServerFactoryContext& server_factory_context_
        -MatchTreeValidationVisitor& visitor_
        -MatchInputFactory match_input_factory_
        +create(config) MatchTreeFactoryCb
        -createTreeMatcher()
        -createListMatcher()
        -createAnyMatcher()
        -createFieldMatcher()
        -createMapMatcher()
        -createOnMatchBase()
        -createInputMatcher()
    }
    class MatchInputFactory~DataType~ {
        -ValidationVisitor& validator_
        -MatchTreeValidationVisitor& validation_visitor_
        +createDataInput(config) DataInputFactoryCb
    }
    class MatchTreeValidationVisitor~DataType~ {
        +validateDataInput(input, type_url)
        +validateOnMatch(on_match)
        +errors() vector~Status~
        #performDataInputValidation()* Status
    }
    class ActionBase~ProtoType_Base~ {
        +typeUrl() string_view
        +staticTypeUrl()$ string_view
    }

    OnMatchFactory~DataType~ <|.. MatchTreeFactory~DataType_Ctx~
    MatchTreeFactory~DataType_Ctx~ o-- MatchInputFactory~DataType~
    MatchTreeFactory~DataType_Ctx~ ..> MatchTreeValidationVisitor~DataType~ : uses
    Action <|-- ActionBase~ProtoType_Base~
```

---

## 6. Factory typed-registry interfaces

These extend `Config::TypedFactory` so consumers can register protocol-specific pieces:

```mermaid
classDiagram
    direction LR
    class TypedFactory { <<interface>> }
    class DataInputFactory~DataType~ { +createDataInputFactoryCb() }
    class CommonProtocolInputFactory { +createCommonProtocolInputFactoryCb() }
    class InputMatcherFactory { +createInputMatcherFactoryCb() }
    class ActionFactory~Ctx~ { +createAction() }
    class CustomMatcherFactory~DataType~ { +createCustomMatcherFactoryCb() }

    TypedFactory <|-- DataInputFactory~DataType~
    TypedFactory <|-- CommonProtocolInputFactory
    TypedFactory <|-- InputMatcherFactory
    TypedFactory <|-- ActionFactory~Ctx~
    TypedFactory <|-- CustomMatcherFactory~DataType~
```

---

## 7. Actions subfolder & helpers

```mermaid
classDiagram
    direction TB
    class Action { <<interface>> }
    class StringReturningAction {
        +getOutputString(stream_info)* string
    }
    class RegexReplace {
        -CompiledMatcherPtr regex_
        -string substitution_
        +create(engine, proto)$ StatusOr~RegexReplace~
        +apply(in) string
    }
    Action <|-- StringReturningAction
```

---

## Key type aliases

| Alias | Definition |
|---|---|
| `MatchTreePtr<DataType>` | `unique_ptr<MatchTree<DataType>>` |
| `MatchTreeSharedPtr<DataType>` | `shared_ptr<MatchTree<DataType>>` |
| `MatchTreeFactoryCb<DataType>` | `function<MatchTreePtr<DataType>()>` |
| `FieldMatcherPtr<DataType>` | `unique_ptr<FieldMatcher<DataType>>` |
| `DataInputPtr<DataType>` | `unique_ptr<DataInput<DataType>>` |
| `InputMatcherPtr` | `unique_ptr<InputMatcher>` |
| `ActionConstSharedPtr` | `shared_ptr<const Action>` |
| `OnMatchFactoryCb<DataType>` | `function<OnMatch<DataType>()>` |

---

## Legend

- `<<interface>>` — pure virtual (mostly in `envoy/matcher/matcher.h`).
- `<<abstract>>` — has pure-virtual members but also shared logic (`MapMatcher`).
- `<|..` implements; `<|--` inherits; `o--` aggregates; `..>` uses.
- `~DataType~` denotes a template parameter.

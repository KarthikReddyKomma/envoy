# Envoy Formatter Architecture

## Overview

The `source/common/formatter/` directory implements Envoy's **substitution formatter** — the engine that powers access logs, header values, and any string template containing `%COMMAND(SUBCOMMAND):LENGTH%` tokens.

The formatter takes a **format string** (or a JSON struct) plus runtime data (`Context` + `StreamInfo`) and produces a rendered output string. It supports two output shapes (text and JSON), pluggable command parsers (extensions register new tokens), and per-token max-length truncation.

---

## 1. High-Level Architecture

```mermaid
graph TB
    subgraph "Configuration Time"
        Config[SubstitutionFormatString proto]
        TextFmt[text_format: 'string template']
        JsonFmt[json_format: Protobuf::Struct]
        TextSrc[text_format_source: DataSource]

        Config --> TextFmt
        Config --> JsonFmt
        Config --> TextSrc
    end

    subgraph "Parsing"
        Util[SubstitutionFormatStringUtils<br/>fromProtoConfig]
        ParseFmters[parseFormatters<br/>extension command parsers]
        FmtImpl[FormatterImpl::create]
        JsonImpl[createJsonFormatter]

        Util --> ParseFmters
        Util --> FmtImpl
        Util --> JsonImpl
    end

    subgraph "Format Engine"
        SubParser[SubstitutionFormatParser::parse<br/>tokenize % … %]
        Regex[commandWithArgsRegex<br/>%COMMAND(SUB):LEN%]
        UserParsers[User CommandParsers]
        BuiltIn[BuiltInCommandParsers<br/>HTTP / StreamInfo]

        SubParser --> Regex
        SubParser --> UserParsers
        SubParser --> BuiltIn
    end

    subgraph "Runtime"
        Formatter[Formatter<br/>FormatterImpl / JsonFormatterImpl]
        Providers[FormatterProviders<br/>Plain / Header / StreamInfo / …]
        Output[Rendered string]

        Formatter --> Providers
        Providers --> Output
    end

    Config --> Util
    Util --> Formatter
    SubParser -.produces.-> Providers
    FmtImpl --> SubParser
    JsonImpl --> SubParser

    style Config fill:#e1f5ff
    style Output fill:#c8e6c9
    style Regex fill:#fff9c4
```

---

## 2. The Two Pipelines: Text vs JSON

### 2.1 Text pipeline

```mermaid
flowchart LR
    A[text_format string] --> B[SubstitutionFormatParser::parse]
    B --> C[std::vector FormatterProviderPtr]
    C --> D[FormatterImpl]
    D -- format ctx, info --> E{for each provider}
    E --> F[provider->format]
    F --> G{value?}
    G -- yes --> H[append value]
    G -- no, !omit --> I[append '-']
    G -- no, omit --> J[skip]
    H --> K[concatenated string]
    I --> K
    J --> K
```

- The format string is split at `%…%` boundaries.
- Each non-`%` run becomes a `PlainStringFormatter`.
- Each `%COMMAND(...)%` becomes a typed `FormatterProvider` (header, stream-info, etc.).
- At runtime, every provider's `format(ctx, info)` is called and the strings are concatenated.

### 2.2 JSON pipeline

```mermaid
flowchart LR
    A[Protobuf Struct] --> B[JsonFormatBuilder::fromStruct]
    B --> C[FormatElements: raw JSON pieces + template strings]
    C --> D[JsonFormatterImpl ctor]
    D --> E[For each template element:<br/>SubstitutionFormatParser::parse]
    E --> F[ParsedFormatElement variants:<br/>string OR Formatters]
    F --> G[JsonFormatterImpl::format]
    G --> H{Element type?}
    H -- raw string --> I[append directly]
    H -- 1 provider --> J[formatValue + appendValueToString<br/>preserves types]
    H -- N providers --> K[stringValueToLogLine<br/>quoted, sanitized]
    I --> L[Final JSON line]
    J --> L
    K --> L
```

- A `Protobuf::Struct` is walked once at config time. Map keys are sorted for deterministic output.
- Pure data (numbers, booleans, strings without `%`) is serialized as **raw JSON pieces**.
- String values containing `%` are kept as **template strings** parsed into `Formatters`.
- At runtime, raw JSON is appended verbatim; templates render their `FormatterProvider`s and emit a quoted, sanitized JSON string. A single-provider template can use `formatValue` to keep numeric / bool / null typing.

---

## 3. Command Token Anatomy

Format: `%COMMAND(SUBCOMMAND):LENGTH%`

```mermaid
graph LR
    A["%REQ(:authority):16%"] --> B[regex match]
    B --> C[COMMAND = REQ<br/>group 1]
    B --> D[SUBCOMMAND = :authority<br/>group 2]
    B --> E[LENGTH = 16<br/>group 3]

    C --> F[lookup parser]
    D --> G[parse subcommand<br/>e.g. main:alt header]
    E --> H[max truncation length]

    F --> I[FormatterProvider]
    G --> I
    H --> I

    style A fill:#e1f5ff
    style I fill:#c8e6c9
```

### 3.1 Regex (from `substitution_formatter.cc`)

```text
^%((?:[A-Z]|[0-9]|_)+)(?:\((.*?)\))?(?::([0-9]+))?%
   |__________________|     |___|        |______|
        COMMAND               SUB           LEN
```

- COMMAND: `[A-Z0-9_]+`, mandatory.
- SUBCOMMAND: any chars except `)`, optional, in parens.
- LENGTH: digits, optional, after a colon.

### 3.2 Syntax flags (`CommandSyntaxChecker::CommandSyntaxFlags`)

| Flag | Meaning |
|------|---------|
| `COMMAND_ONLY` | No params, no length (e.g. `%PROTOCOL%`) |
| `PARAMS_REQUIRED` | Subcommand must be present (e.g. `%REQ(...)%`) |
| `PARAMS_OPTIONAL` | Subcommand may be present (e.g. `%START_TIME%`, `%START_TIME(...)%`) |
| `LENGTH_ALLOWED` | `:N` truncation suffix is allowed |

`CommandSyntaxChecker::verifySyntax` is called by every command parser before constructing a provider.

### 3.3 Parsing loop in `SubstitutionFormatParser::parse`

```mermaid
sequenceDiagram
    autonumber
    participant Caller
    participant Parser as SubstitutionFormatParser
    participant Regex
    participant UserCmds as User CommandParsers
    participant Built as BuiltIn CommandParserFactoryHelper
    participant Out as formatters[]

    Caller->>Parser: parse(format, command_parsers)
    loop for each char
        alt char != '%'
            Parser->>Parser: append to current_token
        else escape '%%'
            Parser->>Parser: append '%' to current_token
        else start of command
            Parser->>Out: push PlainStringFormatter(current_token)
            Parser->>Regex: RE2::Consume(sub_format, ...)
            Regex-->>Parser: command, command_arg, max_len
            loop user parsers (override built-ins)
                Parser->>UserCmds: parse(cmd, arg, len)
                UserCmds-->>Parser: provider | nullptr
            end
            opt none matched
                Parser->>Built: parse(cmd, arg, len)
                Built-->>Parser: provider | nullptr
            end
            alt provider returned
                Parser->>Out: push provider
            else nothing matched
                Parser-->>Caller: InvalidArgumentError
            end
        end
    end
    Parser->>Out: push final PlainStringFormatter(current_token)
    Parser-->>Caller: vector<FormatterProviderPtr>
```

Important subtleties:
- **`%%` escaping** is handled before the regex.
- **User command parsers run first**, so extensions can override built-ins.
- An empty / trailing literal segment still produces a `PlainStringFormatter` (so the renderer can iterate uniformly).
- An unknown command is a hard parse error — discovered at config load, not at runtime.

---

## 4. Formatter Class Hierarchy

```mermaid
classDiagram
    class Formatter {
        <<interface>>
        +format(ctx, info) string
    }
    class FormatterImpl {
        -providers_: vector<FormatterProviderPtr>
        -omit_empty_values_: bool
        +create(format, omit_empty, parsers)
        +format(ctx, info) string
    }
    class JsonFormatterImpl {
        -parsed_elements_: vector<ParsedFormatElement>
        -omit_empty_values_: bool
        +format(ctx, info) string
    }

    class FormatterProvider {
        <<interface>>
        +format(ctx, info) optional<string>
        +formatValue(ctx, info) Protobuf::Value
    }
    class PlainStringFormatter
    class PlainNumberFormatter
    class CoalesceFormatter

    class HttpProviders {
        <<group>>
        RequestHeaderFormatter
        ResponseHeaderFormatter
        ResponseTrailerFormatter
        QueryParameterFormatter
        QueryParametersFormatter
        PathFormatter
        GrpcStatusFormatter
        TraceIDFormatter / SpanIDFormatter
        LocalReplyBodyFormatter
        AccessLogTypeFormatter
        HeadersByteSizeFormatter
    }
    class StreamInfoFormatterProvider {
        <<abstract>>
        +format(info) optional<string>
        +formatValue(info) Protobuf::Value
    }
    class StreamInfoProviders {
        <<group>>
        DynamicMetadataFormatter
        ClusterMetadataFormatter
        UpstreamHostMetadataFormatter
        FilterStateFormatter
        CommonDurationFormatter
        SystemTimeFormatter / StartTimeFormatter
        EnvironmentFormatter
        RequestedServerNameFormatter
    }

    Formatter <|-- FormatterImpl
    Formatter <|-- JsonFormatterImpl
    FormatterProvider <|-- PlainStringFormatter
    FormatterProvider <|-- PlainNumberFormatter
    FormatterProvider <|-- CoalesceFormatter
    FormatterProvider <|-- HttpProviders
    FormatterProvider <|-- StreamInfoFormatterProvider
    StreamInfoFormatterProvider <|-- StreamInfoProviders
    FormatterImpl o-- FormatterProvider : holds N
    JsonFormatterImpl o-- FormatterProvider : holds N
```

Key splits:
- `FormatterProvider` always sees the full `Context` (which carries headers + trailers + access-log type + local reply body) and `StreamInfo`.
- `StreamInfoFormatterProvider` is a convenience base for tokens that ignore the HTTP context — it forwards `format(ctx, info)` to a single-arg `format(info)` overload.

---

## 5. Built-In Command Parsers

Two factories register at static init via `REGISTER_FACTORY`:

```mermaid
graph LR
    A[BuiltInCommandParserFactoryHelper::commandParsers]
    A --> B[DefaultBuiltInHttpCommandParserFactory<br/>name = envoy.built_in_formatters.http.default]
    A --> C[DefaultBuiltInStreamInfoCommandParserFactory<br/>name = envoy.built_in_formatters.stream_info.default]

    B --> B1[BuiltInHttpCommandParser<br/>lookup table]
    C --> C1[BuiltInStreamInfoCommandParser<br/>lookup table]

    B1 --> B2[REQ / REQUEST_HEADER]
    B1 --> B3[RESP / RESPONSE_HEADER]
    B1 --> B4[TRAILER / RESPONSE_TRAILER]
    B1 --> B5[GRPC_STATUS / GRPC_STATUS_NUMBER]
    B1 --> B6[QUERY_PARAM / QUERY_PARAMS / PATH]
    B1 --> B7[TRACE_ID / SPAN_ID]
    B1 --> B8[LOCAL_REPLY_BODY / ACCESS_LOG_TYPE]
    B1 --> B9[REQUEST_HEADERS_BYTES / …]
    B1 --> B10[COALESCE — recursive]

    C1 --> C2[DYNAMIC_METADATA]
    C1 --> C3[CLUSTER_METADATA]
    C1 --> C4[UPSTREAM_HOST_METADATA]
    C1 --> C5[FILTER_STATE]
    C1 --> C6[START_TIME / EMIT_TIME]
    C1 --> C7[DURATION / COMMON_DURATION]
    C1 --> C8[ENVIRONMENT]
    C1 --> C9[REQUESTED_SERVER_NAME]
    C1 --> C10[DOWNSTREAM_PEER_CERT_V_START / V_END]
    C1 --> C11[UPSTREAM_PEER_CERT_V_START / V_END]
```

`BuiltIn{Http,StreamInfo}CommandParser::parse(command, sub, len)`:

1. Look up `command` in a static `CONSTRUCT_ON_FIRST_USE` table.
2. If absent, return `nullptr` (allows other parsers to try).
3. Verify syntax flags via `CommandSyntaxChecker::verifySyntax`.
4. Invoke the captured factory lambda with `(subcommand, max_length)` to construct the `FormatterProvider`.

---

## 6. Header & Length Helpers

```mermaid
graph TB
    subgraph "HeaderFormatter (mixin)"
        F[findHeader]
        F --> M{main_header present?}
        M -- yes --> R[return main_header value]
        M -- no --> A{alternative configured?}
        A -- yes --> AH{alt_header present?}
        AH -- yes --> RA[return alt_header value]
        AH -- no --> N[return nullptr]
        A -- no --> N
    end

    subgraph "RequestHeaderFormatter"
        RHF[uses ctx.requestHeaders]
    end
    subgraph "ResponseHeaderFormatter"
        RPF[uses ctx.responseHeaders]
    end
    subgraph "ResponseTrailerFormatter"
        RTF[uses ctx.responseTrailers]
    end

    HeaderFormatter --> RHF
    HeaderFormatter --> RPF
    HeaderFormatter --> RTF

    RHF --> Trunc[truncateStringView max_length_]
    RPF --> Trunc
    RTF --> Trunc
```

- `parseSubcommandHeaders("X?Y")` splits on `?` to populate `(main, alternative)`.
- `truncate / truncateStringView` are no-ops when `max_length` is unset.
- `parseSubcommand(sub, ':', tok1, tok2, ..., tail_vector)` is a variadic helper that scans `:`-separated subcommands and binds each token, dumping leftovers into an optional vector at the end.

---

## 7. CoalesceFormatter — first non-null wins

`CoalesceFormatter` is itself a `FormatterProvider`. Configured as a JSON object:

```json
{
  "operators": [
    "REQUESTED_SERVER_NAME",
    {"command": "REQ", "param": ":authority"},
    {"command": "REQ", "param": "host"}
  ]
}
```

```mermaid
flowchart TD
    Cfg[JSON config string<br/>operators array] --> Parse[CoalesceFormatter::create]
    Parse --> ParseEach[parseOperatorEntry]
    ParseEach --> CmdLookup[createFormatterForCommand<br/>uses BuiltIn parsers]
    CmdLookup --> Vec[vector FormatterProviderPtr]

    Vec --> Run[CoalesceFormatter::format]
    Run --> Loop{for each provider}
    Loop --> Try[provider->format ctx info]
    Try --> Has{has value?}
    Has -- yes --> Trunc[apply max_length]
    Trunc --> Out[return string]
    Has -- no --> Loop
    Loop -.exhausted.-> NullOut[return nullopt]

    style Out fill:#c8e6c9
    style NullOut fill:#ffe1e1
```

Caveats:
- Each operator entry can be either a bare string (command name only) or `{command, param, max_length}` object.
- The JSON `param` cannot contain literal `)` because the surrounding parser regex would close prematurely.
- `COALESCE` is itself registered as a built-in HTTP command, enabling nested fallback chains.

---

## 8. Configuration Entry Points

```mermaid
graph TB
    Caller[Filter / Access logger / Header formatter]
    Cfg[envoy.config.core.v3.SubstitutionFormatString]

    Caller --> Util[SubstitutionFormatStringUtils::fromProtoConfig]
    Cfg --> Util

    Util --> ParseExt[parseFormatters extensions]
    ParseExt --> Factory[Config::Utility::getFactory<CommandParserFactory>]
    Factory --> CmdParser[CommandParserPtr]

    Util --> Switch{config.format_case}
    Switch -- text_format --> TF[FormatterImpl::create]
    Switch -- json_format --> JF[createJsonFormatter]
    Switch -- text_format_source --> DS[DataSource::read]
    DS --> TF
    Switch -- FORMAT_NOT_SET --> Panic[PANIC_DUE_TO_PROTO_UNSET]

    TF --> Out[FormatterPtr]
    JF --> Out

    style Out fill:#c8e6c9
    style Panic fill:#ff8888
```

- **Extension resolution**: `parseFormatters` walks `config.formatters()` (a list of `TypedExtensionConfig`), looks up each via `Config::Utility::getFactory<CommandParserFactory>`, and produces a `CommandParserPtr` per entry. Errors are returned, not thrown.
- **Single source of truth**: any caller that wants to load a format from proto config must go through `fromProtoConfig` so that the three input modes (inline text, JSON, file source) are normalized.

---

## 9. JSON Format Builder Internals

`JsonFormatBuilder` walks a `Protobuf::Struct` once and produces an alternating sequence of *raw JSON pieces* and *format template strings*.

```mermaid
graph TB
    Start[fromStruct]
    Start --> Walk[formatValueToFormatElements ProtoDict]
    Walk --> Sort[Sort keys lexicographically]
    Sort --> Iter[For each key,value pair]
    Iter --> Emit[Emit key string + colon delimiter via JsonStringSerializer]
    Emit --> Switch{value kind}
    Switch -- null --> Null[serializer.addNull]
    Switch -- number --> Num[serializer.addNumber]
    Switch -- bool --> Bool[serializer.addBool]
    Switch -- string --> S{contains '%'?}
    Switch -- struct --> Walk
    Switch -- list --> List[Iterate list, recurse]

    S -- no --> RawStr[serializer.addString sanitized]
    S -- yes --> FlushPiece[push raw JSON buffer as element]
    FlushPiece --> PushTpl[push template string element]

    Iter -.next or end.-> Iter
    Walk -.end of map.-> CloseMap[serializer.addMapEndDelimiter]
    Start -.after walk.-> FinalPush[push remaining buffer as last raw element]
```

- Sorting keys makes the output deterministic across builds — important for snapshot tests.
- Strings without `%` are emitted as **pre-sanitized** JSON pieces (escaped + quoted at config time).
- Strings with `%` flush the current buffer then emit a `template` element; sanitization is deferred to runtime because the rendered values are unknown.
- `JsonStringSerializer` is a low-level helper that writes bytes via `Json::StringOutput` and uses `Json::sanitize` for escaping. It duplicates a small piece of `BufferStreamer` to allow finer control of delimiters.

---

## 10. Runtime: How `format(ctx, info)` Renders

### 10.1 Text (`FormatterImpl::format`)

```mermaid
sequenceDiagram
    participant App
    participant FI as FormatterImpl
    participant P as Provider[i]
    participant Out as log_line

    App->>FI: format(ctx, info)
    FI->>Out: reserve(256)
    loop for each provider
        FI->>P: format(ctx, info)
        P-->>FI: optional<string>
        alt has value
            FI->>Out: append(value)
        else no value && !omit_empty_values_
            FI->>Out: append("-")
        else omit_empty_values_
            FI->>Out: skip
        end
    end
    FI-->>App: log_line
```

### 10.2 JSON (`JsonFormatterImpl::format`)

For each `ParsedFormatElement`:
- `string` variant → append raw JSON piece directly (already sanitized).
- `Formatters` with **exactly one** provider → call `formatValue(ctx, info)` and `Json::Utility::appendValueToString`. This preserves number / bool / null types in the output JSON.
- `Formatters` with **multiple** providers → call `stringValueToLogLine`:
  - Emit `"`.
  - Concatenate each provider's `format(ctx, info)` value (sanitized) or `"-"` / `""` for missing values.
  - Emit `"`.
- Append a trailing `\n` after the line is built.

The single-vs-many distinction is what lets `{"status": "%RESPONSE_CODE%"}` produce `{"status":200}` (number) instead of `{"status":"200"}` (string).

---

## 11. End-to-End Render Sequence (Text Example)

Format string: `"%PROTOCOL% %REQ(:authority)% %RESPONSE_CODE%\n"`

```mermaid
sequenceDiagram
    autonumber
    participant Cfg as Config Load
    participant Parser as SubstitutionFormatParser
    participant Built as BuiltIn parsers
    participant FI as FormatterImpl
    participant Run as Runtime call
    participant CTX as Context + StreamInfo
    participant Out

    Cfg->>Parser: parse("%PROTOCOL% %REQ(:authority)% %RESPONSE_CODE%\n")
    Parser->>Built: PROTOCOL → ProtocolFormatter (StreamInfo)
    Parser->>Built: REQ(:authority) → RequestHeaderFormatter
    Parser->>Built: RESPONSE_CODE → ResponseCodeFormatter (StreamInfo)
    Parser-->>Cfg: [Plain "", Protocol, Plain " ", ReqHdr, Plain " ", RespCode, Plain "\n"]
    Cfg->>FI: store providers_

    Note over Run,Out: At access-log time
    Run->>FI: format(ctx, info)
    FI->>CTX: provider 1: protocol = HTTP/2
    FI->>Out: append "HTTP/2"
    FI->>Out: append " "
    FI->>CTX: provider 3: req[:authority] = "api.example.com"
    FI->>Out: append "api.example.com"
    FI->>Out: append " "
    FI->>CTX: provider 5: response_code = 200
    FI->>Out: append "200"
    FI->>Out: append "\n"
    FI-->>Run: "HTTP/2 api.example.com 200\n"
```

---

## 12. Extension Points

```mermaid
graph TB
    subgraph "User Extension"
        UF[Extension Factory<br/>CommandParserFactory]
        UP[Custom CommandParser]
        UV[Custom FormatterProvider]
    end

    subgraph "Config"
        TC[TypedExtensionConfig in formatters[] list]
    end

    subgraph "Engine"
        Util[parseFormatters]
        SP[SubstitutionFormatParser::parse]
    end

    TC --> Util
    Util -- getFactory + createCommandParserFromProto --> UF
    UF --> UP
    UP -- parse cmd, sub, len --> UV
    SP -- "tries user parsers first" --> UP

    style UF fill:#fff9c4
    style UV fill:#c8e6c9
```

- A user extension only needs to implement `CommandParserFactory` and `CommandParser`. The latter returns `nullptr` for commands it doesn't recognize.
- User parsers run **before** built-ins in `SubstitutionFormatParser::parse`, allowing extensions to override default behavior.
- The same parser list is threaded through `JsonFormatterImpl`, so JSON configs see the same extension surface.

---

## 13. Quick Reference

| Concept | Class / Function | File |
|---------|------------------|------|
| Top-level config conversion | `SubstitutionFormatStringUtils::fromProtoConfig` | `substitution_format_string.cc` |
| Tokenization | `SubstitutionFormatParser::parse` | `substitution_formatter.cc` |
| Text renderer | `FormatterImpl` | `substitution_formatter.{h,cc}` |
| JSON renderer | `JsonFormatterImpl` + `JsonFormatBuilder` | `substitution_formatter.cc` |
| Syntax flags | `CommandSyntaxChecker` | `substitution_format_utility.h` |
| Header / `?` parsing | `SubstitutionFormatUtils::parseSubcommandHeaders` | `substitution_format_utility.{h,cc}` |
| Generic `:`-split parser | `SubstitutionFormatUtils::parseSubcommand` | `substitution_format_utility.h` |
| Built-in HTTP commands | `BuiltInHttpCommandParser` | `http_specific_formatter.{h,cc}` |
| Built-in StreamInfo commands | `BuiltInStreamInfoCommandParser` | `stream_info_formatter.{h,cc}` |
| First-non-null operator | `CoalesceFormatter` | `coalesce_formatter.{h,cc}` |
| Default access-log format | `HttpSubstitutionFormatUtils::defaultSubstitutionFormatter` | `http_formatter_context.{h,cc}` |


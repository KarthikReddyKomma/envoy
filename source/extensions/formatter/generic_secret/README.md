# Generic Secret Formatter

Adds the `%SECRET(name)%` substitution command. Looks up a named generic
secret from either SDS or static bootstrap resources and emits its current
value. Useful for injecting rotating tokens into outbound headers or
access logs.

Proto: `envoy.extensions.formatter.generic_secret.v3.GenericSecret`.
Factory name: `envoy.formatter.generic_secret`.

## Files
- `config.h/cc` - `GenericSecretFormatterFactory` plus the
  anonymous-namespace `GenericSecretFormatterProvider` and
  `GenericSecretCommandParser`.

## Interface
- Factory base: `Envoy::Formatter::CommandParserFactory`.
- Command parser base: `Envoy::Formatter::CommandParser`.
- Formatter provider base: `Envoy::Formatter::FormatterProvider`.

## Logic
- `createCommandParserFromProto` asserts it's running on the main thread
  (secret providers create thread locals), then iterates
  `typed_config.secret_configs()`:
  - If the entry has an `sds_config`, it subscribes via
    `server_context.secretManager().findOrCreateGenericSecretProvider`.
  - Otherwise it resolves a static provider via
    `findStaticGenericSecretProvider`, throwing if the named resource
    is missing.
  - Each provider is wrapped in a
    `Secret::ThreadLocalGenericSecretProvider` and stored in a
    `ProviderMap` keyed by the configured name.
- `GenericSecretCommandParser::parse` only handles the literal
  `SECRET` command. Unknown names throw
  `EnvoyException("secret '%s' is not configured in secret_configs")`
  at parse time so log configs fail loudly.
- At runtime, `format()` returns the current secret value (truncated to
  `max_length`) or `absl::nullopt` if the secret is empty.

## Key decision points
- `config.cc:107` - looking up the static secret immediately at config
  time detects missing bootstrap secrets before any traffic hits the
  formatter.
- `config.cc:123` - `ThreadLocalGenericSecretProvider::create` is the
  single touch-point for thread local allocation, so the assertion on
  `ASSERT_IS_MAIN_OR_TEST_THREAD` is required to satisfy Envoy's TLS
  invariants.
- The provider holds `shared_ptr` ownership so the formatter survives
  secret rotation callbacks; rotations update the stored value
  in-place.

## Configuration
- `secret_configs` - map of command name to
  `envoy.extensions.transport_sockets.tls.v3.SdsSecretConfig`. The key
  is the name used in `%SECRET(name)%`, not the secret resource name.

## Stats / errors
- None. Missing secrets fail loudly at parse time; empty values return
  `absl::nullopt` at runtime.

# Generic Credential Injector

A `CredentialInjector` plugin consumed by the `credential_injector` HTTP
filter (`source/extensions/filters/http/credential_injector`). It injects a
static secret (loaded from SDS) into a configurable request header, with
an optional value prefix - suitable for bearer tokens, API keys, Basic
auth pre-baked blobs, etc.

Proto: `envoy.extensions.http.injected_credentials.generic.v3.Generic`.

## Files
- `generic_impl.h/cc` - `GenericCredentialInjector`.
- `config.h/cc` - factory that wires SDS into the injector.

## Interface
- Implements `Extensions::Http::InjectedCredentials::Common::CredentialInjector`.
- Factory is a
  `Common::CredentialInjectorFactoryBase<Generic>`, registered via
  `REGISTER_FACTORY` under
  `NamedCredentialInjectorConfigFactory`.

## Logic
- At config load, `config.cc:secretsProvider` resolves the configured
  secret either as a dynamic SDS secret (`sds_config` present) or a static
  generic secret registered in `SecretManager`; the result is wrapped in
  a `Common::SDSSecretReader` for hot-path reads.
- The injector stores the target header name (defaulting to
  `Authorization` when unset, see `config.cc:38`), the value prefix, and
  the reader.
- `inject(headers, overwrite)`:
  1. If `overwrite` is false and the header already exists, returns
     `AlreadyExistsError` - the filter treats this as an authenticated
     pass-through.
  2. If the secret is empty (SDS not yet loaded or cleared), returns
     `NotFoundError`.
  3. Otherwise sets the header to `prefix + secret` via `setCopy`.

## Key decision points
- `config.cc:38` - default header is `Authorization` when
  `config.header()` is empty.
- `generic_impl.cc:13` - overwrite semantics: `get(header).empty()` is the
  existence check; multi-value headers count as existing.
- `generic_impl.cc:17` - empty secret is treated as an error (SDS not
  ready).
- `generic_impl.cc:21` - value is always `prefix + secret` (prefix may be
  empty, e.g. `"Bearer "` for bearer tokens).

## Configuration
- `credential` - SDS `SdsSecretConfig` for the secret.
- `header` - header name; defaults to `Authorization`.
- `header_value_prefix` - string prepended to the secret value.

## Stats / errors
No dedicated stats. The filter tracks `injected`, `already_exists`,
`failed` counters based on the `absl::Status` returned here.

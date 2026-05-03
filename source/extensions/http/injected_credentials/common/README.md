# Injected Credentials - Common

Shared interfaces and helpers for `credential_injector` HTTP filter
plugins. The `credential_injector` filter
(`source/extensions/filters/http/credential_injector`) delegates to a typed
credential extension selected by configuration; every such extension
implements the interface declared here and typically reuses the SDS-backed
secret reader and the templated factory base. This directory contains no
runtime logic of its own - it is the extension SPI.

Proto: none. The consumers live under
`envoy.extensions.http.injected_credentials.*` (generic, oauth2, ...).

## Files
- `credential.h` - abstract `CredentialInjector` interface.
- `factory.h` - `NamedCredentialInjectorConfigFactory` (registry category
  `envoy.http.injected_credentials`).
- `factory_base.h` - `CredentialInjectorFactoryBase<ConfigProto>`, a
  templated helper that handles proto downcast / validation boilerplate.
- `secret_reader.h` - `SecretReader` interface plus `SDSSecretReader`, a
  thread-local-backed implementation that reads from an SDS
  `GenericSecretConfigProvider`.

## Interface
- `Common::CredentialInjector::inject(RequestHeaderMap&, bool overwrite)`
  is the single runtime method every credential extension must implement;
  it returns an `absl::Status` used by the filter to decide whether to
  continue or fail the request.
- `Common::NamedCredentialInjectorConfigFactory::createCredentialInjectorFromProto`
  is the factory contract. The filter's config loader calls
  `Envoy::Config::Utility::getAndCheckFactory<NamedCredentialInjectorConfigFactory>`
  with the configured type URL, validates the proto, and invokes this to
  build a `CredentialInjectorSharedPtr`.
- `Common::SecretReader::credential()` returns the current secret value;
  concrete extensions do not call into SDS themselves - they consume a
  `SecretReaderConstSharedPtr`.

## Logic
- `SDSSecretReader` builds a `ThreadLocalGenericSecretProvider` in its
  ctor, so `credential()` becomes a lock-free TLS read on the hot path
  (`secret_reader.h:26`). The provider rotates the secret when SDS pushes
  a new version.
- `CredentialInjectorFactoryBase<ConfigProto>` supplies
  `createCredentialInjectorFromProto` by downcasting to the concrete
  proto and forwarding to `createCredentialInjectorFromProtoTyped`, which
  subclasses override (`factory_base.h:36`). It also implements
  `createEmptyConfigProto` and `name`, which factories almost never need
  to customize.

## Key decision points
- `factory.h:20` - registry category name
  `envoy.http.injected_credentials` is the string the filter uses to find
  typed factories.
- `secret_reader.h:20` - `SDSSecretReader` is constructed from a
  `GenericSecretConfigProviderSharedPtr`, which is how SDS dynamic secret
  rotation reaches each extension.
- `credential.h:22` - `overwrite` flag decides whether to stomp an
  existing credential header; every concrete injector honors this.

## Configuration
No proto here; each concrete extension owns its own config.

## Stats / errors
No stats in common. `inject` returns standard absl statuses;
`AlreadyExistsError` is the conventional signal for "credential already
present and overwrite=false".

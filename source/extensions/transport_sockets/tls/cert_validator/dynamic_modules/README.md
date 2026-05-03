# Dynamic Modules Certificate Validator (`envoy.tls.cert_validator.dynamic_modules`)

Certificate validator that delegates the verification step to a natively loaded dynamic module (shared library). The module implements a fixed ABI declared in `source/extensions/dynamic_modules/abi/abi.h`; Envoy resolves five symbols on load and invokes them at handshake time. The module also gets access to the connection's filter state via Envoy-provided C callbacks, enabling validators that need connection-scoped context (e.g. host-based policy, client identity).

Proto: `envoy.extensions.transport_sockets.tls.cert_validator.dynamic_modules.v3.DynamicModuleCertValidatorConfig` (fields: `dynamic_module_config` with module `name` / `do_not_close` / `load_globally`, `validator_name`, opaque `validator_config`).

## Files
- `config.h` — declares:
  - `DynamicModuleCertValidatorConfig` — shared state holding the loaded `DynamicModulePtr`, resolved ABI function pointers, the module's in-module config pointer, plus a transient `last_error_details_` and `current_callbacks_` used as thread-local scratch during a validation call.
  - `newDynamicModuleCertValidatorConfig` — resolves the five ABI symbols and initializes the module's in-module config.
  - `DynamicModuleCertValidator` — the `CertValidator` implementation that marshals cert chains to DER and calls the module.
  - `DynamicModuleCertValidatorFactory` — the `CertValidatorFactory` registered under `envoy.tls.cert_validator.dynamic_modules`.
- `config.cc` — implements all of the above plus three `extern "C"` callbacks the module calls back into:
  - `envoy_dynamic_module_callback_cert_validator_set_error_details` (`config.cc:18`) — stores an error string into `config.last_error_details_` for Envoy to surface.
  - `envoy_dynamic_module_callback_cert_validator_set_filter_state` (`config.cc:29`) — writes a string to the connection's filter state (read-only, connection-scoped). Requires `current_callbacks_` to be set (i.e. only valid during a `do_verify_cert_chain` call).
  - `envoy_dynamic_module_callback_cert_validator_get_filter_state` (`config.cc:50`) — reads a string from the connection's filter state.

## Interface / implementation
`DynamicModuleCertValidator::doVerifyCertChain` (`config.cc:165`):
1. Rejects an empty chain with `ClientValidationStatus::NoClientCertificate`.
2. DER-encodes every `X509*` via `i2d_X509` into `der_certs`, then wraps each in an `envoy_dynamic_module_type_envoy_buffer` (`config.cc:180-197`).
3. Resets `last_error_details_`, sets `current_callbacks_ = validation_context.callbacks` so the module's callbacks can reach the connection filter state.
4. Calls `on_do_verify_cert_chain_` with `cert_buffers`, `host_name`, and `is_server` (`config.cc:210-213`).
5. Clears `current_callbacks_`, translates the status (`Successful` vs `Failed`) and detailed status (`NotValidated` / `Validated` / `NoClientCertificate` / `Failed`), extracts an optional TLS alert, and returns a `ValidationResults` with `last_error_details_`.
6. On failure, increments `stats_.fail_verify_error_`.

`addClientValidationContext` (`config.cc:154`) sets `SSL_VERIFY_PEER [| SSL_VERIFY_FAIL_IF_NO_PEER_CERT]` based on `require_client_cert`. Unlike the default validator it does *not* populate the client CA list — the module is expected to handle that if relevant.

`initializeSslContexts` (`config.cc:258`) delegates the `SSL_VERIFY_*` decision to the module via `on_get_ssl_verify_mode_`.

`updateDigestForSessionId` (`config.cc:266`) calls `on_update_digest_` to collect module-specific state, then also hashes the `validator_name` and `validator_config` strings so two different module configurations never share a session ID.

`daysUntilFirstCertExpires` returns `nullopt`; `getCaFileName` returns `""`; `getCaCertInformation` returns `nullptr`. The extension does not expose admin-plane CA details because the module may not store certs in a form Envoy can introspect.

## Factory
`DynamicModuleCertValidatorFactory::createCertValidator` (`config.cc:299`):
- Translates `customValidatorConfig().typed_config()` into the `DynamicModuleCertValidatorConfig` proto.
- Loads the module via `Envoy::Extensions::DynamicModules::newDynamicModuleByName`.
- Serializes the opaque `validator_config` to bytes.
- Calls `newDynamicModuleCertValidatorConfig` (`config.cc:102`) which resolves all five ABI symbols (`_config_new`, `_config_destroy`, `_do_verify_cert_chain`, `_get_ssl_verify_mode`, `_update_digest`) and invokes `_config_new` to create the module's per-validator config. Fails with `InvalidArgumentError` if the module returns null.

Destruction: `DynamicModuleCertValidatorConfig::~DynamicModuleCertValidatorConfig` (`config.cc:95`) calls the module's `on_config_destroy_` to tear down the in-module config before the `DynamicModulePtr` is released.

## Lifecycle
- Config load: module is loaded once per `DynamicModuleCertValidatorFactory::createCertValidator` invocation; the module receives the opaque `validator_config` and can parse it internally.
- Per handshake: the handshaker calls `doVerifyCertChain`, which is a synchronous call into the module.
- Config shutdown: destructor order guarantees the module's `config_destroy` is called before the dylib is closed (unless `do_not_close` is set in the module config).

## Configuration
- `dynamic_module_config.name` — module name / path. `do_not_close` keeps the dylib loaded across config refreshes; `load_globally` uses `RTLD_GLOBAL`.
- `validator_name` — free-form identifier passed into the module to select a variant.
- `validator_config` — opaque Any serialized to bytes and passed verbatim to `on_config_new`.

## Stats / errors
- `fail_verify_error` (from common SSL stats) — incremented on any validation failure.
- Module-supplied error strings are logged at `debug` (`config.cc:252`) and returned in the `ValidationResults::error_details`.
- Module load / symbol resolution failures surface as `absl::StatusOr` errors at config creation time.

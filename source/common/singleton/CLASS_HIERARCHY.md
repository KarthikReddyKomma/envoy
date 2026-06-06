# Singleton — class hierarchy (UML)

UML-style Mermaid diagrams for the managed manager and the helper templates. See
[`OVERVIEW.md`](OVERVIEW.md) for behavior.

---

## Managed manager + registration

```mermaid
classDiagram
    class Instance {
        <<interface>>
        +~Instance()
    }

    class Manager {
        <<interface>>
        +getTyped~T~(name, cb, pin) shared_ptr~T~
        +getTyped~T~(name) shared_ptr~T~
        +get(name, cb, pin)* InstanceSharedPtr
    }

    class ManagerImpl {
        -singletons_ : node_hash_map~string, weak_ptr~Instance~~
        -pinned_singletons_ : vector~InstanceSharedPtr~
        +get(name, cb, pin) InstanceSharedPtr
    }

    class UntypedFactory {
        <<interface>>
    }
    class Registration {
        +category() string  // "envoy.singleton"
    }
    class RegistrationImpl~name~ {
        +name() string
    }

    Manager <|.. ManagerImpl
    ManagerImpl ..> Instance : stores weak_ptr / strong (pinned)
    UntypedFactory <|.. Registration
    Registration <|.. RegistrationImpl
    ManagerImpl ..> Registration : validates name via Registry

    note for ManagerImpl "main-thread only (ASSERT)\nweak by default; pin = also strong"
```

### Concrete singletons derive from `Instance`

```mermaid
classDiagram
    class Instance { <<interface>> }
    class SecretManagerImpl
    class TracerManagerImpl
    class SomeProvider

    Instance <|.. SecretManagerImpl
    Instance <|.. TracerManagerImpl
    Instance <|.. SomeProvider

    note for Instance "Any T returned by getTyped&lt;T&gt;\nmust derive from Instance"
```

---

## Header-only singleton templates

```mermaid
classDiagram
    class ConstSingleton~T~ {
        +get()$ const T&
    }

    class ThreadSafeSingleton~T~ {
        #create_once_ : once_flag$
        #instance_ : T*$
        +get()$ T&
        #Create()$
    }

    class InjectableSingleton~T~ {
        #loader_ : atomic~T*~$
        +get()$ T&
        +getExisting()$ T*
        +initialize(T*)$
        +clear()$
        +replaceForTest(T*)$ T*
    }

    class ThreadLocalInjectableSingleton~T~ {
        #loader_ : thread_local T*$
        +get()$ T&
        +initialize(T*)$
        +clear()$
    }

    class ScopedInjectableLoader~T~ {
        -instance_ : unique_ptr~T~
        +instance() T&
    }
    class StackedScopedInjectableLoaderForTest~T~ {
        -instance_ : unique_ptr~T~
        -original_loader_ : T*
    }

    ScopedInjectableLoader ..> InjectableSingleton : initialize() / clear()
    StackedScopedInjectableLoaderForTest ..> InjectableSingleton : replaceForTest()

    note for ConstSingleton "✅ encouraged — immutable"
    note for ThreadSafeSingleton "⚠️ discouraged — OS shims only"
    note for InjectableSingleton "swappable; needs initialize()"
```

---

## Type & macro reference

| Symbol | Kind | Meaning |
|---|---|---|
| `Singleton::InstanceSharedPtr` | `shared_ptr<Instance>` | What the manager hands out. |
| `Singleton::ManagerPtr` | `unique_ptr<Manager>` | Server owns one. |
| `Singleton::SingletonFactoryCb` | `function<InstanceSharedPtr()>` | Lazy creation callback. |
| `SINGLETON_MANAGER_REGISTRATION(NAME)` | macro | Static-registers `"NAME_singleton"`. |
| `SINGLETON_MANAGER_REGISTERED_NAME(NAME)` | macro | Expands to the registered name symbol. |

---

## Relationship summary

| Relationship | Type | Meaning |
|---|---|---|
| `ManagerImpl` → `Instance` | weak (default) / strong (pinned) | Storage of managed singletons. |
| `ManagerImpl` → `Registration` | validation | Name must be in the registry. |
| `ScopedInjectableLoader` → `InjectableSingleton` | RAII | Install in ctor, clear in dtor. |
| `RegistrationImpl<name>` → `Registration` → `UntypedFactory` | inheritance | Static factory registration. |

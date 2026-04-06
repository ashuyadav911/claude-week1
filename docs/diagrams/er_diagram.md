# Pydantic — Entity-Relationship Diagram

```mermaid
erDiagram

    %% ── Core model layer ──────────────────────────────────────────────────

    BaseModel {
        dict   __pydantic_fields__        "field-name → FieldInfo"
        dict   __pydantic_decorators__    "DecoratorInfos instance"
        dict   __pydantic_core_schema__   "compiled CoreSchema"
        obj    __pydantic_validator__     "SchemaValidator (Rust)"
        obj    __pydantic_serializer__    "SchemaSerializer (Rust)"
        obj    model_config              "ConfigDict"
        dict   __dict__                  "validated field values"
        set    __pydantic_fields_set__   "fields supplied at init"
    }

    RootModel {
        any    root                       "the single root value"
    }

    %% ── Field metadata ────────────────────────────────────────────────────

    FieldInfo {
        type   annotation
        any    default
        func   default_factory
        str    alias
        int    alias_priority
        any    validation_alias           "str | AliasPath | AliasChoices"
        str    serialization_alias
        str    title
        str    description
        list   examples
        bool   exclude
        any    discriminator
        bool   frozen
        bool   validate_default
        bool   repr
        bool   init
        bool   init_var
        bool   kw_only
        list   metadata                   "constraint objects"
    }

    ComputedFieldInfo {
        func   wrapped_property
        type   return_type
        str    alias
        str    title
        str    description
        bool   repr
    }

    ModelPrivateAttr {
        any    default
        func   default_factory
    }

    %% ── Configuration ─────────────────────────────────────────────────────

    ConfigDict {
        str    title
        str    extra                      "allow | ignore | forbid"
        bool   frozen
        bool   strict
        bool   populate_by_name
        bool   validate_assignment
        bool   use_enum_values
        bool   arbitrary_types_allowed
        bool   str_strip_whitespace
        bool   str_to_lower
        bool   str_to_upper
        int    str_min_length
        int    str_max_length
        any    alias_generator
        bool   validate_default
        bool   revalidate_instances
        bool   from_attributes
    }

    %% ── Alias helpers ─────────────────────────────────────────────────────

    AliasPath {
        list   path                       "list[str | int]"
    }

    AliasChoices {
        list   choices                    "list[str | AliasPath]"
    }

    AliasGenerator {
        func   alias
        func   validation_alias
        func   serialization_alias
    }

    %% ── Decorator metadata ────────────────────────────────────────────────

    DecoratorInfos {
        dict   validators                 "V1 @validator"
        dict   field_validators           "@field_validator"
        dict   root_validators            "@root_validator"
        dict   field_serializers          "@field_serializer"
        dict   model_serializers          "@model_serializer"
        dict   model_validators           "@model_validator"
        dict   computed_fields            "@computed_field"
    }

    FieldValidatorDecoratorInfo {
        tuple  fields
        str    mode                       "before | after | plain | wrap"
        bool   check_fields
        any    json_schema_input_type
    }

    ModelValidatorDecoratorInfo {
        str    mode                       "before | after | wrap"
    }

    FieldSerializerDecoratorInfo {
        tuple  fields
        str    mode                       "plain | wrap"
        any    return_type
        str    when_used
        bool   check_fields
    }

    ModelSerializerDecoratorInfo {
        str    mode                       "plain | wrap"
        any    return_type
        str    when_used
    }

    %% ── Network / URL types ───────────────────────────────────────────────

    UrlConstraints {
        int    max_length
        list   allowed_schemes
        bool   host_required
        str    default_host
        int    default_port
        str    default_path
        bool   preserve_empty_path
    }

    AnyUrl {
        str    scheme
        str    host
        int    port
        str    path
        str    query
        str    fragment
    }

    NameEmail {
        str    name
        str    email
    }

    %% ── Schema / validation layer (pydantic-core) ─────────────────────────

    CoreSchema {
        str    type                       "model | str | int | list …"
        dict   schema                     "inner field schemas"
        dict   config
        list   validators
        list   serializers
    }

    SchemaValidator {
        obj    compiled_schema            "Rust bytecode"
    }

    SchemaSerializer {
        obj    compiled_schema            "Rust bytecode"
    }

    %% ── JSON Schema layer ────────────────────────────────────────────────

    GenerateJsonSchema {
        str    schema_dialect
        dict   definitions
        list   warnings
    }

    %% ══════════════════════════════════════════════════════════════════════
    %% Relationships
    %% ══════════════════════════════════════════════════════════════════════

    %% BaseModel owns its parts
    BaseModel ||--o{ FieldInfo            : "has fields"
    BaseModel ||--o{ ComputedFieldInfo    : "has computed_fields"
    BaseModel ||--o{ ModelPrivateAttr     : "has private_attrs"
    BaseModel ||--|| ConfigDict           : "configured by"
    BaseModel ||--|| DecoratorInfos       : "has decorators"
    BaseModel ||--|| CoreSchema           : "described by"
    BaseModel ||--|| SchemaValidator      : "validated by"
    BaseModel ||--|| SchemaSerializer     : "serialized by"

    %% RootModel is a specialised BaseModel
    RootModel ||--|| BaseModel            : "extends"

    %% FieldInfo references alias helpers
    FieldInfo ||--o| AliasPath            : "validation_alias may be"
    FieldInfo ||--o| AliasChoices         : "validation_alias may be"

    %% AliasChoices contains AliasPath entries
    AliasChoices ||--o{ AliasPath         : "contains"

    %% ConfigDict may reference AliasGenerator
    ConfigDict ||--o| AliasGenerator      : "alias_generator"

    %% AliasGenerator produces AliasPath / AliasChoices
    AliasGenerator ||--o{ AliasPath       : "may produce"
    AliasGenerator ||--o{ AliasChoices    : "may produce"

    %% DecoratorInfos groups all decorator info types
    DecoratorInfos ||--o{ FieldValidatorDecoratorInfo    : "field_validators"
    DecoratorInfos ||--o{ ModelValidatorDecoratorInfo    : "model_validators"
    DecoratorInfos ||--o{ FieldSerializerDecoratorInfo   : "field_serializers"
    DecoratorInfos ||--o{ ModelSerializerDecoratorInfo   : "model_serializers"
    DecoratorInfos ||--o{ ComputedFieldInfo              : "computed_fields"

    %% FieldValidatorDecoratorInfo targets FieldInfo by name
    FieldValidatorDecoratorInfo }o--o{ FieldInfo         : "targets fields"
    FieldSerializerDecoratorInfo }o--o{ FieldInfo        : "targets fields"

    %% CoreSchema is compiled into validator / serializer
    CoreSchema ||--|| SchemaValidator     : "compiled into"
    CoreSchema ||--|| SchemaSerializer    : "compiled into"

    %% URL types are constrained by UrlConstraints
    UrlConstraints ||--o{ AnyUrl          : "constrains"

    %% JSON Schema generation consumes CoreSchema
    GenerateJsonSchema ||--|| CoreSchema  : "consumes"
```

## Entity Descriptions

| Entity | Layer | Description |
|--------|-------|-------------|
| `BaseModel` | Public API | Root class every user model inherits; owns fields, config, and compiled validators |
| `RootModel` | Public API | Specialisation of `BaseModel` for single-value models |
| `FieldInfo` | Public API | Full metadata for one field: type, default, alias, constraints, serialization hints |
| `ComputedFieldInfo` | Public API | Metadata for `@computed_field` properties |
| `ModelPrivateAttr` | Public API | Descriptor for `__private__` attributes not included in schema |
| `ConfigDict` | Public API | TypedDict of 50+ knobs controlling validation, serialization & JSON schema behaviour |
| `AliasPath` | Public API | Dot/index path for extracting a nested input value into a field |
| `AliasChoices` | Public API | Ordered list of alias candidates (first match wins) |
| `AliasGenerator` | Public API | Callable trio that derives alias / validation_alias / serialization_alias from a field name |
| `DecoratorInfos` | Internal | Aggregates all decorator metadata collected during class creation |
| `FieldValidatorDecoratorInfo` | Internal | Metadata captured from `@field_validator` |
| `ModelValidatorDecoratorInfo` | Internal | Metadata captured from `@model_validator` |
| `FieldSerializerDecoratorInfo` | Internal | Metadata captured from `@field_serializer` |
| `ModelSerializerDecoratorInfo` | Internal | Metadata captured from `@model_serializer` |
| `CoreSchema` | pydantic-core | Intermediate dict-tree representation of all validation rules |
| `SchemaValidator` | pydantic-core (Rust) | Compiled, optimised validator; called on every `Model(**data)` |
| `SchemaSerializer` | pydantic-core (Rust) | Compiled serializer; called on every `.model_dump()` / `.model_dump_json()` |
| `UrlConstraints` | Network types | Dataclass holding URL validation rules (schemes, host, port, length) |
| `AnyUrl` | Network types | Base validated URL type |
| `NameEmail` | Network types | `"Name <email>"` string parsed into name + email pair |
| `GenerateJsonSchema` | JSON Schema | Walks `CoreSchema` to produce an OpenAPI-compatible JSON Schema dict |

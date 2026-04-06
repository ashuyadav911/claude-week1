# Pydantic Model Validation — Sequence Diagram

Two phases: **class definition** (runs once at import time) and **instance validation** (runs on every `MyModel(**data)` call).

```mermaid
sequenceDiagram
    autonumber

    actor User
    participant Meta  as ModelMetaclass<br/>(_model_construction.py)
    participant Fields as collect_model_fields()<br/>(_fields.py)
    participant GenSchema as GenerateSchema<br/>(_generate_schema.py)
    participant Decorators as DecoratorInfos<br/>(_decorators.py)
    participant CoreAPI  as pydantic-core API<br/>(plugin/_schema_validator.py)
    participant Core     as SchemaValidator<br/>(pydantic-core / Rust)
    participant Model    as BaseModel instance<br/>(main.py)

    rect rgb(219, 234, 254)
        Note over User,Core: ── Phase 1: Class Definition (import time) ──

        User->>Meta: class MyModel(BaseModel):<br/>    name: str<br/>    age: int = Field(gt=0)

        Meta->>Fields: collect_model_fields(cls, bases, config)
        Fields-->>Meta: __pydantic_fields__: Dict[str, FieldInfo]

        Meta->>Decorators: DecoratorInfos.build(cls)
        Note right of Decorators: scans @field_validator,<br/>@model_validator,<br/>@field_serializer methods
        Decorators-->>Meta: __pydantic_decorators__: DecoratorInfos

        Meta->>GenSchema: GenerateSchema(config, decorators)
        GenSchema->>GenSchema: _model_schema(cls, fields)

        loop for each FieldInfo
            GenSchema->>GenSchema: _generate_md_field_schema(field)
            Note right of GenSchema: resolves Annotated metadata,<br/>constraints (gt, min_length…),<br/>Before/After/Wrap validators
        end

        GenSchema->>GenSchema: wrap with @model_validator(mode="before"/"after")
        GenSchema-->>Meta: CoreSchema (dict tree describing all rules)

        Meta->>CoreAPI: create_schema_validator(core_schema, config)
        CoreAPI->>Core: SchemaValidator(core_schema)
        Note right of Core: Rust compiles schema to<br/>optimised validation bytecode
        Core-->>CoreAPI: compiled SchemaValidator
        CoreAPI-->>Meta: __pydantic_validator__

        Meta->>Core: SchemaSerializer(core_schema)
        Core-->>Meta: __pydantic_serializer__

        Meta-->>User: MyModel class (ready to use)
    end

    rect rgb(220, 252, 231)
        Note over User,Core: ── Phase 2: Instance Validation (every call) ──

        User->>Model: MyModel(name="Alice", age=30)

        Model->>Core: __pydantic_validator__.validate_python(data, self_instance=self)

        Note right of Core: ① run @model_validator(mode="before") if any<br/>② for each field in schema order:<br/>   • apply BeforeValidators<br/>   • coerce type (str→int, etc.)<br/>   • check constraints (gt, min_length…)<br/>   • apply AfterValidators<br/>③ run @model_validator(mode="after") if any<br/>④ call __pydantic_post_init__ if defined

        alt validation succeeds
            Core-->>Model: populates __dict__ + __pydantic_fields_set__
            Model-->>User: MyModel instance ✓
        else validation fails
            Core-->>User: raises ValidationError<br/>(structured list of errors with loc, msg, type)
        end
    end
```

## Flow Summary

| Step | Who | What |
|------|-----|-------|
| 1 | `ModelMetaclass` | Intercepts class creation via `__new__` |
| 2 | `collect_model_fields()` | Builds `FieldInfo` objects from annotations + `Field()` calls |
| 3 | `DecoratorInfos.build()` | Collects `@field_validator`, `@model_validator`, `@field_serializer` |
| 4 | `GenerateSchema` | Converts Python types + field metadata → `CoreSchema` (nested dict) |
| 5 | `create_schema_validator()` | Passes `CoreSchema` through the plugin layer to pydantic-core |
| 6 | `SchemaValidator` (Rust) | Compiles the schema to optimised validation logic; stored on the class |
| 7 | `BaseModel.__init__` | On every instantiation, calls `validate_python(data)` on the compiled validator |
| 8 | pydantic-core runtime | Runs before-validators → type coercion → constraints → after-validators → model validators |
| 9 | success / failure | Returns a populated model instance **or** raises a structured `ValidationError` |

## Key Design Principle

Schema compilation (steps 1–6) happens **once** per class at definition time.  
Validation (steps 7–9) is fully delegated to compiled Rust code, making per-instance overhead minimal.

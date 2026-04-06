# Developer Usage Trace — Full Call Path with File Locations

A complete trace of what happens when a developer uses Pydantic, from the first `import` to
serializing validated data. Every step is annotated with the exact file and line number.

---

## The code being traced

```python
from pydantic import BaseModel
from pydantic.fields import Field

class User(BaseModel):
    name: str
    age: int = Field(gt=0, default=18)

user = User(name="Alice", age=30)        # validate
data = user.model_dump()                  # serialize to dict
json_str = user.model_dump_json()         # serialize to JSON string
schema = User.model_json_schema()         # generate JSON Schema
```

---

## Phase 1 — Import (`from pydantic import BaseModel`)

```
pydantic/__init__.py:1
  └── __getattr__(name='BaseModel')              # lazy import facade
        └── importlib.import_module('pydantic.main')
              └── pydantic/main.py:119
                    class BaseModel(metaclass=ModelMetaclass)
                      # metaclass set; class body executed
                      # class-level __pydantic_validator__ = MockValSer(...)  line 240
                      # class-level __pydantic_serializer__ = MockValSer(...) line 245
```

---

## Phase 2 — Class Definition (`class User(BaseModel): ...`)

Python calls `ModelMetaclass.__new__()` because `BaseModel`'s metaclass is `ModelMetaclass`.

### 2.1 — `ModelMetaclass.__new__` entry
```
pydantic/_internal/_model_construction.py:84   ModelMetaclass.__new__()
  line 111   if bases:   # True — User has BaseModel as a base
  line 129   mcs._collect_bases_data(bases)
               # collects field names, class vars, private attrs from parent classes
  line 131   ConfigWrapper.for_model(bases, namespace, raw_annotations, kwargs)
               # reads model_config = ConfigDict(...) from class body or parents
               pydantic/_internal/_config.py:29   ConfigWrapper.__init__()
  line 133   inspect_namespace(namespace, raw_annotations, ...)
```

### 2.2 — `inspect_namespace` — scan the class body
```
pydantic/_internal/_model_construction.py:400  inspect_namespace()
  # iterates namespace items
  # identifies ModelPrivateAttr (PrivateAttr) instances → private_attributes dict
  # validates no leading-underscore field names
  # validates all FieldInfo instances have type annotations
  # returns: dict[str, ModelPrivateAttr]
```

### 2.3 — `DecoratorInfos.build` — collect validators / serializers
```
pydantic/_internal/_model_construction.py:176
  cls.__pydantic_decorators__ = DecoratorInfos.build(cls, replace_wrapped_methods=True)

  pydantic/_internal/_decorators.py  DecoratorInfos.build()
    # scans class MRO for PydanticDescriptorProxy wrappers
    # collects @field_validator  → FieldValidatorDecoratorInfo  (line 56)
    # collects @model_validator  → ModelValidatorDecoratorInfo  (line 135)
    # collects @field_serializer → FieldSerializerDecoratorInfo (line 93)
    # collects @model_serializer → ModelSerializerDecoratorInfo (line 116)
    # collects @computed_field   → ComputedFieldInfo
    # stores in DecoratorInfos dataclass                         (line 416)
```

### 2.4 — `set_model_fields` — build FieldInfo per field
```
pydantic/_internal/_model_construction.py:566  set_model_fields()
  └── pydantic/_internal/_fields.py  collect_model_fields()
        # iterates cls.__annotations__
        # for each annotation:
        #   - wraps bare default values in FieldInfo
        #   - resolves Field(...) calls already in FieldInfo
        #   - resolves forward references via NsResolver
        #   - populates FieldInfo.annotation, .default, .metadata (constraints), .alias…
        # returns Dict[str, FieldInfo]

  line 583  cls.__pydantic_fields__ = fields      # stored on the class
```

### 2.5 — `complete_model_class` — schema generation + compilation
```
pydantic/_internal/_model_construction.py:256  complete_model_class() called
pydantic/_internal/_model_construction.py:600  complete_model_class() definition

  line 660  gen_schema = GenerateSchema(config_wrapper, ns_resolver, typevars_map)
              pydantic/_internal/_generate_schema.py  GenerateSchema.__init__()

  line 667  schema = gen_schema.generate_schema(cls)
              pydantic/_internal/_generate_schema.py:717  generate_schema()
                line 741  _generate_schema_from_get_schema_method(cls, cls)
                            # checks for cls.__get_pydantic_core_schema__ override
                line 744  _generate_schema_inner(cls)
                            └── _model_schema(cls)   line 756
                                  # for each field in __pydantic_fields__:
                                  #   _generate_md_field_schema(field_name, field_info)
                                  #     → resolves annotation type
                                  #     → builds CoreSchema for the type (str_schema, int_schema…)
                                  #     → applies BeforeValidator / AfterValidator / WrapValidator
                                  #       from Annotated metadata
                                  #     → applies @field_validator wrappers
                                  # wraps all fields in model_fields_schema
                                  # wraps with @model_validator(mode="before"/"after") if any
                                  # returns core_schema.model_schema(cls, inner_schema, config)

  line 674  core_config = config_wrapper.core_config(title=cls.__name__)
              # converts ConfigDict → CoreConfig (pydantic-core's config format)

  line 677  schema = gen_schema.clean_schema(schema)
              # resolves $defs / forward references into a flat definitions block

  line 688  cls.__pydantic_core_schema__ = schema    # stored on the class

  line 690  cls.__pydantic_validator__ = create_schema_validator(schema, cls, ...)
              pydantic/plugin/_schema_validator.py:22  create_schema_validator()
                line 40  plugins = get_plugins()
                line 42  if plugins → PluggableSchemaValidator(schema, ...)   line 56
                           # wraps validate_python / validate_json with plugin hooks
                line 53  else → SchemaValidator(schema, core_config)
                           # pydantic-core (Rust): compiles CoreSchema dict to bytecode
                           # stored as cls.__pydantic_validator__

  line 700  cls.__pydantic_serializer__ = SchemaSerializer(schema, core_config)
              # pydantic-core (Rust): compiles serialization rules
              # stored as cls.__pydantic_serializer__

  line 705  cls.__signature__ = LazyClassAttribute(...)
              # synthesizes an __init__ signature for IDE/inspect support
              pydantic/_internal/_signature.py  generate_pydantic_signature()

  line 716  cls.__pydantic_complete__ = True
```

---

## Phase 3 — Instantiation / Validation (`User(name="Alice", age=30)`)

```
pydantic/main.py:253  BaseModel.__init__(self, /, **data)
  # data = {"name": "Alice", "age": 30}

  line 263  validated_self = self.__pydantic_validator__.validate_python(data, self_instance=self)

    ── pydantic-core (Rust) SchemaValidator.validate_python() ──────────────────
    │  Executes compiled validation bytecode built in Phase 2:
    │
    │  1. Run @model_validator(mode="before") if defined
    │
    │  2. For each field in schema order:
    │     a. Look up input key (or alias / validation_alias)
    │     b. Run BeforeValidator(s) from Annotated metadata or @field_validator(mode="before")
    │     c. Type coercion: "Alice" → str  (no-op), "30" → int (if strict=False)
    │     d. Check numeric constraints: gt=0 → 30 > 0 ✓
    │     e. Run AfterValidator(s) / WrapValidator(s) from @field_validator(mode="after")
    │
    │  3. Run @model_validator(mode="after") if defined
    │
    │  4. Call model_post_init(context) if __pydantic_post_init__ is set
    │     (pydantic/main.py:627  BaseModel.model_post_init)
    │
    │  5. Populate self.__dict__ = {"name": "Alice", "age": 30}
    │     Populate self.__pydantic_fields_set__ = {"name", "age"}
    │
    │  On any failure → raise pydantic_core.ValidationError
    │    (structured list of {loc, msg, type, input, url} dicts)
    ────────────────────────────────────────────────────────────────────────────

  line 264  if self is not validated_self: warn(...)   # guard for unusual top-level validators

  # __init__ returns; `user` is a fully populated BaseModel instance
```

---

## Phase 4 — Serialization to dict (`user.model_dump()`)

```
pydantic/main.py:427  BaseModel.model_dump(self, *, mode='python', ...)

  line 475  return self.__pydantic_serializer__.to_python(
                self,
                mode=mode,          # 'python'
                by_alias=by_alias,
                exclude_unset=exclude_unset,
                ...
            )

    ── pydantic-core (Rust) SchemaSerializer.to_python() ───────────────────────
    │  Walks model fields in definition order:
    │    - Reads value from self.__dict__[field_name]
    │    - Applies @field_serializer if defined
    │    - Applies PlainSerializer / WrapSerializer from Annotated metadata
    │    - Applies exclusion filters (exclude_unset, exclude_none, include/exclude sets)
    │  Returns: {"name": "Alice", "age": 30}
    ────────────────────────────────────────────────────────────────────────────
```

---

## Phase 5 — Serialization to JSON string (`user.model_dump_json()`)

```
pydantic/main.py:493  BaseModel.model_dump_json(self, *, indent=None, ...)

  line 542  return self.__pydantic_serializer__.to_json(
                self,
                indent=indent,
                by_alias=by_alias,
                ...
            ).decode()

    ── pydantic-core (Rust) SchemaSerializer.to_json() ─────────────────────────
    │  Same field walk as to_python(), but outputs UTF-8 JSON bytes directly.
    │  .decode() converts bytes → str at line 559.
    │  Returns: '{"name":"Alice","age":30}'
    ────────────────────────────────────────────────────────────────────────────
```

---

## Phase 6 — JSON Schema generation (`User.model_json_schema()`)

```
pydantic/main.py:562  BaseModel.model_json_schema(cls, by_alias=True, mode='validation', ...)

  line 591  return model_json_schema(cls, by_alias=by_alias, ...)
              pydantic/json_schema.py  model_json_schema()
                └── GenerateJsonSchema(by_alias=True, ref_template=...).generate(inputs)
                      # walks cls.__pydantic_core_schema__  (the CoreSchema built in Phase 2)
                      # converts each node type to its JSON Schema equivalent:
                      #   model_schema      → {"type": "object", "properties": {...}}
                      #   str_schema        → {"type": "string"}
                      #   int_schema + gt=0 → {"type": "integer", "exclusiveMinimum": 0}
                      # resolves $defs for recursive / shared schemas
                      # returns dict conforming to JSON Schema Draft 2020-12

  # Returns:
  # {
  #   "title": "User",
  #   "type": "object",
  #   "properties": {
  #     "name": {"title": "Name", "type": "string"},
  #     "age":  {"title": "Age", "default": 18, "exclusiveMinimum": 0, "type": "integer"}
  #   },
  #   "required": ["name"]
  # }
```

---

## Full call-stack summary

```
PHASE 1  Import
  pydantic/__init__.py:1              __getattr__ lazy import
  pydantic/main.py:119               class BaseModel definition

PHASE 2  Class definition  (class User(BaseModel): ...)
  pydantic/_internal/_model_construction.py:84    ModelMetaclass.__new__
  pydantic/_internal/_model_construction.py:131   ConfigWrapper.for_model
  pydantic/_internal/_model_construction.py:400   inspect_namespace
  pydantic/_internal/_model_construction.py:176   DecoratorInfos.build
    pydantic/_internal/_decorators.py             (all decorator info classes)
  pydantic/_internal/_model_construction.py:566   set_model_fields
    pydantic/_internal/_fields.py                 collect_model_fields
    pydantic/fields.py:106                        FieldInfo population
  pydantic/_internal/_model_construction.py:600   complete_model_class
    pydantic/_internal/_generate_schema.py:717    GenerateSchema.generate_schema
    pydantic/_internal/_generate_schema.py:756    GenerateSchema._model_schema
    pydantic/plugin/_schema_validator.py:22       create_schema_validator
      pydantic-core (Rust)                        SchemaValidator(schema)   → __pydantic_validator__
      pydantic-core (Rust)                        SchemaSerializer(schema)  → __pydantic_serializer__
  pydantic/_internal/_model_construction.py:716   cls.__pydantic_complete__ = True

PHASE 3  Instantiation  (User(name="Alice", age=30))
  pydantic/main.py:253               BaseModel.__init__
  pydantic/main.py:263               __pydantic_validator__.validate_python(data)
    pydantic-core (Rust)             run validators, coerce types, check constraints
    pydantic/main.py:627             model_post_init (if overridden)

PHASE 4  Serialize to dict  (user.model_dump())
  pydantic/main.py:427               BaseModel.model_dump
  pydantic/main.py:475               __pydantic_serializer__.to_python(self, ...)
    pydantic-core (Rust)             walk fields, apply serializers, apply filters

PHASE 5  Serialize to JSON  (user.model_dump_json())
  pydantic/main.py:493               BaseModel.model_dump_json
  pydantic/main.py:542               __pydantic_serializer__.to_json(self, ...).decode()
    pydantic-core (Rust)             write UTF-8 JSON bytes

PHASE 6  JSON Schema  (User.model_json_schema())
  pydantic/main.py:562               BaseModel.model_json_schema
  pydantic/main.py:591               model_json_schema(cls, ...)
  pydantic/json_schema.py            GenerateJsonSchema.generate(inputs)
    walks cls.__pydantic_core_schema__ → returns JSON Schema dict
```

---

## Key handoff points

| Handoff | From | To | What crosses the boundary |
|---------|------|----|--------------------------|
| Schema compile | `complete_model_class` (`_model_construction.py:690`) | pydantic-core Rust | `CoreSchema` dict tree |
| Serializer compile | `complete_model_class` (`_model_construction.py:700`) | pydantic-core Rust | same `CoreSchema` dict tree |
| Validate call | `BaseModel.__init__` (`main.py:263`) | pydantic-core Rust | raw `**data` dict |
| Serialize call | `BaseModel.model_dump` (`main.py:475`) | pydantic-core Rust | `self` (model instance) |
| JSON serialize | `BaseModel.model_dump_json` (`main.py:542`) | pydantic-core Rust | `self` (model instance) |
| JSON Schema | `model_json_schema` (`main.py:591`) | `GenerateJsonSchema` (`json_schema.py`) | `cls.__pydantic_core_schema__` |

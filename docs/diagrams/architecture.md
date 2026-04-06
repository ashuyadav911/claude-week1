# Pydantic Architecture Flowchart

```mermaid
flowchart TD
    subgraph USER["User Code"]
        U1["class MyModel(BaseModel)"]
        U2["TypeAdapter(SomeType)"]
        U3["@validate_call"]
    end

    subgraph PUBLIC_API["Public API  —  pydantic/__init__.py"]
        direction LR
        A1["BaseModel\nRootModel\ncreate_model"]
        A2["Field · PrivateAttr\ncomputed_field"]
        A3["@field_validator\n@model_validator\n@validate_call"]
        A4["@field_serializer\n@model_serializer"]
        A5["ConfigDict\nAliasPath/Generator"]
        A6["TypeAdapter"]
    end

    subgraph TYPES["Built-in Types  —  types.py / networks.py"]
        T1["Constrained primitives\nStrictStr · PositiveInt …"]
        T2["Network & identity\nHttpUrl · EmailStr · UUID…"]
        T3["Special-purpose\nSecretStr · Json · Base64…"]
    end

    subgraph SCHEMA_GEN["Schema Generation  —  _internal/_generate_schema.py"]
        S1["GenerateSchema\n(Python types → CoreSchema)"]
        S2["Resolve Annotated metadata\n& field constraints"]
        S3["Handle generics,\nforward refs, recursion"]
        S4["Merge ConfigDict\n& validator decorators"]
        S1 --> S2 --> S3 --> S4
    end

    subgraph JSON_SCHEMA["JSON Schema  —  json_schema.py"]
        J1["GenerateJsonSchema"]
        J2["model_json_schema()\nTypeAdapter.json_schema()"]
        J1 --> J2
    end

    subgraph PLUGIN["Plugin Layer  —  plugin/"]
        P1["PluggableSchemaValidator"]
        P2["Third-party hooks\n(e.g. pydantic-settings)"]
        P1 --> P2
    end

    subgraph PYDANTIC_CORE["pydantic-core  (Rust / C Extension)"]
        direction TB
        C1["SchemaValidator\n(compile CoreSchema)"]
        C2["validate_python()\nvalidate_json()"]
        C3["SchemaSerializer\n(compile serializer)"]
        C4["dump_python()\ndump_json()"]
        C5["ValidationError\n(structured error details)"]
        C1 --> C2
        C3 --> C4
        C2 -->|failure| C5
    end

    subgraph COMPAT["Compatibility  —  deprecated/ · v1/"]
        V1["V1 validators\n@validator · @root_validator"]
        V2["BaseConfig · Extra"]
        V3["parse_obj_as · schema_of"]
    end

    subgraph OUTPUT["Output"]
        O1["Validated Python object"]
        O2["dict / JSON bytes"]
        O3["JSON Schema dict"]
        O4["ValidationError"]
    end

    %% User to Public API
    U1 & U2 & U3 --> PUBLIC_API

    %% Public API feeds schema generation
    PUBLIC_API --> SCHEMA_GEN
    TYPES --> SCHEMA_GEN

    %% Schema gen produces CoreSchema → pydantic-core
    SCHEMA_GEN -->|CoreSchema| C1
    SCHEMA_GEN -->|CoreSchema| C3

    %% Plugin layer wraps SchemaValidator
    C1 --> P1

    %% JSON Schema path
    SCHEMA_GEN --> J1

    %% Compat feeds schema gen
    COMPAT --> SCHEMA_GEN

    %% pydantic-core outputs
    C2 --> O1
    C4 --> O2
    J2 --> O3
    C5 --> O4

    %% Styling
    classDef userBox    fill:#dbeafe,stroke:#3b82f6,color:#1e3a5f
    classDef apiBox     fill:#ede9fe,stroke:#7c3aed,color:#2e1065
    classDef typeBox    fill:#fef9c3,stroke:#ca8a04,color:#713f12
    classDef schemaBox  fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef coreBox    fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
    classDef outputBox  fill:#f1f5f9,stroke:#64748b,color:#1e293b
    classDef compatBox  fill:#fce7f3,stroke:#db2777,color:#500724
    classDef pluginBox  fill:#ffedd5,stroke:#ea580c,color:#431407
    classDef jsonBox    fill:#e0f2fe,stroke:#0284c7,color:#0c4a6e

    class U1,U2,U3 userBox
    class A1,A2,A3,A4,A5,A6 apiBox
    class T1,T2,T3 typeBox
    class S1,S2,S3,S4 schemaBox
    class C1,C2,C3,C4,C5 coreBox
    class O1,O2,O3,O4 outputBox
    class V1,V2,V3 compatBox
    class P1,P2 pluginBox
    class J1,J2 jsonBox
```

## Layer Descriptions

| Layer | Files | Responsibility |
|-------|-------|---------------|
| **User Code** | — | Subclass `BaseModel`, annotate fields, call `.model_validate()` |
| **Public API** | `pydantic/__init__.py` | Lazy-imported facade; single import surface for all symbols |
| **Built-in Types** | `types.py`, `networks.py` | Constrained, network, secret, and encoding types |
| **Schema Generation** | `_internal/_generate_schema.py` | Converts Python type hints + field metadata → `CoreSchema` objects |
| **JSON Schema** | `json_schema.py` | Converts `CoreSchema` → OpenAPI-compatible JSON Schema dicts |
| **Plugin Layer** | `plugin/` | Wraps `SchemaValidator` so third-party tools can hook into validation |
| **pydantic-core** | `pydantic-core/src/` (Rust) | Compiles schemas, runs validation/serialization at native speed |
| **Compatibility** | `deprecated/`, `v1/` | V1 API shims (`@validator`, `BaseConfig`, `parse_obj_as`) |
| **Output** | — | Validated Python objects, serialized dicts/JSON, JSON Schema, errors |
```

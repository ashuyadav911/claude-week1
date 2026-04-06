# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development setup

Requires [uv](https://docs.astral.sh/uv/) and Rust (for `pydantic-core`).

```bash
make install          # sync deps, install pre-commit hooks
```

## Common commands

```bash
# Testing
make test                                              # full test suite
uv run pytest tests/test_main.py                       # single file
uv run pytest tests/test_types.py -k "test_strict"    # single test by name
uv run pytest tests/test_generics.py -x               # stop on first failure
make test-mypy                                         # mypy integration tests (separate run)

# Linting & formatting
make lint                  # ruff check + ruff format --check + Rust lint
make format                # auto-fix ruff + cargo fmt
make typecheck             # pyright via pre-commit

# Docs
make docs-serve            # local preview at localhost:8000

# Coverage
make testcov               # run tests then open htmlcov/
```

`NUM_THREADS` controls parallel test workers: `make test NUM_THREADS=4`.

Benchmarks are disabled by default; enable with `make benchmark`.

## Repository layout

```
pydantic/               # pure-Python library
  __init__.py           # lazy-import facade (all public symbols via __getattr__)
  main.py               # BaseModel
  root_model.py         # RootModel
  fields.py             # FieldInfo, Field(), PrivateAttr, computed_field
  config.py             # ConfigDict
  aliases.py            # AliasPath, AliasChoices, AliasGenerator
  types.py              # constrained & special types (SecretStr, Json, …)
  networks.py           # URL / DSN / email types
  functional_validators.py  # BeforeValidator, AfterValidator, field_validator, model_validator
  functional_serializers.py # field_serializer, model_serializer, PlainSerializer, …
  type_adapter.py       # TypeAdapter
  json_schema.py        # GenerateJsonSchema
  validate_call_decorator.py
  _internal/            # private implementation (do not import from outside pydantic)
pydantic-core/          # Rust extension (git submodule / uv workspace member)
tests/                  # pytest test suite
  mypy/                 # mypy plugin integration tests (need --test-mypy flag)
  typechecking/         # pyright / mypy / pyrefly static analysis tests
  benchmarks/
docs/
  examples/             # runnable .py files embedded in the docs
```

## Architecture: how validation works

**Class-definition time** (runs once per class, at import):

1. `ModelMetaclass.__new__` (`_internal/_model_construction.py`) intercepts `class MyModel(BaseModel)`.
2. `collect_model_fields()` (`_internal/_fields.py`) builds a `Dict[str, FieldInfo]` from annotations and `Field()` calls → stored as `__pydantic_fields__`.
3. `DecoratorInfos.build()` (`_internal/_decorators.py`) scans the class for `@field_validator`, `@model_validator`, `@field_serializer`, `@computed_field` → stored as `__pydantic_decorators__`.
4. `GenerateSchema` (`_internal/_generate_schema.py`) converts Python types + `FieldInfo` + decorator metadata into a `CoreSchema` (a plain-dict tree defined by pydantic-core).
5. `create_schema_validator()` (`plugin/_schema_validator.py`) passes the `CoreSchema` through any registered plugins, then calls pydantic-core's `SchemaValidator(core_schema)` to compile it into optimised Rust bytecode → `__pydantic_validator__`.
6. `SchemaSerializer(core_schema)` is compiled similarly → `__pydantic_serializer__`.

**Instance time** (runs on every `MyModel(**data)`):

`BaseModel.__init__` calls `self.__pydantic_validator__.validate_python(data)`. All validation — type coercion, constraint checks, before/after/wrap validators, model validators, `__pydantic_post_init__` — runs inside pydantic-core (Rust). On failure a structured `ValidationError` is raised.

**Key design consequence:** `_generate_schema.py` is the translation layer between Python semantics and pydantic-core's schema language. When adding support for a new type or annotation, this is where the work happens.

## `pydantic-core` boundary

`pydantic-core` is a compiled Rust extension (a uv workspace member in `pydantic-core/`). Its Python surface is `pydantic_core` (imported as a package). The core schema types live in `pydantic_core.core_schema`. Never bypass `SchemaValidator`/`SchemaSerializer` for validation or serialisation.

## `__init__.py` lazy-import pattern

All public symbols are re-exported lazily via `__getattr__` to keep import time fast. `TYPE_CHECKING` blocks contain the actual `import` statements so that IDEs and type checkers see the full public API without triggering the lazy loading at runtime.

## Deprecation conventions

V1 compatibility shims live in `pydantic/deprecated/` and `pydantic/v1/`. New code should never import from `pydantic/v1/`. Deprecated symbols emit `PydanticDeprecatedSince*` warnings (defined in `warnings.py`).

## Testing notes

- `tests/test_docs.py` runs all code examples embedded in the documentation — keep examples in `docs/examples/` runnable.
- Mypy integration tests (`tests/mypy/`) require `--test-mypy` and are slow; they are skipped in the default `make test` run.
- `xfail_strict = true` in `pyproject.toml` — expected-failure tests that start passing will break CI.
- Warnings are turned into errors (`filterwarnings = ['error', ...]`); tests that intentionally trigger deprecation warnings must use `pytest.warns`.

## Code style

- Line length: 120 (ruff).
- Quote style: single quotes for code, double for multiline strings.
- Docstrings: Google convention (`pydocstyle`).
- `pydantic/_internal/` modules use `from __future__ import annotations` — string-form annotations are the norm there.

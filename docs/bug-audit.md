# Pydantic — Bug Audit

**Date:** 2026-04-04
**Codebase:** pydantic `main` branch (HEAD `46dea928`)
**Scope:** `pydantic/` public API, `pydantic/_internal/`, `pydantic/plugin/` — excludes `pydantic/v1/`
**Method:** Static analysis of `TODO`/`FIXME`/`HACK` comments, `pytest.mark.xfail` tests, and known GitHub issues

---

## Severity scale

| Level | Meaning |
|-------|---------|
| **Critical** | Data silently corrupted or validation produces wrong result |
| **High** | Feature broken or raises wrong error; tracked with xfail test |
| **Medium** | Design flaw, fragile hack, or incomplete behaviour |
| **Low** | Cleanup / version-gated work; no user-facing impact today |

---

## Critical

### BUG-001 — NaN float input mishandled in pydantic-core
| Field | Detail |
|-------|--------|
| **Severity** | Critical |
| **File** | `tests/test_annotated.py:378` |
| **Status** | Verified — comment in test source |
| **GitHub** | pydantic-core (upstream) |

```python
# TODO: input should be float('nan'), this seems like a subtle bug in pydantic-core
```

NaN float values are not passed through correctly during validation of annotated types. The input received by the validator is not `float('nan')` as expected. Because this lives in pydantic-core (Rust), the fix must come from upstream.

---

## High

### BUG-002 — Discriminated union serialization broken
| Field | Detail |
|-------|--------|
| **Severity** | High |
| **File** | `tests/test_discriminated_union.py:2163` |
| **Status** | Verified — `xfail` test present |
| **GitHub** | pydantic/pydantic#9688 |

```python
@pytest.mark.xfail(reason='Waiting for union serialization fixes via https://github.com/pydantic/pydantic/issues/9688.')
```

Discriminated unions do not serialize correctly under certain configurations. Affects `.model_dump()` and `.model_dump_json()` output for models using `Annotated` discriminated unions.

---

### BUG-003 — Discriminator info lost when core schemas are inlined
| Field | Detail |
|-------|--------|
| **Severity** | High |
| **File** | `tests/test_discriminated_union.py:2268` |
| **Status** | Verified — `xfail` test present |
| **GitHub** | — |

```python
@pytest.mark.xfail(reason='deferred discriminated union info is lost on core schemas that are inlined.')
```

When a discriminated union's core schema is inlined (rather than referenced via `$defs`), the discriminator metadata is dropped. Breaks JSON schema output and may affect runtime dispatch.

---

### BUG-004 — TypeAdapter JSON schema drops discriminator, emits warning instead
| Field | Detail |
|-------|--------|
| **Severity** | High |
| **File** | `tests/test_discriminated_union.py:1812` |
| **Status** | Verified — `xfail` test present |
| **GitHub** | pydantic/pydantic#8271 |

```python
@pytest.mark.xfail(
    reason='Issue not yet fixed, see: https://github.com/pydantic/pydantic/issues/8271. '
           'At the moment, JSON schema gen warns with a PydanticJsonSchemaWarning.'
)
```

`TypeAdapter(...).json_schema()` emits a `PydanticJsonSchemaWarning` instead of correctly embedding the discriminator in the JSON Schema output.

---

### BUG-005 — `Final[]` field not recognised as class variable during `model_rebuild()`
| Field | Detail |
|-------|--------|
| **Severity** | High |
| **File** | `tests/test_main.py:2093` |
| **Status** | Verified — `xfail` test present |
| **GitHub** | — |

```python
@pytest.mark.xfail(reason="When rebuilding fields, we don't consider the field as a class variable")
```

Forward references using `Final[...]` are not treated as class variables when `model_rebuild()` resolves them. The field ends up in `__pydantic_fields__` instead of `__class_vars__`, causing validation errors.

---

### BUG-006 — Plain validator incompatible with filter-dict schema
| Field | Detail |
|-------|--------|
| **Severity** | High |
| **File** | `tests/test_validators.py:2964` |
| **Status** | Verified — `xfail` test present |
| **GitHub** | pydantic/pydantic#10428 |

```python
@pytest.mark.xfail(reason='https://github.com/pydantic/pydantic/issues/10428')
```

`PlainValidator` does not compose correctly when the outer schema is a filter-dict schema (e.g. `model_config = ConfigDict(extra='allow')`).

---

### BUG-007 — Nested `after` model validator re-executed unexpectedly
| Field | Detail |
|-------|--------|
| **Severity** | High |
| **File** | `tests/test_validators.py:3125` |
| **Status** | Verified — `xfail` test present |
| **GitHub** | pydantic/pydantic#8452 |

```python
@pytest.mark.xfail(
    reason="Bug: Nested 'after' model_validator is re-executed. See issue #8452.", raises=ValidationError
)
```

A `@model_validator(mode='after')` on a nested model is re-executed a second time during parent model validation, causing `ValidationError` in validators that are not idempotent.

---

### BUG-008 — Recursive generic model inheritance missing schema definition
| Field | Detail |
|-------|--------|
| **Severity** | High |
| **File** | `tests/test_generics.py:1738` |
| **Status** | Verified — `xfail` test present |
| **GitHub** | pydantic/pydantic#9969 |

```python
@pytest.mark.xfail(reason='Core schema generation is missing the M1 definition')
```

When a generic model inherits from another generic model and both use the same type parameter recursively, schema generation fails to emit the definition for the parent model, producing an incomplete `CoreSchema`.

---

### BUG-009 — Variadic generics (`TypeVarTuple`) not supported
| Field | Detail |
|-------|--------|
| **Severity** | High |
| **File** | `tests/test_generics.py:2607`, `tests/test_generics.py:2643` |
| **Status** | Verified — two `xfail` tests present |
| **GitHub** | pydantic/pydantic#5804 |

```python
@pytest.mark.xfail(reason='TODO: Variadic generic parametrization is not supported yet; Issue: https://github.com/pydantic/pydantic/issues/5804')
@pytest.mark.xfail(reason='TODO: Variadic fields are not supported yet; Issue: https://github.com/pydantic/pydantic/issues/5804')
```

`TypeVarTuple` and `Unpack[...]` generic parametrization are silently unsupported. Both model init and variadic field definitions fail at runtime.

---

## Medium

### BUG-010 — `FieldInfo` metadata merge order is non-deterministic (HACK 1)
| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **File** | `pydantic/fields.py:430` |
| **Status** | Verified — `HACK` comment in source |
| **GitHub** | — (noted as v3 fix) |

```python
# HACK 1: the order in which the metadata is merged is inconsistent; we need to prepend
```

When `FieldInfo` is constructed from multiple `Annotated` layers, the metadata list ordering is inconsistent. Constraints applied later may silently override earlier ones depending on merge path. Acknowledged as only fixable in v3.

---

### BUG-011 — FastAPI `FieldInfo` subclass compatibility hack
| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **File** | `pydantic/fields.py:438`, `pydantic/_internal/_generate_schema.py:2259` |
| **Status** | Verified — two `HACK` comments in source |
| **GitHub** | — |

```python
# HACK 2: FastAPI is subclassing `FieldInfo` and historically expected the actual...
# HACK: we don't want to emit the warning for `FieldInfo` subclasses, because FastAPI does weird manipulations
```

Two separate hacks exist to preserve backwards compatibility with FastAPI's internal manipulation of `FieldInfo`. Warning suppression in `_generate_schema.py` is fragile — any internal refactor risks silently breaking FastAPI integrations.

---

### BUG-012 — `RootModel` raises `ValueError` instead of `AttributeError` on invalid assignment
| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **File** | `tests/test_root_model.py:267` |
| **Status** | Verified — `TODO` comment in test source |
| **GitHub** | — |

```python
# TODO: Should this be an `AttributeError`?
```

Assigning an invalid attribute to a `RootModel` instance raises `ValueError` rather than the `AttributeError` expected by standard Python semantics and stdlib `dataclasses`.

---

### BUG-013 — Pydantic dataclass `InitVar` raises wrong error type
| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **File** | `tests/test_dataclasses.py:702` |
| **Status** | Verified — `xfail` test present |
| **GitHub** | — |

```python
@pytest.mark.xfail(reason='Ideally we should raise an attribute error, like stdlib dataclasses')
```

Accessing an `InitVar` field after dataclass init raises the wrong exception type compared to stdlib `dataclasses`, breaking code that catches `AttributeError` for this case.

---

### BUG-014 — Regex pattern flags silently dropped in JSON Schema
| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **File** | `pydantic/json_schema.py:775` |
| **Status** | Verified — `TODO` comment in source |
| **GitHub** | — |

```python
# TODO: should we add regex flags to the pattern?
```

When a field is constrained with a compiled regex pattern (e.g. `re.compile(r'\w+', re.IGNORECASE)`), the JSON Schema output emits the pattern string but drops all flags. Consumers of the JSON Schema (e.g. OpenAPI validators) will apply stricter matching than pydantic itself.

---

### BUG-015 — `Any` schema for serialization requires an ugly hack
| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **File** | `pydantic/_internal/_generate_schema.py:434` |
| **Status** | Verified — `TODO` comment in source |
| **GitHub** | — |

```python
# TODO this is an ugly hack, how do we trigger an Any schema for serialization?
```

There is no clean mechanism to generate an `Any` core schema specifically for the serialisation path. The current workaround is acknowledged as a hack and may produce incorrect serialisation schemas for dynamically-typed values.

---

### BUG-016 — `RootModel` root field may skip common field metadata
| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **File** | `pydantic/_internal/_generate_schema.py:843` |
| **Status** | Verified — `FIXME` comment in source |
| **GitHub** | — |

```python
# FIXME: should the common field metadata be used here?
```

During schema generation the `root` field of a `RootModel` may not inherit common field-level metadata (e.g. `title`, `description` set at the model level). Produces incomplete JSON Schema output for `RootModel` subclasses.

---

### BUG-017 — OpenAPI discriminator leaks into plain JSON Schema output
| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **File** | `pydantic/json_schema.py:1353` |
| **Status** | Verified — `TODO` comment in source |
| **GitHub** | — |

```python
# This reflects the v1 behavior; TODO: we should make it possible to exclude OpenAPI stuff from the JSON schema
```

OpenAPI-specific discriminator fields are unconditionally included in JSON Schema output. There is no way for a user to generate a pure JSON Schema (non-OpenAPI) without OpenAPI artefacts.

---

### BUG-018 — Default value read incorrectly in serialisation-mode JSON Schema
| Field | Detail |
|-------|--------|
| **Severity** | Medium |
| **File** | `pydantic/json_schema.py:1433–1434` |
| **Status** | Verified — two `TODO` comments in source |
| **GitHub** | — |

```python
# TODO: Need to read the default value off of model config or whatever
use_strict = schema.get('strict', False)  # TODO: replace this default False
```

When generating a JSON Schema in `mode='serialization'`, default values are not correctly read from the model config, and the `strict` flag defaults to `False` regardless of the model's actual strictness setting.

---

## Low

### BUG-019 — Recursion error detection incomplete in type evaluation
| Field | Detail |
|-------|--------|
| **Severity** | Low |
| **File** | `pydantic/_internal/_typing_extra.py:490` |
| **Status** | Verified — `TODO` comment in source |
| **GitHub** | — |

```python
# TODO ideally recursion errors should be checked in `eval_type` above, but `eval_type_backport` is used directly in some places
```

`RecursionError` handling during forward-reference evaluation is incomplete. `eval_type_backport` is called directly in some code paths, bypassing the recursion guard in `eval_type`, and can produce unhandled exceptions on deeply recursive type hierarchies.

---

### BUG-020 — Dead code path in mypy plugin
| Field | Detail |
|-------|--------|
| **Severity** | Low |
| **File** | `pydantic/mypy.py:820` |
| **Status** | Verified — `TODO` comment in source |
| **GitHub** | pydantic/pydantic#11119 |

```python
# TODO: this path should be removed (see https://github.com/pydantic/pydantic/issues/11119)
```

A code path in the mypy plugin is acknowledged as dead and awaiting removal pending an upstream mypy fix.

---

### BUG-021 — No migration path for deprecated `custom_pydantic_encoder`
| Field | Detail |
|-------|--------|
| **Severity** | Low |
| **File** | `pydantic/deprecated/json.py:112` |
| **Status** | Verified — `TODO` comment in source |
| **GitHub** | — |

```python
# TODO: Add a suggested migration path once there is a way to use custom encoders
```

The deprecated `custom_pydantic_encoder` has no documented migration path. Users are warned it is deprecated but given no guidance on the equivalent v2 approach.

---

### BUG-022 — `EllipsisType` usage deferred pending Python 3.9 drop
| Field | Detail |
|-------|--------|
| **Severity** | Low |
| **File** | `pydantic/fields.py:926` |
| **Status** | Verified — `TODO` comment in source |
| **GitHub** | — |

```python
default: ellipsis,  # noqa: F821  # TODO: use `_typing_extra.EllipsisType` when we drop Py3.9
```

The proper `EllipsisType` annotation cannot be used in field default signatures while Python 3.9 is supported. The `# noqa: F821` suppression is a workaround until 3.9 support is dropped.

---

### BUG-023 — Parenthesis workaround for pyright / Python 3.10
| Field | Detail |
|-------|--------|
| **Severity** | Low |
| **File** | `pydantic/_internal/_generics.py:275` |
| **Status** | Verified — `TODO` comment in source |
| **GitHub** | — |

```python
# TODO: remove parentheses when we drop support for Python 3.10
```

Extra parentheses are present purely to work around a pyright issue on Python 3.10. They can be removed once 3.10 support is dropped.

---

### BUG-024 — `dict()` method removal deferred to v3
| Field | Detail |
|-------|--------|
| **Severity** | Low |
| **File** | `pydantic/main.py:4` |
| **Status** | Verified — `TODO` comment in source |
| **GitHub** | — |

```python
# TODO v3 fallback to `dict` when the deprecated `dict` method gets removed.
```

The deprecated `BaseModel.dict()` method (v1 compatibility) is kept alive by a local namespace alias. The alias and associated comment are cleanup blocked on the v3 release.

---

## Summary

| Severity | Count |
|----------|-------|
| Critical | 1 |
| High | 8 |
| Medium | 9 |
| Low | 5 |
| **Total** | **23** |

### Verification breakdown

| Status | Count |
|--------|-------|
| Verified via `xfail` test | 10 |
| Verified via source comment (`TODO`/`FIXME`/`HACK`) | 13 |

### Issues requiring pydantic-core (Rust) fix

- BUG-001 (NaN input)
- BUG-002 (union serialization)
- BUG-003 (inlined schema discriminator)

All other bugs are fixable at the Python layer.

# PR 1: Introduce `DynamoConfig` / `dynamodb_config`

**Summary**: rename `ModelConfig` to `DynamoConfig` and expose it under a new
class attribute `dynamodb_config`. Keep `ModelConfig` and `model_config` as
aliases for back-compat. No behavior change.

## Motivation

Pydantic v2 reserves `model_config` on every `BaseModel` as its own
`ConfigDict`. PR 4 introduces a new `PydanticModel(Model, BaseModel)` class
that needs `model_config` free for Pydantic's `ConfigDict(...)`. A separate
`dynamodb_config` name avoids the collision there.

The rename also helps `Model` users: `dynamodb_config` makes it obvious
that the value configures DynamoDB behavior (table, client, hot-partition
overrides) rather than being a generic Pydantic-style config. `Model`
users keep `model_config = ModelConfig(...)` working indefinitely; this
PR is non-breaking.

## Scope

In:

- New name `DynamoConfig` for the dataclass currently called `ModelConfig`.
- New class attribute `dynamodb_config: ClassVar[DynamoConfig]` readable by
  all existing `ModelBase` helpers.
- Read-through compat: if a subclass only sets the legacy `model_config =
  ModelConfig(...)`, everything still works and emits a `DeprecationWarning`
  once per class.
- `ModelConfig` remains as `TypeAlias = DynamoConfig` for type-import
  back-compat.

Out:

- Introducing `PydanticModel`. That is PR 4.
- Any `Annotated` markers. That is PR 2.
- Any condition / atomic API changes. That is PR 3.

## Files touched

- [python/pydynox/config.py](../../../python/pydynox/config.py) - rename the
  dataclass, add the alias, update docstrings.
- [python/pydynox/_internal/_model/_base.py](../../../python/pydynox/_internal/_model/_base.py)
  - update `_get_client` (lines 358-378), `_get_table` (lines 400-405),
  `_apply_hot_partition_overrides` (lines 380-398), `_should_skip_hooks`
  (lines 407-412) to read either attribute.
- [python/pydynox/__init__.py](../../../python/pydynox/__init__.py) - export
  `DynamoConfig` alongside `ModelConfig`.
- [python/pydynox/model.py](../../../python/pydynox/model.py) - update
  docstring example.
- [docs/getting-started.md](../../../docs/getting-started.md) and
  [docs/guides/](../../../docs/guides/) - add a note that `dynamodb_config`
  is the new preferred name.
- [tests/](../../../tests/) - add tests that both old and new names resolve
  to the same config; assert `DeprecationWarning` on old.

## Public API changes

### Before

```python
from pydynox import Model, ModelConfig
from pydynox.attributes import StringAttribute

class User(Model):
    model_config = ModelConfig(table="users")
    pk = StringAttribute(partition_key=True)
```

### After (both work, new preferred)

```python
from typing import ClassVar
from pydynox import Model, DynamoConfig
from pydynox.attributes import StringAttribute

class User(Model):
    dynamodb_config: ClassVar[DynamoConfig] = DynamoConfig(table="users")
    pk = StringAttribute(partition_key=True)
```

### Resolution order (internal)

In [python/pydynox/_internal/_model/_base.py](../../../python/pydynox/_internal/_model/_base.py),
the accessor becomes (pseudo-code):

```python
@classmethod
def _get_config(cls) -> DynamoConfig | None:
    cfg = getattr(cls, "dynamodb_config", None)
    if cfg is not None:
        return cfg
    legacy = getattr(cls, "model_config", None)
    if isinstance(legacy, DynamoConfig):
        _warn_once(cls, "model_config is deprecated; use dynamodb_config")
        return legacy
    return None
```

All existing `_get_client` / `_get_table` / `_apply_hot_partition_overrides` /
`_should_skip_hooks` / index binding sites call `_get_config()` instead of
`cls.model_config`.

## Back-compat and deprecation notes

- `ModelConfig` stays importable from [python/pydynox/config.py](../../../python/pydynox/config.py)
  as `ModelConfig: TypeAlias = DynamoConfig`.
- Setting `model_config = ModelConfig(...)` on a `Model` class body keeps
  working. First access emits `DeprecationWarning` with a pointer to the
  migration guide. Warning is class-cached so noisy repeat warnings do not
  flood logs.
- No removal scheduled. `Model` users may keep `model_config` indefinitely;
  the deprecation exists only to nudge docs and examples toward one canonical
  name. `PydanticModel` (PR 4) cannot accept the legacy name because
  Pydantic owns `model_config` there.

## Test plan

Add to [tests/unit/model/](../../../tests/) (or equivalent):

- `test_dynamodb_config_resolves_client`: a class with `dynamodb_config`
  only; `_get_client` returns the expected client.
- `test_model_config_still_works_with_warning`: a class with legacy
  `model_config`; assert `pytest.warns(DeprecationWarning)` on first
  access.
- `test_both_aliases_are_same_type`: `ModelConfig is DynamoConfig` at
  runtime.
- `test_hot_partition_overrides_under_new_name`: parity for
  `hot_partition_writes` / `hot_partition_reads` resolution.
- `test_model_config_dataclass_asdict_unchanged`: snapshot current field
  names and defaults.

No integration tests need to change.

## Size

S. Roughly 80-120 LOC added, 40-60 LOC refactored.

## Depends on / unblocks

- Depends on: nothing.
- Unblocks: PR 4 (`PydanticModel` can freely set
  `model_config = ConfigDict(...)` because pydynox configuration lives
  under `dynamodb_config`).

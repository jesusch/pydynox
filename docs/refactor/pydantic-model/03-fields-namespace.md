# PR 3: `cls.F` condition / atomic namespace

**Summary**: add a `cls.F` namespace exposing one `AttributeRef` per field.
`AttributeRef` carries the condition and atomic operators currently defined
directly on `Attribute`. Both styles work after this PR; the old style emits
a deprecation warning on class-level access.

## Motivation

Today `User.pk == "x"` works because `Attribute` is a class-level descriptor
whose `__eq__` returns a `ConditionComparison`. PR 4 makes `Model` a
`BaseModel` and class-level `User.pk` becomes a `FieldInfo` - the operator
overloads would be lost.

This PR introduces the replacement access path, `User.F.pk == "x"`, so users
can migrate before the breaking change. The implementation is additive and
delegates to the exact same `Condition*` and `Atomic*` classes used today,
so downstream code (transactions, update, query, scan) does not change.

## Scope

In:

- A new `AttributeRef` class (module
  `python/pydynox/_internal/_model/_refs.py`) wrapping an underlying
  `Attribute`. Exposes:
  - Comparison operators (`__eq__`, `__ne__`, `__lt__`, `__le__`, `__gt__`,
    `__ge__`) returning `ConditionComparison`.
  - `exists()`, `not_exists()`, `begins_with()`, `contains()`, `between()`,
    `is_in()`, `__getitem__` (for nested map / list paths).
  - `set()`, `add()`, `remove()`, `append()`, `prepend()`, `if_not_exists()`.
- A `_FieldsNamespace` exposing one `AttributeRef` per entry in
  `cls._attributes`, accessible as `cls.F` (both class-level and
  instance-level; both return the same namespace).
- `cls.F["pk"]` string lookup for dynamic builders.
- A deprecation shim on the existing `Attribute` class: the operator
  overloads still work but emit `DeprecationWarning` once per class+attr
  pair, pointing at `cls.F.<name>`.

Out:

- Turning `Model` into a `BaseModel`. PR 4.
- Removing the descriptor operators entirely. PR 4.

## Design

```mermaid
flowchart LR
    User["User class"] -->|"cls.F"| FNS["_FieldsNamespace"]
    FNS -->|"cls.F.pk"| RefPk["AttributeRef(attr=pk)"]
    FNS -->|"cls.F.counter"| RefC["AttributeRef(attr=counter)"]

    RefPk -->|"== 'x'"| Cond["ConditionComparison"]
    RefPk -->|"begins_with"| Cond
    RefC -->|"add(1)"| Atom["AtomicAdd"]

    Cond --> Tx["Transaction / update / query\n(unchanged)"]
    Atom --> Tx
```

`AttributeRef` is constructed with the underlying `Attribute` and delegates
to the existing condition/atomic builders in
[python/pydynox/_internal/_conditions.py](../../../python/pydynox/_internal/_conditions.py)
and [python/pydynox/_internal/_atomic.py](../../../python/pydynox/_internal/_atomic.py).

`_FieldsNamespace` is lazy: it is built once per class in
`ModelMeta.__new__` after `_attributes` is populated, and memoized on
`cls._F_namespace`.

## Files touched

- New `python/pydynox/_internal/_model/_refs.py` - `AttributeRef` and
  `_FieldsNamespace`.
- [python/pydynox/_internal/_model/_base.py](../../../python/pydynox/_internal/_model/_base.py)
  - at the end of `ModelMeta.__new__` (after index binding, around line
  167), build and attach the `F` namespace:
  `cls.F = _FieldsNamespace(cls._attributes)`.
- [python/pydynox/attributes/base.py](../../../python/pydynox/attributes/base.py)
  - wrap each condition / atomic method on `Attribute` with a
  `_warn_descriptor_usage()` call on first use per (class, attr) pair.
- [python/pydynox/__init__.py](../../../python/pydynox/__init__.py) - export
  `AttributeRef` for type annotations.
- [tests/](../../../tests/) - full parity tests + deprecation warning tests.
- [docs/guides/](../../../docs/guides/) - update the conditions guide to
  use `cls.F.*`.

## Public API changes

### Conditions

Before:

```python
from pydynox.conditions import Condition

users = User.sync_query(
    partition_key="TENANT#1",
    sort_key_condition=User.sk.begins_with("USER#"),
    filter_condition=(User.status == "active") & (User.age > 18),
)
```

After (old still works with warning; new is idiomatic):

```python
users = User.sync_query(
    partition_key="TENANT#1",
    sort_key_condition=User.F.sk.begins_with("USER#"),
    filter_condition=(User.F.status == "active") & (User.F.age > 18),
)
```

### Atomic updates

Before:

```python
await user.update_by_key(
    pk="USER#1",
    counter=User.counter.add(1),
    tags=User.tags.append(["premium"]),
    cache_key=User.cache_key.if_not_exists("default"),
)
```

After:

```python
await user.update_by_key(
    pk="USER#1",
    counter=User.F.counter.add(1),
    tags=User.F.tags.append(["premium"]),
    cache_key=User.F.cache_key.if_not_exists("default"),
)
```

### String lookup

New, useful for dynamic builders:

```python
def build_filter(field: str, value: Any) -> Condition:
    return User.F[field] == value
```

## Back-compat and deprecation notes

- **All existing code keeps working** after this PR. Nothing is removed.
- First class-level descriptor usage (`User.pk == "x"`) emits
  `DeprecationWarning` once per (class, attribute) pair. Message points to
  `cls.F.pk`.
- Warning filter key is stable, so `warnings.filterwarnings("ignore",
  category=DeprecationWarning, module="pydynox")` still silences it for
  users who want time before migrating.
- PR 4 removes the descriptor-style operators entirely.

## Test plan

### Semantic parity

For every operator and atomic op, assert that the old and new forms produce
`Condition` / `AtomicOp` objects that serialize to byte-identical expression
strings:

```python
assert (User.pk == "x").serialize({}, {}) == (User.F.pk == "x").serialize({}, {})
```

Cover:

- Comparisons (`==`, `!=`, `<`, `<=`, `>`, `>=`).
- `exists`, `not_exists`, `begins_with`, `contains`, `between`, `is_in`.
- Boolean combinators (`&`, `|`, `~`).
- Nested path (`User.F.profile["address"]["city"] == "Berlin"`).
- Atomic ops (`set`, `add`, `remove`, `append`, `prepend`, `if_not_exists`).
- Aliased fields: `User.F.<py_name>` uses the DynamoDB alias in serialized
  output, matching today's behavior.

### Deprecation warning

- `pytest.warns(DeprecationWarning)` on first descriptor use.
- No warning on second descriptor use on the same `(class, attr)` pair.
- No warning on `cls.F.<attr>` use.

### Dynamic lookup

- `User.F["pk"] == "x"` equals `User.F.pk == "x"`.
- `User.F["nonexistent"]` raises `KeyError` with a helpful message.

### Integration

Re-run the existing query / transaction / update integration tests with all
uses rewritten to `cls.F.*`; must pass identically.

## Size

M. Roughly 200-300 LOC added (`_refs.py` + wiring), 30-60 LOC changed
(warning shim in `attributes/base.py`). Tests are the bulk.

## Depends on / unblocks

- Depends on: nothing hard, but lands after PR 1 and PR 2 so the docs
  examples already show the new `dynamodb_config` + `Annotated` style.
- Unblocks: PR 4. The descriptor style is removed in PR 4; users who
  migrated to `cls.F.*` see no change.

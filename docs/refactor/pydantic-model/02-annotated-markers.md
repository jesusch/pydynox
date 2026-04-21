# PR 2: `Annotated[T, Dynamo.*]` field markers

**Summary**: add a `pydynox.Dynamo` namespace of metadata markers and teach
`ModelMeta` to synthesize the existing `Attribute` subclasses from
`Annotated[T, Dynamo.*]` annotations. Users can now declare fields in a
Pydantic-friendly style without losing any DynamoDB semantics.

## Motivation

PR 4 cannot happen unless there is a declaration syntax that Pydantic
accepts. PEP 593 `Annotated[T, ...]` is the canonical way to attach
library-specific metadata to a field while letting the type checker see the
underlying `T`. This PR introduces that syntax and makes it equivalent to
the legacy descriptor declarations. Class-body `Attribute` descriptors
continue to work for now, so this is purely additive.

Once this PR lands, users can start migrating at their own pace.

## Scope

In:

- A new module `python/pydynox/markers.py` (or `pydynox/dynamo.py`)
  exporting the `Dynamo` namespace with frozen dataclasses.
- Metaclass support in
  [python/pydynox/_internal/_model/_base.py](../../../python/pydynox/_internal/_model/_base.py)
  to scan `__annotations__` and synthesize `Attribute` instances.
- Parity with every current `Attribute` subclass (see the full list
  below).
- Coexistence: a single class body may mix descriptor-style and
  `Annotated`-style declarations.

Out:

- Making `Model` a `BaseModel`. PR 4.
- `cls.F` namespace. PR 3.
- Removing the descriptor style. PR 4.

## Marker inventory

One marker per current `Attribute` subclass / orthogonal concern. All frozen
`@dataclass(frozen=True, slots=True)`:

| Marker | Maps to | Parameters |
|--------|---------|------------|
| `Dynamo.PartitionKey` | sets `partition_key=True` on synthesized attr | `template: str \| None = None`, `alias: str \| None = None` |
| `Dynamo.SortKey` | sets `sort_key=True` | `template: str \| None = None`, `alias: str \| None = None` |
| `Dynamo.Alias` | overrides DynamoDB attribute name | `name: str` |
| `Dynamo.Template` | sets a template on a non-key attr | `template: str` |
| `Dynamo.AutoGen` | wraps `AutoGenerate` enum as `default` | `strategy: AutoGenerate` |
| `Dynamo.Required` | sets `required=True` | `-` |
| `Dynamo.Discriminator` | sets `discriminator=True` | `-` |
| `Dynamo.TTL` | picks `TTLAttribute` | `ttl_seconds: int \| None = None` |
| `Dynamo.Version` | picks `VersionAttribute` | `-` |
| `Dynamo.Encrypted` | picks `EncryptedAttribute` | `key_id: str`, `mode: EncryptionMode \| None = None`, `region: str \| None = None`, `context: Mapping[str, str] \| None = None` |
| `Dynamo.Compressed` | picks `CompressedAttribute` | `algorithm: str = "zstd"`, `level: int \| None = None` |
| `Dynamo.S3` | picks `S3Attribute` | `bucket: str`, `key_prefix: str \| None = None`, `threshold_bytes: int \| None = None` |
| `Dynamo.JSON` | picks `JSONAttribute` with typed model | `model: type[BaseModel] \| None = None` |
| `Dynamo.Enum` | picks `EnumAttribute` | `enum: type[Enum]` |
| `Dynamo.Datetime` | picks `DatetimeAttribute` | `format: str = "iso8601"` |
| `Dynamo.StringSet` | picks `StringSetAttribute` | `-` |
| `Dynamo.NumberSet` | picks `NumberSetAttribute` | `-` |

Multiple markers on one field are valid and combined:
`Annotated[str, Dynamo.PartitionKey(), Dynamo.Alias("PK")]`.

## Type inference rule

For each annotated field, the synthesized `Attribute` subclass is chosen by
this precedence:

1. An explicit type marker (`Encrypted`, `Compressed`, `S3`, `JSON`, `Enum`,
   `Datetime`, `TTL`, `Version`, `StringSet`, `NumberSet`) -> that exact
   subclass.
2. Else, map the annotated base type `T`:
   - `str` -> `StringAttribute`
   - `int` / `float` -> `NumberAttribute`
   - `bool` -> `BooleanAttribute`
   - `bytes` -> `BinaryAttribute`
   - `list[...]` -> `ListAttribute`
   - `dict[str, ...]` -> `MapAttribute`
   - `set[str]` -> `StringSetAttribute`
   - `set[int] / set[float]` -> `NumberSetAttribute`
   - `Enum` subclass -> `EnumAttribute`
   - `datetime` -> `DatetimeAttribute`
   - `BaseModel` subclass -> `JSONAttribute[That]`

Unknown types raise `TypeError` at class creation with a clear message
pointing at the marker docs.

## Files touched

- [python/pydynox/__init__.py](../../../python/pydynox/__init__.py) - export
  `Dynamo`.
- New `python/pydynox/markers.py` - the dataclass markers and a small
  `_build_attribute(py_name, annotation, default) -> Attribute` helper.
- [python/pydynox/_internal/_model/_base.py](../../../python/pydynox/_internal/_model/_base.py)
  - in `ModelMeta.__new__` (around lines 46-117), after collecting
  descriptor-style attributes, iterate `namespace.get("__annotations__", {})`,
  detect `Annotated` + `Dynamo.*` metadata, call `_build_attribute`, and
  merge the result into the `attributes` dict. Keep existing ordering rules
  for partition/sort/discriminator detection.
- [python/pydynox/attributes/__init__.py](../../../python/pydynox/attributes/__init__.py)
  - no API removal; ensure `Attribute` subclasses accept the kwargs the
  builder passes (they already do).
- [tests/unit/model/](../../../tests/) - full parity test matrix.
- [docs/guides/](../../../docs/guides/) - add an "Annotated fields" guide.

## Public API changes

### Before

```python
from pydynox import Model, ModelConfig
from pydynox.attributes import StringAttribute, EncryptedAttribute

class User(Model):
    model_config = ModelConfig(table="users")
    pk    = StringAttribute(partition_key=True, template="USER#{email}")
    email = StringAttribute()
    ssn   = EncryptedAttribute(key_id="alias/my-key")
```

### After (both work; `Annotated` is the new preferred)

```python
from typing import Annotated, ClassVar
from pydynox import Model, DynamoConfig, Dynamo

class User(Model):
    dynamodb_config: ClassVar[DynamoConfig] = DynamoConfig(table="users")

    pk:    Annotated[str, Dynamo.PartitionKey(template="USER#{email}")]
    email: str
    ssn:   Annotated[str, Dynamo.Encrypted(key_id="alias/my-key")] = None
```

Note: `Field(description=...)` does not work yet - `Model` is not a
`BaseModel` until PR 4. The `Annotated` syntax lands first so the migration
path exists before the breaking change.

## Back-compat and deprecation notes

- **Zero deprecation warnings** in this PR. Both styles are first-class.
- The synthesized `Attribute` objects are identical to what descriptor-style
  declarations produce. `_attributes`, `_py_to_dynamo`, `_partition_key`,
  `_indexes` are bit-identical for equivalent declarations.
- Mixing styles is supported: a class can declare some fields the old way
  and others the new way. Tests cover that.

## Test plan

### Parity tests

For each `Dynamo.*` marker, a pair of classes: one using descriptors, one
using `Annotated`. Assert:

- Same `cls._attributes` keys and types.
- Same `cls._partition_key`, `cls._sort_key`, `cls._py_to_dynamo`.
- Same `to_dict()` / `from_dict()` round-trip for a sample instance.
- Same wire format for special attributes (encrypted bytes length is
  non-deterministic so assert decryptability, not equality).

### Integration tests

Run the existing [tests/integration/](../../../tests/) suite once with a
shadow class tree declared via `Annotated` and assert identical behavior
(save, get, query, GSI query, TTL expiry, version conflict, encrypted
round-trip, S3 offload).

### Negative tests

- Unknown marker type: `Annotated[str, object()]` -> `TypeError` at class
  creation, with error message naming the field.
- Unsupported base type: `Annotated[complex, Dynamo.PartitionKey()]` ->
  `TypeError`.
- Conflicting markers: `Annotated[str, Dynamo.Encrypted(...), Dynamo.Compressed()]`
  -> `TypeError` (or a documented precedence, decided in PR 2 review).

### Docs tests

`Dynamo` namespace in the API docs; one worked example per marker in the
new guide.

## Size

M. Roughly 300-500 LOC added (markers + builder + tests), 50-80 LOC
changed in `ModelMeta`. No removals.

## Depends on / unblocks

- Depends on: PR 1 (so users can also write `dynamodb_config` in the
  examples; not strictly required).
- Unblocks: PR 4 (declaration syntax must exist before Pydantic takes
  over class body).

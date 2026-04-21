# PR 4: `Model` becomes a `pydantic.BaseModel` (breaking, major version)

**Summary**: `Model` inherits from `pydantic.BaseModel` via a combined
metaclass. Field declarations move entirely to pydantic-native
`Annotated[T, Dynamo.*]` + `Field(...)`. The descriptor class-body style
(`pk = StringAttribute(partition_key=True)`) is removed. `User.F.pk` is the
only condition/atomic access path. `ModelConfig`/`model_config` alias is
dropped.

## Motivation

This is the goal of the whole refactor: one canonical way to declare a
DynamoDB model that is also a fully Pydantic-featured class. Everything
through PRs 1-3 was preparation so this PR is mechanical rather than
exploratory: a user who has already migrated their code to
`dynamodb_config` + `Annotated` + `cls.F.*` should see **no runtime change
beyond what Pydantic itself provides**.

## Scope

In:

- Make `Model` subclass `pydantic.BaseModel`.
- Combined metaclass `_PydynoxMeta(ModelMetaclass)` whose `__new__` runs
  Pydantic's schema build first, then pydynox's schema collection via
  `__pydantic_init_subclass__` (or an explicit hook called from `__new__`).
- Drive `_attributes` / `_py_to_dynamo` / `_dynamo_to_py` /
  `_partition_key` / `_sort_key` / `_discriminator_attr` /
  `_discriminator_registry` / `_indexes` / `_local_indexes` / `_hooks`
  from `cls.model_fields` and `FieldInfo.metadata`.
- Remove class-body `Attribute` descriptor support. `StringAttribute`,
  `EncryptedAttribute`, etc. remain as the internal runtime
  representation; they are no longer valid in class bodies.
- Remove the `model_config = ModelConfig(...)` legacy path. `model_config`
  now belongs to Pydantic (`ConfigDict`). Users must use `dynamodb_config`
  (introduced in PR 1).
- Remove the class-level operator descriptor shim from `Attribute`.
  `User.pk` is now a `FieldInfo`. `User.F.pk` is the only condition path.
- Rewire `to_dict()` / `from_dict()` on top of
  `model_dump(by_alias=True, mode="python")` and `model_validate()`, while
  still invoking `Attribute.serialize` / `deserialize` for fields with
  custom transforms (encrypted, compressed, S3, JSON, sets, enum,
  datetime, TTL, version).
- Cooperate with pydantic's `validate_assignment` so dirty tracking
  (`is_dirty`, `changed_fields`, `_json_snapshots`) keeps working.
- Move template-key building to `@model_validator(mode="after")`, while
  keeping `_build_template_keys` call sites in `save()` and
  `_apply_auto_generate` (order matters when auto-gen feeds templates).

Out:

- Deprecating `@dynamodb_model`. PR 5.
- Recommending native pydantic nested `BaseModel` over `JSONAttribute[M]`.
  PR 5 (docs).

## Files touched

Core:

- [python/pydynox/model.py](../../../python/pydynox/model.py) - change
  `class Model(ModelBase, metaclass=ModelMeta)` to
  `class Model(ModelBase, BaseModel, metaclass=_PydynoxMeta)` (or pull
  `ModelBase` into the same file).
- [python/pydynox/_internal/_model/_base.py](../../../python/pydynox/_internal/_model/_base.py)
  - large rewrite of `ModelMeta` -> `_PydynoxMeta`. Field collection moves
  from reading class-body descriptors to reading `cls.model_fields` and
  `FieldInfo.metadata`. `ModelBase.__init__` is replaced by Pydantic's
  `BaseModel.__init__`; template-key post-processing becomes a
  `@model_validator(mode="after")`. `__setattr__` delegates to
  `super().__setattr__` then updates change-tracking.
- [python/pydynox/attributes/base.py](../../../python/pydynox/attributes/base.py)
  - remove the class-level `__get__` descriptor (or keep `__set_name__`
  only for legacy import use). Remove the deprecation shim on operators;
  the operators themselves move to `AttributeRef` (already in PR 3).
- [python/pydynox/config.py](../../../python/pydynox/config.py) - remove
  the back-compat read of `model_config` as a dataclass; `ModelConfig` stays
  as `TypeAlias = DynamoConfig` for import compat only.
- [python/pydynox/__init__.py](../../../python/pydynox/__init__.py) -
  update exports and example.

Serializers:

- For each `Attribute` subclass with non-trivial `serialize`/`deserialize`,
  wire equivalent `@field_serializer(by_alias=True)` /
  `@field_validator(mode="before")` into the synthesized schema. This can
  be done by attaching pydantic `PlainSerializer` / `BeforeValidator`
  annotations to the field when `_PydynoxMeta` rebuilds the schema.
  Affected classes:
  - `EncryptedAttribute` ([attributes/encrypted.py](../../../python/pydynox/attributes/encrypted.py))
  - `CompressedAttribute` ([attributes/compressed.py](../../../python/pydynox/attributes/compressed.py))
  - `S3Attribute` ([attributes/s3.py](../../../python/pydynox/attributes/s3.py))
  - `JSONAttribute` ([attributes/special.py](../../../python/pydynox/attributes/special.py))
  - `EnumAttribute`, `DatetimeAttribute` (same file)
  - `TTLAttribute` ([attributes/ttl.py](../../../python/pydynox/attributes/ttl.py))
  - `VersionAttribute` ([attributes/version.py](../../../python/pydynox/attributes/version.py))
  - `StringSetAttribute`, `NumberSetAttribute` ([attributes/sets.py](../../../python/pydynox/attributes/sets.py))

Downstream consumers (no changes expected, verified by tests):

- [python/pydynox/_internal/_indexes.py](../../../python/pydynox/_internal/_indexes.py)
  (GSI / LSI).
- [python/pydynox/transaction.py](../../../python/pydynox/transaction.py).
- [python/pydynox/batch_operations.py](../../../python/pydynox/batch_operations.py),
  [python/pydynox/_internal/_model/_batch.py](../../../python/pydynox/_internal/_model/_batch.py).
- [python/pydynox/collection.py](../../../python/pydynox/collection.py).
- [python/pydynox/query.py](../../../python/pydynox/query.py).
- [python/pydynox/hooks.py](../../../python/pydynox/hooks.py).
- [python/pydynox/_internal/_metrics.py](../../../python/pydynox/_internal/_metrics.py).
- [python/pydynox/_internal/_model/_*.py](../../../python/pydynox/_internal/_model/).

Docs + examples:

- [docs/getting-started.md](../../../docs/getting-started.md) - rewrite
  the quickstart against the new syntax.
- [docs/guides/](../../../docs/guides/) - update every guide that shows a
  class-body `StringAttribute(...)` declaration.
- [docs/examples/](../../../docs/examples/) - ditto.

## Public API changes

### Minimal before / after

Before (last minor release):

```python
from pydynox import Model, ModelConfig
from pydynox.attributes import StringAttribute

class User(Model):
    model_config = ModelConfig(table="users")
    pk = StringAttribute(partition_key=True)
    name = StringAttribute()
```

After (major release):

```python
from typing import Annotated, ClassVar
from pydantic import Field
from pydynox import Model, DynamoConfig, Dynamo

class User(Model):
    dynamodb_config: ClassVar[DynamoConfig] = DynamoConfig(table="users")

    pk:   Annotated[str, Dynamo.PartitionKey()] = Field(description="User PK")
    name: str = Field(description="Display name", max_length=120)
```

### Full worked example with complex features

```python
from datetime import datetime
from enum import Enum
from typing import Annotated, ClassVar
from pydantic import BaseModel, Field
from pydynox import Model, DynamoConfig, Dynamo
from pydynox.generators import AutoGenerate
from pydynox.indexes import GlobalSecondaryIndex, LocalSecondaryIndex

class Role(str, Enum):
    admin  = "admin"
    member = "member"

class Address(BaseModel):
    street: str
    city:   str

class User(Model):
    dynamodb_config: ClassVar[DynamoConfig] = DynamoConfig(
        table="users",
        consistent_read=False,
        hot_partition_writes=2000,
    )

    # Keys with template (PEP 750 t-strings also supported)
    pk:    Annotated[str, Dynamo.PartitionKey(template="USER#{email}")] = Field(description="User PK")
    sk:    Annotated[str, Dynamo.SortKey()]        = "PROFILE"

    # Regular fields with full Pydantic validation
    email: str   = Field(pattern=r".+@.+", description="Primary email")
    name:  str   = Field(min_length=1, max_length=120)
    role:  Annotated[Role, Dynamo.Enum(Role)]  = Role.member

    # Nested pydantic model (stored as JSON in DynamoDB)
    address: Annotated[Address, Dynamo.JSON(model=Address)] | None = None

    # DynamoDB-specific features
    ssn:       Annotated[str, Dynamo.Encrypted(key_id="alias/my-key")] | None = None
    avatar:    Annotated[bytes, Dynamo.S3(bucket="user-avatars")] | None      = None
    tags:      Annotated[set[str], Dynamo.StringSet()]                        = Field(default_factory=set)
    version:   Annotated[int, Dynamo.Version()]       = 0
    ttl:       Annotated[datetime, Dynamo.TTL()] | None = None
    created_at: datetime = Field(default_factory=datetime.utcnow)

    # Indexes unchanged
    email_index    = GlobalSecondaryIndex(index_name="email-index", partition_key="email")
    location_index = GlobalSecondaryIndex(
        index_name="loc-index",
        partition_key=["tenant_id", "region"],
        sort_key=["status", "created_at"],
        projection=["email", "status"],
    )
    status_index   = LocalSecondaryIndex(index_name="status-index", sort_key="role")

# Usage unchanged (except User.F.* for conditions)
user = User(email="jane@example.com", name="Jane")
await user.save()

async for u in User.email_index.query(partition_key="jane@example.com"):
    print(u.name)

await user.update(role=Role.admin)
await user.delete()
```

## Back-compat and deprecation notes

- **Breaking**:
  - Class-body `Attribute` descriptors (`pk = StringAttribute(...)`) are no
    longer valid. Use `Annotated[T, Dynamo.*]`. Class creation raises
    `TypeError` with a migration hint if detected.
  - `model_config = ModelConfig(...)` no longer works (conflicts with
    pydantic). Use `dynamodb_config: ClassVar[DynamoConfig] = ...`.
  - `User.pk == "x"` no longer returns a `Condition`. Use `User.F.pk == "x"`.
    Running the old style raises `TypeError` (or returns pydantic's
    `FieldInfo.__eq__` result) rather than silently doing the wrong thing.
- **Preserved**:
  - `ModelConfig` is still importable as a type alias for `DynamoConfig`.
  - `to_dict()` / `from_dict()` signatures unchanged.
  - `Model.sync_query` / `Model.query` / `Model.save` / `Model.get` /
    `Model.update` / `Model.delete` / `Model.batch_get` /
    `Model.create_table` / etc. unchanged.
  - Hooks, indexes, transactions, atomic ops, AutoGenerate, discriminator,
    S3 offload, encryption, compression, TTL, version - all behavior
    preserved.
- **New**:
  - Every Pydantic feature is now available: `Field(description=...)`,
    `Field(pattern=...)`, `@field_validator`, `@model_validator`,
    `model_json_schema()`, `model_dump_json()`, nested `BaseModel` fields,
    `ConfigDict` via pydantic's `model_config`.

## Migration guide

A new section added to [docs/guides/](../../../docs/guides/) with a
step-by-step migration. High level:

1. Upgrade to the minor release that contains PRs 1-3. Migrate at leisure:
   - `model_config = ModelConfig(...)` -> `dynamodb_config: ClassVar[DynamoConfig] = DynamoConfig(...)`.
   - `pk = StringAttribute(partition_key=True)` -> `pk: Annotated[str, Dynamo.PartitionKey()]`.
   - `User.pk == "x"` -> `User.F.pk == "x"`.
2. Upgrade to the major release with this PR. If all three migrations are
   done, no further changes needed.
3. Optional: adopt Pydantic features
   (`Field(description=...)`, validators, nested models).

## Test plan

This is the PR with the largest test footprint.

### Pre-cutover snapshot

Before merging, add a comprehensive behavioral snapshot suite in
[tests/integration/](../../../tests/). For each DynamoDB feature, record the
wire format (raw dict passed to the client) and the round-trip output for
the descriptor-style declaration. Keep this file frozen. The cutover must
produce byte-identical wire output for equivalent `Annotated` declarations.

### Core pydantic integration

- `model_json_schema()` on a `Model` subclass includes descriptions from
  `Field(description=...)`.
- `@field_validator` runs during `__init__` and during `update()`.
- `@model_validator(mode="after")` fires after template-key generation (so
  validators see the computed key values).
- `ConfigDict(validate_assignment=True)` still triggers dirty tracking.

### Dirty tracking regression

Critical regression area. Cover:

- `user = await User.get(...); user.name = "new"` -> `is_dirty == True`,
  `"name" in user.changed_fields`.
- `user.name = user.name` (no-op) -> not dirty.
- `user.address = Address(...)` -> dirty (nested model replacement).
- `user.address.city = "Berlin"` (in-place mutation of a nested model) ->
  dirty via JSON snapshot mechanism.
- `user.tags.add("premium")` (in-place set mutation) -> dirty (covered by
  either `__setattr__` or by a post-save snapshot of set fields).
- `@model_validator(mode="after")` template rebuild does not spuriously
  mark pk/sk as dirty.

### Feature parity matrix

Run one instance per feature:

- Composite GSI partition key, composite sort key, INCLUDE projection.
- LSI with `consistent_read=True`.
- Inverted-index template: `inverted_index.query(order_id="456")` ->
  `"ORDER#456"`.
- Encrypted field round-trip with a fake KMS client.
- Compressed field round-trip.
- S3 field offload on save, delete on delete.
- Version field: two clients, same base version, second save raises
  `ConditionalCheckFailedException`.
- TTL: `is_expired()`, `extend_ttl(...)`, absent TTL.
- Discriminator: `cls.from_dict({"type": "Admin", ...})` returns `Admin`.
- String/Number sets: native DynamoDB `SS`/`NS`, not `L`.
- AutoGenerate: `ULID`, `UUID4`, `KSUID`, `EPOCH`, `EPOCH_MS`, `ISO8601`.
- Hooks: `BEFORE_SAVE`, `AFTER_LOAD`, skip-hooks via `dynamodb_config`.
- `create_table` / `table_exists` / `delete_table` produce identical
  DynamoDB schema.

### Negative tests

- Class body containing `pk = StringAttribute(partition_key=True)` raises
  `TypeError` with migration hint.
- Class body containing `model_config = ModelConfig(...)` raises
  `TypeError` with migration hint.
- `Model` subclass without `dynamodb_config` raises `ValueError` when
  `_get_table()` is called, as today.

## Size

L. Realistic estimate: 800-1500 LOC net change, of which ~60% is test
additions. Metaclass rewrite is the densest part.

## Depends on / unblocks

- Depends on: PR 1, PR 2, PR 3.
- Unblocks: PR 5.
- Ships in: next major version.

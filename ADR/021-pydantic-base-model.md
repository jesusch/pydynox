# ADR 021: Pydantic BaseModel for Model

## Status

Proposed

## Context

pydynox currently offers two overlapping ways to declare a DynamoDB-backed
model, and both have structural problems:

1. The **`Model` class** in [python/pydynox/model.py](../python/pydynox/model.py)
   with `Attribute` descriptors in [python/pydynox/attributes/](../python/pydynox/attributes/).
   Full DynamoDB feature set (GSI/LSI, templates, aliases, encryption, TTL,
   version, S3 offload, discriminator, hooks, auto-generate, dirty tracking,
   transactions, batch). No Pydantic-level features: no `Field(description=...)`,
   no validators, no JSON schema, no `model_dump_json()`.

2. The **`@dynamodb_model` decorator** in
   [python/pydynox/integrations/pydantic.py](../python/pydynox/integrations/pydantic.py).
   Wraps an arbitrary `BaseModel` and adds `save`/`get`/`delete`/`update` only.
   Loses almost every pydynox DynamoDB feature listed above.

Users who want both a rich Pydantic schema (descriptions, validators,
constraints, JSON schema) and pydynox's DynamoDB features today have no path.
Asking them to "put the DynamoDB config into `model_config`" is blocked by
Pydantic v2, which reserves `model_config` as a `ConfigDict`.

## Decision

Evolve `Model` so that it **is a `pydantic.BaseModel`** subclass. Field
declaration uses idiomatic Pydantic:

```python
class User(Model):
    dynamodb_config: ClassVar[DynamoConfig] = DynamoConfig(table="users")

    pk:    Annotated[str, Dynamo.PartitionKey(template="USER#{email}")] = Field(description="User PK")
    email: str = Field(description="Email address")
    ssn:   Annotated[str, Dynamo.Encrypted(key_id="alias/my-key")] = Field(default=None)

    email_index = GlobalSecondaryIndex(index_name="email-index", partition_key="email")
```

Key points:

- The existing `Attribute` subclasses (`StringAttribute`, `EncryptedAttribute`,
  `JSONAttribute`, ...) remain as the **internal runtime representation** that
  carries `serialize` / `deserialize` / `alias` / `attr_type` /
  `has_template` / `placeholders`. They are no longer typed in class bodies.
- Declarations use PEP 593 `Annotated[T, Dynamo.*]` markers to convey DynamoDB
  semantics, plus Pydantic's `Field(...)` for everything else.
- Per-model configuration moves to `dynamodb_config: ClassVar[DynamoConfig]`
  because `model_config` belongs to Pydantic.
- Condition and atomic-op builders move from `User.pk == "x"` to
  `User.F.pk == "x"` (a sibling namespace), because Pydantic owns class-level
  field access.

The refactor ships as five staged PRs so each step is reviewable and,
for PRs 1-3, non-breaking. See
[docs/refactor/pydantic-model/overview.md](../docs/refactor/pydantic-model/overview.md)
for the full plan.

## Reasons

- **Users gain Pydantic's full feature surface for free**: `Field(description=...)`,
  `Field(examples=..., ge=..., pattern=...)`, `@field_validator`,
  `@model_validator`, `model_json_schema()`, `model_dump_json()`, discriminated
  unions, nested `BaseModel` fields.
- **The DynamoDB layer is preserved**: almost all downstream code
  (CRUD, GSI/LSI, transactions, hooks, metrics, `create_table`) consumes the
  `_attributes` / `_py_to_dynamo` / `_indexes` class dunders, not the
  descriptor protocol. Populating those dunders from Pydantic `FieldInfo`
  instead of class-body descriptors is a localized metaclass change.
- **Two worlds collapse to one**: the `@dynamodb_model` decorator becomes
  obsolete once subclassing `Model` already gives you a Pydantic model.
- **Layered rollout**: PRs 1-3 add the new surface without removing the old
  one, so early adopters can migrate gradually; PR 4 performs the cutover
  in a major version.

## Alternatives considered

- **Leave the two worlds as-is**: rejected. The decorator path has been
  steadily accumulating feature gaps (no aliases, no templates, no indexes,
  no hooks, no encryption, ...), and reconciling them ad hoc would double
  the work.
- **Extend the decorator path to reach feature parity**: rejected. It would
  require re-implementing the entire `Attribute` serialize/deserialize
  pipeline, `ModelMeta` collection, and index binding on top of ad hoc
  monkey-patching in [python/pydynox/integrations/_base.py](../python/pydynox/integrations/_base.py).
  The result would be two parallel implementations of the same logic.
- **Keep `Attribute` descriptors and just add Pydantic validation on top**:
  rejected. Pydantic's `BaseModel` owns `__init__`, `__setattr__`, and class
  creation; you cannot mix class-body descriptors with Pydantic fields
  without extreme metaclass gymnastics, and users would lose `Field(...)`
  declarations on the affected attributes anyway.
- **Reuse `model_config` for DynamoDB settings**: rejected. Pydantic v2
  reserves that name for `ConfigDict`; arbitrary keys get warnings and
  type-checked shape. A separate `dynamodb_config` name is the clean fix.

## Consequences

Positive:

- Users get `Field(description=...)` and every other Pydantic feature.
- The existing `Attribute` machinery stays as internal implementation detail,
  so serialization/encryption/compression/S3/TTL/version/sets all work
  unchanged.
- GSI/LSI, hooks, transactions, batch, `create_table`, metrics, collections,
  parallel scan, PartiQL: no changes, they read `cls._attributes` and friends.
- The deprecated decorator path goes away; one canonical way to declare
  a DynamoDB model.

Negative / trade-offs:

- **Single breaking change**: `User.pk == "x"` becomes `User.F.pk == "x"`.
  A descriptor shim is provided for one release with `DeprecationWarning`.
- **PR 4 is a major version**. PRs 1-3 are non-breaking (minor versions).
- **`__setattr__`-based dirty tracking** must cooperate with Pydantic's
  `validate_assignment`. Regression tests for `is_dirty`/`changed_fields`
  must accompany PR 4.
- **Templates** on key fields now rely on a `@model_validator(mode="after")`
  plus the existing `_build_template_keys` call in `save()`; must be
  covered with tests.

## References

- Detailed plan + architecture diagrams:
  [docs/refactor/pydantic-model/overview.md](../docs/refactor/pydantic-model/overview.md)
- PR 1 (non-breaking):
  [01-dynamoconfig-rename.md](../docs/refactor/pydantic-model/01-dynamoconfig-rename.md)
- PR 2 (non-breaking):
  [02-annotated-markers.md](../docs/refactor/pydantic-model/02-annotated-markers.md)
- PR 3 (non-breaking):
  [03-fields-namespace.md](../docs/refactor/pydantic-model/03-fields-namespace.md)
- PR 4 (breaking, major version):
  [04-basemodel-cutover.md](../docs/refactor/pydantic-model/04-basemodel-cutover.md)
- PR 5 (post-cutover cleanup):
  [05-deprecate-decorator.md](../docs/refactor/pydantic-model/05-deprecate-decorator.md)

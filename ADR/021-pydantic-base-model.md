# ADR 021: Optional Pydantic integration via `PydanticModel`

## Status

Proposed (revised after maintainer feedback; Pydantic stays optional and
`Model` is not changed).

## Context

pydynox currently offers two overlapping ways to declare a DynamoDB-backed
model:

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

### Constraint: Pydantic must remain optional

pydynox's core value is speed and a minimal dependency footprint. The Rust
core handles serialization, compression, encryption, and all AWS SDK calls.
Making Pydantic a required runtime dependency goes against that principle.
Pydantic is installed today via the `pydynox[pydantic]` extra and must stay
optional.

This rules out evolving `Model` into a `pydantic.BaseModel` subclass, which
would force every user to depend on Pydantic.

## Decision

Introduce a new, **opt-in** `PydanticModel` class that inherits from both
`Model` and `pydantic.BaseModel`:

```python
from pydynox import Model          # no Pydantic; works exactly as today
from pydynox import PydanticModel  # Model + BaseModel; requires pydynox[pydantic]
```

`PydanticModel` lives in a new module
[python/pydynox/pydantic_model.py](../python/pydynox/pydantic_model.py) and is
lazily imported: a user without Pydantic installed can still
`import pydynox` and use `Model` as today; only `from pydynox import
PydanticModel` triggers the import and raises a clear `ImportError` with a
pointer to `pip install pydynox[pydantic]` if Pydantic is missing.

Supporting changes, all additive and non-breaking:

- Rename `ModelConfig` to `DynamoConfig` and expose a new
  `dynamodb_config: ClassVar[DynamoConfig]` attribute so `PydanticModel`
  subclasses can also set Pydantic's `model_config = ConfigDict(...)`
  without a naming collision. See
  [docs/refactor/pydantic-model/01-dynamoconfig-rename.md](../docs/refactor/pydantic-model/01-dynamoconfig-rename.md).
- Add `Annotated[T, Dynamo.*]` markers as an alternative declaration style
  for fields, since `PydanticModel` cannot use class-body `Attribute`
  descriptors. `Model` users keep both styles. See
  [docs/refactor/pydantic-model/02-annotated-markers.md](../docs/refactor/pydantic-model/02-annotated-markers.md).
- Add a `cls.F` namespace for condition and atomic operators, because
  Pydantic owns class-level field access on `PydanticModel` and
  `User.pk == "x"` cannot work there. `Model` users gain `cls.F` as a
  sibling path but keep the existing descriptor style with no warnings.
  See [docs/refactor/pydantic-model/03-fields-namespace.md](../docs/refactor/pydantic-model/03-fields-namespace.md).
- Deprecate the `@dynamodb_model` decorator and point users at
  `PydanticModel` as the canonical Pydantic path. See
  [docs/refactor/pydantic-model/05-deprecate-decorator.md](../docs/refactor/pydantic-model/05-deprecate-decorator.md).

## Reasons

- **Zero new required dependencies.** Pydantic stays behind the
  `pydynox[pydantic]` extra. Users who do not want it pay nothing and see
  no API change.
- **Single codebase for the DynamoDB feature set.** `PydanticModel`
  inherits from `Model`, so GSI/LSI, transactions, batch, hooks, metrics,
  `create_table`, atomic ops, version, TTL, encryption, compression, S3
  offload, discriminator, and auto-generate are all shared. No duplicated
  implementation.
- **No breaking changes.** All five PRs in the rollout ship on minor
  versions. Existing `Model` subclasses keep working unchanged.
- **Clean deprecation path for `@dynamodb_model`.** Users wanting Pydantic
  migrate to `PydanticModel` and gain the full DynamoDB feature set at
  the same time.

## Alternatives considered

- **Make `Model` a `BaseModel` directly** (original proposal): rejected.
  Forces Pydantic as a required dependency, violates the minimal-dependency
  principle, and would break every existing user with a major-version
  bump. The maintainer pushed back explicitly on this in
  [question.md](../question.md).
- **Leave the two worlds as-is**: rejected. The decorator path keeps
  accumulating feature gaps (no aliases, no templates, no indexes, no
  hooks, no encryption, ...), and reconciling them ad hoc would double
  the work.
- **Extend the decorator path to reach feature parity**: rejected. It would
  require re-implementing the entire `Attribute` serialize/deserialize
  pipeline, `ModelMeta` collection, and index binding on top of ad hoc
  monkey-patching in [python/pydynox/integrations/_base.py](../python/pydynox/integrations/_base.py).
  The result would be two parallel implementations of the same logic.
- **Reuse `model_config` for DynamoDB settings**: rejected. Pydantic v2
  reserves that name for `ConfigDict`; arbitrary keys get warnings and
  type-checked shape. A separate `dynamodb_config` name is the clean fix,
  and it benefits `Model` users too by avoiding confusion with pydantic's
  naming convention.

## Consequences

Positive:

- `Model` users see zero change beyond the additive `cls.F` namespace and
  the optional `dynamodb_config` name.
- `PydanticModel` users get every Pydantic feature
  (`Field(description=...)`, `Field(pattern=..., ge=..., max_length=...)`,
  `@field_validator`, `@model_validator`, `model_json_schema()`,
  `model_dump_json()`, discriminated unions, nested `BaseModel` fields)
  **and** every pydynox DynamoDB feature.
- Only one class to maintain for DynamoDB logic. `PydanticModel` is a
  thin layer over `Model`.
- Clean story for deprecating `@dynamodb_model`: point users at
  `PydanticModel`.

Negative / trade-offs:

- Two public model base classes (`Model` and `PydanticModel`) instead of
  one. Documentation must explain when to pick which.
- `PydanticModel` users must use `Annotated[...]` declarations
  (`pk: Annotated[str, Dynamo.PartitionKey()]`) and `cls.F.pk == "x"` for
  conditions. These are Pydantic's constraints, not pydynox's choice.
- `__setattr__`-based dirty tracking must cooperate with Pydantic's
  `validate_assignment` on `PydanticModel`. Handled in the PR 4 design by
  delegating to `BaseModel.__setattr__` first and then updating change
  tracking.

## References

- Detailed plan + architecture diagrams:
  [docs/refactor/pydantic-model/overview.md](../docs/refactor/pydantic-model/overview.md)
- PR 1 (non-breaking):
  [01-dynamoconfig-rename.md](../docs/refactor/pydantic-model/01-dynamoconfig-rename.md)
- PR 2 (non-breaking):
  [02-annotated-markers.md](../docs/refactor/pydantic-model/02-annotated-markers.md)
- PR 3 (non-breaking):
  [03-fields-namespace.md](../docs/refactor/pydantic-model/03-fields-namespace.md)
- PR 4 (non-breaking; introduces `PydanticModel`):
  [04-pydantic-model.md](../docs/refactor/pydantic-model/04-pydantic-model.md)
- PR 5 (non-breaking; deprecates the decorator):
  [05-deprecate-decorator.md](../docs/refactor/pydantic-model/05-deprecate-decorator.md)
- Maintainer feedback that prompted this revision:
  [question.md](../question.md)

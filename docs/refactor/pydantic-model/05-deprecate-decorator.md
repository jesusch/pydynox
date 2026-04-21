# PR 5: Deprecate `@dynamodb_model` and consolidate docs

**Summary**: now that subclassing `Model` gives users a fully Pydantic-capable
DynamoDB model, the `@dynamodb_model` decorator becomes redundant. This PR
adds a `DeprecationWarning`, updates all docs/examples to point at the
unified `Model(BaseModel)` path, and (optionally) prefers native pydantic
nested `BaseModel` over `JSONAttribute[M]` in guides.

## Motivation

Pre-refactor the decorator existed because subclassing `Model` meant giving
up Pydantic features. After PR 4 that trade-off is gone. Keeping two paths
will confuse users, split documentation, and create maintenance drag.

This PR does **not** remove the decorator. Removal would break users who
pinned to the old path. Deprecation here, removal in a later major.

## Scope

In:

- Add `DeprecationWarning` at decorator-call time in
  [python/pydynox/integrations/pydantic.py](../../../python/pydynox/integrations/pydantic.py)
  and in
  [python/pydynox/integrations/functions.py](../../../python/pydynox/integrations/functions.py).
- Update every doc, guide, example, and ADR that mentions the decorator to
  use the subclass path.
- Add a migration guide entry (separate from the PR 4 migration guide):
  "Migrating from `@dynamodb_model`".
- Mark `pydynox/integrations/pydantic.py` and
  `pydynox/integrations/dataclass.py` as deprecated in docstrings and
  flag them as deprecated in the public API docs.
- Optional doc change: recommend native pydantic nested `BaseModel` fields
  over `JSONAttribute[M]` in new code. Keep `JSONAttribute[M]` working
  for back-compat; do not deprecate it.

Out:

- Actually removing the decorator. Scheduled for a later major version.
- Removing `JSONAttribute[M]`. It stays; only docs guidance shifts.
- Any code changes outside the two integration modules and docs.

## Files touched

Code:

- [python/pydynox/integrations/pydantic.py](../../../python/pydynox/integrations/pydantic.py)
  - add `DeprecationWarning` in `dynamodb_model` (currently at lines
  47-69) and `from_pydantic` (lines 72-120). Message points to
  [docs/refactor/pydantic-model/overview.md](overview.md) and the
  subclass example.
- [python/pydynox/integrations/functions.py](../../../python/pydynox/integrations/functions.py)
  - add the same warning in its `dynamodb_model` entry point.
- [python/pydynox/integrations/dataclass.py](../../../python/pydynox/integrations/dataclass.py)
  - same treatment for `from_dataclass`. Document that pydantic's
  `dataclass` decorator composed with `Model(BaseModel)` is the
  recommended path going forward.
- [python/pydynox/integrations/__init__.py](../../../python/pydynox/integrations/__init__.py)
  - add a module-level deprecation note in the docstring.
- [python/pydynox/__init__.py](../../../python/pydynox/__init__.py) - keep
  `dynamodb_model` in `__all__` but annotate in the docstring.

Docs + examples:

- [README.md](../../../README.md) - remove decorator from the headline
  example.
- [docs/getting-started.md](../../../docs/getting-started.md) - single
  canonical example uses subclass path.
- [docs/guides/](../../../docs/guides/) - purge decorator examples, add
  migration guide.
- [docs/examples/](../../../docs/examples/) - update example files.
- [CHANGELOG.md](../../../CHANGELOG.md) - note the deprecation and link
  to the migration guide.
- [ADR/021-pydantic-base-model.md](../../../ADR/021-pydantic-base-model.md)
  - flip status to "Accepted" if not already, add a "Superseded pieces"
  note.

## Public API changes

None. All existing decorator usage keeps working.

### Warning behavior

Calling `@dynamodb_model(...)` (or `from_pydantic(...)`, `from_dataclass(...)`)
emits:

```text
DeprecationWarning: pydynox @dynamodb_model is deprecated since X.Y.0.
Subclass pydynox.Model directly - you now get Pydantic features and the
full DynamoDB feature set in one place. See
docs/refactor/pydantic-model/overview.md for the migration guide.
```

Emitted once per wrapped class.

## Migration guide (for this PR specifically)

Before:

```python
from pydantic import BaseModel
from pydynox import DynamoDBClient, dynamodb_model

client = DynamoDBClient(region="us-east-1")

@dynamodb_model(table="users", partition_key="pk", client=client)
class User(BaseModel):
    pk: str
    name: str

user = User(pk="USER#1", name="John")
await user.save()
```

After:

```python
from typing import Annotated, ClassVar
from pydantic import Field
from pydynox import Model, DynamoConfig, Dynamo, DynamoDBClient, set_default_client

set_default_client(DynamoDBClient(region="us-east-1"))

class User(Model):
    dynamodb_config: ClassVar[DynamoConfig] = DynamoConfig(table="users")

    pk:   Annotated[str, Dynamo.PartitionKey()] = Field(description="User PK")
    name: str = Field(description="Display name")

user = User(pk="USER#1", name="John")
await user.save()
```

The second form gives you everything the decorator form had, plus
encryption, S3 offload, GSI/LSI query, transactions, hooks, atomic ops,
discriminator, version, TTL, auto-generate, dirty tracking, and metrics.

## Back-compat and deprecation notes

- Deprecation only. No removal in this PR.
- All three deprecated entry points (`dynamodb_model`, `from_pydantic`,
  `from_dataclass`) keep functioning unchanged.
- Removal scheduled for the next major after this one. Record this in
  [CHANGELOG.md](../../../CHANGELOG.md) and in a follow-up issue so it is
  not forgotten.

## Test plan

- `pytest.warns(DeprecationWarning)` when calling `@dynamodb_model(...)`
  on a `BaseModel`.
- `pytest.warns(DeprecationWarning)` when calling `from_pydantic(...)` or
  `from_dataclass(...)` directly.
- All existing tests for the decorator path in
  [tests/unit/integrations/](../../../tests/) and
  [tests/integration/](../../../tests/) keep passing (ignoring the
  deprecation warning).
- Docs build passes with the updated examples.
- Link check on [docs/](../../../docs/) so the migration guide links
  resolve.

## Size

S-M. Mostly doc churn. Code change is ~20-40 LOC of warning plumbing.

## Depends on / unblocks

- Depends on: PR 4 (the subclass path must actually be feature-complete
  before we point users at it).
- Unblocks: a future PR that removes the decorator entirely (next major
  after this).

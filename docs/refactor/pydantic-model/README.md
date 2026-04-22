# Optional Pydantic integration refactor

This folder collects the planning documents for giving pydynox users an
**opt-in** path to Pydantic features (`Field(description=...)`, validators,
JSON schema, `model_dump_json()`, ...) without changing the existing
`Model` class and without forcing Pydantic as a required dependency.

The refactor introduces a new `PydanticModel(Model, BaseModel)` sibling
class. `Model` itself stays exactly as today. Pydantic stays behind the
`pydynox[pydantic]` extra.

## Status

Proposed. See [ADR 021](../../../ADR/021-pydantic-base-model.md).

## Reading order

1. **[overview.md](overview.md)** - start here. Motivation, current vs
   target architecture (with diagrams), feature-by-feature mapping,
   risks, and rollout plan.
2. **[01-dynamoconfig-rename.md](01-dynamoconfig-rename.md)** - PR 1,
   non-breaking. Introduce `DynamoConfig` / `dynamodb_config` alongside
   the existing `ModelConfig` / `model_config`. Unblocks PR 4 because
   `PydanticModel` needs `model_config` for Pydantic's `ConfigDict`.
3. **[02-annotated-markers.md](02-annotated-markers.md)** - PR 2,
   non-breaking. Add `Annotated[T, Dynamo.*]` markers as an alternative
   declaration style. Optional on `Model`; required on `PydanticModel`.
4. **[03-fields-namespace.md](03-fields-namespace.md)** - PR 3,
   non-breaking. Add `cls.F` namespace carrying the condition / atomic
   operators. The old `User.pk == "x"` keeps working on `Model` with no
   warning. `PydanticModel` requires `cls.F` because Pydantic owns
   class-level field access.
5. **[04-pydantic-model.md](04-pydantic-model.md)** - PR 4, non-breaking.
   Introduce the opt-in `PydanticModel(Model, BaseModel)` class. Users
   who want Pydantic features subclass this instead of `Model`.
6. **[05-deprecate-decorator.md](05-deprecate-decorator.md)** - PR 5,
   non-breaking. Deprecate `@dynamodb_model` and point users at
   `PydanticModel`. Update docs and examples.

## At a glance

| PR | Title | Breaking? | Size | Ships in |
|----|-------|-----------|------|----------|
| 1  | `DynamoConfig` rename | no | S | minor |
| 2  | `Annotated` markers | no | M | minor |
| 3  | `cls.F` namespace | no | M | minor |
| 4  | `PydanticModel` sibling class | no | M | minor |
| 5  | Deprecate `@dynamodb_model` | no | S-M | minor |

Every PR ships on a minor version. No major-version bump required.

## Conventions for these docs

Each PR document uses the same sections:

- **Title** + one-line summary
- **Motivation**
- **Scope** (in / out)
- **Files touched**
- **Public API changes** (before / after)
- **Back-compat + deprecation notes**
- **Test plan**
- **Size estimate**
- **Depends on / unblocks**

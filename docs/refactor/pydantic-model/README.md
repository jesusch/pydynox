# Pydantic BaseModel Refactor

This folder collects the planning documents for migrating `pydynox.Model`
onto `pydantic.BaseModel`, preserving every existing DynamoDB feature while
unlocking Pydantic niceties (`Field(description=...)`, validators, JSON
schema, ...).

## Status

Proposed. See [ADR 021](../../../ADR/021-pydantic-base-model.md).

## Reading order

1. **[overview.md](overview.md)** - start here. Motivation, current vs target
   architecture (with diagrams), feature-by-feature mapping, risks, and
   rollout plan.
2. **[01-dynamoconfig-rename.md](01-dynamoconfig-rename.md)** - PR 1, non-breaking.
   Introduce `DynamoConfig` / `dynamodb_config` alongside the existing
   `ModelConfig` / `model_config`.
3. **[02-annotated-markers.md](02-annotated-markers.md)** - PR 2, non-breaking.
   Add `Annotated[T, Dynamo.*]` markers as an alternative to class-body
   `Attribute` descriptors.
4. **[03-fields-namespace.md](03-fields-namespace.md)** - PR 3, non-breaking.
   Add `cls.F` namespace carrying the condition / atomic operators. Old
   `User.pk == "x"` keeps working with a deprecation warning.
5. **[04-basemodel-cutover.md](04-basemodel-cutover.md)** - PR 4, breaking
   (major version). Make `Model(BaseModel)`. Drop the descriptor style in
   class bodies. `User.F.pk` is now the only way.
6. **[05-deprecate-decorator.md](05-deprecate-decorator.md)** - PR 5, after
   PR 4 stabilizes. Deprecate `@dynamodb_model`, update docs and examples.

## At a glance

| PR | Title | Breaking? | Size | Ships in |
|----|-------|-----------|------|----------|
| 1  | `DynamoConfig` rename | no | S | minor |
| 2  | `Annotated` markers | no | M | minor |
| 3  | `cls.F` namespace | no | M | minor |
| 4  | `BaseModel` cutover | yes | L | major |
| 5  | Deprecate decorator | no | S-M | minor after 4 |

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

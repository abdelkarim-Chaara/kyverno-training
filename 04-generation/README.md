# Module 04 — Generation Policies

Generation policies automatically **create new resources** when a trigger resource is created. They are managed by the Background Controller.

## Key Concepts

- **Downstream resource** — the new resource created by the generate rule
- **Synchronize** — when `synchronize: true`, Kyverno keeps the downstream resource in sync with the policy. If the policy is deleted, the downstream resource is also deleted.
- **generateExisting** — set to `true` to apply the generation rule to resources that already exist
- **clone vs data** — use `data` to define the resource inline; use `clone` to copy from an existing resource

## Clone vs Data

| | `data` | `clone` |
|---|---|---|
| Source | Defined inline in the policy | Copied from an existing resource |
| On policy delete | Downstream deleted (policy is the source) | Downstream NOT deleted (cloned resource is the source) |
| Use case | Standard generated resources | Spreading secrets/configs across namespaces |

## Sub-modules

| Sub-module | Topic |
|---|---|
| [01 - Data](./01-data/README.md) | Generate resources defined inline |
| [02 - Clone](./02-clone/README.md) | Copy existing resources to new namespaces |
| [03 - CloneList](./03-clonelist/README.md) | Clone multiple resources by label selector |

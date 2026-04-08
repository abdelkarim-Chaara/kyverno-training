# Module 03 — Mutation Policies

Mutation policies automatically modify resources **before** they are stored in etcd. They run at admission time and are transparent to users.

> **Important:** Mutation always happens BEFORE validation. If you have both mutation and validation policies, the mutated resource is what gets validated.

## Sub-modules

| Sub-module | Topic |
|---|---|
| [01 - patchStrategicMerge](./01-patch-strategic-merge/README.md) | Overlay/merge approach for adding and modifying fields |
| [02 - JSON Patch (json6902)](./02-json6902/README.md) | Precise add/replace/remove operations |
| [03 - ForEach Mutation](./03-foreach/README.md) | Mutate each item in a list |
| [04 - Mutate Existing](./04-mutate-existing/README.md) | Modify existing resources via background controller |

## Two Mutation Strategies

### patchStrategicMerge — The Overlay Approach

You provide a partial YAML definition that gets **merged** onto the original resource.

- Best for: adding/overlaying data, setting defaults
- Supports: conditional logic (anchors)
- Supports: auto-gen rules for Deployments, Jobs, etc.
- Readability: High

### JSON Patch (patchesJson6902) — Precise Operations

A list of operations (add, replace, remove) targeting specific JSON paths.

- Best for: removing fields, modifying specific list elements
- Does NOT support: auto-gen, conditional logic (use preconditions instead)
- Readability: Lower

## Auto-Gen Rules

When you write a mutation rule for Pods, Kyverno automatically generates equivalent rules for controllers that create pods:

- `Deployment`
- `DaemonSet`
- `StatefulSet`
- `Job`
- `CronJob`

**Check auto-gen rules:**
```bash
kubectl get clusterpolicy <name> -o yaml
# Look for autogen- rules in the spec
```

**Disable or customize auto-gen:**
```yaml
# Restrict to specific controllers only:
metadata:
  annotations:
    pod-policies.kyverno.io/autogen-controllers: "Deployment,Job"
```

```yaml
# Disable auto-gen entirely:
metadata:
  annotations:
    pod-policies.kyverno.io/autogen-controllers: "none"
```

**Auto-gen is NOT generated when your rule has:**
- `names` in the match/exclude block
- `selector` in the match/exclude block
- `annotations` in the match/exclude block
- Multiple kinds in the same rule (e.g., `kinds: [Pod, ConfigMap]`)

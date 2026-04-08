# Module 02 — Validation Policies

Validation policies are the most common Kyverno use case. They intercept admission requests and **block** (Enforce) or **log** (Audit) resources that don't meet your requirements.

## Sub-modules

| Sub-module | Topic |
|---|---|
| [01 - Basic Validation](./01-basic/README.md) | Pattern matching, match/exclude, preconditions |
| [02 - Anchors](./02-anchors/README.md) | Conditional (`()`), equality `=()`, add-if-not-present `+()`, existence `^()`, negation `X()`, global `<()` |
| [03 - AnyPattern](./03-anypattern/README.md) | Multiple acceptable patterns |
| [04 - Deny](./04-deny/README.md) | CEL-style condition-based blocking |
| [05 - ForEach](./05-foreach/README.md) | Iterating over lists |
| [06 - Pod Security Standards](./06-pod-security-standards/README.md) | Built-in PSS baseline/restricted |
| [07 - CEL](./07-cel/README.md) | Common Expression Language validation |

## Key Settings

```yaml
spec:
  validationFailureAction: Enforce   # Block non-compliant | Audit = log only
  background: true                   # Scan existing resources
  admission: true                    # Enforce at admission time
```

## Rule Ordering

Kyverno processes anchors in this order:
1. Check strict conditional anchors `()` — if any fail, the entire mutation/validation stops
2. Check `+()` (add-if-not-present) and plain values

## failureAction vs validationFailureAction

In newer Kyverno versions, you can override `validationFailureAction` **per rule** using `validate.failureAction`:

```yaml
spec:
  validationFailureAction: Audit   # default for all rules
  rules:
    - name: my-strict-rule
      validate:
        failureAction: Enforce     # override just this rule
```

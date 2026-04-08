# Module 10 — Enforcement Strategies

## Concept

Rolling out a new policy directly in Enforce mode is risky — it could block existing workloads. Kyverno provides a gradual rollout path:

```
Silent Audit → Audit + Warnings → Enforce (allow existing) → Full Enforce
```

---

## The Four Phases

### Phase 1: Silent Audit (Week 1-2)
Collect baseline data. No impact on developers.

```yaml
spec:
  emitWarning: false
  rules:
    - name: require-ns-purpose-label
      match:
        any:
          - resources:
              kinds: [Namespace]
      validate:
        failureAction: Audit
        message: "You must have label 'purpose: production' on all namespaces."
        pattern:
          metadata:
            labels:
              purpose: production
```

---

### Phase 2: Audit + Warnings (Week 3-4)
Developers see warnings in their `kubectl` output. No blocking.

```yaml
spec:
  emitWarning: true    # ← warning shown in kubectl output
  rules:
    - name: require-ns-purpose-label
      match:
        any:
          - resources:
              kinds: [Namespace]
      validate:
        failureAction: Audit
        message: "You must have label 'purpose: production' on all namespaces."
        pattern:
          metadata:
            labels:
              purpose: production
```

**Expected warning output:**
```
namespace/test-namespace created
Warning: [require-ns-purpose-label] You must have label 'purpose: production' on all namespaces.
```

---

### Phase 3: Enforce — New Resources Blocked, Existing Tolerated (Week 5-6)

```yaml
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-ns-purpose-label
      match:
        any:
          - resources:
              kinds: [Namespace]
      validate:
        allowExistingViolations: true   # ← give existing resources time to be fixed
        message: "You must have label 'purpose: production' on all new namespaces."
        pattern:
          metadata:
            labels:
              purpose: production
```

**Behavior:**
- NEW namespace without label → **BLOCKED**
- EXISTING namespace without label → **ALLOWED** to be updated/modified
- `allowExistingViolations: true` gives developers time to remediate

---

### Phase 4: Full Enforcement (Week 7+)

```yaml
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-ns-purpose-label
      match:
        any:
          - resources:
              kinds: [Namespace]
      validate:
        allowExistingViolations: false  # ← all resources must comply
        message: "You must have label 'purpose: production' on all namespaces."
        pattern:
          metadata:
            labels:
              purpose: production
```

---

## Migration Summary Table

| Phase | failureAction | emitWarning | allowExistingViolations | Developer impact |
|---|---|---|---|---|
| 1 — Silent Audit | Audit | false | N/A | None — only logged |
| 2 — Visible Audit | Audit | true | N/A | Warning in kubectl output |
| 3 — Partial Enforce | Enforce | false | true | New resources blocked |
| 4 — Full Enforce | Enforce | false | false | All resources blocked |

---

## Switching Between Modes

```bash
# Patch a policy to Audit mode
kubectl patch clusterpolicy require-app-label \
  --type=merge \
  -p '{"spec":{"validationFailureAction":"Audit"}}'

# Test — now the violating pod is ALLOWED
kubectl run audit-test --image=nginx
kubectl get clusterpolicyreport -o yaml | grep -A10 "audit-test"

# Switch back to Enforce
kubectl patch clusterpolicy require-app-label \
  --type=merge \
  -p '{"spec":{"validationFailureAction":"Enforce"}}'

# Cleanup
kubectl delete pod audit-test --ignore-not-found=true
```

---

## emitWarning Deep Dive

| Setting | Admission result | kubectl output | PolicyReport |
|---|---|---|---|
| `failureAction: Audit, emitWarning: false` | PASS (allowed) | No warning | fail entry created |
| `failureAction: Audit, emitWarning: true` | PASS (allowed) | Warning shown | fail entry created |
| `failureAction: Enforce` | BLOCKED | Error message | No report (blocked) |

---

## allowExistingViolations Deep Dive

`allowExistingViolations` controls what happens when a resource that **already violates** the policy is **updated**:

| Setting | Update existing violating resource | Fix the violating field |
|---|---|---|
| `true` | ALLOWED (other fields can be updated) | Still required |
| `false` | BLOCKED until violation is fixed | Required before any update |

**Use case for `true`:** Developers need to update other fields of a violating resource (e.g., scale a deployment) without being forced to fix the policy violation immediately. They can work on compliance in parallel.

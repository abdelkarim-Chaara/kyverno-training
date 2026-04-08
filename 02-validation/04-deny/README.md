# 02-04 — Deny Rules

## Concept

The `deny` block allows condition-based blocking using JMESPath expressions, giving you more flexibility than pattern matching. It's useful when you need to evaluate computed values or complex comparisons.

## When to Use `deny` Instead of `pattern`

- Block resources based on computed/compared values (e.g., replicas > 2)
- Block based on operation type (CREATE, UPDATE, DELETE)
- Block based on dynamic values from the admission request context
- Block ConfigMap deletion (cannot use pattern for operation-based checks)

---

## Example 1: Block DELETE on a ConfigMap

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: protect-configmap
spec:
  validationFailureAction: Enforce
  rules:
    - name: block-cm-delete
      match:
        any:
          - resources:
              kinds: [ConfigMap]
              names: ["critical-config"]
      validate:
        message: "Deleting critical-config is not allowed."
        deny:
          conditions:
            any:
              - key: "{{ request.operation }}"
                operator: Equals
                value: "DELETE"
```

---

## Example 2: Limit Deployment Replicas

```yaml
validate:
  message: "Deployment replicas must not exceed 2."
  deny:
    conditions:
      any:
        - key: "{{ request.object.spec.replicas }}"
          operator: GreaterThan
          value: 2
```

---

## Example 3: Block ConfigMaps with Specific Labels

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-cm-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: cm-labels
      match:
        any:
          - resources:
              kinds: [ConfigMap]
      validate:
        deny:
          conditions:
            any:
              - key: "{{ request.object.metadata.labels.type || '' }}"
                operator: Equals
                value: testing
              - key: "{{ request.object.metadata.labels.type || '' }}"
                operator: Equals
                value: staging
```

```bash
# FAIL — type=testing
kubectl create configmap test-cm --from-literal=key=val
kubectl label configmap test-cm type=testing

# Apply inline:
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: bad-cm
  labels:
    type: testing
data:
  key: value
EOF

# PASS — no type label
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: good-cm
data:
  key: value
EOF

kubectl delete configmap bad-cm good-cm --ignore-not-found=true
kubectl delete clusterpolicy check-cm-labels
```

---

## Example 4: Empty Deny Block

Used as a "catch-all" at the end of a verification loop — unconditionally blocks anything that reaches it:

```yaml
validate:
  message: "Updating or deleting this resource is not allowed."
  deny: {}
```

---

## Available Operators for Deny Conditions

| Operator | Description |
|---|---|
| `Equals` | Exact match |
| `NotEquals` | Not equal |
| `GreaterThan` | Numeric greater than |
| `GreaterThanOrEquals` | Numeric greater than or equal |
| `LessThan` | Numeric less than |
| `LessThanOrEquals` | Numeric less than or equal |
| `In` | Value is in a list |
| `NotIn` | Value is not in a list |
| `AnyIn` | Any element is in the list |
| `AnyNotIn` | Any element is NOT in the list |
| `AllIn` | All elements are in the list |
| `AllNotIn` | All elements are NOT in the list |

---

## Cleanup

```bash
kubectl delete clusterpolicy protect-configmap check-cm-labels --ignore-not-found=true
```

# 02-07 — CEL (Common Expression Language) Validation

## Concept

CEL is a fast, sandboxed expression language developed by Google and adopted by Kubernetes. It's the standard validation language used in Kubernetes ValidatingAdmissionPolicy (VAP). Kyverno supports CEL as an alternative to pattern-based validation.

## Available Variables

| Variable | Description | Returns null for |
|---|---|---|
| `object` | The incoming resource from the admission request | DELETE operations |
| `oldObject` | The existing resource in the cluster | CREATE operations |
| `request` | Attributes of the admission request (userInfo, operation type) | — |
| `namespaceObject` | The full resource for the object's namespace | Cluster-scoped objects |
| `params` | A parameter resource (for configurable policies) | — |
| `authorizer` | Perform authorization checks | — |

---

## Basic Syntax

```yaml
validate:
  failureAction: Enforce
  cel:
    expressions:
      - expression: "<CEL expression that returns bool>"
        message: "Error message"
```

---

## Example 1: Disallow Host Namespaces

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-host-namespace
spec:
  validationFailureAction: Enforce
  rules:
    - name: disallow-host-namespace
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        cel:
          expressions:
            - expression: >-
                (!has(object.spec.hostNetwork) || object.spec.hostNetwork == false) &&
                (!has(object.spec.hostIPC) || object.spec.hostIPC == false) &&
                (!has(object.spec.hostPID) || object.spec.hostPID == false)
              message: >-
                Sharing host namespaces is disallowed. spec.hostNetwork,
                spec.hostIPC, and spec.hostPID must be unset or false.
```

**Key CEL patterns:**
- `has(object.spec.field)` — check if a field exists
- `!has(...)` — field does not exist
- `||` — OR
- `&&` — AND

---

## Example 2: CEL Preconditions (celPreconditions)

Apply the rule only when a specific condition is met (e.g., a label has a certain value):

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-host-path
spec:
  validationFailureAction: Enforce
  rules:
    - name: host-path
      match:
        any:
          - resources:
              kinds: [Pod]
      celPreconditions:
        - name: "Label should be red"
          expression: "object.metadata.labels['color'] == 'red'"
      validate:
        cel:
          expressions:
            - expression: >-
                !has(object.spec.volumes) ||
                object.spec.volumes.all(volume, !has(volume.hostPath))
              message: "HostPath volumes are forbidden."
```

---

## Example 3: Compare Old vs New on UPDATE

CEL can access `oldObject` to compare before/after values during UPDATE operations:

```yaml
validate:
  cel:
    expressions:
      - expression: >-
          request.operation != 'UPDATE' ||
          object.spec.replicas >= oldObject.spec.replicas
        message: "Scaling down is not allowed."
```

---

## Common CEL Patterns

```
# Check if field exists
has(object.spec.securityContext)

# String matching
object.metadata.name.startsWith("prod-")
object.metadata.name.endsWith("-svc")
object.metadata.name.contains("database")

# List operations
object.spec.containers.all(c, c.image.contains("/"))  # all containers use registry
object.spec.containers.exists(c, c.name == "sidecar")  # at least one named sidecar
object.spec.containers.size() > 0                       # non-empty list

# Numeric comparison
object.spec.replicas >= 2
object.spec.replicas <= 10

# Conditional (ternary)
has(object.spec.replicas) ? object.spec.replicas >= 2 : true
```

---

## When to Use CEL vs Pattern

| Use Case | Recommended |
|---|---|
| Simple field presence/value checks | Pattern (`?*`, `!default`) |
| Comparing values (old vs new) | CEL |
| Complex boolean logic | CEL |
| Kubernetes-native style (VAP compatible) | CEL |
| Quick readable rules | Pattern |

---

## Cleanup

```bash
kubectl delete clusterpolicy disallow-host-namespace disallow-host-path --ignore-not-found=true
```

# 02-02 — Anchors

Anchors allow conditional logic within the `pattern` block without needing `preconditions`. They connect elements at the same or parent-child hierarchy levels.

---

## Anchor Types

| Anchor | Syntax | Meaning |
|---|---|---|
| Conditional | `(field)` | IF this peer element condition is true, THEN check the rest |
| Equality | `=(field)` | IF this field exists, THEN check its children |
| Add-if-not-present | `+(field)` | Set this value only if the field is not already present |
| Existence | `^(field)` | At least one element in the array must match |
| Negation | `X(field)` | Deny if this field EXISTS (regardless of value) |
| Global | `<(field)` | IF this condition is true, THEN check a DIFFERENT part of the resource |

---

## 1. Conditional Anchor `()` — IF...THEN for Peers

**Use case:** If a container uses `:latest` image tag, then `imagePullPolicy` must NOT be `IfNotPresent`.

```yaml
validate:
  pattern:
    spec:
      containers:
        - (image): "*:latest"           # IF image matches *:latest
          imagePullPolicy: "!IfNotPresent"  # THEN this peer field must not be IfNotPresent
```

The `()` connects **elements at the same hierarchy level** (peers in the same list item).

---

## 2. Equality Anchor `=()` — IF...THEN for Children

**Use case:** IF a `hostPath` volume exists, THEN its `path` must not be `/var/lib`.

```yaml
validate:
  pattern:
    spec:
      volumes:
        - =(hostPath):         # IF hostPath object exists
            path: "!/var/lib"  # THEN its child field must NOT be /var/lib
```

The `=()` connects a **parent to its children**.

---

## 3. Add-if-not-present Anchor `+()` — Set Default Value

**Use case:** Add label `lfx-mentorship: kyverno` to ConfigMaps, but only if that label doesn't already exist.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: lfx-add-labels
spec:
  rules:
    - name: lfx-mentorship
      match:
        any:
          - resources:
              kinds: [ConfigMap]
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              +(lfx-mentorship): kyverno   # Only set if label doesn't exist
```

Commonly used in mutation policies to set defaults without overriding existing values.

---

## 4. Existence Anchor `^()` — At Least One

**Use case:** At least one container must use an approved image.

```yaml
validate:
  pattern:
    spec:
      ^(containers):           # IF containers array exists
        - image: nginx:latest  # at least ONE element must match this
```

**Difference vs wildcard `*`:**

```yaml
# Wildcard (*) — EVERY container must use nginx:latest
pattern:
  spec:
    containers:
      - name: "*"
        image: nginx:latest

# Existence (^) — AT LEAST ONE container must use nginx:latest
pattern:
  spec:
    ^(containers):
      - image: nginx:latest
```

---

## 5. Negation Anchor `X()` — Forbid if Field Exists

**Use case:** Forbid developers from using a `hostPath` volume.

```yaml
validate:
  pattern:
    spec:
      volumes:
        - name: "*"
          X(hostPath): "null"   # IF hostPath exists in ANY volume → FAIL
                                # Kyverno doesn't check the value, only presence
```

---

## 6. Global Anchor `<()` — Cross-section Condition

**Use case:** If a container uses an image from `nexus.com`, THEN the pod must define `imagePullSecrets`.

The condition (`nexus.com/*`) is in a different part of the spec from the requirement (`imagePullSecrets`). Standard conditional anchors only work on peers.

```yaml
validate:
  pattern:
    spec:
      containers:
        - name: "*"
          <(image): "nexus.com/*"   # IF any container uses nexus.com image
      imagePullSecrets:              # THEN this other section must also exist
        - name: mynexus-secrets
```

The `<()` anchor is checked first — if it's false, the **entire rule is skipped** (not failed).

---

## Lab 1: Restrict NodePort Services

Block Services that use `type: NodePort`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-nodeport
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-nodeport
      match:
        any:
          - resources:
              kinds: [Service]
      validate:
        message: "Services of type NodePort are not allowed."
        pattern:
          spec:
            =(type): "!NodePort"   # IF type exists, THEN it must not be NodePort
```

```bash
# Test — should fail
kubectl create service nodeport my-svc --tcp=5678:8080
# Error: Services of type NodePort are not allowed.

# Test — should pass
kubectl create service clusterip my-svc --tcp=8080:8080

kubectl delete service my-svc --ignore-not-found=true
```

---

## Lab 2: Validate HostPath Volumes

Ensure `hostPath` volumes don't use `/var/run/docker.sock`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: validate-pods
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-pods
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        pattern:
          =(spec):
            =(volumes):
              - =(hostPath):
                  path: "!/var/run/docker.sock"
```

---

## Lab 3: Conditional Label Check on HostPath

If a Pod uses a `hostPath` volume with path `/var/run/docker.sock`, it must have label `allow-docker: "true"`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: validate-pods
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-pods
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        pattern:
          metadata:
            labels:
              allow-docker: "true"
          (spec):                  # () = conditional: if spec matches...
            (volumes):
              - (hostPath):
                  path: "/var/run/docker.sock"
```

---

## Anchor Processing Order

Kyverno processes anchors in this priority order:
1. **Global `<()`** — if false, skip the entire rule
2. **Conditional `()`** — if false, the pattern for that element passes trivially
3. **Equality `=()`** — if field absent, the children are not checked
4. **Add-if-not-present `+()`** and **plain values** — applied last

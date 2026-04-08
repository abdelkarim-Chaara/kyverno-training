# 02-03 — AnyPattern

## Concept

`anyPattern` allows you to define **multiple acceptable patterns**. A resource passes if it matches **at least one** pattern.

> A rule can use either `pattern` OR `anyPattern` — never both.

---

## Use Case: Require Non-Root Containers

`runAsNonRoot` can be set at the **Pod level** or the **container level**. We need to accept both configurations.

```yaml
validate:
  anyPattern:
    # Pattern 1: Set at Pod securityContext level
    - spec:
        securityContext:
          runAsNonRoot: true
        containers:
          - =(securityContext):        # IF container has a securityContext
              =(runAsNonRoot): true    # THEN it must also be true
        =(initContainers):
          - =(securityContext):
              =(runAsNonRoot): true

    # Pattern 2: Set at container level (every container must define it)
    - spec:
        containers:
          - securityContext:
              runAsNonRoot: true
        =(initContainers):
          - securityContext:
              runAsNonRoot: true
```

**Full policy:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-run-as-non-root
  annotations:
    policies.kyverno.io/title: Require Non-Root Containers
    policies.kyverno.io/category: Pod Security
    policies.kyverno.io/severity: medium
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: run-as-non-root
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: >-
          Containers must run as non-root. Set runAsNonRoot: true at the
          Pod level or in every container's securityContext.
        anyPattern:
          - spec:
              securityContext:
                runAsNonRoot: true
              containers:
                - =(securityContext):
                    =(runAsNonRoot): true
          - spec:
              containers:
                - securityContext:
                    runAsNonRoot: true
```

---

## Test Cases

```bash
kubectl apply -f require-run-as-non-root.yaml

# PASS — runAsNonRoot at pod level
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-pod-level
  labels:
    app: test
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: nginx
      image: nginx:1.25
EOF

# PASS — runAsNonRoot at container level
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-container-level
  labels:
    app: test
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      securityContext:
        runAsNonRoot: true
EOF

# FAIL — neither level set
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-no-security
  labels:
    app: test
spec:
  containers:
    - name: nginx
      image: nginx:1.25
EOF

# Cleanup
kubectl delete pod pod-pod-level pod-container-level --ignore-not-found=true
kubectl delete clusterpolicy require-run-as-non-root
```

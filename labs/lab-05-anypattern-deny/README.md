# Lab 05 — AnyPattern and Deny Rules

## Objectives

- Use `anyPattern` to allow multiple valid configurations for a rule
- Use `deny` blocks with conditions and operators to block requests
- Combine multiple conditions using `all` inside a `deny` block

---

## Exercise 1: AnyPattern — Non-Root Containers

### Goal

Write a `ClusterPolicy` named `require-nonroot` that ensures pods run as non-root. The policy should accept a pod if **either** of these is true:

- `spec.securityContext.runAsNonRoot: true` (pod-level setting), **OR**
- Every container has `securityContext.runAsNonRoot: true` (container-level setting)

Use the `anyPattern` field instead of `pattern`.

### Expected Behavior

| Pod Configuration | Result |
|---|---|
| Pod-level `runAsNonRoot: true` | Pass |
| Container-level `runAsNonRoot: true` | Pass |
| Neither set | Fail |

<details>
<summary>Solution</summary>

**ClusterPolicy — `require-nonroot.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-nonroot
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-nonroot
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Pods must run as non-root. Set runAsNonRoot: true at pod or container level."
        anyPattern:
          - spec:
              securityContext:
                runAsNonRoot: true
          - spec:
              containers:
                - securityContext:
                    runAsNonRoot: true
```

**Apply the policy:**

```bash
kubectl apply -f require-nonroot.yaml
```

**Test 1 — Pod-level setting (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-level
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: app
      image: nginx:1.25
EOF
```

**Test 2 — Container-level setting (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-container-level
spec:
  containers:
    - name: app
      image: nginx:1.25
      securityContext:
        runAsNonRoot: true
EOF
```

**Test 3 — Neither set (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-no-nonroot
spec:
  containers:
    - name: app
      image: nginx:1.25
EOF
# Expected error:
# Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
# resource Pod/default/test-no-nonroot was blocked due to the following policies
# require-nonroot:
#   check-nonroot: Pods must run as non-root. Set runAsNonRoot: true at pod or container level.
```

**Cleanup:**

```bash
kubectl delete pod test-pod-level test-container-level --ignore-not-found
kubectl delete clusterpolicy require-nonroot
```

</details>

---

## Exercise 2: Deny — Block Deployments with More Than 5 Replicas

### Goal

Write a `ClusterPolicy` named `limit-replicas` that uses a `deny` block to reject any Deployment that requests more than 5 replicas. Use the `GreaterThan` operator.

### Expected Behavior

| Deployment replicas | Result |
|---|---|
| `replicas: 3` | Pass |
| `replicas: 6` | Fail |

<details>
<summary>Solution</summary>

**ClusterPolicy — `limit-replicas.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: limit-replicas
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-replica-count
      match:
        any:
          - resources:
              kinds:
                - Deployment
      validate:
        message: "Deployments may not have more than 5 replicas."
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.replicas }}"
                operator: GreaterThan
                value: 5
```

**Apply the policy:**

```bash
kubectl apply -f limit-replicas.yaml
```

**Test 1 — 6 replicas (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: too-many-replicas
spec:
  replicas: 6
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: nginx:1.25
EOF
# Expected error:
# Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
# resource Deployment/default/too-many-replicas was blocked due to the following policies
# limit-replicas:
#   check-replica-count: Deployments may not have more than 5 replicas.
```

**Test 2 — 3 replicas (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: normal-replicas
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: nginx:1.25
EOF
```

**Cleanup:**

```bash
kubectl delete deployment normal-replicas --ignore-not-found
kubectl delete clusterpolicy limit-replicas
```

</details>

---

## Exercise 3: Deny — Block ConfigMaps with Label `env: dev` in the `production` Namespace

### Goal

Write a `ClusterPolicy` named `no-dev-cms-in-prod` that denies creation of ConfigMaps that meet **all** of the following conditions:

- The ConfigMap has label `env: dev`
- The request is targeting the `production` namespace

Use a `deny` block with an `all` conditions list.

### Expected Behavior

| ConfigMap | Namespace | Result |
|---|---|---|
| label `env: dev` | `production` | Fail |
| label `env: dev` | `staging` | Pass |
| label `env: prod` | `production` | Pass |

<details>
<summary>Solution</summary>

**ClusterPolicy — `no-dev-cms-in-prod.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: no-dev-cms-in-prod
spec:
  validationFailureAction: Enforce
  rules:
    - name: block-dev-configmaps
      match:
        any:
          - resources:
              kinds:
                - ConfigMap
      validate:
        message: "ConfigMaps labeled 'env: dev' are not allowed in the production namespace."
        deny:
          conditions:
            all:
              - key: "{{ request.object.metadata.labels.env }}"
                operator: Equals
                value: "dev"
              - key: "{{ request.namespace }}"
                operator: Equals
                value: "production"
```

**Setup — create namespaces:**

```bash
kubectl create namespace production --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
```

**Apply the policy:**

```bash
kubectl apply -f no-dev-cms-in-prod.yaml
```

**Test 1 — env=dev in production (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: dev-config
  namespace: production
  labels:
    env: dev
data:
  key: value
EOF
# Expected error:
# Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
# resource ConfigMap/production/dev-config was blocked due to the following policies
# no-dev-cms-in-prod:
#   block-dev-configmaps: ConfigMaps labeled 'env: dev' are not allowed in the production namespace.
```

**Test 2 — env=dev in staging (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: dev-config
  namespace: staging
  labels:
    env: dev
data:
  key: value
EOF
```

**Test 3 — env=prod in production (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: prod-config
  namespace: production
  labels:
    env: prod
data:
  key: value
EOF
```

**Cleanup:**

```bash
kubectl delete configmap dev-config -n staging --ignore-not-found
kubectl delete configmap prod-config -n production --ignore-not-found
kubectl delete clusterpolicy no-dev-cms-in-prod
```

</details>

---

## Summary

| Concept | Key Point |
|---|---|
| `anyPattern` | Rule passes if **any** listed pattern matches |
| `pattern` | Rule passes only if **all** conditions in the single pattern match |
| `deny` with `any` | Block if **at least one** condition is true |
| `deny` with `all` | Block only if **every** condition is true |
| `GreaterThan` operator | Works on numeric values extracted via JMESPath expressions |

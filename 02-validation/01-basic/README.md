# 02-01 — Basic Validation

## Concept

A `validate` rule defines a **pattern** that the incoming resource must match. If the pattern fails, the request is blocked (Enforce) or logged (Audit).

---

## Example 1: Require a Label on All Pods

**Policy:** `require-app-label.yaml`

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-app-label
  annotations:
    policies.kyverno.io/title: Require App Label
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/description: >-
      All pods must have an 'app' label for proper identification.
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: check-for-app-label
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "The 'app' label is required for all pods."
        pattern:
          metadata:
            labels:
              app: "?*"    # ?* means: at least one character
```

```bash
kubectl apply -f require-app-label.yaml

# FAIL — no label
kubectl run nginx --image=nginx
# Error: admission webhook denied: The 'app' label is required for all pods.

# PASS — has label
kubectl run nginx --image=nginx --labels="app=nginx"
kubectl delete pod nginx
```

---

## Example 2: Match with ANY Logic

Apply the policy only to Pods with `app: critical` **OR** `type: database` labels:

```yaml
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-env-label
      match:
        any:
          - resources:
              kinds: [Pod]
              selector:
                matchLabels:
                  app: critical
          - resources:
              kinds: [Pod]
              selector:
                matchLabels:
                  type: database
      validate:
        message: "The label 'env' is required."
        pattern:
          metadata:
            labels:
              env: "?*"
```

```bash
# FAIL — has app=critical but no env label
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-critical
  labels:
    app: critical
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF

# PASS — no special labels, policy doesn't match
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-normal
spec:
  containers:
  - image: nginx
    name: nginx
EOF

# PASS — type=database AND has env label
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-database
  labels:
    type: database
    env: production
spec:
  containers:
  - image: nginx
    name: nginx
EOF

kubectl delete pod pod-normal pod-database --ignore-not-found=true
```

---

## Example 3: Match with ALL Logic

Apply only when a Pod has **BOTH** `app: critical` AND `type: database`:

```yaml
match:
  all:
    - resources:
        kinds: [Pod]
        selector:
          matchLabels:
            app: critical
    - resources:
        kinds: [Pod]
        selector:
          matchLabels:
            type: database
```

---

## Example 4: Match by Name Pattern

Match Pods with `app: critical` **OR** Deployments named `database-*` with `type: database`:

```yaml
match:
  any:
    - resources:
        kinds: [Pod]
        selector:
          matchLabels:
            app: critical
    - resources:
        kinds: [Deployment]
        selector:
          matchLabels:
            type: database
        names:
          - "database-*"    # wildcard name matching
```

---

## Example 5: Exclude by Namespace

Require `env` label on all Pods **except** those in `testing` namespace:

```yaml
rules:
  - name: check-env-label
    match:
      any:
        - resources:
            kinds: [Pod]
    exclude:
      any:
        - resources:
            namespaces:
              - testing
    validate:
      message: "label 'env' is required"
      pattern:
        metadata:
          labels:
            env: "?*"
```

```bash
kubectl create namespace testing

# FAIL — default namespace, no env label
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-prod
  namespace: default
spec:
  containers:
  - image: nginx
    name: nginx
EOF

# PASS — testing namespace is excluded
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-test
  namespace: testing
spec:
  containers:
  - image: nginx
    name: nginx
EOF

kubectl delete pod pod-test -n testing
kubectl delete namespace testing
```

---

## Example 6: Exclude by Label Selector

Require `type` label on all Deployments **except** those with `app: critical`:

```yaml
rules:
  - name: check-type-label
    match:
      any:
        - resources:
            kinds: [Deployment]
    exclude:
      any:
        - resources:
            selector:
              matchLabels:
                app: critical
    validate:
      message: "label 'type' is required"
      pattern:
        metadata:
          labels:
            type: "?*"
```

---

## Example 7: Preconditions

Only require `env` label if the Pod has `color=blue` OR `app=busybox`:

```yaml
rules:
  - name: check-env-label
    match:
      any:
        - resources:
            kinds: [Pod]
    preconditions:
      any:
        - key: "{{ request.object.metadata.labels.color || '' }}"
          operator: Equals
          value: blue
        - key: "{{ request.object.metadata.labels.app || '' }}"
          operator: Equals
          value: busybox
    validate:
      message: "label 'env' is required"
      pattern:
        metadata:
          labels:
            env: "?*"
```

```bash
# FAIL — has color=blue but no env
kubectl run test-blue --image=nginx --labels=color=blue

# PASS — no special labels, precondition not met
kubectl run test-normal --image=nginx

# PASS — color=blue AND env set
kubectl run test-blue-env --image=nginx --labels=color=blue,env=testing

kubectl delete pod test-normal test-blue-env --ignore-not-found=true
```

---

## Example 8: Common Validation Patterns

```yaml
# Disallow pods in default namespace
validate:
  pattern:
    metadata:
      namespace: "!default"

# Require at least 2 replicas
validate:
  pattern:
    spec:
      replicas: ">=2"

# Restrict service port range
validate:
  pattern:
    spec:
      ports:
        - port: 32000-33000
```

---

## Pattern Operators

| Operator | Example | Meaning |
|---|---|---|
| `?*` | `"?*"` | At least one character (not empty) |
| `!` | `"!default"` | NOT equal |
| `>=` | `">=2"` | Greater than or equal |
| `>` | `">0"` | Greater than |
| `range` | `"32000-33000"` | Numeric range |
| `\|` | `"production\|staging"` | OR — one of these values |

---

## Files in This Directory

- `require-app-label.yaml` — Basic label requirement policy

## Cleanup

```bash
kubectl delete clusterpolicy require-app-label --ignore-not-found=true
kubectl delete clusterpolicy check-env-label --ignore-not-found=true
kubectl delete clusterpolicy check-type-label --ignore-not-found=true
```

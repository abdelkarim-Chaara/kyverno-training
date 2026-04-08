# 09-02 — `kyverno test` Command

The `test` command runs **declarative unit tests** for your policies. You define the expected outcome (pass/fail/skip) for each policy-rule-resource combination, and Kyverno verifies your assertions.

## Command

```bash
kyverno test <path-to-directory>
```

Kyverno looks for a `kyverno-test.yaml` file in the directory and runs all tests defined there.

## Test File Anatomy

```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: my-test-suite
policies:
  - policy.yaml            # policies to load
resources:
  - good-resource.yaml     # resources to test against
  - bad-resource.yaml
exceptions:                # optional
  - exception.yaml
variables: values.yaml     # optional — supplies context data
results:
  - policy: my-policy      # policy name
    rule: my-rule           # rule name
    resources:
      - good-resource       # resource name
    kind: Pod
    result: pass            # expected: pass | fail | skip | warn | error
  - policy: my-policy
    rule: my-rule
    resources:
      - bad-resource
    kind: Pod
    result: fail
```

---

## Example 1: Test a Validation Policy

**Directory structure:**
```
disallow-latest-tag/
├── policy.yaml
├── resources.yaml
└── kyverno-test.yaml
```

**resources.yaml:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.21.0
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
spec:
  containers:
    - name: busybox
      image: busybox:latest
---
apiVersion: v1
kind: Pod
metadata:
  name: redis-pod
spec:
  containers:
    - name: redis
      image: redis:latest
```

**kyverno-test.yaml:**
```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: disallow-latest-tag
policies:
  - policy.yaml
resources:
  - resources.yaml
results:
  - policy: disallow-latest-tag
    rule: no-latest-tag
    resources: [nginx-pod]
    kind: Pod
    result: pass
  - policy: disallow-latest-tag
    rule: no-latest-tag
    resources: [busybox-pod]
    kind: Pod
    result: fail
  - policy: disallow-latest-tag
    rule: no-latest-tag
    resources: [redis-pod]
    kind: Pod
    result: fail
```

```bash
kyverno test disallow-latest-tag/

# Output:
# Loading test (kyverno-test.yaml) ...
# Applying 1 policy to 3 resources ...
# Checking results ...
#
# │ ID │ POLICY              │ RULE          │ RESOURCE        │ RESULT │ REASON │
# │ 1  │ disallow-latest-tag │ no-latest-tag │ Pod/nginx-pod   │ Pass   │ Ok     │
# │ 2  │ disallow-latest-tag │ no-latest-tag │ Pod/busybox-pod │ Pass   │ Ok     │
# │ 3  │ disallow-latest-tag │ no-latest-tag │ Pod/redis-pod   │ Pass   │ Ok     │
#
# Test Summary: 3 tests passed and 0 tests failed
```

---

## Example 2: Test with PolicyException

When using exceptions, the expected result changes to `skip`:

```yaml
results:
  - policy: disallow-latest-tag
    rule: no-latest-tag
    resources: [busybox-pod]
    kind: Pod
    result: skip   # ← was fail, now skip because exception applies
```

Add the exception to the test file:

```yaml
exceptions:
  - exception.yaml
```

---

## Example 3: Test a Mutation Policy

For mutations, you provide a `patchedResource` — the **expected state after mutation**:

**patched-resource.yaml** (what the Deployment should look like after mutation):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    cost-center: research   # ← added by mutation
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.21.0
```

**kyverno-test.yaml for mutation:**
```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: add-label
policies:
  - policy.yaml
resources:
  - deployment.yaml
results:
  - policy: add-label
    rule: add-cost-center-label
    resources: [my-app]
    kind: Deployment
    patchedResource: patched-resource.yaml   # ← compare against this
    result: pass
```

```bash
kyverno test add-team-label/
```

---

## Example 4: Test with Values File (Context Data)

When a policy uses a `configMap` or `apiCall` context, supply mock data:

**values.yaml:**
```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Value
metadata:
  name: test-values
policies:
  - name: add-annotation-from-configmap
    rules:
      - name: add-annotation-from-configmap
        values:
          dictionary.data.region: "us-east-1"
```

**kyverno-test.yaml:**
```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: add-annotation
policies:
  - policy.yaml
resources:
  - deployment.yaml
variables: values.yaml   # ← supplies context data
results:
  - policy: add-annotation-from-configmap
    rule: add-annotation-from-configmap
    resources: [my-app]
    kind: Deployment
    patchedResource: patched-deployment.yaml
    result: pass
```

---

## CI/CD Integration

```bash
# In a GitHub Actions workflow or CI pipeline:
kyverno test ./policies/

# Non-zero exit code on failure — integrate with CI gates
if ! kyverno test ./policies/; then
  echo "Policy tests failed!"
  exit 1
fi
```

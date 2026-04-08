# 09-01 — `kyverno apply` Command

The `apply` command tests one or more policies against resource manifests and shows pass/fail results.

## Basic Usage

```bash
kyverno apply <path-to-policy> --resource <path-to-resource>
```

---

## Example 1: Basic Validation

**Policy:** `disallow-latest-tag.yaml`

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Audit
  rules:
    - name: no-latest-tag
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Using ':latest' tag is not allowed."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

**Resources:** `pods.yaml`

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
```

```bash
kyverno apply disallow-latest-tag.yaml --resource pods.yaml

# Output:
# Applying 3 policy rule(s) to 2 resource(s)...
# pass: 1, fail: 1, warn: 0, error: 0, skip: 0
```

---

## Example 2: PolicyReport Output

```bash
kyverno apply disallow-latest-tag.yaml --resource pods.yaml --policy-report

# Output shows full PolicyReport YAML with details:
# - nginx-pod: pass
# - busybox-pod: fail (rule no-latest-tag failed at /spec/containers/0/image/)
```

---

## Example 3: Apply with a PolicyException

```yaml
# exception.yaml
apiVersion: kyverno.io/v2
kind: PolicyException
metadata:
  name: exclude-busybox-pod
spec:
  exceptions:
    - policyName: disallow-latest-tag
      ruleNames: [no-latest-tag]
  match:
    any:
      - resources:
          kinds: [Pod]
          names: ["busybox-pod"]
```

```bash
kyverno apply disallow-latest-tag.yaml \
  --resource pods.yaml \
  --exception exception.yaml

# Output:
# Applying 3 policy rule(s) to 2 resource(s) with 1 exception(s)...
# pass: 1, fail: 0, warn: 0, error: 0, skip: 1
# busybox-pod is now "skip" (exception applied)
```

---

## Example 4: Preview Mutation

```bash
# add-label.yaml mutates Deployments by adding cost-center: research

kyverno apply add-label.yaml --resource deployment.yaml

# Output shows the PATCHED resource:
# mutate policy add-label applied to default/Deployment/my-app:
# apiVersion: apps/v1
# kind: Deployment
# metadata:
#   labels:
#     cost-center: research     ← added by mutation
#   name: my-app
# ...
```

---

## Example 5: Supply ConfigMap Data with --values-file

When a policy uses a `configMap` context, you can simulate it with a values file:

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

```bash
kyverno apply add-annotations.yaml \
  --resource deployment.yaml \
  --values-file values.yaml

# OR use --set for a single value:
kyverno apply add-annotations.yaml \
  --resource deployment.yaml \
  --set dictionary.data.region=us-east-1
```

---

## Files in This Directory

- `disallow-latest-tag/` — validation policy test files
  - `policy.yaml` — the ClusterPolicy
  - `pods.yaml` — test resources (pass and fail cases)
  - `exception.yaml` — PolicyException to skip busybox-pod
- `add-team-label/` — mutation policy test files
  - `policy.yaml` — mutation policy
  - `deployment.yaml` — test Deployment
  - `patched-resource.yaml` — expected result after mutation

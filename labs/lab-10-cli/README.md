# Lab 10 — Kyverno CLI

## Objectives

- Use `kyverno apply` to validate resources against policies locally without a running cluster
- Apply PolicyExceptions via the CLI to see `skip` results
- Preview mutation policies with `kyverno apply`
- Write and run a full test suite using `kyverno test`

---

## Prerequisites

Install the Kyverno CLI:

```bash
# macOS (Homebrew)
brew install kyverno

# Linux (direct download)
curl -LO https://github.com/kyverno/kyverno/releases/latest/download/kyverno-cli_linux_x86_64.tar.gz
tar -xvf kyverno-cli_linux_x86_64.tar.gz
chmod +x kyverno
sudo mv kyverno /usr/local/bin/

# Verify
kyverno version
```

---

## Exercise 1: `kyverno apply` — Validate a Policy Locally

### Goal

Use the Kyverno CLI to run a policy against a set of pod resources offline. Interpret which pods pass and which fail. Then re-run with the `--policy-report` flag to see structured output.

The policy and resources are located at:
- Policy: `../09-cli/01-apply/disallow-latest-tag/policy.yaml`
- Resources: `../09-cli/01-apply/disallow-latest-tag/pods.yaml`

<details>
<summary>Solution</summary>

**Inspect the policy:**

```bash
cat ../09-cli/01-apply/disallow-latest-tag/policy.yaml
```

Expected content:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-image-tag
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Using a mutable image tag such as 'latest' is not allowed."
        pattern:
          spec:
            containers:
              - image: "!*:latest"
```

**Inspect the resources:**

```bash
cat ../09-cli/01-apply/disallow-latest-tag/pods.yaml
```

Expected content (two pods — one passing, one failing):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-versioned
spec:
  containers:
    - name: nginx
      image: nginx:1.25
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

**Run `kyverno apply`:**

```bash
kyverno apply ../09-cli/01-apply/disallow-latest-tag/policy.yaml \
  --resource ../09-cli/01-apply/disallow-latest-tag/pods.yaml
```

Expected output:

```
Applying 1 policy rule(s) to 2 resource(s)...

policy disallow-latest-tag -> resource Pod/default/nginx-versioned: PASS
policy disallow-latest-tag -> resource Pod/default/busybox-pod: FAIL (error: [check-image-tag] Validation error: Using a mutable image tag such as 'latest' is not allowed.; Validation rule 'require-image-tag' failed at '/spec/containers/0/image/' for resource Pod//busybox-pod.)

pass: 1, fail: 1, warn: 0, error: 0, skip: 0
```

**Interpretation:**
- `nginx-versioned` uses `nginx:1.25` — a pinned, non-latest tag — **PASS**
- `busybox-pod` uses `busybox:latest` — a mutable tag — **FAIL**

**Run with `--policy-report` flag:**

```bash
kyverno apply ../09-cli/01-apply/disallow-latest-tag/policy.yaml \
  --resource ../09-cli/01-apply/disallow-latest-tag/pods.yaml \
  --policy-report
```

Expected output format (a `PolicyReport` YAML object):

```yaml
apiVersion: wgpolicyk8s.io/v1alpha2
kind: PolicyReport
metadata:
  name: merged
results:
  - message: 'validation error: Using a mutable image tag such as ''latest'' is not allowed.
      rule require-image-tag failed at path /spec/containers/0/image/'
    policy: disallow-latest-tag
    resources:
      - apiVersion: v1
        kind: Pod
        name: busybox-pod
        namespace: default
    result: fail
    rule: require-image-tag
    scored: true
    source: kyverno
  - message: 'validation rule ''require-image-tag'' passed.'
    policy: disallow-latest-tag
    resources:
      - apiVersion: v1
        kind: Pod
        name: nginx-versioned
        namespace: default
    result: pass
    rule: require-image-tag
    scored: true
    source: kyverno
summary:
  error: 0
  fail: 1
  pass: 1
  skip: 0
  warn: 0
```

</details>

---

## Exercise 2: `kyverno apply` with Exception

### Goal

Re-run the same policy and resources from Exercise 1, but also pass the exception file. The `busybox-pod` should now receive a `skip` result instead of `fail`.

The exception file is at:
- Exception: `../09-cli/01-apply/disallow-latest-tag/exception.yaml`

<details>
<summary>Solution</summary>

**Inspect the exception file:**

```bash
cat ../09-cli/01-apply/disallow-latest-tag/exception.yaml
```

Expected content:

```yaml
apiVersion: kyverno.io/v2
kind: PolicyException
metadata:
  name: busybox-exception
  namespace: default
spec:
  exceptions:
    - policyName: disallow-latest-tag
      ruleNames:
        - require-image-tag
  match:
    any:
      - resources:
          kinds:
            - Pod
          names:
            - "busybox-pod"
```

**Run `kyverno apply` with the exception:**

```bash
kyverno apply ../09-cli/01-apply/disallow-latest-tag/policy.yaml \
  --resource ../09-cli/01-apply/disallow-latest-tag/pods.yaml \
  --exception ../09-cli/01-apply/disallow-latest-tag/exception.yaml
```

Expected output:

```
Applying 1 policy rule(s) to 2 resource(s)...

policy disallow-latest-tag -> resource Pod/default/nginx-versioned: PASS
policy disallow-latest-tag -> resource Pod/default/busybox-pod: SKIP

pass: 1, fail: 0, warn: 0, error: 0, skip: 1
```

**With `--policy-report`:**

```bash
kyverno apply ../09-cli/01-apply/disallow-latest-tag/policy.yaml \
  --resource ../09-cli/01-apply/disallow-latest-tag/pods.yaml \
  --exception ../09-cli/01-apply/disallow-latest-tag/exception.yaml \
  --policy-report
```

The `busybox-pod` result changes from `fail` to `skip` in the report:

```yaml
results:
  - message: ""
    policy: disallow-latest-tag
    resources:
      - kind: Pod
        name: busybox-pod
        namespace: default
    result: skip
    rule: require-image-tag
    scored: true
    source: kyverno
  - message: "validation rule 'require-image-tag' passed."
    policy: disallow-latest-tag
    resources:
      - kind: Pod
        name: nginx-versioned
        namespace: default
    result: pass
    rule: require-image-tag
    scored: true
    source: kyverno
summary:
  fail: 0
  pass: 1
  skip: 1
```

</details>

---

## Exercise 3: `kyverno apply` — Preview a Mutation

### Goal

Write a simple mutation policy that adds the label `env: dev` to all Deployments using `patchesJson6902`. Save it locally and run `kyverno apply` to preview the mutation output without applying anything to a cluster.

<details>
<summary>Solution</summary>

**Create the mutation policy — `add-env-label.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-env-label
spec:
  rules:
    - name: add-env-dev-label
      match:
        any:
          - resources:
              kinds:
                - Deployment
      mutate:
        patchesJson6902: |-
          - path: "/metadata/labels/env"
            op: add
            value: dev
```

**Create the test Deployment — `test-deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
  labels:
    app: my-app
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
        - name: app
          image: nginx:1.25
```

**Run `kyverno apply` to preview the mutation:**

```bash
kyverno apply add-env-label.yaml --resource test-deployment.yaml
```

Expected output:

```
Applying 1 policy rule(s) to 1 resource(s)...

mutate policy add-env-label applied to default/Deployment/my-deployment:
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-app
    env: dev          # <-- label was added
  name: my-deployment
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
        - image: nginx:1.25
          name: app

pass: 1, fail: 0, warn: 0, error: 0, skip: 0
```

The output shows the full mutated resource with the added label, without making any changes to a live cluster.

**Save the mutated output to a file:**

```bash
kyverno apply add-env-label.yaml \
  --resource test-deployment.yaml \
  -o mutated-deployment.yaml

cat mutated-deployment.yaml
# Contains the Deployment with env: dev label
```

</details>

---

## Exercise 4: `kyverno test` — Write a Test Suite

### Goal

Create a self-contained test directory with a policy, resources, and a `kyverno-test.yaml` test file. The policy should block pods created in the `default` namespace. Run `kyverno test` to execute the suite and verify the expected pass/fail results.

### Directory Structure to Create

```
disallow-default-ns/
  policy.yaml
  resources.yaml
  kyverno-test.yaml
```

<details>
<summary>Solution</summary>

**Create the directory:**

```bash
mkdir -p disallow-default-ns
```

**`disallow-default-ns/policy.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-default-namespace
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-namespace
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Pods must not be created in the default namespace."
        pattern:
          metadata:
            namespace: "!default"
```

**`disallow-default-ns/resources.yaml`**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-default
  namespace: default
spec:
  containers:
    - name: app
      image: nginx:1.25
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-staging
  namespace: staging
spec:
  containers:
    - name: app
      image: nginx:1.25
```

**`disallow-default-ns/kyverno-test.yaml`**

```yaml
apiVersion: cli.kyverno.io/v1alpha1
kind: Test
metadata:
  name: disallow-default-namespace-test
policies:
  - policy.yaml
resources:
  - resources.yaml
results:
  - policy: disallow-default-namespace
    rule: check-namespace
    resource: pod-default
    namespace: default
    result: fail
  - policy: disallow-default-namespace
    rule: check-namespace
    resource: pod-staging
    namespace: staging
    result: pass
```

**Run the test suite:**

```bash
kyverno test disallow-default-ns/
```

Expected output:

```
Loading test  ( disallow-default-ns/kyverno-test.yaml ) ...
  Loading values/variables ...
  Loading policies ...
  Loading resources ...
  Applying 1 policy to 2 resources ...
  Checking results ...

  policy/rule                                          resource                              result
  -------------------------------------------------    ------------------------------------  ------
  disallow-default-namespace/check-namespace           default/Pod/pod-default               Pass
  disallow-default-namespace/check-namespace           staging/Pod/pod-staging               Pass

Test Summary: 2 tests passed and 0 tests failed
```

> The "Pass" in the result column of `kyverno test` output means the **test assertion matched** the actual result. Since we asserted `pod-default` should `fail` and it did fail, the test itself passes. Similarly, we asserted `pod-staging` should `pass` and it did — so that test passes too.

**Run with verbose output:**

```bash
kyverno test disallow-default-ns/ --detailed-results
```

**Run with a specific test file name:**

```bash
kyverno test disallow-default-ns/ -f kyverno-test.yaml
```

**Intentionally break a test to see failure output:**

Edit `kyverno-test.yaml` and change `pod-staging`'s expected result to `fail`, then re-run:

```bash
# After editing to have wrong expectation:
kyverno test disallow-default-ns/

# Output shows test failure:
# Test Summary: 1 tests passed and 1 tests failed
# FAILED TEST: disallow-default-namespace/check-namespace - pod-staging
#   Expected: fail, Got: pass
```

**Revert the change:**

```bash
# Restore the correct result: pass for pod-staging
```

</details>

---

## Summary

| CLI Command | Purpose |
|---|---|
| `kyverno apply <policy> --resource <res>` | Validate or preview mutation of a resource against a policy offline |
| `kyverno apply ... --exception <file>` | Include exception files to see `skip` results |
| `kyverno apply ... --policy-report` | Output results as a `PolicyReport` YAML object |
| `kyverno apply ... -o <file>` | Save mutated resource output to a file |
| `kyverno test <dir>` | Run a test suite from a directory containing a `kyverno-test.yaml` |
| `kyverno test ... --detailed-results` | Show per-result verbose output |
| `kyverno version` | Print the CLI version |

**`kyverno-test.yaml` result values:**

| Value | Meaning |
|---|---|
| `pass` | Resource satisfies the policy rule |
| `fail` | Resource violates the policy rule |
| `skip` | Resource is exempted by a PolicyException |
| `warn` | Resource triggers a warning (Audit mode with warning action) |

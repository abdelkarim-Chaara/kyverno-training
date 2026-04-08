# Lab 09 — Policy Exceptions

## Objectives

- Create a `PolicyException` to exempt specific workloads from a policy without modifying the policy
- Scope exceptions using `podSelector`-style name matching
- Add conditions to an exception to restrict when it applies
- Interpret `skip` results in Policy Reports

---

## Exercise 1: Create a PolicyException for a Privileged Workload

### Goal

1. Apply a policy `disallow-privileged` that blocks pods with `privileged: true`.
2. Create a `PolicyException` in namespace `ops` that exempts pods whose name starts with `ops-agent` from this policy.

### Expected Behavior

| Pod Name | Privileged | Namespace | Result |
|---|---|---|---|
| `ops-agent-1` | `true` | `ops` | Pass (exempted) |
| `app-server` | `true` | `ops` | Fail (not exempted) |
| `ops-agent-2` | `false` | `ops` | Pass (policy not triggered) |

<details>
<summary>Solution</summary>

**Step 1 — Create the `ops` namespace:**

```bash
kubectl create namespace ops
```

**ClusterPolicy — `disallow-privileged.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-privileged
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Privileged containers are not allowed."
        pattern:
          spec:
            containers:
              - =(securityContext):
                  =(privileged): "false"
```

**Apply the policy:**

```bash
kubectl apply -f disallow-privileged.yaml
```

**PolicyException — `ops-exception.yaml`**

```yaml
apiVersion: kyverno.io/v2
kind: PolicyException
metadata:
  name: ops-exception
  namespace: ops
spec:
  exceptions:
    - policyName: disallow-privileged
      ruleNames:
        - check-privileged
  match:
    any:
      - resources:
          kinds:
            - Pod
          namespaces:
            - ops
          names:
            - "ops-agent*"
```

**Apply the exception:**

```bash
kubectl apply -f ops-exception.yaml
```

**Test 1 — `ops-agent-1` with privileged=true in ops (should PASS — exempted):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ops-agent-1
  namespace: ops
spec:
  containers:
    - name: agent
      image: busybox:latest
      securityContext:
        privileged: true
EOF

kubectl get pod ops-agent-1 -n ops
# Pod should be running
```

**Test 2 — `app-server` with privileged=true in ops (should FAIL — not exempted):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: app-server
  namespace: ops
spec:
  containers:
    - name: app
      image: nginx:1.25
      securityContext:
        privileged: true
EOF
# Expected error:
# Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
# resource Pod/ops/app-server was blocked due to the following policies
#
# disallow-privileged:
#   check-privileged: Privileged containers are not allowed.
```

**Test 3 — Verify the exception status:**

```bash
kubectl get policyexception ops-exception -n ops
kubectl describe policyexception ops-exception -n ops
```

**Cleanup (keep for Exercise 2):**

```bash
kubectl delete pod ops-agent-1 -n ops --ignore-not-found
```

</details>

---

## Exercise 2: Exception with Conditions

### Goal

Extend the `PolicyException` from Exercise 1 by adding a `conditions` block that restricts the exception to only apply when the request is made by the user `admin-user`. Regular users submitting the same pod name will still be blocked.

> Note: In most lab environments you cannot easily impersonate a specific username. The purpose of this exercise is to understand the YAML structure and behavior. The `request.userInfo.username` field is populated by the API server based on the authenticated identity.

<details>
<summary>Solution</summary>

**Updated PolicyException — `ops-exception-with-conditions.yaml`**

```yaml
apiVersion: kyverno.io/v2
kind: PolicyException
metadata:
  name: ops-exception
  namespace: ops
spec:
  exceptions:
    - policyName: disallow-privileged
      ruleNames:
        - check-privileged
  match:
    any:
      - resources:
          kinds:
            - Pod
          namespaces:
            - ops
          names:
            - "ops-agent*"
  conditions:
    any:
      - key: "{{ request.userInfo.username }}"
        operator: Equals
        value: "admin-user"
```

**Apply the updated exception:**

```bash
kubectl apply -f ops-exception-with-conditions.yaml
```

**Behavior explanation:**

Without the `conditions` block, the exception applies to any request that matches the resource criteria (name starts with `ops-agent*` in namespace `ops`).

With the `conditions` block added:
- A request from `admin-user` creating `ops-agent-1` with `privileged: true` in `ops` → **Pass** (both resource and condition match)
- A request from `developer` creating `ops-agent-1` with `privileged: true` in `ops` → **Fail** (resource matches but condition does not; exception is not applied; policy blocks it)
- A request from `admin-user` creating `app-server` with `privileged: true` in `ops` → **Fail** (resource name does not match; exception never considered)

**To test with a specific username (requires cluster admin or impersonation rights):**

```bash
# Test as admin-user (should pass if your identity is admin-user)
kubectl apply -f - --as=admin-user <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ops-agent-2
  namespace: ops
spec:
  containers:
    - name: agent
      image: busybox:latest
      securityContext:
        privileged: true
EOF

# Test as a different user (should fail)
kubectl apply -f - --as=developer <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ops-agent-3
  namespace: ops
spec:
  containers:
    - name: agent
      image: busybox:latest
      securityContext:
        privileged: true
EOF
# Expected: admission denied — developer does not satisfy the condition
```

**Cleanup (keep namespace and policy for Exercise 3):**

```bash
kubectl delete pod ops-agent-2 ops-agent-3 -n ops --ignore-not-found
```

</details>

---

## Exercise 3: View Exception Status in Policy Reports

### Goal

After creating the PolicyException, observe how exempted and blocked resources appear in the `PolicyReport` generated by Kyverno. Understand what the `skip` result means.

<details>
<summary>Solution</summary>

**Step 1 — Create resources to populate the report:**

```bash
# This should pass (exempted by ops-exception — remove conditions block first if using basic exception)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ops-agent-report
  namespace: ops
spec:
  containers:
    - name: agent
      image: busybox:latest
      securityContext:
        privileged: true
EOF
```

**Step 2 — Wait for the Policy Report to be generated (Kyverno generates it asynchronously):**

```bash
sleep 5
kubectl get policyreport -n ops
# NAME                        PASS   FAIL   WARN   ERROR   SKIP   AGE
# cpol-disallow-privileged    0      0      0      0       1      10s
```

**Step 3 — Inspect the report:**

```bash
kubectl get policyreport -n ops -o yaml | grep -A 10 result
```

Example output:

```yaml
results:
  - message: ""
    policy: disallow-privileged
    resources:
      - apiVersion: v1
        kind: Pod
        name: ops-agent-report
        namespace: ops
        uid: abc123...
    result: skip
    rule: check-privileged
    scored: true
    source: kyverno
```

**Step 4 — View all results with their statuses:**

```bash
kubectl get policyreport -n ops -o jsonpath='{range .items[*].results[*]}{.policy}{"\t"}{.rule}{"\t"}{.result}{"\t"}{.resources[0].name}{"\n"}{end}'
```

**Explanation of `result: skip`:**

| Result | Meaning |
|---|---|
| `pass` | The resource was evaluated against the rule and satisfied all conditions |
| `fail` | The resource was evaluated and violated one or more rules |
| `skip` | The resource matched a `PolicyException` — the rule was not enforced |
| `warn` | The resource violated a rule in `Audit` mode with `validationFailureActionOverrides` warning |
| `error` | An internal error occurred during policy evaluation |

A `skip` result means the resource **would have failed** the policy, but a `PolicyException` was found that matched both the resource and (if present) the conditions. The admission was allowed because of the exception, and the report records this transparently so security teams can audit all exemptions.

**Step 5 — Check cluster-wide policy reports:**

```bash
kubectl get clusterpolicyreport -o wide
```

**Cleanup:**

```bash
kubectl delete pod ops-agent-report -n ops --ignore-not-found
kubectl delete policyexception ops-exception -n ops --ignore-not-found
kubectl delete clusterpolicy disallow-privileged
kubectl delete namespace ops
```

</details>

---

## Summary

| Concept | Key Point |
|---|---|
| `PolicyException` | A namespaced resource that exempts matching resources from specific policy rules |
| `spec.exceptions[].policyName` | The name of the `ClusterPolicy` (or `Policy`) to exempt from |
| `spec.exceptions[].ruleNames` | List of rule names within that policy to exempt |
| `spec.match` | Resource selector — same syntax as a policy `match` block |
| `spec.conditions` | Optional additional conditions; exception only applies if all conditions are met |
| `result: skip` | Means a PolicyException matched; the resource was admitted but the exemption is recorded |
| Exception namespace | A `PolicyException` must be in the **same namespace** as the resources it exempts (for namespace-scoped resources) |

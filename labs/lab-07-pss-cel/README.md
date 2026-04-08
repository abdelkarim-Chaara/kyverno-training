# Lab 07 — Pod Security Standards (PSS) and CEL Validation

## Objectives

- Apply Kyverno's built-in Pod Security Standards using the `podSecurity` validate subrule
- Configure PSS exemptions at the control level for specific images
- Write CEL (Common Expression Language) validation rules with `validate.cel`
- Use `celPreconditions` to scope CEL rules to specific namespaces

---

## Exercise 1: Pod Security Standards — Enforce Baseline

### Goal

Apply a `ClusterPolicy` that enforces the PSS `baseline` profile cluster-wide. The baseline profile blocks known privilege escalation vectors such as `hostIPC`, `hostPID`, `hostNetwork`, and privileged containers.

### Expected Behavior

| Pod Configuration | Result |
|---|---|
| Normal pod (no host access) | Pass |
| Pod with `hostIPC: true` | Fail |

<details>
<summary>Solution</summary>

**ClusterPolicy — `pss-baseline.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: pss-baseline
spec:
  validationFailureAction: Enforce
  rules:
    - name: baseline
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        podSecurity:
          level: baseline
          version: latest
```

**Apply the policy:**

```bash
kubectl apply -f pss-baseline.yaml
```

**Test 1 — Normal pod (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: safe-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
EOF
```

**Test 2 — Pod with `hostIPC: true` (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hostipc-pod
spec:
  hostIPC: true
  containers:
    - name: app
      image: nginx:1.25
EOF
```

Expected error output from Kyverno:

```
Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
resource Pod/default/hostipc-pod was blocked due to the following policies

pss-baseline:
  baseline: |
    Validation rule 'baseline' failed. It violates PodSecurity "baseline:latest":
    ({message:host namespaces
    reason:  pods "hostipc-pod" violates PodSecurity "baseline:latest": hostIPC=true})
```

**Test 3 — Pod with `hostPID: true` (should also FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hostpid-pod
spec:
  hostPID: true
  containers:
    - name: app
      image: nginx:1.25
EOF
```

**Cleanup:**

```bash
kubectl delete pod safe-pod --ignore-not-found
kubectl delete clusterpolicy pss-baseline
```

</details>

---

## Exercise 2: PSS with Container-Level Exemption

### Goal

Apply the same PSS `baseline` policy but add an **exemption** for the `Capabilities` control for containers whose image matches `nginx*`. This allows nginx containers to use additional Linux capabilities even under the baseline profile.

### Expected Behavior

| Container Image | Capability | Result |
|---|---|---|
| `nginx:1.25` | `SYS_ADMIN` | Pass (exempted) |
| `busybox:latest` | `SYS_ADMIN` | Fail |

<details>
<summary>Solution</summary>

**ClusterPolicy — `pss-baseline-exemption.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: pss-baseline-exemption
spec:
  validationFailureAction: Enforce
  rules:
    - name: baseline-with-exemption
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        podSecurity:
          level: baseline
          version: latest
          exemptions:
            - controlName: Capabilities
              images:
                - "nginx*"
```

**Apply the policy:**

```bash
kubectl apply -f pss-baseline-exemption.yaml
```

**Test 1 — nginx with SYS_ADMIN (should PASS — exempted):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-privileged
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      securityContext:
        capabilities:
          add:
            - SYS_ADMIN
EOF
```

**Test 2 — busybox with SYS_ADMIN (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: busybox-privileged
spec:
  containers:
    - name: app
      image: busybox:latest
      securityContext:
        capabilities:
          add:
            - SYS_ADMIN
EOF
# Expected error:
# Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
# resource Pod/default/busybox-privileged was blocked due to the following policies
#
# pss-baseline-exemption:
#   baseline-with-exemption: |
#     Validation rule 'baseline-with-exemption' failed. It violates PodSecurity "baseline:latest":
#     ({message:non-default capabilities
#      reason:  pods "busybox-privileged" violates PodSecurity "baseline:latest": container "app" must not include
#               SYS_ADMIN in securityContext.capabilities.add})
```

**Test 3 — Confirm nginx without capabilities still passes:**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-normal
spec:
  containers:
    - name: nginx
      image: nginx:1.25
EOF
```

**Cleanup:**

```bash
kubectl delete pod nginx-privileged nginx-normal --ignore-not-found
kubectl delete clusterpolicy pss-baseline-exemption
```

</details>

---

## Exercise 3: CEL Validation — Disallow hostNetwork

### Goal

Write a `ClusterPolicy` named `disallow-host-network` that uses `validate.cel` to block pods with `hostNetwork: true`. Add a `celPrecondition` so the rule only applies outside the `kube-system` namespace.

The CEL expression to use:

```
!has(object.spec.hostNetwork) || object.spec.hostNetwork == false
```

### Expected Behavior

| Pod Configuration | Namespace | Result |
|---|---|---|
| `hostNetwork: true` | `default` | Fail |
| `hostNetwork: true` | `kube-system` | Pass (skipped by celPrecondition) |
| `hostNetwork: false` | `default` | Pass |
| No `hostNetwork` field | `default` | Pass |

<details>
<summary>Solution</summary>

**ClusterPolicy — `disallow-host-network-cel.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-host-network
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-host-network
      match:
        any:
          - resources:
              kinds:
                - Pod
      celPreconditions:
        - name: not-kube-system
          expression: "object.metadata.namespace != 'kube-system'"
      validate:
        cel:
          expressions:
            - expression: "!has(object.spec.hostNetwork) || object.spec.hostNetwork == false"
              message: "Pods must not use the host network. Set hostNetwork to false or remove the field."
```

**Apply the policy:**

```bash
kubectl apply -f disallow-host-network-cel.yaml
```

**Test 1 — `hostNetwork: true` in default namespace (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hostnet-pod
  namespace: default
spec:
  hostNetwork: true
  containers:
    - name: app
      image: nginx:1.25
EOF
# Expected error:
# Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
# resource Pod/default/hostnet-pod was blocked due to the following policies
#
# disallow-host-network:
#   check-host-network: Pods must not use the host network. Set hostNetwork to false or remove the field.
```

**Test 2 — `hostNetwork: true` in kube-system (should PASS — celPrecondition skips it):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hostnet-pod-ks
  namespace: kube-system
spec:
  hostNetwork: true
  containers:
    - name: app
      image: nginx:1.25
EOF
# This pod is admitted because the celPrecondition filters out kube-system requests.
```

**Test 3 — No hostNetwork field in default (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: normal-pod
  namespace: default
spec:
  containers:
    - name: app
      image: nginx:1.25
EOF
```

**Verify the CEL expression behavior:**

The expression `!has(object.spec.hostNetwork) || object.spec.hostNetwork == false` evaluates as:
- If `hostNetwork` field does not exist: `!has(...)` is `true` → expression passes
- If `hostNetwork` is `false`: second clause is `true` → expression passes
- If `hostNetwork` is `true`: both clauses are `false` → expression fails → admission denied

**Cleanup:**

```bash
kubectl delete pod normal-pod --ignore-not-found
kubectl delete pod hostnet-pod-ks -n kube-system --ignore-not-found
kubectl delete clusterpolicy disallow-host-network
```

</details>

---

## Summary

| Concept | Key Point |
|---|---|
| `validate.podSecurity` | Built-in PSS enforcement using `level: baseline/restricted/privileged` |
| PSS `exemptions` | Exempt specific images from specific PSS controls using `controlName` + `images` |
| `validate.cel` | Write validation logic in CEL expressions instead of YAML patterns |
| `celPreconditions` | CEL expressions that gate whether the rule fires at all |
| `has()` in CEL | Checks if a field exists before accessing it (avoids null pointer errors) |
| PSS baseline controls | Blocks: hostIPC, hostPID, hostNetwork, privileged containers, unsafe capabilities |

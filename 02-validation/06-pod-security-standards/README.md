# 02-06 — Pod Security Standards (PSS)

## Concept

Kyverno implements the Kubernetes Pod Security Standards as a native validation type. Instead of writing individual rules for each security control, you declare a `podSecurity` block with a `level` and Kyverno enforces all controls at that level.

## PSS Levels

| Level | Description |
|---|---|
| `baseline` | Minimally restrictive. Prevents known privilege escalations. |
| `restricted` | Heavily restricted. Follows pod hardening best practices. |
| `privileged` | Unrestricted (no controls). |

---

## Example 1: Enforce Baseline Across the Cluster

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: enforce-pss-baseline
spec:
  validationFailureAction: Enforce
  rules:
    - name: enforce-baseline
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        failureAction: Enforce
        podSecurity:
          level: baseline
          version: latest
```

```bash
kubectl apply -f enforce-pss-baseline.yaml

# FAIL — uses hostIPC (violates baseline)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-bad
spec:
  hostIPC: true
  containers:
    - name: nginx
      image: nginx
EOF

# PASS — no host namespace usage
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-good
spec:
  containers:
    - name: nginx
      image: nginx
EOF

kubectl delete pod pod-good --ignore-not-found=true
```

---

## Example 2: PSS with Exemptions at the Pod Level

Allow `hostIPC` and `hostPID` (Host Namespaces control):

```yaml
validate:
  podSecurity:
    level: baseline
    version: latest
    exclude:
      - controlName: Host Namespaces   # allow hostIPC and hostPID
```

---

## Example 3: PSS with Container-Level Exemptions

Allow specific capabilities only for nginx and redis images:

```yaml
validate:
  podSecurity:
    level: baseline
    version: latest
    exclude:
      - controlName: Capabilities
        images:
          - nginx*
          - redis*
```

---

## Example 4: Pod and Container Level Seccomp Exemption

```yaml
validate:
  podSecurity:
    level: restricted
    version: latest
    exclude:
      - controlName: Seccomp          # pod-level exemption
      - controlName: Seccomp          # container-level exemption
        images:
          - "*"
```

---

## Baseline Controls (examples)

| Control | What it blocks |
|---|---|
| Host Namespaces | `hostPID`, `hostIPC`, `hostNetwork: true` |
| Privileged Containers | `securityContext.privileged: true` |
| Capabilities | Adding dangerous Linux capabilities |
| HostPath Volumes | Mounting host paths |
| Host Ports | Container ports binding on the host |
| AppArmor | Overriding default AppArmor profiles |
| SELinux | Custom SELinux options |
| /proc Mount Type | Non-default /proc mount types |
| Seccomp | `Unconfined` seccomp profiles |
| Sysctls | Unsafe sysctls |

---

## Install PSS Policies via Helm

```bash
# Baseline (recommended starting point)
helm install kyverno-policies kyverno/kyverno-policies \
  -n kyverno \
  --set podSecurityStandard=baseline \
  --set validationFailureAction=Audit

# Restricted (tighter security)
helm install kyverno-policies kyverno/kyverno-policies \
  -n kyverno \
  --set podSecurityStandard=restricted \
  --set validationFailureAction=Audit
```

---

## Cleanup

```bash
kubectl delete clusterpolicy enforce-pss-baseline --ignore-not-found=true
```

# Module 08 — Reporting

## Concept

Kyverno generates **PolicyReports** and **ClusterPolicyReports** that consolidate compliance results from:
1. **Admission reviews** — real-time results when resources are created/updated
2. **Background scans** — periodic scans of existing resources

## Report Types

| Resource | Scope | Covers |
|---|---|---|
| `PolicyReport` | Namespaced | Pods, Deployments, Services, ConfigMaps, etc. |
| `ClusterPolicyReport` | Cluster-wide | Namespaces, Nodes, ClusterRoles, etc. |

---

## Anatomy of a PolicyReport

```yaml
kind: PolicyReport
metadata:
  namespace: default
results:
  - policy: require-app-label        # which policy
    rule: check-for-app-label        # which rule
    result: pass                     # pass | fail | skip | warn | error
    message: "validation rule passed"
    resources:
      - kind: Pod
        name: my-pod
        namespace: default
  - policy: require-app-label
    rule: check-for-app-label
    result: fail
    message: "The 'app' label is required for all pods."
    resources:
      - kind: Pod
        name: bad-pod
        namespace: default
scope:
  kind: Pod
  name: bad-pod
  namespace: default
summary:
  pass: 3
  fail: 2
  warn: 0
  skip: 0
  error: 0
```

## Result Values

| Result | Meaning |
|---|---|
| `pass` | Resource matches the rule (compliant) |
| `fail` | Resource violates the rule |
| `skip` | Rule did not apply (precondition not met, or PolicyException) |
| `warn` | Resource failed but policy is annotated with `policies.kyverno.io/scored: "false"` (soft fail) |
| `error` | Policy itself has an error (bad variable substitution, etc.) |

---

## Viewing Policy Reports

```bash
# List all namespaced policy reports
kubectl get policyreport -A
kubectl get polr -A   # short alias

# List cluster-wide policy reports
kubectl get clusterpolicyreport
kubectl get cpolr   # short alias

# View details of a specific report
kubectl describe policyreport <name> -n <namespace>

# View full YAML (shows all results)
kubectl get policyreport <name> -n <namespace> -o yaml

# Filter to see only failures
kubectl get policyreport -A -o json | \
  jq '.items[].results[] | select(.result == "fail")'
```

---

## Policy Application Modes

```yaml
spec:
  admission: true    # (default) enforce at admission time — CREATE, UPDATE
  background: true   # (default) periodically scan existing resources
```

> You can disable one but not both.

---

## Enforce vs Audit — Effect on Reports

| Mode | Compliant resource | Non-compliant resource |
|---|---|---|
| **Audit** | Request PASSES, PolicyReport generated (pass) | Request PASSES, PolicyReport generated (fail) |
| **Enforce** | Request PASSES, PolicyReport generated (pass) | Request BLOCKED, NO PolicyReport generated |

In Enforce mode, non-compliant resources are rejected at the door — they never exist, so no report is created.

---

## Lab 1: Admission-Based Report (Audit Mode)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-pod-owner
spec:
  admission: true
  background: false
  validationFailureAction: Audit
  rules:
    - name: check-pod-owner-label
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "The owner label is required for all Pods."
        pattern:
          metadata:
            labels:
              owner: "?*"
```

```bash
kubectl apply -f require-pod-owner.yaml

kubectl run pod-with-owner -n default --image=nginx:1.14.2 --labels="owner=app-team"
kubectl run pod-without-owner -n default --image=busybox:1.28

# Both pods are created (Audit mode)
# Check the policy report
kubectl get policyreport -n default
kubectl get policyreport -n default -o yaml | grep -A5 result
```

---

## Lab 2: Background Scan Report

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-env-label
spec:
  admission: false      # don't block at admission
  background: true      # only scan existing resources
  validationFailureAction: Enforce
  rules:
    - name: check-env-label
      match:
        any:
          - resources:
              kinds: [Deployment]
      validate:
        message: "The 'env' label is required for all Deployments."
        pattern:
          metadata:
            labels:
              env: "?*"
```

```bash
# Create resources BEFORE applying the policy
kubectl create deployment good-deployment --image=nginx:1.23.0
kubectl label deployment good-deployment env=production

kubectl create deployment bad-deployment --image=nginx:1.23.0

# Now apply the policy
kubectl apply -f check-env-label-background.yaml

# Wait ~30 seconds for background scan
kubectl get policyreport -n default

# Output:
# NAME         KIND         NAME               PASS   FAIL
# abc123...    Deployment   good-deployment    1      0
# def456...    Deployment   bad-deployment     0      1

kubectl get policyreport <bad-report-name> -o yaml
# result: fail
# message: "The 'env' label is required for all Deployments."
# properties:
#   process: background scan

kubectl delete deployment good-deployment bad-deployment
kubectl delete clusterpolicy check-env-label
```

---

## Intermediate Reports

Before final PolicyReports are created, Kyverno uses intermediate resources:
- `AdmissionReport` — from admission events
- `ClusterAdmissionReport` — cluster-scoped admission events
- `BackgroundScanReport` — from background scans

These are merged by the Reports Controller into final PolicyReports.

---

## Policy Reporter UI (Optional)

For a visual dashboard of policy reports, install Policy Reporter:

```bash
helm repo add policy-reporter https://kyverno.github.io/policy-reporter
helm install policy-reporter policy-reporter/policy-reporter \
  -n policy-reporter \
  --create-namespace \
  --set ui.enabled=true \
  --set kyvernoPlugin.enabled=true

kubectl port-forward service/policy-reporter-ui 8082:8080 -n policy-reporter
# Open http://localhost:8082
```

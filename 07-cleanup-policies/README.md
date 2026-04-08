# Module 07 — Cleanup Policies

## Concept

`ClusterCleanupPolicy` and `CleanupPolicy` allow you to **automatically delete resources** on a cron schedule based on match criteria and conditions. They are processed by the **Cleanup Controller**.

## Types

| Resource | Scope |
|---|---|
| `CleanupPolicy` | Namespaced — deletes resources in a specific namespace |
| `ClusterCleanupPolicy` | Cluster-wide — deletes resources across all namespaces |

## RBAC Requirement

The cleanup controller needs permission to list and delete the target resources. Grant it via a ClusterRole with the aggregate label:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kyverno:delete-deployments
  labels:
    rbac.kyverno.io/aggregate-to-cleanup-controller: "true"   # auto-aggregates to cleanup controller
rules:
  - apiGroups: [apps]
    resources: [deployments]
    verbs: [get, watch, list, delete]
```

---

## Example 1: Delete Stale Deployments on Schedule

Delete Deployments labeled `canremove=true` that have fewer than 2 replicas. Run every minute.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kyverno:delete-deployments
  labels:
    rbac.kyverno.io/aggregate-to-cleanup-controller: "true"
rules:
  - apiGroups: [apps]
    resources: [deployments]
    verbs: [get, watch, list, delete]
---
apiVersion: kyverno.io/v2
kind: ClusterCleanupPolicy
metadata:
  name: cleanup-stale-deployments
spec:
  match:
    any:
      - resources:
          kinds: [Deployment]
          selector:
            matchLabels:
              canremove: "true"
  conditions:
    any:
      - key: "{{ target.spec.replicas }}"
        operator: LessThan
        value: 2
  schedule: "*/1 * * * *"   # every minute (cron syntax)
```

```bash
kubectl apply -f cleanup-stale-deployments.yaml

# This deployment will be cleaned up (has label, < 2 replicas)
kubectl create deployment test-deployment-1 --image=nginx:1.25
kubectl label deployment test-deployment-1 canremove="true"
# replicas defaults to 1 < 2 → will be deleted on next schedule run

# This deployment will NOT be cleaned up (no canremove label)
kubectl create deployment test-deployment-2 --image=nginx:1.25

# Wait up to 1 minute
kubectl get deployment test-deployment-1   # disappears
kubectl get deployment test-deployment-2   # stays

kubectl delete deployment test-deployment-2 --ignore-not-found=true
kubectl delete clustercleanuppolicy cleanup-stale-deployments
```

---

## Example 2: TTL-Based Deletion via Label

Set a TTL on individual resources by adding a label. The cleanup controller checks and deletes them after the TTL expires.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-server
  labels:
    cleanup.kyverno.io/ttl: 2m    # delete after 2 minutes
  annotations:
    cleanup.kyverno.io/propagation-policy: Foreground   # wait for all pods to terminate
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-server
  template:
    metadata:
      labels:
        app: nginx-server
    spec:
      containers:
        - name: nginx-server
          image: nginx
```

```bash
kubectl apply -f ttl-deployment.yaml

# After 2 minutes
kubectl get deployment nginx-server   # no longer exists
```

**Supported TTL formats:**
- `2m` — 2 minutes
- `1h` — 1 hour
- `24h` — 24 hours
- `7d` — 7 days

---

## Cron Schedule Reference

```
# ┌───── minute (0-59)
# │ ┌───── hour (0-23)
# │ │ ┌───── day of month (1-31)
# │ │ │ ┌───── month (1-12)
# │ │ │ │ ┌───── day of week (0-6, Sun=0)
# │ │ │ │ │
  * * * * *

"*/1 * * * *"    # every minute
"0 * * * *"      # every hour
"0 0 * * *"      # every day at midnight
"0 0 * * 0"      # every Sunday at midnight
"0 2 1 * *"      # first day of every month at 2am
```

---

## Cleanup Controllers

```bash
# Check cleanup controller
kubectl get pods -n kyverno | grep cleanup

# Check cleanup policy status
kubectl get clustercleanuppolicies
kubectl describe clustercleanuppolicy cleanup-stale-deployments
```

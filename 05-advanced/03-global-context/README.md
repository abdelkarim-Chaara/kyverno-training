# 05-03 — Global Context (GlobalContextEntry)

## Concept

`GlobalContextEntry` is a Custom Resource that tells Kyverno to **pre-fetch and cache** data from the Kubernetes API or an external service. Policies then reference this cache instead of making a live API call for each request.

**Benefits:**
- Reduces API server load under high admission rates
- Faster policy evaluation (cache hit vs live call)

## Two Cache Types

| Type | Description |
|---|---|
| `kubernetesResource` | Cache a specific Kubernetes resource (e.g., all Deployments in a namespace) |
| `apiCall` | Cache results of a Kubernetes API call or an external HTTPS call |

## apiCall types:
- `urlPath` — calls the Kubernetes API (internal)
- `service.url` — calls an external HTTPS service (requires CA bundle)

---

## Example: Cache Deployments and Enforce Existence

```yaml
# Step 1: Define what to cache
apiVersion: kyverno.io/v2alpha1
kind: GlobalContextEntry
metadata:
  name: blue-deployments
spec:
  kubernetesResource:
    group: apps
    version: v1
    resource: deployments
    namespace: default   # omit for cluster-scoped
```

```yaml
# Step 2: Reference the cache in your policy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-main-deployment
spec:
  rules:
    - name: main-deployment-exists
      match:
        any:
          - resources:
              kinds: [Pod]
      context:
        - name: deploymentCount
          globalReference:
            name: blue-deployments     # must match GlobalContextEntry metadata.name
            jmesPath: "length(@)"      # count items in cache
      validate:
        message: "At least one Deployment must exist before creating Pods."
        deny:
          conditions:
            any:
              - key: "{{ deploymentCount }}"
                operator: Equals
                value: 0
```

```bash
kubectl apply -f global-context-deployments.yaml
kubectl get globalcontextentries
```

---

## Refresh Interval

By default, Kyverno refreshes the cache periodically. You can configure the refresh interval:

```yaml
spec:
  kubernetesResource:
    group: apps
    version: v1
    resource: deployments
```

The cache is automatically invalidated and refreshed when the watched resources change.

---

## Cleanup

```bash
kubectl delete globalcontextentry blue-deployments --ignore-not-found=true
kubectl delete clusterpolicy require-main-deployment --ignore-not-found=true
```

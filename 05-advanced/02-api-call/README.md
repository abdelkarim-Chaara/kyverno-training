# 05-02 — API Call Context

## Concept

Use `apiCall` in the context to query the **Kubernetes API Server** (or an external HTTPS endpoint) at policy evaluation time. This gives you access to live cluster data.

## Finding API Paths

```bash
# List all resource types with their API groups and versions
kubectl api-resources

# List all API versions
kubectl api-versions

# Test a path
kubectl get --raw /api/v1/namespaces/default/services
kubectl get --raw "/api/v1/namespaces/default/services?fieldSelector=metadata.name=myservice"
```

## API Path Patterns

| Resource Type | Path Pattern |
|---|---|
| Core resources (Pod, Service) | `/api/v1/namespaces/{NS}/{type}` |
| Grouped resources (Deployment) | `/apis/{group}/{version}/namespaces/{NS}/{type}` |
| Cluster-scoped (Node, Namespace) | `/api/v1/{type}` or `/apis/{group}/{version}/{type}` |

## JMESPath Filtering

The `jmesPath` field filters/transforms the API response:

```yaml
apiCall:
  urlPath: "/api/v1/namespaces/{{ request.namespace }}/services"
  jmesPath: "items[?spec.type == 'LoadBalancer'] | length(@)"
  # Counts LoadBalancer services in the namespace
```

---

## Example 1: Limit LoadBalancer Services Per Namespace

Allow only one LoadBalancer service per namespace.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: one-lb-per-namespace
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-existing-lb
      match:
        any:
          - resources:
              kinds: [Service]
      context:
        - name: existing_lb
          apiCall:
            urlPath: "/api/v1/namespaces/{{ request.namespace }}/services"
            jmesPath: "items[?spec.type == 'LoadBalancer'] | length(@)"
      validate:
        message: "Only one LoadBalancer service is allowed per namespace."
        deny:
          conditions:
            any:
              - key: "{{ existing_lb }}"
                operator: GreaterThanOrEquals
                value: 1
```

---

## Example 2: Propagate Namespace Labels to Deployments

Automatically add the namespace's `environment` label to all new Deployments in that namespace.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-label
spec:
  rules:
    - name: add-label
      match:
        any:
          - resources:
              kinds: [Deployment]
      context:
        - name: namespaceLabels
          apiCall:
            urlPath: "/api/v1/namespaces/{{ request.namespace }}"
            jmesPath: "metadata.labels"   # extract only the labels
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              environment: "{{ namespaceLabels.environment }}"
```

```bash
kubectl apply -f add-label-from-ns.yaml

# Label the namespace
kubectl create namespace production
kubectl label namespace production environment=prod

# Create a deployment — it gets the label automatically
kubectl create deployment my-prod-app \
  --image=nginx:1.25.0 \
  --namespace=production

kubectl get deployment my-prod-app -n production --show-labels
# LABELS: ...,environment=prod

kubectl delete namespace production
```

---

## Cleanup

```bash
kubectl delete clusterpolicy one-lb-per-namespace add-label --ignore-not-found=true
```

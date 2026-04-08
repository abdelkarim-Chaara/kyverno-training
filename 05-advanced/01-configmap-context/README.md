# 05-01 — ConfigMap Context

## Concept

Fetch values from a ConfigMap and use them in your policy rules. This is useful for:
- Centralizing configuration (e.g., allowed registries, region names)
- Updating policy behavior without changing the policy YAML

## Syntax

```yaml
context:
  - name: cmData           # variable name
    configMap:
      name: cluster-info   # ConfigMap name
      namespace: default   # ConfigMap namespace

# Access data: {{ cmData.data.KEY_NAME }}
```

---

## Example: Add Deployment Label from ConfigMap

Add a `deployment-region` label to all Deployments, with the value sourced from a ConfigMap.

```bash
kubectl create configmap cluster-info \
  --from-literal=region=us-east-1
```

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
        - name: cmData
          configMap:
            name: cluster-info
            namespace: default
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              deployment-region: "{{ cmData.data.region }}"
```

```bash
kubectl apply -f add-label-from-cm.yaml

kubectl create deployment sample-app --image=nginx:1.25.0

kubectl get deploy sample-app --show-labels
# NAME         READY   LABELS
# sample-app   1/1     app=sample-app,deployment-region=us-east-1

kubectl delete deployment sample-app
```

---

## Accessing Arrays in ConfigMap

To convert an array stored as JSON in a ConfigMap key, use `parse_json(@)`:

```yaml
# ConfigMap:
# data:
#   allowed_registries: '["gcr.io", "docker.io", "quay.io"]'

context:
  - name: registryConfig
    configMap:
      name: registry-config
      namespace: default

# Access: {{ parse_json(registryConfig.data.allowed_registries) }}
```

---

## Cleanup

```bash
kubectl delete clusterpolicy add-label --ignore-not-found=true
kubectl delete configmap cluster-info --ignore-not-found=true
```

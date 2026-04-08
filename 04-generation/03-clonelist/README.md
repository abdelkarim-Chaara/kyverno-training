# 04-03 — Generate with CloneList

## Concept

`cloneList` allows you to clone **multiple resources of different types** from a source namespace, selected by label.

## Use Case

A team in the `staging` namespace has multiple ConfigMaps and Secrets that should be available in every new namespace. Instead of writing one clone rule per resource, use `cloneList` with a shared label.

---

## Setup: Label the Source Resources

```bash
# Create source resources in staging namespace
kubectl create namespace staging

kubectl create configmap my-config-app1 \
  -n staging \
  --from-literal=db-host=postgres.staging \
  --from-literal=cache-host=redis.staging

kubectl create secret generic my-secret-app1 \
  -n staging \
  --from-literal=api-key=supersecret123

# Label them as "allowed to be cloned"
kubectl label configmap my-config-app1 -n staging allowedToBeCloned=true
kubectl label secret my-secret-app1 -n staging allowedToBeCloned=true

# Verify
kubectl get cm,secret -n staging --show-labels
```

---

## CloneList Policy

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-cm-and-secret
spec:
  rules:
    - name: generate-cm-and-secret
      match:
        any:
          - resources:
              kinds: [Namespace]
      generate:
        namespace: "{{ request.object.metadata.name }}"
        synchronize: true
        cloneList:
          namespace: staging          # source namespace
          kinds:
            - v1/Secret
            - v1/ConfigMap
          selector:
            matchLabels:
              allowedToBeCloned: "true"  # only resources with this label
```

```bash
kubectl apply -f generate-cm-and-secret.yaml

kubectl create namespace my-new-ns

# Both ConfigMap and Secret are cloned
kubectl get cm,secret -n my-new-ns

kubectl delete namespace my-new-ns
```

---

## Cleanup

```bash
kubectl delete clusterpolicy generate-cm-and-secret --ignore-not-found=true
kubectl delete namespace staging --ignore-not-found=true
```

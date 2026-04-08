# 04-02 — Generate with Clone

## Concept

Use `generate.clone` to **copy an existing resource** into new namespaces. The original resource is the "source":
- When the source changes, all clones are updated (with `synchronize: true`)
- When the **policy** is deleted, the clones are **NOT** deleted (unlike data-based rules)
- When the source is deleted, clones still exist but are no longer updated

## Use Case

Spreading image pull secrets, TLS certificates, or shared ConfigMaps across all namespaces automatically.

---

## Example: Auto-Clone Registry Secret to All New Namespaces

```bash
# First, create the source secret
kubectl create secret generic regcred \
  -n default \
  --from-literal=username='testuser' \
  --from-literal=password='testpass123'
```

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sync-secrets
spec:
  rules:
    - name: sync-image-pull-secret
      match:
        any:
          - resources:
              kinds: [Namespace]
      generate:
        apiVersion: v1
        kind: Secret
        name: regcred
        namespace: "{{ request.object.metadata.name }}"
        synchronize: true
        clone:
          namespace: default   # source namespace
          name: regcred        # source resource name
```

```bash
kubectl apply -f sync-secrets.yaml

kubectl create namespace new-team-ns

# Secret is automatically cloned
kubectl get secret regcred -n new-team-ns
kubectl get secret regcred -n new-team-ns -o yaml

# Test sync: update the source
kubectl patch secret regcred -n default \
  -p '{"stringData":{"password":"newpassword456"}}'

# The clone should be updated automatically (may take a few seconds)
kubectl get secret regcred -n new-team-ns -o jsonpath='{.data.password}' | base64 -d

kubectl delete namespace new-team-ns
```

---

## Cleanup

```bash
kubectl delete clusterpolicy sync-secrets --ignore-not-found=true
kubectl delete secret regcred -n default --ignore-not-found=true
```

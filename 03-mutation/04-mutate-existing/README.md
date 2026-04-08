# 03-04 — Mutate Existing Resources

## Concept

Standard mutation rules apply to resources at **admission time** (when they are created/updated). Mutate Existing rules are processed by the **Background Controller** and can modify resources that **already exist** in the cluster.

This is useful for:
- Propagating labels/annotations to existing resources when a trigger changes
- Keeping related resources in sync

## Anatomy

```yaml
match:       # Trigger: WHEN this resource is created/updated
  any: ...

mutate:
  targets:   # Target: APPLY changes to this other resource
    - apiVersion: v1
      kind: Secret
      name: my-secret
      namespace: my-namespace
  patchStrategicMerge:
    # changes to apply to the target
```

---

## Example: Sync Label to a Secret When ConfigMap Changes

When a ConfigMap named `dictionary-1` is created/updated in the `staging` namespace, add label `foo: bar` to Secret `secret-1` in the same namespace.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: mutate-existing-secret-policy
spec:
  rules:
    - name: mutate-secret-on-configmap-event
      match:
        any:
          - resources:
              kinds: [ConfigMap]
              names: ["dictionary-1"]
              namespaces: ["staging"]
      mutate:
        targets:
          - apiVersion: v1
            kind: Secret
            name: secret-1
            namespace: staging
        patchStrategicMerge:
          metadata:
            labels:
              foo: bar
```

```bash
kubectl apply -f mutate-existing-secret-policy.yaml

# Create the trigger — this causes the Secret to be mutated
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: dictionary-1
  namespace: staging
data:
  trigger: "data"
EOF

# Check the Secret was mutated
kubectl get secret secret-1 -n staging -o jsonpath='{.metadata.labels}'
# {"foo":"bar"}
```

---

## Cleanup

```bash
kubectl delete clusterpolicy mutate-existing-secret-policy --ignore-not-found=true
```

# 03-01 — patchStrategicMerge

## Concept

`patchStrategicMerge` takes a **partial YAML definition** and merges it onto the original resource. Think of it as writing what you _want_ the resource to look like, and Kyverno handles the merge.

---

## Example 1: Add imagePullPolicy for Latest Images

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-imagepullpolicy
spec:
  background: false
  rules:
    - name: add-imagepullpolicy-always
      match:
        any:
          - resources:
              kinds: [Pod]
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - (image): "*:latest"   # Conditional anchor: IF image ends with :latest
                imagePullPolicy: Always  # THEN set this peer field
```

```bash
kubectl apply -f add-imagepullpolicy.yaml

# The mutation applies automatically
kubectl run test --image=nginx:latest --labels="app=test"
kubectl get pod test -o yaml | grep imagePullPolicy
# imagePullPolicy: Always  <- automatically added

kubectl delete pod test
```

---

## Example 2: Add a Default Label

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-label
spec:
  rules:
    - name: add-default-label
      match:
        any:
          - resources:
              kinds: [ConfigMap]
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              +(managed-by): kyverno  # Add only if not already present
```

---

## Example 3: Add SecurityContext to All Containers

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-securitycontext
spec:
  rules:
    - name: add-securitycontext
      match:
        any:
          - resources:
              kinds: [Pod]
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - (name): "*"    # match all containers
                securityContext:
                  +(allowPrivilegeEscalation): false
```

---

## Test Mutation Was Applied

```bash
# Check mutation annotations
kubectl get pod <name> -o jsonpath='{.metadata.annotations}' | grep kyverno

# Check the specific field
kubectl get pod <name> -o yaml | grep imagePullPolicy
```

---

## Cleanup

```bash
kubectl delete clusterpolicy add-imagepullpolicy add-default-label add-securitycontext --ignore-not-found=true
```

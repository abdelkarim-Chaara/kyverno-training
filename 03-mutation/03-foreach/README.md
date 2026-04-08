# 03-03 — ForEach Mutation

## Concept

`foreach` in a mutate rule lets you loop over a list of items in a resource and apply a patch to each one individually.

Special variables:
- `element` — the current item in the loop
- `elementIndex` — the zero-based index of the current item

---

## Example 1: Add SecurityContext to Every Container

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: mutate-containers
spec:
  rules:
    - name: mutate-containers
      match:
        any:
          - resources:
              kinds: [Pod]
              operations: [CREATE]
      mutate:
        foreach:
          - list: "request.object.spec.containers"
            patchesJson6902: |-
              - path: /spec/containers/{{elementIndex}}/securityContext
                op: add
                value:
                  runAsNonRoot: true
```

```bash
kubectl apply -f mutate-containers.yaml

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  labels:
    app: test
spec:
  containers:
    - name: nginx-container
      image: nginx:1.25
    - name: busybox-container
      image: busybox:1.36
      command: ["sleep", "3600"]
EOF

kubectl get pods test-pod -o yaml | grep -A3 securityContext
# Both containers get runAsNonRoot: true injected

kubectl delete pod test-pod
```

---

## Example 2: Using patchStrategicMerge in ForEach

```yaml
mutate:
  foreach:
    - list: "request.object.spec.containers[]"
      patchStrategicMerge:
        spec:
          containers:
            - (name): "{{ element.name }}"  # identify which container to patch
              securityContext:
                +(allowPrivilegeEscalation): false  # only if not already set
```

---

## Cleanup

```bash
kubectl delete clusterpolicy mutate-containers --ignore-not-found=true
```

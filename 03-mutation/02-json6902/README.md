# 03-02 — JSON Patch (patchesJson6902)

## Concept

JSON Patch is a list of precise operations targeting specific paths in the resource. Use it when you need to:
- **Remove** a field entirely
- **Modify** a specific element in a list by index
- Perform operations that simple merge cannot handle

## Anatomy

```yaml
mutate:
  patchesJson6902: |-
    - op: <operation>       # add, replace, remove
      path: <JSONPath>      # e.g., /metadata/labels/app
      value: <new value>    # not required for remove
```

## Special Character Escaping

JSON Pointer paths use `/` as a separator. To include these characters literally:
- `/` -> `~1`
- `~` -> `~0`

```yaml
# Example: annotation key contains /
- op: add
  path: /metadata/annotations/config.linkerd.io~1skip-ports
  value: "8200"
```

---

## Example 1: Add and Remove Fields from a ConfigMap

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: mutate-configmap
spec:
  rules:
    - name: mutate-configmap
      match:
        any:
          - resources:
              names: ["config-game"]
              kinds: [ConfigMap]
      mutate:
        patchesJson6902: |-
          - path: "/data/ship.properties"
            op: add
            value: |-
              type=startship
              owner=google
          - path: "/data/newkey1"
            op: add
            value: newValue
          - path: "/data/labels/unwanted"
            op: remove
```

---

## Example 2: Add a Label to Deployments

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-team-label
spec:
  rules:
    - name: add-team-label
      match:
        any:
          - resources:
              kinds: [Deployment]
      mutate:
        patchesJson6902: |-
          - op: add
            path: "/metadata/labels/team"
            value: "operations"
```

```bash
kubectl apply -f add-team-label.yaml

kubectl create deployment nginx-deployment --image=nginx --replicas=3
kubectl get deployment nginx-deployment -o jsonpath='{.metadata.labels}'
# {"app":"nginx-deployment","team":"operations"}

kubectl delete deployment nginx-deployment
```

---

## Example 3: Remove a Label from ConfigMaps

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: remove-purpose-label
spec:
  rules:
    - name: remove-purpose-label
      match:
        any:
          - resources:
              kinds: [ConfigMap]
      mutate:
        patchesJson6902: |-
          - op: remove
            path: /metadata/labels/purpose
```

```bash
kubectl apply -f remove-purpose-label.yaml

kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-game
  labels:
    purpose: game
    type: application
data:
  initial.setting: "true"
EOF

kubectl get cm config-game -o yaml | grep -E "labels|purpose|type"
# labels:
#   type: application    <- purpose label was removed!
```

---

## Example 4: Inject a Sidecar Container

```yaml
mutate:
  patchesJson6902: |-
    - path: "/spec/containers/1"   # list is 0-indexed; this adds at position 1
      op: add
      value:
        name: sidecar
        image: busybox:latest
        command: ["sleep", "3600"]
```

---

## patchStrategicMerge vs JSON Patch — When to Use Which

| | patchStrategicMerge | patchesJson6902 |
|---|---|---|
| Adding / setting defaults | Ideal | Works but verbose |
| Removing a field | Not supported | Use `remove` op |
| Conditional logic (anchors) | Supported | Use preconditions |
| Auto-gen for Deployment/Job | Yes | No |
| Readability | High | Lower |

---

## Cleanup

```bash
kubectl delete clusterpolicy mutate-configmap add-team-label remove-purpose-label --ignore-not-found=true
```

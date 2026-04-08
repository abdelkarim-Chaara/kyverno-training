# Lab 02 — Mutation

## Exercises

### Exercise 1: Add a Team Label to Deployments
Create a ClusterPolicy that adds label `team: operations` to all new Deployments.

<details>
<summary>Solution</summary>

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

**Test:**
```bash
kubectl create deployment nginx-deployment --image=nginx --replicas=3
kubectl get deployment nginx-deployment -o jsonpath='{.metadata.labels}'
# {"app":"nginx-deployment","team":"operations"}
kubectl delete deployment nginx-deployment
```

</details>

---

### Exercise 2: Remove a Label from ConfigMaps
Create a ClusterPolicy that removes the `purpose` label from all ConfigMaps.

<details>
<summary>Solution</summary>

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

</details>

---

### Exercise 3: Auto-add imagePullPolicy
Create a mutation policy that sets `imagePullPolicy: Always` for any container using a `:latest` image tag.

<details>
<summary>Solution</summary>

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
              - (image): "*:latest"
                imagePullPolicy: Always
```

</details>

---

## Cleanup

```bash
kubectl delete clusterpolicy add-team-label remove-purpose-label add-imagepullpolicy --ignore-not-found=true
```

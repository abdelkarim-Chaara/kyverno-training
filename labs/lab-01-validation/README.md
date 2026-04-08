# Lab 01 — Validation

## Exercises

### Exercise 1: Require Labels on Pods
Write a ClusterPolicy that requires all Pods to have both an `app` label AND an `owner` label.

**Expected behavior:**
- Pod with both labels → PASS
- Pod with only `app` label → FAIL
- Pod with no labels → FAIL

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-pod-labels
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-pod-labels
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Pods must have both 'app' and 'owner' labels."
        pattern:
          metadata:
            labels:
              app: "?*"
              owner: "?*"
```

</details>

---

### Exercise 2: Restrict Default Namespace
Prevent pods from being created in the `default` namespace.

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-default-namespace
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-namespace
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Using 'default' namespace is not allowed."
        pattern:
          metadata:
            namespace: "!default"
```

</details>

---

### Exercise 3: Enforce HA for Deployments
Ensure all Deployments have at least 2 replicas.

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-deployment-replicas
spec:
  validationFailureAction: Audit
  rules:
    - name: check-replicas
      match:
        any:
          - resources:
              kinds: [Deployment]
      validate:
        message: "Replica count must be >= 2 for HA."
        pattern:
          spec:
            replicas: ">=2"
```

</details>

---

### Exercise 4: Block NodePort Services
Block creation of Services with `type: NodePort`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-nodeport
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-nodeport
      match:
        any:
          - resources:
              kinds: [Service]
      validate:
        message: "Services of type NodePort are not allowed."
        pattern:
          spec:
            =(type): "!NodePort"
```

</details>

---

### Exercise 5: Deny Deployments with Replicas > 2

**Hint:** Use the `deny` block instead of `pattern`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: limit-deployment-replicas
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-replicas
      match:
        any:
          - resources:
              kinds: [Deployment]
      validate:
        message: "Deployment replicas must not exceed 2."
        deny:
          conditions:
            any:
              - key: "{{ request.object.spec.replicas }}"
                operator: GreaterThan
                value: 2
```

</details>

---

## Cleanup

```bash
kubectl delete clusterpolicy require-pod-labels disallow-default-namespace \
  check-deployment-replicas restrict-nodeport limit-deployment-replicas \
  --ignore-not-found=true
```

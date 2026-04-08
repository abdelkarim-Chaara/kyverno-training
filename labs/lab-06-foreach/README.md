# Lab 06 — ForEach Rules

## Objectives

- Use `foreach` to iterate over lists of sub-resources (containers, ports, ingress rules)
- Apply validation patterns to each element in a list
- Combine `foreach` with `deny` conditions and preconditions for fine-grained control

---

## Exercise 1: ForEach — All Container Images Must Come From Approved Registries

### Goal

Write a `ClusterPolicy` named `require-approved-registry` that iterates over **both** `initContainers` and `containers` and validates that each container image starts with `gcr.io/` or `registry.corp/`.

Use two `foreach` loops with `pattern.image`.

### Expected Behavior

| Pod | Result |
|---|---|
| All containers from `gcr.io/` | Pass |
| All containers from `registry.corp/` | Pass |
| Any container from `docker.io/` | Fail |

<details>
<summary>Solution</summary>

**ClusterPolicy — `require-approved-registry.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-approved-registry
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-init-container-images
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Init container images must come from gcr.io/ or registry.corp/."
        foreach:
          - list: "request.object.spec.initContainers"
            pattern:
              image: "gcr.io/* | registry.corp/*"
    - name: check-container-images
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Container images must come from gcr.io/ or registry.corp/."
        foreach:
          - list: "request.object.spec.containers"
            pattern:
              image: "gcr.io/* | registry.corp/*"
```

**Apply the policy:**

```bash
kubectl apply -f require-approved-registry.yaml
```

**Test 1 — Approved registry (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: approved-pod
spec:
  initContainers:
    - name: init
      image: gcr.io/myproject/init-tool:v1
  containers:
    - name: app
      image: gcr.io/myapp:v1
EOF
```

**Test 2 — Unapproved registry (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: unapproved-pod
spec:
  containers:
    - name: app
      image: docker.io/nginx:latest
EOF
# Expected error:
# Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
# resource Pod/default/unapproved-pod was blocked due to the following policies
# require-approved-registry:
#   check-container-images: Container images must come from gcr.io/ or registry.corp/.
```

**Test 3 — Mix of approved and unapproved (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: mixed-pod
spec:
  containers:
    - name: app
      image: gcr.io/myapp:v1
    - name: sidecar
      image: docker.io/busybox:latest
EOF
```

**Cleanup:**

```bash
kubectl delete pod approved-pod --ignore-not-found
kubectl delete clusterpolicy require-approved-registry
```

</details>

---

## Exercise 2: ForEach — Block Wildcard Ingress Hosts

### Goal

Write a `ClusterPolicy` named `no-wildcard-ingress` that iterates over `spec.rules` of an Ingress resource and uses a `deny` condition to block any rule whose `host` field contains a wildcard (`*`).

Use the JMESPath `contains()` function as the condition key.

### Expected Behavior

| Ingress Host | Result |
|---|---|
| `app.example.com` | Pass |
| `*.example.com` | Fail |

<details>
<summary>Solution</summary>

**ClusterPolicy — `no-wildcard-ingress.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: no-wildcard-ingress
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-wildcard-hosts
      match:
        any:
          - resources:
              kinds:
                - Ingress
      validate:
        message: "Ingress hosts must not use wildcard (*) patterns."
        foreach:
          - list: "request.object.spec.rules"
            deny:
              conditions:
                any:
                  - key: "{{ contains(element.host, '*') }}"
                    operator: Equals
                    value: true
```

**Apply the policy:**

```bash
kubectl apply -f no-wildcard-ingress.yaml
```

**Test 1 — Specific host (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: safe-ingress
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
EOF
```

**Test 2 — Wildcard host (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-ingress
spec:
  rules:
    - host: "*.example.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
EOF
# Expected error:
# Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
# resource Ingress/default/wildcard-ingress was blocked due to the following policies
# no-wildcard-ingress:
#   check-wildcard-hosts: Ingress hosts must not use wildcard (*) patterns.
```

**Test 3 — Multiple rules, one wildcard (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mixed-ingress
spec:
  rules:
    - host: safe.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: svc1
                port:
                  number: 80
    - host: "*.example.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: svc2
                port:
                  number: 80
EOF
```

**Cleanup:**

```bash
kubectl delete ingress safe-ingress --ignore-not-found
kubectl delete clusterpolicy no-wildcard-ingress
```

</details>

---

## Exercise 3: ForEach + Preconditions — LoadBalancer Only Allows Ports 80 and 443

### Goal

Write a `ClusterPolicy` named `restrict-lb-ports` that:

1. Only applies to Services of type `LoadBalancer` (use a `preconditions` block)
2. Iterates over `spec.ports[]`
3. Denies any port that is **not** 80 or 443 (use `AnyNotIn` operator)

### Expected Behavior

| Service | Port(s) | Result |
|---|---|---|
| LoadBalancer | 80 | Pass |
| LoadBalancer | 443 | Pass |
| LoadBalancer | 3000 | Fail |
| ClusterIP | 3000 | Pass (precondition skips it) |

<details>
<summary>Solution</summary>

**ClusterPolicy — `restrict-lb-ports.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-lb-ports
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-lb-ports
      match:
        any:
          - resources:
              kinds:
                - Service
      validate:
        message: "LoadBalancer Services may only expose ports 80 and 443."
        foreach:
          - list: "request.object.spec.ports"
            preconditions:
              all:
                - key: "{{ request.object.spec.type }}"
                  operator: Equals
                  value: LoadBalancer
            deny:
              conditions:
                any:
                  - key: "{{ element.port }}"
                    operator: AnyNotIn
                    value:
                      - 80
                      - 443
```

**Apply the policy:**

```bash
kubectl apply -f restrict-lb-ports.yaml
```

**Test 1 — LoadBalancer on port 80 (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: lb-port-80
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
EOF
```

**Test 2 — LoadBalancer on port 443 (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: lb-port-443
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 443
      targetPort: 8443
EOF
```

**Test 3 — LoadBalancer on port 3000 (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: lb-port-3000
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 3000
      targetPort: 3000
EOF
# Expected error:
# Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
# resource Service/default/lb-port-3000 was blocked due to the following policies
# restrict-lb-ports:
#   check-lb-ports: LoadBalancer Services may only expose ports 80 and 443.
```

**Test 4 — ClusterIP on port 3000 (should PASS — precondition skips non-LB services):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: clusterip-port-3000
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 3000
      targetPort: 3000
EOF
```

**Cleanup:**

```bash
kubectl delete service lb-port-80 lb-port-443 clusterip-port-3000 --ignore-not-found
kubectl delete clusterpolicy restrict-lb-ports
```

</details>

---

## Summary

| Concept | Key Point |
|---|---|
| `foreach` | Iterates over a list; each item is available as `element` |
| `foreach` with `pattern` | Validates each element matches the pattern |
| `foreach` with `deny` | Blocks if the deny condition is met for any element |
| `preconditions` inside `foreach` | Skips iteration entirely if precondition is not met |
| `AnyNotIn` operator | Fails if the key value is not in the provided list |
| `contains()` in JMESPath | Returns true/false; usable as a condition key |

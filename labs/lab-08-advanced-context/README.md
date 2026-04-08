# Lab 08 — Advanced Context: ConfigMap and API Call

## Objectives

- Use a `configMap` context entry to read external data and inject it into mutation rules
- Use an `apiCall` context entry to fetch live Kubernetes API data
- Build policies that propagate namespace-level metadata to workloads
- Enforce namespace-scoped uniqueness constraints using API call context

---

## Exercise 1: ConfigMap Context — Add Region Label from ConfigMap

### Goal

1. Create a ConfigMap named `cluster-info` in the `default` namespace with a `region` key.
2. Write a mutation `ClusterPolicy` named `add-region-label` that reads the `region` value from that ConfigMap and adds it as a label to every new Pod.

### Expected Behavior

Any Pod created in any namespace receives an additional label `region: <value-from-configmap>` automatically.

<details>
<summary>Solution</summary>

**Step 1 — Create the ConfigMap:**

```bash
kubectl create configmap cluster-info \
  --from-literal=region=eu-west-1 \
  -n default
```

Verify:

```bash
kubectl get configmap cluster-info -n default -o yaml
# data:
#   region: eu-west-1
```

**ClusterPolicy — `add-region-label.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-region-label
spec:
  rules:
    - name: add-region-from-configmap
      match:
        any:
          - resources:
              kinds:
                - Pod
      context:
        - name: cmData
          configMap:
            name: cluster-info
            namespace: default
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              region: "{{ cmData.data.region }}"
```

**Apply the policy:**

```bash
kubectl apply -f add-region-label.yaml
```

**Test — Create a pod and verify the label:**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-region-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
EOF

# Verify the label was added:
kubectl get pod test-region-pod --show-labels
# NAME              READY   STATUS    RESTARTS   AGE   LABELS
# test-region-pod   1/1     Running   0          10s   region=eu-west-1,...

kubectl get pod test-region-pod -o jsonpath='{.metadata.labels.region}'
# eu-west-1
```

**Test — Change the ConfigMap value and create another pod:**

```bash
kubectl patch configmap cluster-info -n default --type merge \
  -p '{"data":{"region":"us-east-1"}}'

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-region-pod-2
spec:
  containers:
    - name: app
      image: nginx:1.25
EOF

kubectl get pod test-region-pod-2 -o jsonpath='{.metadata.labels.region}'
# us-east-1
```

**Cleanup:**

```bash
kubectl delete pod test-region-pod test-region-pod-2 --ignore-not-found
kubectl delete configmap cluster-info -n default
kubectl delete clusterpolicy add-region-label
```

</details>

---

## Exercise 2: API Call Context — Propagate Namespace Label to Deployments

### Goal

1. Create a namespace with the label `team=platform`.
2. Write a mutation `ClusterPolicy` named `add-team-label` that fetches the namespace object via an `apiCall` context entry and propagates the `team` label from the namespace to any Deployment created within it.

### Expected Behavior

A Deployment created in a namespace labeled `team=platform` automatically receives the label `team: platform`.

<details>
<summary>Solution</summary>

**Step 1 — Create a labeled namespace:**

```bash
kubectl create namespace platform-ns
kubectl label namespace platform-ns team=platform

# Verify:
kubectl get namespace platform-ns --show-labels
# NAME           STATUS   AGE   LABELS
# platform-ns    Active   5s    kubernetes.io/metadata.name=platform-ns,team=platform
```

**ClusterPolicy — `add-team-label.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-team-label
spec:
  rules:
    - name: propagate-team-label
      match:
        any:
          - resources:
              kinds:
                - Deployment
      context:
        - name: namespaceLabels
          apiCall:
            urlPath: "/api/v1/namespaces/{{ request.namespace }}"
            jmesPath: "metadata.labels"
      mutate:
        patchStrategicMerge:
          metadata:
            labels:
              team: "{{ namespaceLabels.team }}"
```

**Apply the policy:**

```bash
kubectl apply -f add-team-label.yaml
```

**Test — Deploy in the labeled namespace and verify:**

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-app
  namespace: platform-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: platform-app
  template:
    metadata:
      labels:
        app: platform-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
EOF

# Verify the label was propagated:
kubectl get deployment platform-app -n platform-ns --show-labels
# NAME           READY   UP-TO-DATE   AVAILABLE   AGE   LABELS
# platform-app   1/1     1            1           10s   team=platform,...

kubectl get deployment platform-app -n platform-ns \
  -o jsonpath='{.metadata.labels.team}'
# platform
```

**Test — Deployment in namespace without team label (label will be empty or absent):**

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: other-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: other-app
  template:
    metadata:
      labels:
        app: other-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
EOF

kubectl get deployment other-app -n default --show-labels
# The 'team' label will not be present since default namespace has no team label
```

**Cleanup:**

```bash
kubectl delete deployment platform-app -n platform-ns --ignore-not-found
kubectl delete deployment other-app -n default --ignore-not-found
kubectl delete namespace platform-ns
kubectl delete clusterpolicy add-team-label
```

</details>

---

## Exercise 3: API Call — Limit to One LoadBalancer Service Per Namespace

### Goal

Write a validation `ClusterPolicy` named `one-lb-per-ns` that queries the Kubernetes API to count existing LoadBalancer Services in the requesting namespace. Deny the request if there is already one or more LoadBalancer Services present.

This enforces a limit of exactly one LoadBalancer per namespace.

### Expected Behavior

| Action | Result |
|---|---|
| First LoadBalancer Service in a namespace | Pass |
| Second LoadBalancer Service in the same namespace | Fail |
| LoadBalancer Service in a different namespace | Pass |

<details>
<summary>Solution</summary>

**ClusterPolicy — `one-lb-per-ns.yaml`**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: one-lb-per-ns
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-lb-count
      match:
        any:
          - resources:
              kinds:
                - Service
      preconditions:
        all:
          - key: "{{ request.object.spec.type }}"
            operator: Equals
            value: LoadBalancer
      context:
        - name: lbCount
          apiCall:
            urlPath: "/api/v1/namespaces/{{ request.namespace }}/services"
            jmesPath: "items[?spec.type == 'LoadBalancer'] | length(@)"
      validate:
        message: "Only one LoadBalancer Service is allowed per namespace. Found {{ lbCount }} existing."
        deny:
          conditions:
            any:
              - key: "{{ lbCount }}"
                operator: GreaterThanOrEquals
                value: 1
```

**Setup — create a test namespace:**

```bash
kubectl create namespace lb-test
```

**Apply the policy:**

```bash
kubectl apply -f one-lb-per-ns.yaml
```

**Test 1 — First LoadBalancer (should PASS):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: first-lb
  namespace: lb-test
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
EOF

kubectl get service first-lb -n lb-test
# NAME       TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# first-lb   LoadBalancer   10.96.x.x      <pending>     80:xxxxx/TCP   5s
```

**Test 2 — Second LoadBalancer in the same namespace (should FAIL):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: second-lb
  namespace: lb-test
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 443
      targetPort: 8443
EOF
# Expected error:
# Error from server: error when creating "STDIN": admission webhook "validate.kyverno.svc-fail" denied the request:
# resource Service/lb-test/second-lb was blocked due to the following policies
#
# one-lb-per-ns:
#   check-lb-count: Only one LoadBalancer Service is allowed per namespace. Found 1 existing.
```

**Test 3 — LoadBalancer in a different namespace (should PASS):**

```bash
kubectl create namespace lb-test-2

kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: another-lb
  namespace: lb-test-2
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
EOF
```

**Test 4 — ClusterIP in the same namespace (should PASS — precondition skips):**

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: clusterip-svc
  namespace: lb-test
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
EOF
```

**Cleanup:**

```bash
kubectl delete namespace lb-test lb-test-2
kubectl delete clusterpolicy one-lb-per-ns
```

</details>

---

## Summary

| Concept | Key Point |
|---|---|
| `context[].configMap` | Loads a ConfigMap's data into a named variable accessible in JMESPath expressions |
| `context[].apiCall` | Fetches any Kubernetes API endpoint; `jmesPath` extracts the desired field |
| `urlPath` | Must be a valid Kubernetes API path; use `{{ request.namespace }}` for dynamic scoping |
| `jmesPath` on apiCall | Applied server-side to filter/transform the API response before it becomes the context variable |
| `cmData.data.<key>` | How to reference a ConfigMap value — `cmData` is the context name, `data.<key>` navigates into it |
| Dynamic counting | `items[?spec.type == 'LoadBalancer'] | length(@)` counts matching items from an API list response |

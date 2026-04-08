# Lab 04 — Anchors

## Prerequisites

Kyverno installed, `require-app-label` policy from Lab 01 removed.

---

## Exercise 1: Conditional Anchor `()` — Image Pull Policy

Write a **validation** policy that checks: IF a container uses a `:latest` image tag, THEN `imagePullPolicy` must NOT be `IfNotPresent`.

**Expected behavior:**
- Pod with `nginx:latest` and `imagePullPolicy: IfNotPresent` → **FAIL**
- Pod with `nginx:latest` and `imagePullPolicy: Always` → **PASS**
- Pod with `nginx:1.25` and `imagePullPolicy: IfNotPresent` → **PASS** (anchor doesn't match)

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: validate-imagepullpolicy
spec:
  validationFailureAction: Enforce
  rules:
    - name: validate-imagepullpolicy
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Pods with :latest images must use imagePullPolicy: Always or Never, not IfNotPresent."
        pattern:
          spec:
            containers:
              - (image): "*:latest"
                imagePullPolicy: "!IfNotPresent"
```

**Test:**
```bash
kubectl apply -f validate-imagepullpolicy.yaml

# FAIL — latest + IfNotPresent
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  labels:
    app: bad
spec:
  containers:
    - name: nginx
      image: nginx:latest
      imagePullPolicy: IfNotPresent
EOF

# PASS — latest + Always
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  labels:
    app: good
spec:
  containers:
    - name: nginx
      image: nginx:latest
      imagePullPolicy: Always
EOF

kubectl delete pod good-pod --ignore-not-found=true
```

</details>

---

## Exercise 2: Equality Anchor `=()` — HostPath Restriction

Write a policy that checks: IF a Pod defines a `hostPath` volume, THEN its `path` must NOT be `/var/run/docker.sock`.

**Expected behavior:**
- Pod with `hostPath.path: /var/run/docker.sock` → **FAIL**
- Pod with `hostPath.path: /tmp/data` → **PASS**
- Pod with no volumes at all → **PASS** (equality anchor skips absent field)

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-docker-sock
spec:
  validationFailureAction: Enforce
  rules:
    - name: restrict-docker-sock
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Mounting /var/run/docker.sock is not allowed."
        pattern:
          spec:
            =(volumes):
              - =(hostPath):
                  path: "!/var/run/docker.sock"
```

**Test:**
```bash
kubectl apply -f restrict-docker-sock.yaml

# FAIL — docker.sock mount
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  labels:
    app: bad
spec:
  containers:
    - name: app
      image: nginx:1.25
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
EOF

# PASS — safe hostpath
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  labels:
    app: good
spec:
  containers:
    - name: app
      image: nginx:1.25
  volumes:
    - name: data
      hostPath:
        path: /tmp/data
EOF

# PASS — no volumes
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: novol-pod
  labels:
    app: novol
spec:
  containers:
    - name: app
      image: nginx:1.25
EOF

kubectl delete pod good-pod novol-pod --ignore-not-found=true
```

</details>

---

## Exercise 3: Negation Anchor `X()` — Forbid HostPath Volumes Entirely

Write a policy that **forbids** any Pod from defining a `hostPath` volume (regardless of path).

**Hint:** Use the `X()` negation anchor — it denies if the field exists.

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-hostpath
spec:
  validationFailureAction: Enforce
  rules:
    - name: disallow-hostpath
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "HostPath volumes are not allowed."
        pattern:
          spec:
            volumes:
              - name: "*"
                X(hostPath): "null"
```

</details>

---

## Exercise 4: Global Anchor `<()` — Require imagePullSecrets for Private Registry

Write a policy: IF any container uses an image from `nexus.corp/*`, THEN the Pod must define `imagePullSecrets` with name `nexus-creds`.

**Hint:** The condition (image prefix) and the requirement (imagePullSecrets) are in different parts of the spec — use the global anchor `<()`.

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-nexus-pullsecrets
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-nexus-pullsecrets
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Pods using nexus.corp images must define imagePullSecrets: nexus-creds."
        pattern:
          spec:
            containers:
              - name: "*"
                <(image): "nexus.corp/*"
            imagePullSecrets:
              - name: nexus-creds
```

</details>

---

## Cleanup

```bash
kubectl delete clusterpolicy validate-imagepullpolicy restrict-docker-sock \
  disallow-hostpath require-nexus-pullsecrets --ignore-not-found=true
```

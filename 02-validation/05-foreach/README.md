# 02-05 — ForEach Validation

## Concept

`foreach` allows you to iterate over a list of items within a resource and validate each one. This is useful when a resource contains multiple items (containers, ports, rules) that all need to pass a check.

You can use `pattern`, `deny`, or `anyPattern` inside a `foreach` block.

The special variable `element` holds the current item in the loop.

---

## Example 1: Validate All Container Images Come from a Trusted Registry

A Pod can have both `containers` and `initContainers`. Use `foreach` to check all of them with a single rule.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-registry
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-registry
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "All images must come from trust-registry.io"
        foreach:
          # Loop 1: check initContainers
          - list: "request.object.spec.initContainers"  # no {{ }} here
            pattern:
              image: "trust-registry.io/*"
          # Loop 2: check containers
          - list: "request.object.spec.containers"
            pattern:
              image: "trust-registry.io/*"
```

---

## Example 2: Block Wildcard Ingress Hosts

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-ingress-wildcard
spec:
  validationFailureAction: Enforce
  rules:
    - name: block-ingress-wildcard
      match:
        any:
          - resources:
              kinds: [Ingress]
      validate:
        message: "Wildcards are not permitted as hosts."
        foreach:
          - list: "request.object.spec.rules"
            deny:
              conditions:
                any:
                  - key: "{{ contains(element.host, '*') }}"  # element = current item
                    operator: Equals
                    value: true
```

---

## Example 3: ForEach with Preconditions — EmptyDir Storage Limits

Any container that mounts an `emptyDir` volume must define ephemeral storage requests and limits.

```yaml
validate:
  message: "Containers mounting emptyDir volumes must specify ephemeral storage requests and limits."
  foreach:
    - list: "request.object.spec.[initContainers, containers][]"  # merge both lists
      preconditions:   # only check containers that actually use emptyDir
        any:
          - key: "{{ element.volumeMounts[].name }}"
            operator: AnyIn
            value: "{{ emptydirnames }}"  # variable defined in context
      pattern:
        resources:
          requests:
            ephemeral-storage: "?*"
          limits:
            ephemeral-storage: "?*"
```

---

## Example 4: Restrict LoadBalancer Port Range

Only allow ports 22, 80, and 443 for LoadBalancer services.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-service-ports
spec:
  validationFailureAction: Enforce
  rules:
    - name: restrict-port-range
      match:
        any:
          - resources:
              kinds: [Service]
      preconditions:
        all:
          - key: "{{ request.object.spec.type }}"
            operator: Equals
            value: LoadBalancer
      validate:
        message: "Only ports 22, 80, and 443 are allowed for LoadBalancer services."
        foreach:
          - list: "request.object.spec.ports[]"
            deny:
              conditions:
                all:
                  - key: "{{ element.port }}"
                    operator: AnyNotIn
                    value: [22, 80, 443]
```

**What happens with this Service:**

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 80    # allowed
    - port: 3000  # not in [22, 80, 443] -> BLOCKED
```

The precondition (type: LoadBalancer) passes. The foreach iterates over ports. Port 80 is fine. Port 3000 triggers the deny condition — entire Service is rejected.

---

## Test Cases for Registry Check

```bash
kubectl apply -f check-registry.yaml

# FAIL — untrusted registry
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
      image: docker.io/nginx:latest
EOF

# PASS — trusted registry
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
      image: trust-registry.io/nginx:1.25
EOF

kubectl delete pod good-pod --ignore-not-found=true
kubectl delete clusterpolicy check-registry
```

---

## Cleanup

```bash
kubectl delete clusterpolicy check-registry block-ingress-wildcard restrict-service-ports --ignore-not-found=true
```

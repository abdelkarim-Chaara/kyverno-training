# Module 06 — Policy Exceptions

## Concept

`PolicyException` is a Custom Resource that allows you to **bypass specific rules** for specific resources without modifying the original policy. This is useful for:
- Granting temporary exceptions during migrations
- Allowing specific workloads that legitimately need to violate a policy (e.g., a monitoring agent that needs `hostNetwork`)
- Namespace-scoped exceptions for specific teams

## PolicyException vs Exclude Block

| | `exclude` in policy | `PolicyException` CR |
|---|---|---|
| Who manages it | Policy owner | Namespace team / platform team |
| Scope | Policy-wide | Per resource |
| Requires policy edit | Yes | No |
| Namespaced | No (cluster-wide) | Yes (lives in a namespace) |

## Anatomy

```yaml
apiVersion: kyverno.io/v2
kind: PolicyException
metadata:
  name: my-exception
  namespace: my-namespace   # exception lives in a namespace
spec:
  exceptions:
    - policyName: my-policy           # which policy to bypass
      ruleNames:
        - my-rule                     # which rules to bypass (use * for all)
        - autogen-my-rule             # don't forget auto-gen rules!
  match:                              # which resources get the exception
    any:
      - resources:
          kinds: [Pod, Deployment]
          namespaces: [my-namespace]
          names: ["important-tool*"]  # supports wildcards
  conditions:                         # optional extra conditions
    any:
      - key: "{{ request.userInfo.username }}"
        operator: Equals
        value: "admin"
```

---

## Example 1: Exception for a Specific Workload

**Policy:** Block pods that use host namespaces.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-host-namespaces
spec:
  validationFailureAction: Enforce
  rules:
    - name: host-namespaces
      match:
        any:
          - resources:
              kinds: [Pod, Deployment]
      validate:
        message: >-
          Sharing the host namespaces is disallowed.
          spec.hostNetwork, spec.hostIPC, and spec.hostPID must be unset or false.
        pattern:
          spec:
            =(hostPID): "false"
            =(hostIPC): "false"
            =(hostNetwork): "false"
          template:
            spec:
              =(hostPID): "false"
              =(hostIPC): "false"
              =(hostNetwork): "false"
```

**Exception:** Allow Pods/Deployments named `important-tool*` in the `delta` namespace.

```yaml
apiVersion: kyverno.io/v2
kind: PolicyException
metadata:
  name: delta-exception
  namespace: delta
spec:
  exceptions:
    - policyName: disallow-host-namespaces
      ruleNames:
        - host-namespaces
        - autogen-host-namespaces   # auto-gen rule also needs to be listed
  match:
    any:
      - resources:
          kinds: [Pod, Deployment]
          namespaces: [delta]
          names: ["important-tool*"]
```

```bash
kubectl apply -f disallow-host-namespaces.yaml

kubectl create namespace delta
kubectl apply -f delta-exception.yaml
kubectl get polex -n delta
# NAME              AGE
# delta-exception   34s

# PASSES — exception applies
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: important-tool
  namespace: delta
spec:
  replicas: 1
  selector:
    matchLabels:
      app: busybox
  template:
    metadata:
      labels:
        app: busybox
    spec:
      hostIPC: true   # would normally be blocked
      containers:
        - image: busybox:1.35
          name: busybox
          command: ["sleep", "1d"]
EOF

# FAILS — no exception for this name
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: regular-pod
  namespace: delta
spec:
  hostIPC: true
  containers:
    - name: test
      image: busybox:1.35
      command: ["sleep", "1d"]
EOF

kubectl delete namespace delta
kubectl delete clusterpolicy disallow-host-namespaces
```

---

## Example 2: Exception for Pod Security Standards

Bypass specific PSS controls for specific images:

```yaml
apiVersion: kyverno.io/v2
kind: PolicyException
metadata:
  name: pss-exception
  namespace: delta
spec:
  exceptions:
    - policyName: psa
      ruleNames:
        - restricted
  match:
    any:
      - resources:
          namespaces: [delta]
  podSecurity:
    - controlName: "Running as Non root"   # exempt this specific control
    - controlName: Capabilities
      images:
        - nginx*
        - redis*
```

---

## Checking Exceptions

```bash
# List policy exceptions (short alias: polex)
kubectl get polex -A

# Describe a specific exception
kubectl describe polex delta-exception -n delta

# Check if a resource was skipped due to exception
kubectl get policyreport -n delta -o yaml | grep result
# result: skip
```

---

## Important Notes

- PolicyException is **namespaced** — it must be in the same namespace as the resources it exempts, or you must match the namespace explicitly
- Always include **auto-gen rule names** (`autogen-<rule-name>`) when excepting Pods/Deployments
- Use the `conditions` block to add extra guardrails (e.g., only admin users can create excepted resources)
- Kyverno must be configured to allow PolicyExceptions (`--enablePolicyException=true` is default in recent versions)

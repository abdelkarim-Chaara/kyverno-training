# Lab 03 — Generation

## Exercises

### Exercise 1: Generate a NetworkPolicy for New Namespaces
Auto-create a default deny-all ingress NetworkPolicy in every new namespace (except system namespaces).

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-networkpolicy
spec:
  rules:
    - name: default-deny-ingress
      match:
        any:
          - resources:
              kinds: [Namespace]
      exclude:
        any:
          - resources:
              namespaces:
                - kube-system
                - kube-public
                - kube-node-lease
                - kyverno
      generate:
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        name: default-deny-ingress
        namespace: "{{ request.object.metadata.name }}"
        synchronize: true
        data:
          spec:
            podSelector: {}
            policyTypes:
              - Ingress
```

**Test:**
```bash
kubectl create namespace lab-ns
kubectl get networkpolicy -n lab-ns
# default-deny-ingress   <none>   Xm
kubectl delete namespace lab-ns
```

</details>

---

### Exercise 2: Generate a ConfigMap for New Namespaces
Auto-create a ConfigMap named `app-config` with default values in every new namespace.

<details>
<summary>Solution</summary>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-app-config
spec:
  rules:
    - name: generate-app-config
      match:
        any:
          - resources:
              kinds: [Namespace]
      exclude:
        any:
          - resources:
              namespaces: [kube-system, default, kube-public, kyverno]
      generate:
        synchronize: true
        apiVersion: v1
        kind: ConfigMap
        name: app-config
        namespace: "{{ request.object.metadata.name }}"
        data:
          data:
            LOG_LEVEL: "info"
            MAX_CONNECTIONS: "100"
```

</details>

---

## Cleanup

```bash
kubectl delete clusterpolicy add-networkpolicy generate-app-config --ignore-not-found=true
```

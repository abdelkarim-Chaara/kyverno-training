# 04-01 — Generate with Data (Inline)

## Concept

Use `generate.data` to define the resource inline in the policy. The policy itself is the "source" — if you delete the policy, the downstream resources are also deleted (when `synchronize: true`).

---

## Example 1: Auto-generate NetworkPolicy for New Namespaces

Every new namespace gets a default deny-all ingress NetworkPolicy.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-networkpolicy
  annotations:
    policies.kyverno.io/title: Add Default NetworkPolicy
    policies.kyverno.io/category: Multi-Tenancy
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
        namespace: "{{ request.object.metadata.name }}"  # dynamic: uses the new namespace name
        synchronize: true
        data:
          spec:
            podSelector: {}
            policyTypes:
              - Ingress
```

```bash
kubectl apply -f add-networkpolicy.yaml

kubectl create namespace demo-ns

# NetworkPolicy is automatically created!
kubectl get networkpolicy -n demo-ns
# NAME                   POD-SELECTOR   AGE
# default-deny-ingress   <none>         2s

kubectl describe networkpolicy default-deny-ingress -n demo-ns
kubectl get networkpolicy default-deny-ingress -n demo-ns -o yaml

# Cleanup — deleting the namespace also deletes the NetworkPolicy
kubectl delete namespace demo-ns
```

---

## Example 2: Auto-generate ConfigMap for New Namespaces

Generate a ConfigMap with Kafka/ZooKeeper addresses in every new namespace.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-configmap
spec:
  rules:
    - name: generate-configmap
      match:
        any:
          - resources:
              kinds: [Namespace]
      exclude:
        any:
          - resources:
              namespaces:
                - kube-system
                - default
                - kube-public
                - kyverno
      generate:
        synchronize: true
        apiVersion: v1
        kind: ConfigMap
        name: zk-kafka-address
        namespace: "{{ request.object.metadata.name }}"
        data:
          kind: ConfigMap
          metadata:
            labels:
              somekey: somevalue
          data:
            ZK_ADDRESS: "192.168.10.10:2181"
            KAFKA_ADDRESS: "192.168.10.13:9092"
```

```bash
kubectl apply -f generate-configmap.yaml

kubectl create namespace my-app-ns

kubectl get configmap zk-kafka-address -n my-app-ns
kubectl get configmap zk-kafka-address -n my-app-ns -o yaml

kubectl delete namespace my-app-ns
```

---

## generateExisting — Apply to Existing Namespaces

To also generate resources for namespaces that already exist, add `generateExisting: true`:

```yaml
generate:
  generateExisting: true
  synchronize: true
  # ...rest of generate block
```

---

## Cleanup

```bash
kubectl delete clusterpolicy add-networkpolicy generate-configmap --ignore-not-found=true
```

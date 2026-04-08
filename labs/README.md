# Labs — Hands-On Exercises

Practice what you've learned with these guided exercises.

## Prerequisites

All labs require a running Kubernetes cluster with Kyverno installed.

```bash
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=kyverno -n kyverno --timeout=300s
```

---

## Lab Index

| Lab | Topic | Difficulty |
|---|---|---|
| [Lab 01 — Validation](./lab-01-validation/README.md) | Write and test validation rules | Beginner |
| [Lab 02 — Mutation](./lab-02-mutation/README.md) | Write mutation policies | Intermediate |
| [Lab 03 — Generation](./lab-03-generation/README.md) | Auto-generate resources | Intermediate |
| [Lab 04 — Anchors](./lab-04-anchors/README.md) | Conditional, equality, negation, global anchors | Intermediate |
| [Lab 05 — AnyPattern & Deny](./lab-05-anypattern-deny/README.md) | Multi-pattern validation and deny rules | Intermediate |
| [Lab 06 — ForEach](./lab-06-foreach/README.md) | Validate lists: registries, Ingress, ports | Intermediate |
| [Lab 07 — PSS & CEL](./lab-07-pss-cel/README.md) | Pod Security Standards and CEL expressions | Advanced |
| [Lab 08 — Advanced Context](./lab-08-advanced-context/README.md) | ConfigMap context, API calls, dynamic data | Advanced |
| [Lab 09 — Policy Exceptions](./lab-09-policy-exceptions/README.md) | Bypass policies with PolicyException | Advanced |
| [Lab 10 — CLI](./lab-10-cli/README.md) | kyverno apply and kyverno test | Advanced |

---

## Quick Reference — All Demo Commands

```bash
# Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
kubectl get pods -n kyverno

# Validation
kubectl apply -f 02-validation/01-basic/require-app-label.yaml
kubectl run nginx --image=nginx                      # Should FAIL
kubectl run nginx --image=nginx --labels="app=nginx" # Should PASS
kubectl delete pod nginx

# Mutation
kubectl apply -f 03-mutation/01-patch-strategic-merge/add-imagepullpolicy.yaml
kubectl run test --image=nginx:latest --labels="app=test"
kubectl get pod test -o yaml | grep imagePullPolicy  # Should show: Always
kubectl delete pod test

# Policy Reports
kubectl get clusterpolicyreport -A
kubectl get policyreport -A
kubectl get events --field-selector reason=PolicyViolation

# Policy Details
kubectl get clusterpolicy
kubectl describe clusterpolicy require-app-label
kubectl get clusterpolicy require-app-label -o jsonpath='{.status}' | jq

# Generation
kubectl apply -f 04-generation/01-data/add-networkpolicy.yaml
kubectl create namespace demo-ns
kubectl get networkpolicy -n demo-ns
kubectl delete namespace demo-ns

# Cleanup
kubectl delete clusterpolicy --all
helm uninstall kyverno -n kyverno
kubectl delete namespace kyverno
```

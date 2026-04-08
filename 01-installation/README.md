# Module 01 — Installation

## Install Kyverno with Helm (Recommended)

```bash
# 1. Add the Kyverno Helm repository
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

# 2. Install Kyverno (standalone — single replica, good for learning)
helm install kyverno kyverno/kyverno \
  -n kyverno \
  --create-namespace

# 3. Install Kyverno (HA — 3 replicas, for production)
helm install kyverno kyverno/kyverno \
  -n kyverno \
  --create-namespace \
  --set admissionController.replicas=3 \
  --set backgroundController.replicas=2 \
  --set cleanupController.replicas=2 \
  --set reportsController.replicas=2

# 4. Wait for pods to be ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=kyverno \
  -n kyverno \
  --timeout=300s

# 5. Verify installation
kubectl get pods -n kyverno
```

**Expected output (standalone):**
```
NAME                                            READY   STATUS    RESTARTS   AGE
kyverno-admission-controller-xxx-xxx            1/1     Running   0          2m
kyverno-background-controller-xxx-xxx           1/1     Running   0          2m
kyverno-cleanup-controller-xxx-xxx              1/1     Running   0          2m
kyverno-reports-controller-xxx-xxx              1/1     Running   0          2m
```

---

## Verify Webhook Registration

```bash
# Check that Kyverno registered its webhooks
kubectl get validatingwebhookconfigurations | grep kyverno
kubectl get mutatingwebhookconfigurations | grep kyverno
```

---

## Install Kyverno CLI (Optional but Recommended)

The Kyverno CLI lets you test policies locally without a cluster.

```bash
# macOS (Homebrew)
brew install kyverno

# Linux (direct binary)
curl -LO https://github.com/kyverno/kyverno/releases/latest/download/kyverno-cli_linux_x86_64.tar.gz
tar -xvf kyverno-cli_linux_x86_64.tar.gz
chmod +x kyverno
sudo mv kyverno /usr/local/bin/

# Verify
kyverno version
```

---

## Install the Kyverno Policy Library (Optional)

The official policy library contains 200+ production-ready policies.

```bash
# Install Pod Security Standards (Baseline — less restrictive)
helm install kyverno-policies kyverno/kyverno-policies \
  -n kyverno \
  --set podSecurityStandard=baseline \
  --set validationFailureAction=Audit

# Install Pod Security Standards (Restricted — more restrictive)
helm install kyverno-policies kyverno/kyverno-policies \
  -n kyverno \
  --set podSecurityStandard=restricted \
  --set validationFailureAction=Audit
```

Browse the full library: https://kyverno.io/policies

---

## Verify Your Environment

```bash
kubectl cluster-info
kubectl get nodes
kubectl config current-context
helm version
kubectl version --client
kubectl get clusterpolicy   # Should be empty initially
```

---

## Uninstall

```bash
# Remove all policies first
kubectl delete clusterpolicy --all

# Uninstall Kyverno
helm uninstall kyverno -n kyverno
kubectl delete namespace kyverno

# Verify cleanup
kubectl get validatingwebhookconfigurations | grep kyverno
kubectl get mutatingwebhookconfigurations | grep kyverno
```

---

## Next Steps

→ [Module 02 — Validation Policies](../02-validation/README.md)

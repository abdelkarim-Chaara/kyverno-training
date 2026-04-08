# Kyverno Training — Complete Hands-On Guide

A structured, hands-on training repository for learning Kyverno — the Kubernetes-native policy engine.

**Author:** Abdelkarim Chaara | Site Reliability Engineering

---

## What is Kyverno?

Kyverno (Greek for "govern") is a policy engine designed specifically for Kubernetes. It works as an **Admission Controller** — intercepting every request to the Kubernetes API and applying rules to **validate**, **mutate**, **generate**, and **clean up** resources.

Unlike OPA/Gatekeeper which uses Rego, Kyverno policies are plain **Kubernetes YAML** — no new language to learn.

---

## How Kyverno Works (Admission Flow)

```
kubectl apply → API Server → [Kyverno Webhook] → etcd
                                    │
                        ┌───────────┴────────────┐
                        │                        │
                   MutatingWebhook       ValidatingWebhook
                   (mutate rules)        (validate rules)
```

Every resource request is wrapped in an **AdmissionReview** object and sent to Kyverno. Kyverno evaluates your policies and responds with allow/deny/mutate.

---

## Training Modules

| Module | Topic | Difficulty |
|--------|-------|------------|
| [00 - Introduction](./00-introduction/README.md) | Architecture & Concepts | Beginner |
| [01 - Installation](./01-installation/README.md) | Install Kyverno with Helm | Beginner |
| [02 - Validation](./02-validation/README.md) | Block non-compliant resources | Beginner → Advanced |
| [03 - Mutation](./03-mutation/README.md) | Auto-modify resources at admission | Intermediate |
| [04 - Generation](./04-generation/README.md) | Auto-create resources | Intermediate |
| [05 - Advanced Context](./05-advanced/README.md) | ConfigMaps, API calls, image registry | Advanced |
| [06 - Policy Exceptions](./06-policy-exceptions/README.md) | Bypass specific rules | Intermediate |
| [07 - Cleanup Policies](./07-cleanup-policies/README.md) | Scheduled resource deletion | Intermediate |
| [08 - Reporting](./08-reporting/README.md) | PolicyReports & compliance | Intermediate |
| [09 - Kyverno CLI](./09-cli/README.md) | Test policies locally | Intermediate |
| [10 - Enforcement Strategies](./10-enforcement-strategies/README.md) | Audit → Warn → Enforce | Intermediate |
| [Labs](./labs/) | Hands-on exercises | All levels |

---

## Prerequisites

- Kubernetes cluster (v1.24+) — minikube, kind, k3d, or cloud
- `kubectl` configured
- `helm` v3+
- Optional: `kyverno` CLI for local testing

---

## Quick Start

```bash
# 1. Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace

# 2. Verify
kubectl get pods -n kyverno

# 3. Apply your first policy (require a label on all pods)
kubectl apply -f 02-validation/01-basic/require-app-label.yaml

# 4. Test it
kubectl run nginx --image=nginx              # Should FAIL
kubectl run nginx --image=nginx --labels="app=nginx"  # Should PASS
```

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| `ClusterPolicy` | Cluster-wide policy |
| `Policy` | Namespace-scoped policy |
| `validationFailureAction` | `Enforce` (block) or `Audit` (log only) |
| `background` | Enable periodic scanning of existing resources |
| `match` | Which resources the rule applies to |
| `exclude` | Resources to skip |
| `preconditions` | Conditional rule execution |

---

## Policy Rule Types

```
ClusterPolicy
└── spec
    └── rules[]
        ├── validate   → Block or warn on non-compliant resources
        ├── mutate     → Modify resources before they are stored
        ├── generate   → Create new resources as a side-effect
        └── verifyImages → Verify image signatures
```

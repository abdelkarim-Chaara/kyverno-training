# 05-05 — Image Verification (verifyImages)

## Concept

`verifyImages` rules ensure that container images have been **cryptographically signed** by a trusted authority. This prevents unauthorized or tampered images from running in your cluster.

The signing process:
1. Image owner signs the image using a **private key** (generates a signature/attestation pushed to the registry)
2. Kyverno uses the corresponding **public key** to verify the signature at admission time

Supported signing tools:
- **Cosign** (by Sigstore) — keyless or key-based signing
- **Notary v2** — enterprise signing

---

## How Cosign Works

```
Developer                Registry              Kyverno
    │                       │                     │
    ├── Build image ────────►│                     │
    ├── Sign with cosign ───►│ (stores signature)  │
    │                       │                     │
User/CI applies Pod ────────────────────────────►│
                                                  ├── Fetch image
                                                  ├── Fetch signature from registry
                                                  ├── Verify with public key
                                                  └── Allow or Deny
```

---

## Example 1: Verify Image Signature (Key-Based)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - imageReferences:
            - "ghcr.io/my-org/*"   # apply to these images
          attestors:
            - count: 1
              entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
                      -----END PUBLIC KEY-----
```

---

## Example 2: Keyless Signing (Sigstore/Fulcio)

Keyless signing uses the signer's OIDC identity (GitHub Actions, Google, etc.) instead of a static key.

```yaml
verifyImages:
  - imageReferences:
      - "ghcr.io/my-org/*"
    attestors:
      - count: 1
        entries:
          - keyless:
              subject: "https://github.com/my-org/my-repo/.github/workflows/build.yaml@refs/heads/main"
              issuer: "https://token.actions.githubusercontent.com"
              rekor:
                url: https://rekor.sigstore.dev
```

---

## Example 3: Verify with Attestations (SBOM, Vulnerability Scan)

You can also verify attached attestations (Software Bill of Materials, scan results):

```yaml
verifyImages:
  - imageReferences:
      - "ghcr.io/my-org/*"
    attestors:
      - entries:
          - keys:
              publicKeys: |-
                <public key>
    attestations:
      - type: https://spdx.dev/Document   # SBOM type
        attestors:
          - entries:
              - keys:
                  publicKeys: |-
                    <public key>
        conditions:
          - all:
              - key: "{{ attestation.statement.predicate.packages | length(@) }}"
                operator: GreaterThan
                value: 0
```

---

## Sign an Image with Cosign (Example Workflow)

```bash
# Install cosign
brew install cosign

# Generate a key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key ghcr.io/my-org/my-app:v1.0.0

# Verify manually
cosign verify --key cosign.pub ghcr.io/my-org/my-app:v1.0.0
```

---

## Important Notes

- `verifyImages` rules always run in Enforce mode for matched images
- Kyverno mutates the Pod to replace the image tag with the digest after verification (immutable reference)
- For private registries, configure image pull secrets in the Kyverno namespace

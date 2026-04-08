# 05-04 — Image Registry Context

## Concept

The `imageRegistry` context type allows Kyverno to fetch **image metadata** (manifest and config) from an OCI registry at admission time. This lets you make policy decisions based on image properties like:

- Who the image runs as (User field in config)
- Image labels and annotations
- Image creation timestamp
- Image digest

## Syntax

```yaml
context:
  - name: imageData
    imageRegistry:
      reference: "{{ element.image }}"  # or a static image ref
```

After fetching, two variables are available:
- `imageData.manifest` — the OCI image manifest
- `imageData.configData` — the image configuration (entrypoint, user, labels, etc.)

---

## Example: Block Images That Run as Root

Ubuntu and many base images run as root by default (empty User field = root).

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: no-root-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: no-root-images
      match:
        any:
          - resources:
              kinds: [Pod]
      validate:
        message: "Images that run as root are not allowed."
        foreach:
          - list: "request.object.spec.containers"
            context:
              - name: imageData
                imageRegistry:
                  reference: "{{ element.image }}"
            deny:
              conditions:
                any:
                  - key: "{{ imageData.configData.config.User || '' }}"
                    operator: Equals
                    value: ""   # empty string = root
```

---

## Available Image Data Fields

```
imageData.manifest.schemaVersion
imageData.manifest.mediaType
imageData.manifest.layers[]
imageData.configData.config.User         ← image runtime user
imageData.configData.config.Entrypoint[]
imageData.configData.config.Cmd[]
imageData.configData.config.Labels       ← image labels
imageData.configData.created             ← image creation timestamp
imageData.configData.os
imageData.configData.architecture
```

---

## Cleanup

```bash
kubectl delete clusterpolicy no-root-images --ignore-not-found=true
```

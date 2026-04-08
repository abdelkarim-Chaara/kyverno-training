# Module 05 — Advanced Context & External Data

Rules can fetch data from external sources at evaluation time using a `context` block. This enables dynamic policies that adapt based on cluster state.

## Context Types

| Type | Description |
|---|---|
| `configMap` | Fetch data from a ConfigMap in the cluster |
| `apiCall` | Call the Kubernetes API or an external HTTPS endpoint |
| `globalReference` | Reference a pre-cached GlobalContextEntry |
| `imageRegistry` | Fetch image metadata from an OCI registry |

## Basic Syntax

```yaml
rules:
  - name: my-rule
    context:
      - name: myVar          # variable name to use in expressions
        configMap:
          name: my-config
          namespace: default
    match:
      any:
        - resources:
            kinds: [Pod]
    mutate:         # or validate:
      patchStrategicMerge:
        metadata:
          labels:
            region: "{{ myVar.data.region }}"   # Access context: {{ myVar.data.key }}
```

## Sub-modules

| Sub-module | Topic |
|---|---|
| [01 - ConfigMap Context](./01-configmap-context/README.md) | Read data from ConfigMaps |
| [02 - API Call Context](./02-api-call/README.md) | Call the Kubernetes API dynamically |
| [03 - Global Context](./03-global-context/README.md) | Cache data for performance |
| [04 - Image Registry](./04-image-registry/README.md) | Inspect image metadata |
| [05 - Image Verify](./05-image-verify/README.md) | Verify image signatures with Cosign |

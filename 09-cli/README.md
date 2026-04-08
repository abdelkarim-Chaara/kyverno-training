# Module 09 — Kyverno CLI

## Concept

The Kyverno CLI is a standalone command-line tool for working with policies **outside of a live cluster**. It enables:

- **Shift-left testing** — validate policies while writing resources, not after deploying
- **CI/CD integration** — gate pipelines on policy compliance
- **Local preview** — see what a mutation or generation policy would produce
- **No cluster needed** — everything runs locally

## Installation

```bash
# macOS
brew install kyverno

# Linux
curl -LO https://github.com/kyverno/kyverno/releases/latest/download/kyverno-cli_linux_x86_64.tar.gz
tar -xvf kyverno-cli_linux_x86_64.tar.gz
chmod +x kyverno
sudo mv kyverno /usr/local/bin/

kyverno version
```

## Core Commands

| Command | Description |
|---|---|
| `kyverno apply <policy> --resource <resource>` | Apply policy to resource(s), show results |
| `kyverno test <directory>` | Run declarative test suite from a directory |

## Sub-modules

| Sub-module | Topic |
|---|---|
| [01 - apply command](./01-apply/README.md) | Test policies against manifests |
| [02 - test command](./02-test/README.md) | Declarative unit tests for policies |

## Key Flags for `apply`

| Flag | Description |
|---|---|
| `--resource` / `-r` | Path to resource manifest(s) |
| `--policy-report` / `-p` | Show output in PolicyReport format |
| `--exception` / `-e` | Apply a PolicyException file |
| `--values-file` / `-f` | Supply external data (replaces ConfigMaps/APIcalls) |
| `--set` / `-s` | Supply a single value inline |
| `--cluster` | Audit against live cluster resources (requires kubeconfig) |
| `--output` / `-o` | Output file for patched/generated manifests |

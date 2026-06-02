# hypershift-cluster-setup command

Manages the full lifecycle of a self-managed Azure HyperShift hosted cluster: Azure workload identity setup, hosted cluster creation, worker node provisioning, and validation. Designed for testing OpenShift Sandboxed Containers (OSC) DaemonSet mode on HCP clusters.

## Invocation

```
/hypershift-cluster-setup [setup [--cluster-name <NAME>] [--location <REGION>] [--node-count <N>] | validate | teardown [--cluster-name <NAME>]]
```

Run with no arguments for an interactive prompt.

## Subcommands

| Subcommand | What it does |
|---|---|
| `setup` | Creates Azure OIDC issuer, managed identities, and hosted cluster with worker nodes |
| `validate` | Verifies worker nodes joined, checks MCO/MachineConfig CRDs are absent, tests cluster readiness for OSC DaemonSet |
| `teardown` | Deletes hosted cluster resources, Azure VMs, managed identities, and OIDC infrastructure |

## Flags

| Flag | Applies to | Effect |
|---|---|---|
| `--cluster-name <name>` | `setup`, `teardown`, `validate` | Hosted cluster name (default: `hcp-<timestamp>`) |
| `--location <region>` | `setup` | Azure region (default: `eastus`) |
| `--node-count <n>` | `setup` | Number of worker nodes (default: `2`) |
| `--release-image <image>` | `setup` | OCP release image (default: `quay.io/openshift-release-dev/ocp-release:4.21.5-x86_64`) |

## Prerequisites

- Management cluster with MCE + HyperShift installed (OCP 4.21+)
- Azure credentials configured (`az login`)
- `ccoctl-native` (ARM64 macOS) or `ccoctl` (Linux amd64)
- `hypershift` CLI (ARM64 macOS) or from MCE operator image
- `oc`, `kubectl`, `az`, `jq` installed
- Azure DNS zone for base domain
- Pull secret from Red Hat

## Expected environment

```
Management cluster:
- KUBECONFIG pointing to management cluster
- MCE 2.11+ with HyperShift operator running
- CRDs: hostedclusters.hypershift.openshift.io, nodepools.hypershift.openshift.io

Azure:
- Subscription with required quotas (compute, networking)
- DNS zone for base domain (e.g., azure.sandboxedcontainers.com)
- EncryptionAtHost feature enabled

Binaries:
~/Work/azure-hcp/bin/
├── ccoctl-native    ← ARM64 macOS or use ccoctl on Linux
└── hypershift       ← ARM64 macOS or extract from operator image
```

## Defaults

| Parameter | Default |
|---|---|
| Location | `eastus` |
| Node count | `2` |
| VM size | `Standard_D4s_v5` |
| Release | OCP 4.21.5 |
| Base domain | `azure.sandboxedcontainers.com` |

## Known issues

### Federated credential audience must be "openshift"
OpenShift service account tokens use `"openshift"` audience, not the Azure standard `"api://AzureADTokenExchange"`. The setup step creates federated credentials with the correct audience.

### Missing capi-provider federated credential
The `capi-provider` service account (used by Cluster API to provision Azure VMs) is not in standard CredentialsRequest manifests. Setup adds it manually to the nodePoolManagement managed identity.

### EncryptionAtHost feature must be enabled
Azure subscription requires `Microsoft.Compute/EncryptionAtHost` feature. Setup checks and enables it automatically:
```bash
az feature register --namespace Microsoft.Compute --name EncryptionAtHost
az provider register -n Microsoft.Compute
```

### Worker nodes stuck pending with "no such file or directory" errors
The cloud-token-minter sidecar may time out connecting to the hosted kube-apiserver during initial startup. This usually resolves within 5-10 minutes as the control plane stabilizes.

### Etcd performance issues in long-running clusters
Hosted clusters experiencing etcd disk I/O issues (requests >100ms) or control plane pod restart loops (etcd, kube-controller-manager) should be recreated. Bootstrap deadlock can occur when workers can't join due to apiserver instability.

### Control plane says "Waiting for etcd to reach quorum"
This usually indicates worker nodes haven't joined. Check:
- Machines are in Phase=Provisioned with provider IDs
- Azure VMs exist and are running
- cloud-token-minter logs show successful token creation
- No authentication errors in capi-provider logs

## Install

```bash
cp hypershift-cluster-setup/hypershift-cluster-setup.md ~/.claude/commands/hypershift-cluster-setup.md
```

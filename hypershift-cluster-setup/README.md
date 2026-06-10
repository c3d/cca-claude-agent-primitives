# hypershift-cluster-setup command

Complete Azure HyperShift setup from management cluster creation to OSC installation. Automates IPI OpenShift installation, MCE/HyperShift deployment, Azure workload identity configuration, hosted cluster provisioning, and OpenShift Sandboxed Containers (OSC) DaemonSet installation and validation on Azure HCP.

## Invocation

```
/hypershift-cluster-setup management <create|setup|teardown> [--name <NAME>]
/hypershift-cluster-setup hosted <setup|validate|teardown> [--cluster-name <NAME>] [--location <REGION>] [--node-count <N>]
/hypershift-cluster-setup osc <install|validate|reboot> [--cluster-name <NAME>]
```

Run with no arguments for an interactive prompt.

## Management Cluster Operations

| Command | What it does |
|---|---|
| `management create` | Creates IPI OpenShift 4.21+ cluster on Azure using openshift-install (3 control plane + 3 worker nodes) |
| `management setup` | Installs MCE 2.11+ and HyperShift operator, configures Azure prerequisites (EncryptionAtHost), verifies binaries (ccoctl, hypershift CLI) |
| `management teardown` | Destroys management cluster and all Azure resources using openshift-install |

## Hosted Cluster Operations

| Command | What it does |
|---|---|
| `hosted setup` | Creates Azure OIDC issuer, managed identities with federated credentials, and hosted cluster with worker nodes |
| `hosted validate` | Verifies worker nodes joined, checks MCO is non-functional (required for DaemonSet mode), tests cluster readiness |
| `hosted teardown` | Deletes hosted cluster resources, Azure VMs, managed identities, and OIDC infrastructure |

## OSC Operations

| Command | What it does |
|---|---|
| `osc install` | Installs OSC operator, configures DaemonSet mode, creates KataConfig, labels worker nodes for kata |
| `osc validate` | Verifies kata runtime installed on nodes, deploys test kata pod to validate functionality |
| `osc reboot` | Sequentially cordons, drains, and reboots worker nodes (required after kata installation to load runtime) |

## Flags

| Flag | Applies to | Effect |
|---|---|---|
| `--name <name>` | `management create`, `management teardown` | Management cluster name (default: `$USER-hcp-host-$VERSION` extracted from release image) |
| `--cluster-name <name>` | `hosted *`, `osc *` | Hosted cluster name (default: `$USER-hcp-YYYYMMDD`) |
| `--location <region>` | `management create`, `hosted setup` | Azure region (default: `eastus`) |
| `--node-count <n>` | `hosted setup` | Number of worker nodes for hosted cluster (default: `2`) |
| `--release-image <image>` | `hosted setup` | OCP release image (default: `quay.io/openshift-release-dev/ocp-release:4.21.5-x86_64`) |

## Prerequisites

### For Management Cluster Creation
- Azure subscription with sufficient quotas (compute, networking, storage)
- Azure DNS zone for base domain (e.g., `azure.sandboxedcontainers.com` in resource group `osc-clusters`)
- `openshift-install` binary from [mirror.openshift.com](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.21/)
- `az` CLI with active login (`az login`)
- `oc`, `kubectl`, `jq` installed
- Pull secret from [Red Hat](https://console.redhat.com/openshift/install/pull-secret)
- SSH public key (~/.ssh/id_rsa.pub)

### For Hosted Cluster Creation  
- Running management cluster with MCE + HyperShift (created via `management create` + `management install-mce`)
- `ccoctl-native` (ARM64 macOS) or `ccoctl` (Linux amd64) - built from source or downloaded
- `hypershift` CLI (ARM64 macOS) - built from source
- EncryptionAtHost feature enabled on Azure subscription

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

### MCO CRDs present but non-functional
Azure HCP hosted clusters have MachineConfig CRDs registered in the API, but the MCO operator is not functional:
- `kubectl get machineconfigpools` returns "server doesn't have a resource type"
- `kubectl get machineconfigs` returns "server doesn't have a resource type"  
- No machine-config-daemon DaemonSet running on workers
- No MCO controller pods managing the cluster

**Impact:** OSC DaemonSet mode is **required** (MachineConfig mode will not work). The validation step checks for functional MCO by testing MachineConfigPool availability, not just CRD presence.

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

## Complete Workflow Examples

### From Zero to Hosted Cluster

```bash
# 1. Create management cluster (IPI OpenShift on Azure)
/hypershift-cluster-setup management create --name my-mgmt --location eastus
# Takes ~40 minutes

# 2. Setup management cluster (MCE + HyperShift + prerequisites)
/hypershift-cluster-setup management setup
# Takes ~5-10 minutes

# 3. Create hosted cluster
/hypershift-cluster-setup hosted setup --cluster-name my-hcp --node-count 2
# Takes ~15-20 minutes (control plane) + ~10 minutes (workers)

# 4. Validate hosted cluster
/hypershift-cluster-setup hosted validate --cluster-name my-hcp

# 5. Use the hosted cluster
export KUBECONFIG=~/Work/azure-hcp/my-hcp-kubeconfig
oc get nodes
```

### Complete End-to-End with OSC

```bash
# 1-4. Same as above (create management + hosted cluster)
/hypershift-cluster-setup management create --name my-mgmt
/hypershift-cluster-setup management setup
/hypershift-cluster-setup hosted setup --cluster-name my-hcp
/hypershift-cluster-setup hosted validate --cluster-name my-hcp

# 5. Install OSC operator and configure DaemonSet mode
/hypershift-cluster-setup osc install --cluster-name my-hcp
# Takes ~3-5 minutes

# 6. Reboot worker nodes (required to load kata runtime)
/hypershift-cluster-setup osc reboot --cluster-name my-hcp
# Takes ~5-10 minutes per node (sequential)

# 7. Validate kata runtime installation
/hypershift-cluster-setup osc validate --cluster-name my-hcp

# 8. Deploy kata workload
export KUBECONFIG=~/Work/azure-hcp/my-hcp-kubeconfig
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: my-kata-pod
spec:
  runtimeClassName: kata
  containers:
  - name: nginx
    image: nginx:latest
EOF

# Check kata pod is running
oc get pod my-kata-pod -o wide
```

### Cleanup

```bash
# Delete hosted cluster (keeps management cluster)
/hypershift-cluster-setup hosted teardown --cluster-name my-hcp

# Delete management cluster (destroys everything)
/hypershift-cluster-setup management teardown --name my-mgmt
```

## Install

```bash
ln -s ~/Work/claude-agent-primitives/hypershift-cluster-setup/hypershift-cluster-setup.md ~/.claude/commands/hypershift-cluster-setup.md
```

---
name: hypershift-cluster-setup
description: Set up self-managed Azure HyperShift hosted cluster — creates OIDC issuer, managed identities, and hosted cluster with workers. Validates cluster readiness for OSC DaemonSet testing. Supports teardown. Azure credentials must be configured via az login.
argument-hint: "setup [--cluster-name <NAME>] [--location <REGION>] [--node-count <N>] | validate | teardown [--cluster-name <NAME>]"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Automate the full lifecycle of a self-managed Azure HyperShift hosted cluster for testing OpenShift Sandboxed Containers (OSC) DaemonSet mode:

- **setup** — create Azure OIDC issuer, managed identities with workload identity federation, and hosted cluster with worker nodes
- **validate** — verify workers joined, confirm MCO/MachineConfig CRDs are absent (required for DaemonSet mode), test cluster readiness
- **teardown** — delete hosted cluster, Azure VMs, managed identities, and OIDC infrastructure

Azure credentials are assumed to be configured via `az login`. Management cluster KUBECONFIG must be set.

Default values (all overridable via flags):
- Location: `eastus`
- Node count: `2`
- VM size: `Standard_D4s_v5`
- Release: OCP 4.21.5
</objective>

<environment>
```
Management cluster (KUBECONFIG):
- OCP 4.21+ with MCE 2.11+ installed
- HyperShift operator running
- CRDs: hostedclusters.hypershift.openshift.io, nodepools.hypershift.openshift.io

Azure:
- Subscription with EncryptionAtHost feature enabled
- DNS zone for base domain in resource group (e.g., azure.sandboxedcontainers.com in osc-clusters RG)
- Quotas: VMs, VNets, NSGs, load balancers

Binaries ($PATH or ~/Work/azure-hcp/bin):
- ccoctl-native (ARM64 macOS) or ccoctl (Linux amd64)
- hypershift (ARM64 macOS) or extracted from MCE operator
- oc, kubectl, az, jq

Files:
- Pull secret: ~/Work/notes/openshift-pull-secret.txt
```
</environment>

<process>

## 0. Parse Arguments

Parse `$ARGUMENTS`:
- First positional arg: subcommand (`setup`, `validate`, `teardown`)
- `--cluster-name <name>` — hosted cluster name (default: `hcp-$(date +%s)`)
- `--location <region>` — Azure region (default: `eastus`)
- `--node-count <n>` — worker nodes (default: `2`)
- `--release-image <image>` — OCP release (default: `quay.io/openshift-release-dev/ocp-release:4.21.5-x86_64`)

If no subcommand given, ask:
```
Which phase do you want to run?
  1) setup    — create OIDC, identities, and hosted cluster
  2) validate — check worker nodes and cluster readiness
  3) teardown — delete hosted cluster and Azure resources
```

Set defaults:
```bash
CLUSTER_NAME="${CLUSTER_NAME:-hcp-$(date +%s)}"
LOCATION="${LOCATION:-eastus}"
NODE_COUNT="${NODE_COUNT:-2}"
RELEASE_IMAGE="${RELEASE_IMAGE:-quay.io/openshift-release-dev/ocp-release:4.21.5-x86_64}"
BASE_DOMAIN="azure.sandboxedcontainers.com"
DNS_ZONE_RG="osc-clusters"
NAMESPACE="clusters"
PULL_SECRET="${PULL_SECRET:-$HOME/Work/notes/openshift-pull-secret.txt}"

# Derived names
OIDC_RG="${CLUSTER_NAME}-oidc"
INSTALL_RG="${CLUSTER_NAME}-rg"
STORAGE_ACCOUNT=$(echo "${CLUSTER_NAME}oidc" | tr -d '-' | cut -c1-24)  # Azure storage names: 3-24 chars, lowercase/digits only
OIDC_ISSUER_URL="https://${STORAGE_ACCOUNT}.blob.core.windows.net/${CLUSTER_NAME}"
```

---

## 1. Preflight Check (all subcommands)

```bash
# Management cluster
KUBECONFIG="${KUBECONFIG:-$HOME/.kube/config}"
oc cluster-info >/dev/null 2>&1 || { echo "BLOCK: Management cluster not accessible (KUBECONFIG=$KUBECONFIG)"; exit 1; }

# Check MCE/HyperShift
oc get crd hostedclusters.hypershift.openshift.io >/dev/null 2>&1 || {
    echo "BLOCK: HyperShift CRDs not found. Install MCE first:"
    echo "  oc create namespace multicluster-engine"
    echo "  # Create OperatorGroup and Subscription for multicluster-engine"
    exit 1
}

# Azure credentials
SUBSCRIPTION_ID=$(az account show --query id -o tsv 2>/dev/null)
TENANT_ID=$(az account show --query tenantId -o tsv 2>/dev/null)
[[ -n "$SUBSCRIPTION_ID" ]] || { echo "BLOCK: Azure credentials not found. Run: az login"; exit 1; }
echo "Using subscription: $SUBSCRIPTION_ID"

# Required tools
for tool in oc kubectl az jq; do
    command -v "$tool" &>/dev/null && echo "  $tool: ok" || { echo "BLOCK: $tool not found"; exit 1; }
done

# ccoctl and hypershift (either in PATH or ~/Work/azure-hcp/bin)
export PATH="$HOME/Work/azure-hcp/bin:$PATH"
CCOCTL=$(command -v ccoctl-native 2>/dev/null || command -v ccoctl 2>/dev/null)
HYPERSHIFT=$(command -v hypershift 2>/dev/null)
[[ -n "$CCOCTL" ]] || { echo "BLOCK: ccoctl-native or ccoctl not found"; exit 1; }
[[ -n "$HYPERSHIFT" ]] || { echo "BLOCK: hypershift CLI not found"; exit 1; }
echo "  ccoctl: $CCOCTL"
echo "  hypershift: $HYPERSHIFT"

# Pull secret
[[ -f "$PULL_SECRET" ]] || { echo "BLOCK: Pull secret not found at $PULL_SECRET"; exit 1; }

# EncryptionAtHost feature (check only; setup will enable if needed)
ENCR_STATE=$(az feature show --namespace Microsoft.Compute --name EncryptionAtHost --query properties.state -o tsv 2>/dev/null || echo "NotRegistered")
[[ "$ENCR_STATE" == "Registered" ]] || echo "WARN: EncryptionAtHost feature not registered (will enable during setup)"
```

---

## 2. Setup Phase

### 2.1. Create OIDC Key Pair

```bash
mkdir -p ~/Work/azure-hcp/oidc
cd ~/Work/azure-hcp

$CCOCTL azure create-key-pair --output-dir ./oidc
# Creates:
#   oidc/serviceaccount-signer.private
#   oidc/serviceaccount-signer.public
```

### 2.2. Create OIDC Issuer

```bash
$CCOCTL azure create-oidc-issuer \
  --name ${CLUSTER_NAME} \
  --region ${LOCATION} \
  --subscription-id ${SUBSCRIPTION_ID} \
  --tenant-id ${TENANT_ID} \
  --oidc-resource-group-name ${OIDC_RG} \
  --storage-account-name ${STORAGE_ACCOUNT} \
  --public-key-file ./oidc/serviceaccount-signer.public \
  --output-dir ./oidc

# Creates:
#   - Resource group: $OIDC_RG
#   - Storage account: $STORAGE_ACCOUNT
#   - Blob container: $CLUSTER_NAME
#   - OIDC discovery docs uploaded to blob storage
#   - Issuer URL: $OIDC_ISSUER_URL
```

### 2.3. Extract Credential Requests and Create Managed Identities

```bash
# Extract CredentialsRequests from release image
mkdir -p credreqs
RELEASE_IMAGE_FOR_EXTRACT=$(oc get clusterversion -o jsonpath='{.items[0].status.desired.image}')
oc adm release extract --credentials-requests --cloud=azure --to=./credreqs "${RELEASE_IMAGE_FOR_EXTRACT}"

# Create managed identities
$CCOCTL azure create-managed-identities \
  --name ${CLUSTER_NAME} \
  --region ${LOCATION} \
  --subscription-id ${SUBSCRIPTION_ID} \
  --oidc-resource-group-name ${OIDC_RG} \
  --installation-resource-group-name ${INSTALL_RG} \
  --issuer-url ${OIDC_ISSUER_URL} \
  --output-dir ./oidc \
  --credentials-requests-dir ./credreqs

# Creates:
#   - Resource group: $INSTALL_RG (for cluster resources)
#   - 7 managed identities in $OIDC_RG with federated credentials
```

### 2.4. Add Missing Federated Credential for capi-provider

**Critical:** The `capi-provider` service account is not in standard CredentialsRequest manifests.

```bash
# Get the nodePoolManagement managed identity name
IDENTITY_NAME="${CLUSTER_NAME}-openshift-machine-api-azure-cloud-credentials"

# Create federated credential with CORRECT audience ("openshift", not "api://AzureADTokenExchange")
az identity federated-credential create \
  --name capi-provider \
  --identity-name ${IDENTITY_NAME} \
  --resource-group ${OIDC_RG} \
  --issuer ${OIDC_ISSUER_URL} \
  --subject "system:serviceaccount:kube-system:capi-provider" \
  --audiences openshift

echo "✓ Added capi-provider federated credential with audience=openshift"
```

### 2.5. Enable EncryptionAtHost Feature (if not already enabled)

```bash
if [[ "$ENCR_STATE" != "Registered" ]]; then
    echo "Enabling EncryptionAtHost feature..."
    az feature register --namespace Microsoft.Compute --name EncryptionAtHost
    az provider register -n Microsoft.Compute
    echo "✓ EncryptionAtHost feature enabled"
fi
```

### 2.6. Create Workload Identities Mapping File

```bash
# Query managed identity client IDs
cat > workload-identities.json <<EOF
{
  "cloudProvider": {
    "clientId": "$(az identity show -n ${CLUSTER_NAME}-openshift-cloud-controller-manager-azure-cloud-credentials -g ${OIDC_RG} --query clientId -o tsv)"
  },
  "nodePoolManagement": {
    "clientId": "$(az identity show -n ${CLUSTER_NAME}-openshift-machine-api-azure-cloud-credentials -g ${OIDC_RG} --query clientId -o tsv)"
  },
  "ingress": {
    "clientId": "$(az identity show -n ${CLUSTER_NAME}-openshift-ingress-operator-cloud-credentials -g ${OIDC_RG} --query clientId -o tsv)"
  },
  "disk": {
    "clientId": "$(az identity show -n ${CLUSTER_NAME}-openshift-cluster-csi-drivers-azure-disk-credentials -g ${OIDC_RG} --query clientId -o tsv)"
  },
  "file": {
    "clientId": "$(az identity show -n ${CLUSTER_NAME}-openshift-cluster-csi-drivers-azure-file-credentials -g ${OIDC_RG} --query clientId -o tsv)"
  },
  "network": {
    "clientId": "$(az identity show -n ${CLUSTER_NAME}-openshift-cloud-network-config-controller-cloud-credentials -g ${OIDC_RG} --query clientId -o tsv)"
  },
  "imageRegistry": {
    "clientId": "$(az identity show -n ${CLUSTER_NAME}-openshift-image-registry-installer-cloud-credentials -g ${OIDC_RG} --query clientId -o tsv)"
  }
}
EOF
```

### 2.7. Create Azure Credentials File

```bash
cat > azure-creds.json <<EOF
{
  "subscriptionId": "${SUBSCRIPTION_ID}",
  "tenantId": "${TENANT_ID}"
}
EOF
```

### 2.8. Create Hosted Cluster

```bash
# Create namespace if needed
oc create namespace ${NAMESPACE} 2>/dev/null || true

# Create hosted cluster
$HYPERSHIFT create cluster azure \
  --name ${CLUSTER_NAME} \
  --namespace ${NAMESPACE} \
  --base-domain ${BASE_DOMAIN} \
  --location ${LOCATION} \
  --pull-secret ${PULL_SECRET} \
  --release-image ${RELEASE_IMAGE} \
  --node-pool-replicas ${NODE_COUNT} \
  --azure-creds ./azure-creds.json \
  --resource-group-name ${INSTALL_RG} \
  --workload-identities-file ./workload-identities.json \
  --oidc-issuer-url ${OIDC_ISSUER_URL} \
  --sa-token-issuer-private-key-path ./oidc/serviceaccount-signer.private \
  --dns-zone-rg-name ${DNS_ZONE_RG}

echo ""
echo "✓ Hosted cluster created: ${CLUSTER_NAME}"
echo "  Namespace: ${NAMESPACE}"
echo "  Watch: oc get hostedcluster,nodepool -n ${NAMESPACE} -w"
```

### 2.9. Monitor Cluster Creation

```bash
echo ""
echo "Monitoring hosted cluster creation (press Ctrl+C to stop watching)..."
echo "Cluster becomes Available when control plane is ready (~10-15 min)"
echo "Workers join after Azure VMs provision (~5-10 min after control plane ready)"
echo ""

oc wait --for=condition=Available hostedcluster/${CLUSTER_NAME} -n ${NAMESPACE} --timeout=30m || {
    echo "WARN: HostedCluster not Available after 30 min. Check:"
    echo "  oc get hostedcluster -n ${NAMESPACE} ${CLUSTER_NAME} -o yaml"
    echo "  oc get pods -n ${NAMESPACE}-${CLUSTER_NAME}"
}

echo ""
echo "✓ Control plane is Available"
echo "Waiting for worker nodes to join..."

# Wait for machines to be provisioned
oc wait --for=jsonpath='{.status.phase}'=Provisioned \
  machine -n ${NAMESPACE}-${CLUSTER_NAME} --all --timeout=20m || {
    echo "WARN: Machines not Provisioned after 20 min. Check:"
    echo "  oc get machines -n ${NAMESPACE}-${CLUSTER_NAME}"
    echo "  oc logs -n ${NAMESPACE}-${CLUSTER_NAME} -l app=capi-provider-controller-manager"
}

echo ""
echo "✓ Setup complete!"
echo "  Hosted cluster: ${CLUSTER_NAME}"
echo "  OIDC issuer: ${OIDC_ISSUER_URL}"
echo "  Resource groups: ${OIDC_RG}, ${INSTALL_RG}"
echo ""
echo "Next: Run validation to check worker nodes joined"
```

---

## 3. Validate Phase

```bash
echo "Validating hosted cluster: ${CLUSTER_NAME}"
echo ""

# Get hosted cluster kubeconfig
mkdir -p ~/Work/azure-hcp/
HOSTED_KUBECONFIG="$HOME/Work/azure-hcp/${CLUSTER_NAME}-kubeconfig"
$HYPERSHIFT create kubeconfig --name ${CLUSTER_NAME} --namespace ${NAMESPACE} > ${HOSTED_KUBECONFIG}
echo "✓ Generated kubeconfig: ${HOSTED_KUBECONFIG}"

# Check worker nodes
echo ""
echo "=== Worker Nodes ==="
KUBECONFIG=${HOSTED_KUBECONFIG} oc get nodes || {
    echo "WARN: Cannot access hosted cluster or no nodes joined yet"
    echo "Check control plane:"
    echo "  oc get pods -n ${NAMESPACE}-${CLUSTER_NAME} | grep -E 'etcd|kube-apiserver|kube-controller'"
    exit 1
}

NODE_COUNT_ACTUAL=$(KUBECONFIG=${HOSTED_KUBECONFIG} oc get nodes --no-headers 2>/dev/null | wc -l)
echo ""
echo "Worker nodes: ${NODE_COUNT_ACTUAL}/${NODE_COUNT}"

# Verify MCO is not functional (required for OSC DaemonSet mode)
echo ""
echo "=== MCO/MachineConfig Check ==="
echo "Checking if MCO is functional (it should NOT be on Azure HCP)..."

# Check for MachineConfigPools (the key indicator of functional MCO)
KUBECONFIG=${HOSTED_KUBECONFIG} kubectl get machineconfigpools 2>&1 | grep -q "error: the server doesn't have a resource type" && {
    echo "✓ MachineConfigPools not available (MCO not functional)"
    MCO_FUNCTIONAL=false
} || {
    MCP_COUNT=$(KUBECONFIG=${HOSTED_KUBECONFIG} kubectl get machineconfigpools --no-headers 2>/dev/null | wc -l)
    if [[ "$MCP_COUNT" -gt 0 ]]; then
        echo "ERROR: Found $MCP_COUNT MachineConfigPools - MCO is functional!"
        echo "This cluster has a working MCO and should use MachineConfig mode, not DaemonSet."
        exit 1
    else
        echo "✓ No MachineConfigPools (MCO not functional)"
        MCO_FUNCTIONAL=false
    fi
}

# Verify machine-config-daemon is not running
KUBECONFIG=${HOSTED_KUBECONFIG} kubectl get daemonset -A 2>/dev/null | grep -q machine-config-daemon && {
    echo "ERROR: machine-config-daemon is running - MCO is functional!"
    exit 1
} || {
    echo "✓ No machine-config-daemon DaemonSet"
}

echo ""
echo "✓ MCO is not functional on this cluster (correct for Azure HCP)"
echo "  OSC DaemonSet mode is required and will work correctly"

# Check cluster version
echo ""
echo "=== Cluster Version ==="
KUBECONFIG=${HOSTED_KUBECONFIG} oc get clusterversion

# Check node details
echo ""
echo "=== Node Details ==="
KUBECONFIG=${HOSTED_KUBECONFIG} oc get nodes -o wide

echo ""
echo "✓ Validation complete"
echo "  Cluster is ready for OSC DaemonSet installation"
echo ""
echo "To install OSC:"
echo "  KUBECONFIG=${HOSTED_KUBECONFIG} oc create namespace openshift-sandboxed-containers-operator"
echo "  # Install OSC operator from OperatorHub"
echo "  # Create osc-feature-gates ConfigMap with deploymentMode: DaemonSetFallback"
echo "  # Create KataConfig"
```

---

## 4. Teardown Phase

```bash
echo "Tearing down hosted cluster: ${CLUSTER_NAME}"
echo ""

# Confirm deletion
read -p "Delete hosted cluster ${CLUSTER_NAME} and all Azure resources? [y/N] " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    echo "Teardown cancelled"
    exit 0
fi

# Delete hosted cluster (this triggers Azure VM deletion)
oc delete hostedcluster ${CLUSTER_NAME} -n ${NAMESPACE} --wait=false
oc delete nodepool ${CLUSTER_NAME} -n ${NAMESPACE} --wait=false 2>/dev/null || true

echo "Waiting for cluster deletion to propagate (30s)..."
sleep 30

# Delete Azure resources
echo ""
echo "Deleting Azure resource groups..."

# Delete installation resource group (contains VMs, VNet, NSG, etc.)
az group delete --name ${INSTALL_RG} --yes --no-wait 2>/dev/null && echo "  ${INSTALL_RG}: deletion started" || echo "  ${INSTALL_RG}: not found or already deleted"

# Delete OIDC resource group (contains managed identities, storage account)
az group delete --name ${OIDC_RG} --yes --no-wait 2>/dev/null && echo "  ${OIDC_RG}: deletion started" || echo "  ${OIDC_RG}: not found or already deleted"

echo ""
echo "✓ Teardown initiated"
echo "  Azure resource groups deletion in progress (can take 5-10 min)"
echo "  Monitor: az group list --query \"[?starts_with(name,'${CLUSTER_NAME}')].{name:name,state:properties.provisioningState}\" -o table"
echo ""
echo "Local files to clean up manually:"
echo "  rm -rf ~/Work/azure-hcp/oidc"
echo "  rm ~/Work/azure-hcp/${CLUSTER_NAME}-kubeconfig"
echo "  rm ~/Work/azure-hcp/workload-identities.json"
echo "  rm ~/Work/azure-hcp/azure-creds.json"
```

---

</process>

<tips>

## Troubleshooting

### Workers not joining after 20+ minutes

1. Check capi-provider logs:
   ```bash
   oc logs -n ${NAMESPACE}-${CLUSTER_NAME} -l app=capi-provider-controller-manager -c manager --tail=50
   ```

2. Look for authentication errors:
   - `AADSTS700213: No matching federated identity record` → federated credential issue
   - `invalid_client` → wrong audience (must be "openshift", not "api://AzureADTokenExchange")
   - `EncryptionAtHost feature is not enabled` → run setup step 2.5

3. Check cloud-token-minter sidecar:
   ```bash
   oc logs -n ${NAMESPACE}-${CLUSTER_NAME} -l app=capi-provider-controller-manager -c cloud-token-minter
   ```
   Should show: `Successfully wrote token to /var/run/secrets/openshift/serviceaccount/token`

### Control plane "Waiting for etcd to reach quorum"

This usually means no workers have joined. etcd needs a functioning cluster to reach quorum, but workers need a functioning control plane to join.

If etcd shows performance issues (requests >100ms) or restart loops, **recreate the cluster** - bootstrap deadlock is difficult to recover from.

### Stale DNS records blocking cluster creation

If recreating a cluster with the same name, delete old DNS records first:
```bash
# In DNS zone resource group (osc-clusters)
az network dns record-set cname delete -g ${DNS_ZONE_RG} -z ${BASE_DOMAIN} -n "api.${CLUSTER_NAME}" --yes 2>/dev/null
az network dns record-set a delete -g ${DNS_ZONE_RG} -z ${BASE_DOMAIN} -n "*.apps.${CLUSTER_NAME}" --yes 2>/dev/null
```

</tips>

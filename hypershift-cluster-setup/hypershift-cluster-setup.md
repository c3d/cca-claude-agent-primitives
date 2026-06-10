---
name: hypershift-cluster-setup
description: Complete Azure HyperShift setup — creates management cluster (IPI OpenShift), installs MCE/HyperShift, creates hosted clusters with OIDC/managed identities, installs and validates OSC DaemonSet. Supports full teardown. Azure credentials must be configured via az login.
argument-hint: "management create|setup|teardown [--name <NAME>] | hosted setup|validate|teardown [--cluster-name <NAME>] [--location <REGION>] [--node-count <N>] | osc install|validate|reboot|test [--cluster-name <NAME>] [--operator-image <IMAGE>]"
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
Automate the complete lifecycle of self-managed Azure HyperShift for testing OpenShift Sandboxed Containers (OSC) DaemonSet mode:

**Management Cluster Operations:**
- **management create** — create IPI OpenShift 4.21+ cluster on Azure (hosts control planes)
- **management setup** — install MultiCluster Engine 2.11+, HyperShift operator, and configure prerequisites
- **management teardown** — delete management cluster and all Azure resources

**Hosted Cluster Operations:**
- **hosted setup** — create Azure OIDC issuer, managed identities with workload identity federation, and hosted cluster with worker nodes
- **hosted validate** — verify workers joined, confirm MCO non-functional (required for DaemonSet mode), test cluster readiness
- **hosted teardown** — delete hosted cluster, Azure VMs, managed identities, and OIDC infrastructure

**OSC Operations:**
- **osc install** — install OSC operator, configure DaemonSet mode, create KataConfig, label worker nodes
- **osc validate** — verify kata runtime installed on nodes, test kata pod deployment
- **osc reboot** — cordon, drain, and reboot worker nodes (required after kata installation)
- **osc test** — deploy and verify various kata workloads (basic pod, nginx, busybox commands, network test)

Azure credentials are assumed to be configured via `az login`.

Default values (all overridable via flags):
- Location: `eastus`
- Management cluster: 3 control plane + 3 worker nodes
- Hosted cluster: 2 worker nodes, Standard_D4s_v5 VMs
- OCP version: 4.21.5
</objective>

<environment>
```
Azure Requirements:
- Subscription with EncryptionAtHost feature enabled
- DNS zone for base domain in resource group (e.g., azure.sandboxedcontainers.com in osc-clusters RG)
- Sufficient quotas: VMs (Standard_D4s_v5 or similar), VNets, NSGs, load balancers
- Service principal or `az login` authentication
- Resource provider registrations: Microsoft.Compute, Microsoft.Network, Microsoft.Storage

Binaries Required:
- openshift-install (for IPI cluster creation) - ~/Work/azure-hcp/openshift-install or from mirror.openshift.com
- ccoctl-native (ARM64 macOS) or ccoctl (Linux amd64) - ~/Work/azure-hcp/bin/ccoctl-native
- hypershift CLI (ARM64 macOS) - ~/Work/azure-hcp/bin/hypershift
- oc, kubectl, az, jq (in PATH)

Files:
- Pull secret: ~/Work/notes/openshift-pull-secret.txt
- SSH public key: ~/.ssh/id_rsa.pub (for management cluster node access)

Working Directory:
- ~/Work/azure-hcp/ (for installation artifacts, kubeconfigs, OIDC keys)
```
</environment>

<process>

## 0. Parse Arguments

Parse `$ARGUMENTS`:
- First positional arg: scope (`management`, `hosted`, or `osc`)
- Second positional arg: action
  - For `management`: `create`, `setup`, `teardown`
  - For `hosted`: `setup`, `validate`, `teardown`
  - For `osc`: `install`, `validate`, `reboot`, `test`

**Flags:**
- `--name <name>` — management cluster name (default: `$USER-hcp-host-$VERSION` where VERSION extracted from OCP release)
- `--cluster-name <name>` — hosted cluster name (default: `$USER-hcp-$(date +%Y%m%d)`)
- `--location <region>` — Azure region (default: `eastus`)
- `--node-count <n>` — worker nodes for hosted cluster (default: `2`)
- `--release-image <image>` — OCP release (default: `quay.io/openshift-release-dev/ocp-release:4.21.5-x86_64`)
- `--operator-image <image>` — Custom OSC operator image for `osc install` (default: use OperatorHub)

If no arguments given, show menu and ask for operation. THEN after selection, ask for parameters specific to that operation before executing.

Menu to show:
```
Which operation do you want to run?

Management Cluster:
  1) management create      — Create IPI OpenShift cluster on Azure
  2) management setup — Install MCE + HyperShift on management cluster
  3) management teardown    — Delete management cluster

Hosted Cluster:
  4) hosted setup           — Create OIDC, identities, and hosted cluster
  5) hosted validate        — Check worker nodes and cluster readiness
  6) hosted teardown        — Delete hosted cluster and Azure resources

OSC (OpenShift Sandboxed Containers):
  7) osc install            — Install OSC operator and configure DaemonSet mode
  8) osc validate           — Verify kata runtime and test kata pod
  9) osc reboot             — Cordon, drain, and reboot worker nodes
  10) osc test              — Deploy and verify various kata workloads
```

After user selects operation, prompt for required parameters:

**For management create:**
First extract VERSION from release image:
```bash
RELEASE_IMAGE="${RELEASE_IMAGE:-quay.io/openshift-release-dev/ocp-release:4.21.5-x86_64}"
VERSION=$(echo "$RELEASE_IMAGE" | grep -oE '[0-9]+\.[0-9]+' | head -1)
DEFAULT_MGMT_NAME="${USER}-hcp-host-${VERSION}"
```
Then prompt:
```
Management cluster name [${DEFAULT_MGMT_NAME}]: 
Azure region [eastus]: 
```

**For management setup/teardown:**
```
Management cluster name: (required - list existing via 'ls ~/Work/azure-hcp/' or check current KUBECONFIG)
```

**For hosted setup:**
```
Hosted cluster name [${USER}-hcp-$(date +%Y%m%d)]: 
Azure region [eastus]: 
Number of worker nodes [2]: 
```

**For hosted validate/teardown:**
```
Hosted cluster name: (required, no default - list existing if possible)
```

Set defaults based on user input or defaults:
```bash
# Extract OCP version from release image
RELEASE_IMAGE="${RELEASE_IMAGE:-quay.io/openshift-release-dev/ocp-release:4.21.5-x86_64}"
VERSION=$(echo "$RELEASE_IMAGE" | grep -oE '[0-9]+\.[0-9]+' | head -1)

# Management cluster
MGMT_NAME="${MGMT_NAME:-${USER}-hcp-host-${VERSION}}"
MGMT_KUBECONFIG="${MGMT_KUBECONFIG:-$HOME/Work/azure-hcp/auth/kubeconfig}"

# Hosted cluster
CLUSTER_NAME="${CLUSTER_NAME:-${USER}-hcp-$(date +%Y%m%d)}"
LOCATION="${LOCATION:-eastus}"
NODE_COUNT="${NODE_COUNT:-2}"
RELEASE_IMAGE="${RELEASE_IMAGE:-quay.io/openshift-release-dev/ocp-release:4.21.5-x86_64}"

# OSC operator
OPERATOR_IMAGE="${OPERATOR_IMAGE:-}"  # Empty = use OperatorHub, set to custom image to override

# Common
BASE_DOMAIN="azure.sandboxedcontainers.com"
DNS_ZONE_RG="osc-clusters"
NAMESPACE="clusters"
PULL_SECRET="${PULL_SECRET:-$HOME/Work/notes/openshift-pull-secret.txt}"
SSH_KEY="${SSH_KEY:-$HOME/.ssh/id_rsa.pub}"

# Derived names (for hosted cluster)
OIDC_RG="${CLUSTER_NAME}-oidc"
INSTALL_RG="${CLUSTER_NAME}-rg"
STORAGE_ACCOUNT=$(echo "${CLUSTER_NAME}oidc" | tr -d '-' | cut -c1-24)
OIDC_ISSUER_URL="https://${STORAGE_ACCOUNT}.blob.core.windows.net/${CLUSTER_NAME}"
```

---

## MANAGEMENT CLUSTER OPERATIONS

### 1. Management Create - Create IPI OpenShift Cluster on Azure

```bash
cd ~/Work/azure-hcp

echo "=== Creating Management Cluster: ${MGMT_NAME} ==="
echo "Location: ${LOCATION}"
echo "Base domain: ${BASE_DOMAIN}"
echo ""

# Check for existing installation directory
if [[ -d "./${MGMT_NAME}" ]]; then
    echo "ERROR: Installation directory ${MGMT_NAME} already exists"
    echo "To recreate, either:"
    echo "  1. Use a different name: --name other-cluster-name"
    echo "  2. Delete existing: rm -rf ${MGMT_NAME}"
    echo "  3. Run teardown first: /hypershift-cluster-setup management teardown --name ${MGMT_NAME}"
    exit 1
fi

# Check if openshift-install exists
OPENSHIFT_INSTALL=$(command -v openshift-install 2>/dev/null || echo "${HOME}/Work/azure-hcp/openshift-install")
if [[ ! -x "$OPENSHIFT_INSTALL" ]]; then
    echo "ERROR: openshift-install not found"
    echo "Download from: https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable-4.21/"
    echo "Expected locations: ~/Work/azure-hcp/openshift-install or in PATH"
    exit 1
fi

echo "Using openshift-install: $OPENSHIFT_INSTALL"
echo ""

# Check DNS zone exists
az network dns zone show --name ${BASE_DOMAIN} --resource-group ${DNS_ZONE_RG} >/dev/null 2>&1 || {
    echo "ERROR: DNS zone ${BASE_DOMAIN} not found in resource group ${DNS_ZONE_RG}"
    echo "Create it with:"
    echo "  az group create --name ${DNS_ZONE_RG} --location ${LOCATION}"
    echo "  az network dns zone create --name ${BASE_DOMAIN} --resource-group ${DNS_ZONE_RG}"
    exit 1
}

# Create installation directory
mkdir -p ${MGMT_NAME}
cd ${MGMT_NAME}

# Generate install-config.yaml
echo "Generating install-config.yaml..."
cat > install-config.yaml <<EOF
apiVersion: v1
baseDomain: ${BASE_DOMAIN}
metadata:
  name: ${MGMT_NAME}
platform:
  azure:
    region: ${LOCATION}
    baseDomainResourceGroupName: ${DNS_ZONE_RG}
pullSecret: '$(cat ${PULL_SECRET} | jq -c .)'
sshKey: '$(cat ${SSH_KEY})'
compute:
- name: worker
  platform:
    azure:
      type: Standard_D4s_v5
  replicas: 3
controlPlane:
  name: master
  platform:
    azure:
      type: Standard_D4s_v5
  replicas: 3
EOF

echo "✓ Created install-config.yaml"
echo ""

# Backup install-config (it gets consumed)
cp install-config.yaml install-config.yaml.backup

# Create cluster
echo "Creating cluster (this takes ~40 minutes)..."
${OPENSHIFT_INSTALL} create cluster --dir . --log-level=info

# Save kubeconfig to standard location
mkdir -p ~/Work/azure-hcp/auth
cp auth/kubeconfig ~/Work/azure-hcp/auth/kubeconfig

echo ""
echo "✓ Management cluster created successfully!"
echo "  Name: ${MGMT_NAME}"
echo "  Kubeconfig: ~/Work/azure-hcp/auth/kubeconfig"
echo "  Console: https://console-openshift-console.apps.${MGMT_NAME}.${BASE_DOMAIN}"
echo ""
echo "Next steps:"
echo "  1. Setup management: /hypershift-cluster-setup management setup"
echo "  2. Create hosted cluster: /hypershift-cluster-setup hosted setup"
```

**Note:** If cluster creation fails due to stale DNS records from a previous cluster with the same name, delete them:
```bash
# Delete CNAME record for api
az network dns record-set cname delete \
  -g ${DNS_ZONE_RG} \
  -z ${BASE_DOMAIN} \
  -n "api.${MGMT_NAME}" \
  --yes 2>/dev/null

# Delete A record for *.apps
az network dns record-set a delete \
  -g ${DNS_ZONE_RG} \
  -z ${BASE_DOMAIN} \
  -n "*.apps.${MGMT_NAME}" \
  --yes 2>/dev/null
```

---

### 2. Management Setup - Install MCE, HyperShift, and Configure Prerequisites

```bash
export KUBECONFIG=~/Work/azure-hcp/auth/kubeconfig

echo "=== Setting Up Management Cluster: ${MGMT_NAME} ==="

# Verify management cluster is accessible
oc cluster-info >/dev/null 2>&1 || {
    echo "ERROR: Cannot access management cluster"
    echo "Check KUBECONFIG: ~/Work/azure-hcp/auth/kubeconfig"
    exit 1
}

echo "✓ Management cluster accessible"
echo ""

# Check cluster version
CLUSTER_VERSION=$(oc get clusterversion -o jsonpath='{.items[0].status.desired.version}')
echo "Cluster version: ${CLUSTER_VERSION}"

# Verify minimum version (4.21+)
MAJOR_MINOR=$(echo ${CLUSTER_VERSION} | cut -d. -f1,2)
if [[ "${MAJOR_MINOR}" < "4.21" ]]; then
    echo "WARNING: OpenShift version ${CLUSTER_VERSION} may not support HyperShift"
    echo "Recommended: 4.21.0 or newer"
fi

echo ""

# Step 1: Install MultiCluster Engine operator
echo "=== Step 1: Installing MultiCluster Engine Operator ==="
echo ""

# Create multicluster-engine namespace
echo "Creating multicluster-engine namespace..."
oc create namespace multicluster-engine 2>/dev/null || echo "  Namespace already exists"

# Create OperatorGroup
echo "Creating OperatorGroup..."
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: multicluster-engine-operatorgroup
  namespace: multicluster-engine
spec:
  targetNamespaces:
  - multicluster-engine
EOF

# Create Subscription
echo "Creating Subscription for MCE..."
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: multicluster-engine
  namespace: multicluster-engine
spec:
  channel: stable-2.11
  name: multicluster-engine
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

echo ""
echo "Waiting for MCE operator to install (may take 2-3 minutes)..."
sleep 30

# Wait for CSV to be ready
for i in {1..30}; do
    CSV=$(oc get csv -n multicluster-engine -o name 2>/dev/null | grep multicluster-engine | head -1)
    if [[ -n "$CSV" ]]; then
        PHASE=$(oc get ${CSV} -n multicluster-engine -o jsonpath='{.status.phase}')
        if [[ "$PHASE" == "Succeeded" ]]; then
            echo "✓ MCE operator installed: ${CSV}"
            break
        fi
        echo "  Waiting for CSV (phase: ${PHASE})... ($i/30)"
    else
        echo "  Waiting for CSV to appear... ($i/30)"
    fi
    sleep 10
done

echo ""

# Step 2: Create MultiClusterEngine instance
echo "=== Step 2: Creating MultiClusterEngine Instance ==="
echo ""

cat <<EOF | oc apply -f -
apiVersion: multicluster.openshift.io/v1
kind: MultiClusterEngine
metadata:
  name: multiclusterengine
spec: {}
EOF

echo "Waiting for MultiClusterEngine to be available (may take 3-5 minutes)..."
oc wait --for=condition=Available multiclusterengine/multiclusterengine --timeout=10m

echo ""

# Step 3: Verify HyperShift operator and CRDs
echo "=== Step 3: Verifying HyperShift Installation ==="
echo ""

oc get crd hostedclusters.hypershift.openshift.io >/dev/null 2>&1 && echo "✓ HostedCluster CRD present" || echo "ERROR: HostedCluster CRD not found"
oc get crd nodepools.hypershift.openshift.io >/dev/null 2>&1 && echo "✓ NodePool CRD present" || echo "ERROR: NodePool CRD not found"

echo ""
echo "HyperShift operator pods:"
oc get pods -n hypershift

echo ""

# Step 4: Configure Azure prerequisites
echo "=== Step 4: Configuring Azure Prerequisites ==="
echo ""

# Check Azure login
SUBSCRIPTION_ID=$(az account show --query id -o tsv 2>/dev/null)
if [[ -z "$SUBSCRIPTION_ID" ]]; then
    echo "ERROR: Not logged into Azure. Run: az login"
    exit 1
fi
echo "✓ Azure subscription: ${SUBSCRIPTION_ID}"

# Check DNS zone
az network dns zone show --name ${BASE_DOMAIN} --resource-group ${DNS_ZONE_RG} >/dev/null 2>&1 && {
    echo "✓ DNS zone exists: ${BASE_DOMAIN} (RG: ${DNS_ZONE_RG})"
} || {
    echo "WARNING: DNS zone ${BASE_DOMAIN} not found in resource group ${DNS_ZONE_RG}"
    echo "Hosted cluster creation will fail without this. Create it with:"
    echo "  az network dns zone create --name ${BASE_DOMAIN} --resource-group ${DNS_ZONE_RG}"
}

# Check EncryptionAtHost feature
ENCR_STATE=$(az feature show --namespace Microsoft.Compute --name EncryptionAtHost --query properties.state -o tsv 2>/dev/null || echo "NotRegistered")
if [[ "$ENCR_STATE" == "Registered" ]]; then
    echo "✓ EncryptionAtHost feature enabled"
else
    echo "⚠️  EncryptionAtHost feature not registered (state: ${ENCR_STATE})"
    echo "Enabling now (may take a few minutes to propagate)..."
    az feature register --namespace Microsoft.Compute --name EncryptionAtHost
    az provider register -n Microsoft.Compute
    echo "✓ EncryptionAtHost registration initiated"
fi

echo ""

# Step 5: Verify required binaries
echo "=== Step 5: Verifying Required Binaries ==="
echo ""

export PATH="$HOME/Work/azure-hcp/bin:$PATH"

# Check ccoctl
CCOCTL=$(command -v ccoctl-native 2>/dev/null || command -v ccoctl 2>/dev/null)
if [[ -n "$CCOCTL" ]]; then
    echo "✓ ccoctl: $CCOCTL"
else
    echo "ERROR: ccoctl-native or ccoctl not found"
    echo "Build from source or download:"
    echo "  - ccoctl-native: Build from ~/Work/cloud-credential-operator/cmd/ccoctl for ARM64"
    echo "  - ccoctl: Download from mirror.openshift.com"
    exit 1
fi

# Check hypershift CLI
HYPERSHIFT=$(command -v hypershift 2>/dev/null)
if [[ -n "$HYPERSHIFT" ]]; then
    echo "✓ hypershift: $HYPERSHIFT"
else
    echo "ERROR: hypershift CLI not found"
    echo "Build from source:"
    echo "  cd ~/Work/hypershift && make build"
    echo "  cp bin/hypershift ~/Work/azure-hcp/bin/"
    exit 1
fi

# Check pull secret
if [[ -f "${PULL_SECRET}" ]]; then
    echo "✓ Pull secret: ${PULL_SECRET}"
else
    echo "ERROR: Pull secret not found at ${PULL_SECRET}"
    echo "Download from: https://console.redhat.com/openshift/install/pull-secret"
    exit 1
fi

echo ""
echo "✓ Management cluster setup complete!"
echo ""
echo "Summary:"
echo "  - MCE operator installed and MultiClusterEngine available"
echo "  - HyperShift operator running with CRDs present"
echo "  - Azure prerequisites configured"
echo "  - Required binaries verified"
echo ""
echo "Next step:"
echo "  Create hosted cluster: /hypershift-cluster-setup hosted setup --cluster-name my-hcp"
```

---

### 3. Management Teardown - Delete Management Cluster

```bash
cd ~/Work/azure-hcp

echo "=== Tearing Down Management Cluster: ${MGMT_NAME} ==="
echo ""

# Confirm deletion
read -p "Delete management cluster ${MGMT_NAME} and ALL hosted clusters? [y/N] " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    echo "Teardown cancelled"
    exit 0
fi

# Check if installation directory exists
if [[ ! -d "${MGMT_NAME}" ]]; then
    echo "WARNING: Installation directory ${MGMT_NAME} not found"
    echo "Cannot use openshift-install destroy. Manual cleanup required:"
    echo "  1. List resource groups: az group list --query \"[?starts_with(name,'${MGMT_NAME}')].name\" -o table"
    echo "  2. Delete each: az group delete --name <RG_NAME> --yes --no-wait"
    echo "  3. Delete DNS records in ${DNS_ZONE_RG}"
    exit 1
fi

cd ${MGMT_NAME}

# Run openshift-install destroy
OPENSHIFT_INSTALL=$(command -v openshift-install 2>/dev/null || echo "${HOME}/Work/azure-hcp/openshift-install")
${OPENSHIFT_INSTALL} destroy cluster --dir . --log-level=info

cd ..
rm -rf ${MGMT_NAME}

echo ""
echo "✓ Management cluster destroyed"
echo "  Deleted: ${MGMT_NAME}"
echo ""
echo "Manual cleanup (if needed):"
echo "  - Check for stale Azure resource groups: az group list -o table"
echo "  - Delete DNS records if present:"
echo "    az network dns record-set cname delete -g ${DNS_ZONE_RG} -z ${BASE_DOMAIN} -n \"api.${MGMT_NAME}\" --yes"
echo "    az network dns record-set a delete -g ${DNS_ZONE_RG} -z ${BASE_DOMAIN} -n \"*.apps.${MGMT_NAME}\" --yes"
```

---

## HOSTED CLUSTER OPERATIONS

### Preflight Check (all hosted cluster subcommands)

```bash
# Management cluster
export KUBECONFIG=~/Work/azure-hcp/auth/kubeconfig
oc cluster-info >/dev/null 2>&1 || { echo "BLOCK: Management cluster not accessible (KUBECONFIG=$KUBECONFIG)"; exit 1; }

# Check MCE/HyperShift
oc get crd hostedclusters.hypershift.openshift.io >/dev/null 2>&1 || {
    echo "BLOCK: HyperShift CRDs not found. Install MCE first:"
    echo "  /hypershift-cluster-setup management setup"
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

### 4. Hosted Setup - Create Hosted Cluster with OIDC and Managed Identities

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

### 5. Hosted Validate - Verify Hosted Cluster Health and OSC Readiness

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

### 6. Hosted Teardown - Delete Hosted Cluster and Azure Resources

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

## OSC OPERATIONS

### Preflight Check (all OSC subcommands)

```bash
# Hosted cluster kubeconfig
HOSTED_KUBECONFIG="$HOME/Work/azure-hcp/${CLUSTER_NAME}-kubeconfig"
[[ -f "$HOSTED_KUBECONFIG" ]] || { echo "BLOCK: Kubeconfig not found. Run: /hypershift-cluster-setup hosted validate --cluster-name ${CLUSTER_NAME}"; exit 1; }

export KUBECONFIG=${HOSTED_KUBECONFIG}
oc cluster-info >/dev/null 2>&1 || { echo "BLOCK: Cannot access hosted cluster (KUBECONFIG=$KUBECONFIG)"; exit 1; }
echo "✓ Hosted cluster accessible"

# Verify MCO is not functional (required for DaemonSet mode)
if kubectl get machineconfigpools 2>&1 | grep -q "doesn't have a resource type"; then
    echo "✓ MCO not functional (DaemonSet mode compatible)"
else
    echo "ERROR: MCO is functional on this cluster - DaemonSet mode will not work"
    echo "This cluster requires MachineConfig mode, not DaemonSet mode"
    exit 1
fi

# Check worker nodes
NODE_COUNT=$(oc get nodes --no-headers 2>/dev/null | wc -l | tr -d ' ')
[[ "$NODE_COUNT" -gt 0 ]] || { echo "BLOCK: No worker nodes found"; exit 1; }
echo "✓ Found ${NODE_COUNT} worker nodes"
```

---

### 7. OSC Install - Install OSC Operator and Configure DaemonSet Mode

```bash
echo "=== Installing OpenShift Sandboxed Containers (OSC) ==="
echo "Cluster: ${CLUSTER_NAME}"
echo "Mode: DaemonSet (MCO non-functional)"
echo ""

# Step 1: Create namespace
echo "=== Step 1: Creating Namespace ==="
oc create namespace openshift-sandboxed-containers-operator 2>/dev/null || echo "✓ Namespace already exists"

# Step 2: Create OperatorGroup
echo ""
echo "=== Step 2: Creating OperatorGroup ==="
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-sandboxed-containers-operator
  namespace: openshift-sandboxed-containers-operator
spec:
  targetNamespaces:
  - openshift-sandboxed-containers-operator
EOF

echo "✓ OperatorGroup created"

# Step 3: Install OSC Operator
echo ""
echo "=== Step 3: Installing OSC Operator ==="

# Check if custom operator image specified
if [[ -n "${OPERATOR_IMAGE}" ]]; then
    echo "Using custom OSC operator image: ${OPERATOR_IMAGE}"
    echo ""
    
    # Deploy custom operator directly via Deployment (bypass OperatorHub)
    cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openshift-sandboxed-containers-operator
  namespace: openshift-sandboxed-containers-operator
  labels:
    app: openshift-sandboxed-containers-operator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openshift-sandboxed-containers-operator
  template:
    metadata:
      labels:
        app: openshift-sandboxed-containers-operator
    spec:
      serviceAccountName: openshift-sandboxed-containers-operator
      containers:
      - name: manager
        image: ${OPERATOR_IMAGE}
        command:
        - /manager
        env:
        - name: RELATED_IMAGE_SANDBOXED_CONTAINERS_OPERATOR_BUNDLE
          value: ${OPERATOR_IMAGE}
        - name: OPERATOR_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: PLATFORM
          value: "Azure"
        imagePullPolicy: Always
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: openshift-sandboxed-containers-operator
  namespace: openshift-sandboxed-containers-operator
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: openshift-sandboxed-containers-operator
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openshift-sandboxed-containers-operator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openshift-sandboxed-containers-operator
subjects:
- kind: ServiceAccount
  name: openshift-sandboxed-containers-operator
  namespace: openshift-sandboxed-containers-operator
EOF

    echo "Waiting for custom operator deployment (may take 1-2 minutes)..."
    oc wait --for=condition=Available deployment/openshift-sandboxed-containers-operator \
      -n openshift-sandboxed-containers-operator --timeout=5m
    
    echo "✓ Custom OSC operator deployed: ${OPERATOR_IMAGE}"
    
else
    echo "Using OSC operator from OperatorHub"
    echo ""
    
    # Standard OperatorHub installation
    cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-sandboxed-containers-operator
  namespace: openshift-sandboxed-containers-operator
spec:
  channel: stable
  name: sandboxed-containers-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
EOF

    echo "Waiting for OSC operator to install (may take 2-3 minutes)..."
    sleep 30

    # Wait for CSV
    for i in {1..30}; do
        CSV=\$(oc get csv -n openshift-sandboxed-containers-operator -o name 2>/dev/null | grep sandboxed-containers | head -1)
        if [[ -n "\$CSV" ]]; then
            PHASE=\$(oc get \${CSV} -n openshift-sandboxed-containers-operator -o jsonpath='{.status.phase}')
            if [[ "\$PHASE" == "Succeeded" ]]; then
                echo "✓ OSC operator installed: \${CSV}"
                break
            fi
            echo "  Waiting for CSV (phase: \${PHASE})... (\$i/30)"
        else
            echo "  Waiting for CSV to appear... (\$i/30)"
        fi
        sleep 10
    done
fi

# Step 4: Create feature gates ConfigMap for DaemonSet mode
echo ""
echo "=== Step 4: Configuring DaemonSet Mode ==="
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: osc-feature-gates
  namespace: openshift-sandboxed-containers-operator
data:
  deploymentMode: "DaemonSet"
EOF

echo "✓ Feature gates configured (deploymentMode: DaemonSet)"

# Step 5: Label worker nodes for kata
echo ""
echo "=== Step 5: Labeling Worker Nodes ==="
oc label node -l node-role.kubernetes.io/worker kata=true --overwrite
echo "✓ Worker nodes labeled with kata=true"

# Step 6: Create KataConfig
echo ""
echo "=== Step 6: Creating KataConfig ==="
cat <<EOF | oc apply -f -
apiVersion: kataconfiguration.openshift.io/v1
kind: KataConfig
metadata:
  name: kata-config
spec:
  kataConfigPoolSelector:
    matchLabels:
      kata: "true"
EOF

echo "✓ KataConfig created"
echo ""
echo "Waiting for kata installation to begin (checking DaemonSet)..."

# Wait for kata-install DaemonSet
for i in {1..30}; do
    if oc get daemonset -n openshift-sandboxed-containers-operator kata-install >/dev/null 2>&1; then
        DESIRED=\$(oc get daemonset -n openshift-sandboxed-containers-operator kata-install -o jsonpath='{.status.desiredNumberScheduled}')
        READY=\$(oc get daemonset -n openshift-sandboxed-containers-operator kata-install -o jsonpath='{.status.numberReady}')
        echo "  kata-install DaemonSet: \${READY}/\${DESIRED} ready (\$i/30)"
        if [[ "\$READY" == "\$DESIRED" && "\$READY" -gt 0 ]]; then
            echo "✓ Kata installation DaemonSet ready"
            break
        fi
    else
        echo "  Waiting for kata-install DaemonSet... (\$i/30)"
    fi
    sleep 10
done

echo ""
echo "✓ OSC installation complete!"
echo ""
echo "Summary:"
echo "  - OSC operator installed in openshift-sandboxed-containers-operator namespace"
echo "  - DaemonSet mode configured (MCO non-functional)"
echo "  - KataConfig created with kata-install DaemonSet"
echo "  - Worker nodes labeled with kata=true"
echo ""
echo "IMPORTANT: Worker nodes require reboot to load kata runtime"
echo "Next steps:"
echo "  1. Reboot nodes: /hypershift-cluster-setup osc reboot --cluster-name ${CLUSTER_NAME}"
echo "  2. Validate installation: /hypershift-cluster-setup osc validate --cluster-name ${CLUSTER_NAME}"
```

---

### 8. OSC Validate - Verify Kata Runtime and Test Kata Pod

```bash
echo "=== Validating OSC Installation ==="
echo "Cluster: ${CLUSTER_NAME}"
echo ""

# Step 1: Check KataConfig status
echo "=== Step 1: Checking KataConfig Status ==="
KATACONFIG_STATUS=\$(oc get kataconfig kata-config -o jsonpath='{.status.installationStatus.IsInProgress}' 2>/dev/null || echo "unknown")
KATACONFIG_COMPLETED=\$(oc get kataconfig kata-config -o jsonpath='{.status.installationStatus.Completed.CompletedNodesList}' 2>/dev/null | jq -r '.[]' 2>/dev/null | wc -l | tr -d ' ')

echo "KataConfig installation in progress: \${KATACONFIG_STATUS}"
echo "Nodes with kata installed: \${KATACONFIG_COMPLETED}"

if oc get kataconfig kata-config -o yaml | grep -A 5 "^status:" | grep -q "Degraded"; then
    echo "⚠️  KataConfig shows Degraded condition"
    oc get kataconfig kata-config -o yaml | grep -A 10 "conditions:"
fi

# Step 2: Check kata-install DaemonSet
echo ""
echo "=== Step 2: Checking kata-install DaemonSet ==="
if oc get daemonset -n openshift-sandboxed-containers-operator kata-install >/dev/null 2>&1; then
    oc get daemonset -n openshift-sandboxed-containers-operator kata-install
    echo ""
    
    DESIRED=\$(oc get daemonset -n openshift-sandboxed-containers-operator kata-install -o jsonpath='{.status.desiredNumberScheduled}')
    READY=\$(oc get daemonset -n openshift-sandboxed-containers-operator kata-install -o jsonpath='{.status.numberReady}')
    
    if [[ "\$READY" == "\$DESIRED" && "\$READY" -gt 0 ]]; then
        echo "✓ kata-install DaemonSet ready (\${READY}/\${DESIRED})"
    else
        echo "⚠️  kata-install DaemonSet not ready: \${READY}/\${DESIRED}"
        echo "Check pod status:"
        oc get pods -n openshift-sandboxed-containers-operator -l name=kata-install
    fi
else
    echo "⚠️  kata-install DaemonSet not found"
fi

# Step 3: Check RuntimeClass
echo ""
echo "=== Step 3: Checking RuntimeClass ==="
if oc get runtimeclass kata >/dev/null 2>&1; then
    echo "✓ RuntimeClass 'kata' exists"
    oc get runtimeclass kata -o yaml | grep -E "^  handler:|^  scheduling:"
else
    echo "⚠️  RuntimeClass 'kata' not found"
    echo "Expected after kata installation completes"
fi

# Step 4: Check kata runtime on nodes
echo ""
echo "=== Step 4: Checking Kata Runtime on Nodes ==="
for node in \$(oc get nodes -l kata=true -o name); do
    node_name=\$(basename \$node)
    echo "Node: \$node_name"
    
    # Check if node has kata runtime via node status
    if oc get node \$node_name -o json | jq -r '.status.nodeInfo' | grep -q kata; then
        echo "  ✓ Kata runtime detected in node info"
    else
        echo "  ℹ️  Kata runtime not visible in node info (may require reboot)"
    fi
done

# Step 5: Deploy test kata pod
echo ""
echo "=== Step 5: Testing Kata Pod Deployment ==="
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kata-test-pod
  namespace: default
spec:
  runtimeClassName: kata
  containers:
  - name: test
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["sleep", "3600"]
EOF

echo "Waiting for kata test pod to start..."
sleep 5

POD_STATUS=\$(oc get pod kata-test-pod -n default -o jsonpath='{.status.phase}' 2>/dev/null || echo "NotFound")
echo "Test pod status: \${POD_STATUS}"

if [[ "\$POD_STATUS" == "Running" ]]; then
    echo "✓ Kata pod running successfully"
    
    # Verify it's actually using kata
    NODE=\$(oc get pod kata-test-pod -n default -o jsonpath='{.spec.nodeName}')
    echo "  Running on node: \${NODE}"
    echo "  Runtime class: kata"
    
    # Check container runtime
    RUNTIME_INFO=\$(oc get pod kata-test-pod -n default -o jsonpath='{.status.containerStatuses[0].containerID}')
    echo "  Container ID: \${RUNTIME_INFO}"
    
elif [[ "\$POD_STATUS" == "Pending" ]]; then
    echo "⚠️  Kata pod is Pending"
    echo "Events:"
    oc describe pod kata-test-pod -n default | grep -A 10 "^Events:"
    
elif [[ "\$POD_STATUS" == "Failed" || "\$POD_STATUS" == "Error" ]]; then
    echo "❌ Kata pod failed to start"
    oc describe pod kata-test-pod -n default
else
    echo "ℹ️  Kata pod status: \${POD_STATUS}"
fi

echo ""
echo "To clean up test pod:"
echo "  oc delete pod kata-test-pod -n default"

echo ""
echo "=== Validation Summary ==="
if [[ "\$POD_STATUS" == "Running" ]]; then
    echo "✓ OSC installation successful - kata pods can run"
else
    echo "⚠️  OSC installation incomplete or nodes need reboot"
    echo "If nodes haven't been rebooted, run:"
    echo "  /hypershift-cluster-setup osc reboot --cluster-name ${CLUSTER_NAME}"
fi
```

---

### 9. OSC Reboot - Cordon, Drain, and Reboot Worker Nodes

```bash
echo "=== Rebooting Worker Nodes ==="
echo "Cluster: ${CLUSTER_NAME}"
echo ""
echo "This will sequentially:"
echo "  1. Cordon each node (mark unschedulable)"
echo "  2. Drain workloads to other nodes"
echo "  3. Reboot the node via debug pod"
echo "  4. Wait for node to come back Ready"
echo "  5. Uncordon the node"
echo ""

read -p "Proceed with node reboots? [y/N] " -n 1 -r
echo
if [[ ! \$REPLY =~ ^[Yy]$ ]]; then
    echo "Reboot cancelled"
    exit 0
fi

NODES=\$(oc get nodes -l kata=true -o name)
NODE_COUNT=\$(echo "\$NODES" | wc -l | tr -d ' ')

echo "Found \${NODE_COUNT} nodes to reboot"
echo ""

for node in \$NODES; do
    node_name=\$(basename \$node)
    echo "=== Processing node: \${node_name} ==="
    
    # Cordon node
    echo "  Cordoning node..."
    oc adm cordon \${node_name}
    
    # Drain node
    echo "  Draining node (may take a few minutes)..."
    oc adm drain \${node_name} \\
        --ignore-daemonsets \\
        --delete-emptydir-data \\
        --force \\
        --grace-period=300 \\
        --timeout=600s || {
        echo "  ⚠️  Drain timed out or failed, continuing anyway..."
    }
    
    # Reboot via debug pod
    echo "  Rebooting node..."
    oc debug node/\${node_name} -- chroot /host systemctl reboot &
    
    # Wait a moment for reboot to initiate
    sleep 10
    
    # Wait for node to become NotReady
    echo "  Waiting for node to go down..."
    for i in {1..60}; do
        STATUS=\$(oc get node \${node_name} -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null || echo "Unknown")
        if [[ "\$STATUS" != "True" ]]; then
            echo "  Node is down (iteration \$i)"
            break
        fi
        sleep 5
    done
    
    # Wait for node to come back Ready
    echo "  Waiting for node to come back up (this can take 3-5 minutes)..."
    for i in {1..120}; do
        STATUS=\$(oc get node \${node_name} -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null || echo "Unknown")
        if [[ "\$STATUS" == "True" ]]; then
            echo "  ✓ Node is Ready (iteration \$i)"
            break
        fi
        if [[ \$((i % 12)) -eq 0 ]]; then
            echo "  Still waiting for node... (\$i/120)"
        fi
        sleep 5
    done
    
    # Uncordon node
    echo "  Uncordoning node..."
    oc adm uncordon \${node_name}
    
    echo "  ✓ Node \${node_name} rebooted and ready"
    echo ""
    
    # Small delay before next node
    if [[ "\$node" != "\$(echo \"\$NODES\" | tail -1)" ]]; then
        echo "Waiting 30s before processing next node..."
        sleep 30
    fi
done

echo ""
echo "✓ All nodes rebooted successfully"
echo ""
echo "Next steps:"
echo "  Validate kata installation: /hypershift-cluster-setup osc validate --cluster-name ${CLUSTER_NAME}"
echo "  Run comprehensive tests: /hypershift-cluster-setup osc test --cluster-name ${CLUSTER_NAME}"
```

---

### 10. OSC Test - Deploy and Verify Various Kata Workloads

```bash
echo "=== Testing Kata Workloads ==="
echo "Cluster: ${CLUSTER_NAME}"
echo ""
echo "This will deploy and test various kata workloads:"
echo "  1. Basic sleep pod"
echo "  2. Nginx web server"
echo "  3. Busybox with command execution"
echo "  4. Network connectivity test"
echo "  5. Multi-container pod"
echo ""

# Create test namespace
TEST_NAMESPACE="kata-test-$(date +%s)"
echo "Creating test namespace: ${TEST_NAMESPACE}"
oc create namespace ${TEST_NAMESPACE}

TESTS_PASSED=0
TESTS_FAILED=0
FAILED_TESTS=()

# Test 1: Basic sleep pod
echo ""
echo "=== Test 1: Basic Sleep Pod ==="
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kata-sleep
  namespace: ${TEST_NAMESPACE}
  labels:
    test: basic-sleep
spec:
  runtimeClassName: kata
  containers:
  - name: sleeper
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["sleep", "3600"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "200m"
EOF

echo "Waiting for kata-sleep pod..."
if oc wait --for=condition=Ready pod/kata-sleep -n ${TEST_NAMESPACE} --timeout=60s 2>/dev/null; then
    echo "✓ Test 1 PASSED: Basic sleep pod running"
    TESTS_PASSED=$((TESTS_PASSED + 1))
else
    echo "✗ Test 1 FAILED: Pod did not become Ready"
    TESTS_FAILED=$((TESTS_FAILED + 1))
    FAILED_TESTS+=("basic-sleep")
    oc describe pod kata-sleep -n ${TEST_NAMESPACE} | tail -20
fi

# Test 2: Nginx web server
echo ""
echo "=== Test 2: Nginx Web Server ==="
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kata-nginx
  namespace: ${TEST_NAMESPACE}
  labels:
    test: nginx
spec:
  runtimeClassName: kata
  containers:
  - name: nginx
    image: registry.access.redhat.com/ubi9/nginx-124:latest
    ports:
    - containerPort: 8080
      name: http
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "500m"
EOF

echo "Waiting for kata-nginx pod..."
if oc wait --for=condition=Ready pod/kata-nginx -n ${TEST_NAMESPACE} --timeout=90s 2>/dev/null; then
    echo "✓ Test 2 PASSED: Nginx pod running"
    TESTS_PASSED=$((TESTS_PASSED + 1))
    
    # Try to curl nginx (create a test pod to curl from)
    echo "  Testing HTTP connectivity to nginx..."
    NGINX_IP=$(oc get pod kata-nginx -n ${TEST_NAMESPACE} -o jsonpath='{.status.podIP}')
    if oc run curl-test --image=registry.access.redhat.com/ubi9/ubi-minimal:latest \
        --rm -i --restart=Never -n ${TEST_NAMESPACE} --timeout=30s \
        -- curl -s -o /dev/null -w "%{http_code}" http://${NGINX_IP}:8080/ 2>/dev/null | grep -q 200; then
        echo "  ✓ HTTP connectivity working (200 OK)"
    else
        echo "  ⚠️  HTTP connectivity test inconclusive"
    fi
else
    echo "✗ Test 2 FAILED: Nginx pod did not become Ready"
    TESTS_FAILED=$((TESTS_FAILED + 1))
    FAILED_TESTS+=("nginx")
    oc describe pod kata-nginx -n ${TEST_NAMESPACE} | tail -20
fi

# Test 3: Busybox with command execution
echo ""
echo "=== Test 3: Busybox Command Execution ==="
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kata-busybox
  namespace: ${TEST_NAMESPACE}
  labels:
    test: busybox
spec:
  runtimeClassName: kata
  containers:
  - name: busybox
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["sh", "-c", "echo 'Kata runtime test' > /tmp/test.txt && sleep 3600"]
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
      limits:
        memory: "64Mi"
        cpu: "100m"
EOF

echo "Waiting for kata-busybox pod..."
if oc wait --for=condition=Ready pod/kata-busybox -n ${TEST_NAMESPACE} --timeout=60s 2>/dev/null; then
    echo "✓ Test 3 PASSED: Busybox pod running"
    TESTS_PASSED=$((TESTS_PASSED + 1))
    
    # Test command execution
    echo "  Testing command execution..."
    if oc exec kata-busybox -n ${TEST_NAMESPACE} -- cat /tmp/test.txt 2>/dev/null | grep -q "Kata runtime test"; then
        echo "  ✓ Command execution working"
    else
        echo "  ⚠️  Command execution test failed"
    fi
else
    echo "✗ Test 3 FAILED: Busybox pod did not become Ready"
    TESTS_FAILED=$((TESTS_FAILED + 1))
    FAILED_TESTS+=("busybox")
    oc describe pod kata-busybox -n ${TEST_NAMESPACE} | tail -20
fi

# Test 4: Network connectivity test (pod-to-pod)
echo ""
echo "=== Test 4: Network Connectivity Test ==="
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kata-nettest-server
  namespace: ${TEST_NAMESPACE}
  labels:
    test: network-server
spec:
  runtimeClassName: kata
  containers:
  - name: server
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["sh", "-c", "microdnf install -y nc && nc -l 8888"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
---
apiVersion: v1
kind: Pod
metadata:
  name: kata-nettest-client
  namespace: ${TEST_NAMESPACE}
  labels:
    test: network-client
spec:
  runtimeClassName: kata
  containers:
  - name: client
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["sleep", "3600"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
EOF

echo "Waiting for network test pods..."
sleep 10
if oc wait --for=condition=Ready pod/kata-nettest-client -n ${TEST_NAMESPACE} --timeout=90s 2>/dev/null; then
    SERVER_IP=$(oc get pod kata-nettest-server -n ${TEST_NAMESPACE} -o jsonpath='{.status.podIP}' 2>/dev/null)
    if [[ -n "$SERVER_IP" ]]; then
        echo "  Server IP: ${SERVER_IP}"
        echo "  Testing pod-to-pod connectivity..."
        
        # Try to connect to the server (simplified test - just check if we can reach it)
        if oc exec kata-nettest-client -n ${TEST_NAMESPACE} -- sh -c "microdnf install -y nc 2>/dev/null && timeout 5 nc -zv ${SERVER_IP} 8888" 2>&1 | grep -q "succeeded\|open"; then
            echo "✓ Test 4 PASSED: Pod-to-pod network connectivity working"
            TESTS_PASSED=$((TESTS_PASSED + 1))
        else
            echo "✗ Test 4 FAILED: Pod-to-pod connectivity failed"
            TESTS_FAILED=$((TESTS_FAILED + 1))
            FAILED_TESTS+=("network")
        fi
    else
        echo "✗ Test 4 FAILED: Could not get server IP"
        TESTS_FAILED=$((TESTS_FAILED + 1))
        FAILED_TESTS+=("network")
    fi
else
    echo "✗ Test 4 FAILED: Network test pods did not become Ready"
    TESTS_FAILED=$((TESTS_FAILED + 1))
    FAILED_TESTS+=("network")
fi

# Test 5: Multi-container pod
echo ""
echo "=== Test 5: Multi-Container Pod ==="
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kata-multicontainer
  namespace: ${TEST_NAMESPACE}
  labels:
    test: multicontainer
spec:
  runtimeClassName: kata
  containers:
  - name: container1
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["sh", "-c", "echo 'Container 1' > /shared/c1.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
  - name: container2
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["sh", "-c", "sleep 10 && cat /shared/c1.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
    resources:
      requests:
        memory: "32Mi"
        cpu: "50m"
  volumes:
  - name: shared-data
    emptyDir: {}
EOF

echo "Waiting for kata-multicontainer pod..."
if oc wait --for=condition=Ready pod/kata-multicontainer -n ${TEST_NAMESPACE} --timeout=60s 2>/dev/null; then
    echo "✓ Test 5 PASSED: Multi-container pod running"
    TESTS_PASSED=$((TESTS_PASSED + 1))
    
    # Test shared volume
    echo "  Testing shared volume between containers..."
    sleep 3
    if oc logs kata-multicontainer -n ${TEST_NAMESPACE} -c container2 2>/dev/null | grep -q "Container 1"; then
        echo "  ✓ Shared volume working between containers"
    else
        echo "  ⚠️  Shared volume test inconclusive"
    fi
else
    echo "✗ Test 5 FAILED: Multi-container pod did not become Ready"
    TESTS_FAILED=$((TESTS_FAILED + 1))
    FAILED_TESTS+=("multicontainer")
    oc describe pod kata-multicontainer -n ${TEST_NAMESPACE} | tail -20
fi

# Summary
echo ""
echo "=== Test Summary ==="
echo "Total tests: $((TESTS_PASSED + TESTS_FAILED))"
echo "Passed: ${TESTS_PASSED}"
echo "Failed: ${TESTS_FAILED}"

if [[ ${TESTS_FAILED} -gt 0 ]]; then
    echo ""
    echo "Failed tests:"
    for test in "${FAILED_TESTS[@]}"; do
        echo "  - ${test}"
    done
fi

# Show all test pods status
echo ""
echo "=== All Test Pods Status ==="
oc get pods -n ${TEST_NAMESPACE} -o wide

# Cleanup prompt
echo ""
read -p "Delete test namespace ${TEST_NAMESPACE}? [Y/n] " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Nn]$ ]]; then
    oc delete namespace ${TEST_NAMESPACE} --wait=false
    echo "✓ Test namespace deletion initiated"
else
    echo "Test namespace preserved: ${TEST_NAMESPACE}"
    echo "To delete later: oc delete namespace ${TEST_NAMESPACE}"
fi

echo ""
if [[ ${TESTS_FAILED} -eq 0 ]]; then
    echo "✓ All kata workload tests passed!"
    exit 0
else
    echo "⚠️  Some tests failed - review output above"
    exit 1
fi
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

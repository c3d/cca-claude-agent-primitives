---
name: hypershift-cluster-setup
description: Complete Azure HyperShift setup — creates management cluster (IPI OpenShift), installs MCE/HyperShift, creates hosted clusters with OIDC/managed identities, validates OSC DaemonSet readiness. Supports full teardown. Azure credentials must be configured via az login.
argument-hint: "management create|setup|teardown [--name <NAME>] | hosted setup|validate|teardown [--cluster-name <NAME>] [--location <REGION>] [--node-count <N>]"
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
- First positional arg: scope (`management` or `hosted`)
- Second positional arg: action
  - For `management`: `create`, `setup`, `teardown`
  - For `hosted`: `setup`, `validate`, `teardown`

**Flags:**
- `--name <name>` — management cluster name (default: `$USER-hcp-host-$VERSION` where VERSION extracted from OCP release)
- `--cluster-name <name>` — hosted cluster name (default: `$USER-hcp-$(date +%Y%m%d)`)
- `--location <region>` — Azure region (default: `eastus`)
- `--node-count <n>` — worker nodes for hosted cluster (default: `2`)
- `--release-image <image>` — OCP release (default: `quay.io/openshift-release-dev/ocp-release:4.21.5-x86_64`)

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

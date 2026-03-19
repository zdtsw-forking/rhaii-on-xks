# cert-manager Operator Helm Chart

Deploy Red Hat cert-manager Operator on any Kubernetes cluster without OLM.

## Overview

This chart uses **olm-extractor** to extract manifests directly from Red Hat's OLM bundle, enabling deployment on non-OLM Kubernetes clusters (AKS, CoreWeave) while:

- **Minimizing OCP team burden** - Uses exact manifests from Red Hat's OLM bundles
- **Easy upgrades** - Update bundle with `./scripts/update-bundle.sh <version>`, then `make deploy`
- **Incremental consolidation** - Helm templating can be added gradually
- **No breaking changes** - Only minimal patches for non-OLM environments

## Prerequisites

- `kubectl` configured for your cluster
- `helm` 3.17+ (for Server-Side Apply support)
- `helmfile` installed
- Red Hat account for pull secret

## Quick Start

This chart is part of the [rhaii-on-xks](https://github.com/opendatahub-io/rhaii-on-xks) monorepo and is deployed via the top-level helmfile:

```bash
# From the repo root
make deploy-cert-manager

# Or selectively via helmfile
helmfile apply --selector name=cert-manager-operator

# Verify
kubectl get pods -n cert-manager-operator
kubectl get pods -n cert-manager
```

## Configuration

### Step 1: Get Red Hat Pull Secret

1. Go to: https://console.redhat.com/openshift/install/pull-secret
2. Login and download pull secret
3. Save as `~/pull-secret.txt`

### Step 2: Setup Auth (Choose ONE)

#### Option A: Persistent Podman Auth (Recommended)

```bash
# Copy pull secret to persistent location
mkdir -p ~/.config/containers
cp ~/pull-secret.txt ~/.config/containers/auth.json
```

Then in `environments/default.yaml`:
```yaml
useSystemPodmanAuth: true
```

#### Option B: Pull Secret File

```bash
# Verify pull secret works
podman pull --authfile ~/pull-secret.txt registry.redhat.io/ubi8/ubi-minimal --quiet && echo "Auth works!"
```

Then in `environments/default.yaml`:
```yaml
pullSecretFile: ~/pull-secret.txt
```

### Values

| Parameter | Description | Default |
|-----------|-------------|---------|
| `operatorNamespace` | Operator namespace | `cert-manager-operator` |
| `operandNamespace` | Operand namespace | `cert-manager` |
| `bundle.version` | OLM bundle version | `v1.15.2` |
| `pullSecret.name` | Pull secret name | `redhat-pull-secret` |
| `pullSecret.dockerConfigJson` | Docker config (set by helmfile) | `""` |
| `createInfrastructure` | Create Infrastructure CR (set to `false` on OpenShift, defaults to `true` if not set) | `true` |
| `certManager.enabled` | Create CertManager CR | `true` |

## What Gets Deployed

**Presync hooks** (before Helm install):
- Operand namespace (`cert-manager`)
- CertManager CR (`cluster`)

**Helm install** (with Server-Side Apply):
- cert-manager CRDs + Infrastructure CRD stub - installed from `crds/` directory with SSA
- Infrastructure CR (only on non-OpenShift clusters, controlled by `createInfrastructure`)
- Operator namespace (`cert-manager-operator`)
- Pull secrets (in both namespaces)
- cert-manager ServiceAccounts with `imagePullSecrets` (cert-manager, cert-manager-cainjector, cert-manager-webhook)
- cert-manager Operator deployment + RBAC

> **Note:** Helm 3.17+ is required for Server-Side Apply (SSA) support.

**Post-install** (automatic):
- Operator deploys cert-manager components (controller, webhook, cainjector)
- Operator reconciles ServiceAccounts (adds labels, preserves `imagePullSecrets`)

## Version Compatibility

| Component | Version |
|-----------|---------|
| cert-manager Operator | v1.15.2 |
| cert-manager | v1.15.x |

**OLM Bundle:** `registry.redhat.io/cert-manager/cert-manager-operator-bundle` ([Red Hat Catalog](https://catalog.redhat.com/en/software/container-stacks/detail/64f1ad6f3af5362f09c9ce16))

## Verify Installation

```bash
# Check operator
kubectl get pods -n cert-manager-operator

# Check cert-manager components
kubectl get pods -n cert-manager

# Check CertManager CR
kubectl get certmanager cluster -o yaml
```

## Create an Issuer

```bash
# Self-signed ClusterIssuer
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF
```

## Uninstall

```bash
./scripts/cleanup.sh
```

## Upgrade

```bash
# (Optional) Update to a new bundle version
./scripts/update-bundle.sh <version>

# Deploy
make deploy
```

## Update Pull Secret

Personal Red Hat pull secrets and tokens typically expire (yearly). Registry
Service Accounts created via the [Red Hat terms-based registry](https://access.redhat.com/terms-based-registry/)
do not expire and are recommended for production (see Section 1.3 of the
[deployment guide](../../docs/deploying-llm-d-on-managed-kubernetes.md)).

To update expiring credentials:

```bash
# Option A: Using system podman auth (after re-login)
podman login registry.redhat.io
./scripts/update-pull-secret.sh

# Option B: Using pull secret file
./scripts/update-pull-secret.sh ~/new-pull-secret.txt

# Restart pods to use new secret
kubectl rollout restart deployment -n cert-manager --all
```

## Platform Compatibility

This chart works on both OpenShift and non-OpenShift Kubernetes clusters:

### Non-OpenShift Clusters (AKS, CoreWeave, etc.)

The chart includes workarounds for running the Red Hat operator outside OpenShift:

1. **Infrastructure CRD/CR stub** - The operator requires `infrastructures.config.openshift.io` API. The chart creates a stub Infrastructure CR (default behavior when `createInfrastructure` is not set or set to `true`).
2. **CertManager CR pre-creation** - Created in presync to avoid race conditions
3. **OLM pattern replacement** - `olm.targetNamespaces` handled via olm-extractor's `--watch-namespace=""` flag
4. **Pre-created ServiceAccounts with imagePullSecrets** - On non-OpenShift clusters, there's no global pull secret mechanism. The chart pre-creates cert-manager ServiceAccounts with `imagePullSecrets` configured. The operator preserves these when it reconciles.

### OpenShift Clusters

On OpenShift, the Infrastructure CR already exists and is managed by the cluster. Set `createInfrastructure: false` in your values to skip creating it:

```yaml
# values.yaml or root helmfile
certManager:
  createInfrastructure: false
```

## File Structure

```
charts/cert-manager-operator/
├── Chart.yaml
├── values.yaml                  # Default values
├── helmfile.yaml.gotmpl         # Deploy with: helmfile apply
├── .helmignore
├── environments/
│   └── default.yaml             # User config
├── crds/                        # cert-manager CRDs + Infrastructure stub (installed by Helm with SSA)
├── templates/
│   ├── deployment-*.yaml                  # Operator deployment
│   ├── pull-secret.yaml                   # Registry pull secrets
│   ├── serviceaccounts-cert-manager.yaml  # Operand SAs with imagePullSecrets
│   └── *.yaml                             # RBAC, ServiceAccount, etc.
└── scripts/
    ├── cleanup.sh               # Uninstall and delete CRDs
    ├── update-bundle.sh         # Update to new bundle version
    ├── update-pull-secret.sh    # Update expired pull secret
    └── post-install-message.sh  # Post-install instructions
```

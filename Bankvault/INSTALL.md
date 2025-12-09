# Bankvault Helm Chart - Installation Guide

This guide walks through the complete process of converting your kubectl commands into an automated Helm chart deployment.

## What This Chart Does

Automatically deploys:
1. **Vault Operator** (v1.23.1) - Manages Vault Custom Resources
2. **Vault Secrets Webhook** (v1.22.2) - Injects secrets into pods
3. **Vault Instance** (3 replicas with Raft storage) - Secret management engine
4. **RBAC & ServiceAccounts** - Proper authentication and authorization
5. **Unsealing** - Automatic unsealing using Kubernetes secrets

## Quick Start

### Step 1: Update Helm Dependencies

```bash
cd Bankvault
helm dependency update
```

This pulls the latest versions of vault-operator and vault-secrets-webhook charts.

### Step 2: Install in Default Namespace

```bash
helm install bankvault . -n default
```

### Step 3: Monitor Installation

```bash
# Watch pods coming up
kubectl get pods -w

# Check Vault CR status
kubectl get vaults -w

# Verify all are ready (takes 2-3 minutes)
kubectl get pods | grep vault
```

### Step 4: Access Vault

```bash
# Port forward to UI
kubectl port-forward vault-0 8200:8200

# Get root token
kubectl get secret bank-vaults -o jsonpath='{.data.vault-root}' | base64 -d
```

Access UI at: `https://localhost:8200/ui`

## Advanced Usage

### Custom Namespace Deployment

```bash
kubectl create namespace my-vault
helm install bankvault . -n my-vault
```

### Override Configuration

Create a `custom-values.yaml`:

```yaml
vault:
  size: 5                           # 5 replicas instead of 3
  image: hashicorp/vault:1.15.0    # Newer version
  storage: 10Gi                     # More storage
  
vault-secrets-webhook:
  replicaCount: 3                   # More webhook replicas
```

Then install:

```bash
helm install bankvault . -f custom-values.yaml
```

### Upgrade Vault Version

```bash
helm upgrade bankvault . --set vault.image=hashicorp/vault:1.15.0
```

## File Breakdown

| File | Purpose |
|------|---------|
| `Chart.yaml` | Chart metadata & dependencies (vault-operator 1.23.1, vault-secrets-webhook 1.22.2) |
| `values.yaml` | Default configuration values for size, image, storage, and secrets |
| `rbac.yaml` | ServiceAccount and ClusterRoleBinding for Vault auth |
| `cr-raft.yaml` | Vault Custom Resource with Raft storage backend |
| `README.md` | User-facing documentation |
| `INSTALL.md` | This file |

## What Gets Created

```
Namespace: <your-namespace>
├── Deployment
│   ├── vault-operator (from dependency)
│   └── vault-secrets-webhook (from dependency, 2 replicas)
├── StatefulSet
│   └── vault (3 replicas with Raft)
├── ServiceAccount
│   ├── vault
│   ├── vault-secrets-webhook (from dependency)
│   └── vault-operator (from dependency)
├── Service
│   ├── vault (ClusterIP)
│   ├── vault-operator (from dependency)
│   └── vault-secrets-webhook (from dependency)
├── PersistentVolumeClaims
│   ├── vault-raft-vault-0
│   ├── vault-raft-vault-1
│   └── vault-raft-vault-2
├── Secret
│   ├── vault-unseal-keys
│   ├── vault-tls
│   ├── bank-vaults (root token + keys)
│   └── vault-secrets-webhook-webhook-tls (from dependency)
└── ConfigMaps
    └── (Various Vault configs)
```

## Verification Steps

### 1. Check Operator is Running

```bash
kubectl get deployment vault-operator
kubectl logs -l app.kubernetes.io/name=vault-operator
```

### 2. Check Webhook is Running

```bash
kubectl get deployment vault-secrets-webhook
kubectl get pods -l app.kubernetes.io/name=vault-secrets-webhook
```

### 3. Check Vault Stateful Set

```bash
kubectl get statefulset vault
kubectl get pods vault-0 vault-1 vault-2
```

### 4. Verify Vault is Unsealed

```bash
kubectl exec vault-0 -- vault status
# Should show: Sealed: false
```

### 5. Check External Config Applied

```bash
kubectl exec vault-0 -- vault auth list
# Should show: kubernetes/ enabled

kubectl exec vault-0 -- vault secrets list
# Should show: secret/ enabled (kv2)
```

### 6. List Seeded Secrets

```bash
kubectl exec vault-0 -- vault list secret/data
# Should show: accounts/aws, dockerrepo, mysql
```

## Using Secrets in Your Pods

Example deployment with injected secrets:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        vault.security.banzaicloud.io/mutate: "true"
        vault.security.banzaicloud.io/vault-addr: "https://vault:8200"
        vault.security.banzaicloud.io/vault-role: "default"
        vault.security.banzaicloud.io/vault-path: "kubernetes"
        vault.security.banzaicloud.io/vault-skip-verify: "true"
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
        - name: AWS_KEY
          value: "vault:secret/data/accounts/aws#AWS_ACCESS_KEY_ID"
        - name: DOCKER_USER
          value: "vault:secret/data/dockerrepo#DOCKER_REPO_USER"
```

## Uninstall

```bash
# Remove Helm release
helm uninstall bankvault -n <namespace>

# Clean up PVCs (if you want to delete data)
kubectl delete pvc -l app.kubernetes.io/name=vault -n <namespace>

# Remove namespace (optional)
kubectl delete namespace <namespace>
```

## Troubleshooting

### Vault Pods Stuck in Pending

```bash
# Check PVC status
kubectl describe pvc vault-raft-vault-0

# Check node capacity
kubectl top nodes
kubectl describe nodes
```

### Vault Not Unsealing

```bash
# Check unsealing logs
kubectl logs vault-0

# Check unseal keys exist
kubectl get secret bank-vaults -o yaml

# Manually unseal if needed
kubectl exec vault-0 -- vault status
```

### Webhook Not Injecting

```bash
# Check webhook running
kubectl get pods -l app.kubernetes.io/name=vault-secrets-webhook

# Check webhook logs
kubectl logs -l app.kubernetes.io/name=vault-secrets-webhook

# Verify pod has correct annotations
kubectl describe pod <pod-name> | grep vault
```

## Key Differences from Manual kubectl

| Manual kubectl | Helm Chart |
|---|---|
| Apply YAML files one by one | Single `helm install` command |
| Manual image version management | Configured in values.yaml |
| Manual secret seeding | Automatic via externalConfig |
| Manual RBAC setup | Included templates |
| Manual dependency tracking | Managed by Helm dependencies |
| Hard to upgrade | `helm upgrade` simple upgrade path |
| Hard to customize per environment | values.yaml for different environments |

## Next Steps

1. **Test in Development**: Deploy to dev cluster
2. **Customize**: Adjust vault size, storage, secrets in values.yaml
3. **Document**: Create environment-specific values files
4. **ArgoCD**: Integrate with ArgoCD for GitOps deployment
5. **Backup**: Set up Velero integration for backup/restore

## Additional Resources

- Bank-Vaults: https://github.com/banzaicloud/bank-vaults
- Vault Docs: https://www.vaultproject.io/docs
- Helm Documentation: https://helm.sh/docs/

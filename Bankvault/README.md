# Bankvault Helm Chart

A comprehensive Helm chart for deploying HashiCorp Vault with bank-vaults operator and webhook integration on Kubernetes clusters.

## Overview

This chart automates the deployment of:
- **Vault Operator**: Manages Vault Custom Resources (CRs)
- **Vault Secrets Webhook**: Injects Vault secrets into pods via annotations
- **Vault Instance**: HashiCorp Vault running with Raft storage backend

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+

## Installation

### 1. Add the Bank-Vaults Helm Repository

```bash
helm repo add banzaicloud https://charts.banzaicloud.com
helm repo update
```

### 2. Install the Chart

**In the default namespace:**
```bash
helm install bankvault ./Bankvault
```

**In a custom namespace (create if needed):**
```bash
kubectl create namespace vault-infra
helm install bankvault ./Bankvault -n vault-infra
```

### 3. Verify Installation

```bash
# Check deployments
kubectl get deployments -n <namespace>

# Check Vault status
kubectl get vaults -n <namespace>

# Check pods
kubectl get pods -n <namespace>

# Check Vault is unsealed
kubectl logs -n <namespace> vault-0 | grep "Unsealed"
```

## Configuration

### Default Values

The chart comes with sensible defaults. Key configurable values in `values.yaml`:

```yaml
vault:
  size: 3                    # Number of Vault replicas
  image: hashicorp/vault:1.14.1  # Vault version
  storage: 1Gi              # PVC storage size
  
  secrets:
    aws:
      accessKeyId: secretId
      secretAccessKey: s3cr3t
    dockerrepo:
      user: dockerrepouser
      password: dockerrepopassword
    mysql:
      password: s3cr3t
```

### Custom Values

Override values during installation:

```bash
helm install bankvault ./Bankvault \
  --set vault.size=5 \
  --set vault.image=hashicorp/vault:1.15.0 \
  --set vault.storage=5Gi
```

Or use a custom values file:

```bash
helm install bankvault ./Bankvault -f custom-values.yaml
```

## Chart Structure

```
Bankvault/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default configuration values
├── README.md              # This file
├── rbac.yaml              # ServiceAccount and RBAC for Vault
├── cr-raft.yaml          # Vault Custom Resource definition
└── templates/            # Helm templates directory (if needed)
```

## Features

✅ **Automated Unsealing**: Uses Kubernetes secrets for automatic unsealing
✅ **Raft Storage**: Built-in distributed storage backend
✅ **TLS Support**: Secure communication with generated certificates
✅ **Secret Injection**: Automatic secret injection via annotations
✅ **Kubernetes Auth**: Native Kubernetes authentication method
✅ **Multi-Replica HA**: Highly available Vault deployment
✅ **Velero Ready**: Compatible with Velero backup and restore

## Accessing Vault UI

### Port Forward to Vault UI

```bash
kubectl port-forward -n <namespace> vault-0 8200:8200
```

Then access the UI at: `https://localhost:8200/ui`

### Get Root Token

The root token is stored in a Kubernetes secret:

```bash
kubectl get secret -n <namespace> bank-vaults -o jsonpath='{.data.vault-root}' | base64 -d
```

## Using Vault Secrets

### Example: Inject AWS Credentials

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  annotations:
    vault.security.banzaicloud.io/vault-addr: "https://vault.<namespace>:8200"
    vault.security.banzaicloud.io/vault-role: "default"
    vault.security.banzaicloud.io/vault-path: "kubernetes"
    vault.security.banzaicloud.io/vault-skip-verify: "true"  # Dev only
spec:
  serviceAccountName: default
  containers:
  - name: app
    image: my-app:latest
    env:
    - name: AWS_ACCESS_KEY_ID
      value: "vault:secret/data/accounts/aws#AWS_ACCESS_KEY_ID"
    - name: AWS_SECRET_ACCESS_KEY
      value: "vault:secret/data/accounts/aws#AWS_SECRET_ACCESS_KEY"
```

## Uninstallation

```bash
helm uninstall bankvault -n <namespace>
```

Note: This removes the Helm release but Vault data in PVCs will persist. To fully clean up:

```bash
kubectl delete pvc -l app.kubernetes.io/name=vault -n <namespace>
```

## Troubleshooting

### Check Vault Status

```bash
kubectl describe vault vault -n <namespace>
```

### View Vault Logs

```bash
kubectl logs -n <namespace> vault-0
kubectl logs -n <namespace> vault-1
kubectl logs -n <namespace> vault-2
```

### Check Webhook Logs

```bash
kubectl logs -n <namespace> -l app.kubernetes.io/name=vault-secrets-webhook
```

### Verify Unseal Keys

```bash
kubectl get secret -n <namespace> bank-vaults -o yaml
```

## Common Issues

### Vault Pods Not Starting
- Check PVC: `kubectl get pvc -n <namespace>`
- Check node resources: `kubectl top nodes`
- Check events: `kubectl describe pod vault-0 -n <namespace>`

### Webhook Not Injecting Secrets
- Verify webhook deployment: `kubectl get deployment vault-secrets-webhook -n <namespace>`
- Check annotations on pod
- Verify Vault Kubernetes auth is configured

## Support & Documentation

- [Bank-Vaults GitHub](https://github.com/banzaicloud/bank-vaults)
- [Vault Documentation](https://www.vaultproject.io/docs)
- [Bank-Vaults Helm Charts](https://github.com/banzaicloud/bank-vaults/tree/main/charts)

## License

See LICENSE file in the repository root.

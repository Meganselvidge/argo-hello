# ArgoCD Bankvault Installation Guide

ArgoCD kommer automatiskt att installera och hantera Bankvault Helm-installationen på ditt Kubernetes-kluster.

## Förklarning av ArgoCD Applications

Tre olika ArgoCD Application YAML-filer har skapats för olika scenarier:

### 1. bankvault-application.yaml (Grundläggande)
**Användning**: Quick-start med inline values
- Installeras i `default` namespace
- Använder standard values inline i Application-specifikationen
- Perfekt för att snabbt testa eller demonstrera

**Applicera:**
```bash
kubectl apply -f argocd/bankvault-application.yaml
```

### 2. bankvault-application-dev.yaml (Development)
**Användning**: Development-miljö med separata values
- Installeras i `default` namespace
- Använder `values-dev.yaml` för dev-konfiguration
- 1 Vault-replik, 1 webhook-replik (mindre resurser)

**Applicera:**
```bash
kubectl apply -f argocd/bankvault-application-dev.yaml
```

### 3. bankvault-application-prod.yaml (Production)
**Användning**: Production-miljö
- Installeras i `vault-infra` namespace
- Använder `values-prod.yaml` för prod-konfiguration
- 3 Vault-repliker, 2 webhook-repliker, 5Gi lagring (HA-setup)

**Applicera:**
```bash
kubectl apply -f argocd/bankvault-application-prod.yaml
```

## Steg-för-steg Installation

### 1. Förutsättningar
```bash
# Kontrollera att ArgoCD är installerat
kubectl get deployment -n argocd argocd-application-controller

# Kontrollera att Git-repo är tillgänglig för ArgoCD
# ArgoCD behöver åtkomst till: https://github.com/Meganselvidge/argo-hello.git
```

### 2. Skapa Application (välj en av tre)

**Option A: Snabbtesta med inline values**
```bash
kubectl apply -f argocd/bankvault-application.yaml
```

**Option B: Development setup**
```bash
kubectl apply -f argocd/bankvault-application-dev.yaml
```

**Option C: Production setup**
```bash
kubectl apply -f argocd/bankvault-application-prod.yaml
```

### 3. Övervaka Installation

**Via kubectl:**
```bash
# Se Application-status
kubectl get application -n argocd bankvault

# Se detaljerad status
kubectl describe application -n argocd bankvault

# Se Real-time synkronisering
kubectl get application -n argocd bankvault -w
```

**Via ArgoCD CLI:**
```bash
# Logga in (om inte redan inloggad)
argocd login localhost:6443 --insecure

# Se status
argocd app get bankvault
```

**Via ArgoCD UI:**
```bash
# Port-forward till ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Öppna browser: https://localhost:8080
# Logga in och se "bankvault" application
```

### 4. Verifiera Installation

```bash
# Se om alla resurser är deployade
kubectl get pods -n default | grep vault
kubectl get vault -n default

# Se om Vault är unsealed
kubectl exec vault-0 -n default -- vault status

# Se Webhook status
kubectl get deployment vault-secrets-webhook -n default
```

## ArgoCD Application Explained

Varje Application-fil innehåller:

```yaml
metadata:
  name: bankvault              # Application-namn i ArgoCD
  namespace: argocd            # Måste vara argocd namespace

spec:
  project: default             # ArgoCD project (default finns alltid)
  
  source:
    repoURL: <git-repo>        # Git-repo att synka från
    targetRevision: main       # Branch att synka från
    path: Bankvault            # Path till Helm-chart i repo
    helm:
      releaseName: bankvault   # Helm release-namn
      valueFiles:              # Values-filer att använda
      - values-dev.yaml
  
  destination:
    server: <cluster>          # Target Kubernetes cluster
    namespace: default          # Target namespace
  
  syncPolicy:
    automated:                 # Automatisk synkronisering
      prune: true              # Ta bort resurser som inte är i Git
      selfHeal: true           # Åtgärda drift från Git
```

## Miljö-specifik Konfiguration

### Development (values-dev.yaml)
```yaml
vault:
  size: 1              # En Vault-instans
  storage: 1Gi         # Minimal lagring
webhook:
  replicaCount: 1      # En webhook-instans
```

### Production (values-prod.yaml)
```yaml
vault:
  size: 3              # HA setup (3 repliker)
  storage: 5Gi         # Mer lagring
webhook:
  replicaCount: 2      # 2 webhook-instanser för redundans
```

## Automatisk Synkronisering

ArgoCD kommer automatiskt att:

1. **Övervaka Git-repo** - Kollar varje minut om något har ändrats
2. **Detektera ändringar** - Om Bankvault-filerna ändras i Git
3. **Synka automatiskt** - Applicerar ändringarna på klustret
4. **Reparera drift** - Om någon manuellt ändrar resurser återställs de till Git-tillståndet

**Inaktivera automatisk synkronisering** (manual sync):
```yaml
syncPolicy:
  syncOptions:
  - PruneLast=true
  # Automatisk synkronisering är INAKTIVERAD
```

## Uppdatera Vault Configuration

### Via Git (rekommenderat)

1. Ändra `values-prod.yaml`:
```yaml
vault:
  size: 5  # Öka repliker från 3 till 5
  storage: 10Gi  # Öka lagring
```

2. Pusha till GitHub:
```bash
git add Bankvault/values-prod.yaml
git commit -m "Increase Vault replicas to 5"
git push
```

3. ArgoCD detekterar och applicerar automatiskt!

### Manuell Sync (om automation är disabled)

```bash
argocd app sync bankvault
```

## Ta bort Bankvault

### Ta bort Application (behåller resurser)
```bash
kubectl delete application -n argocd bankvault
```

### Ta bort allt inklusive Vault (cascade delete)
```bash
kubectl delete application -n argocd bankvault --cascade=foreground
```

Detta kommer ta bort:
- Vault StatefulSet
- Vault-configurer
- Webhook Deployment
- Alla relaterade resurser

PVC:erna (Persistent Volume Claims) behålls för datasäkerhet. Ta bort manuellt om nödvändigt:
```bash
kubectl delete pvc -l app.kubernetes.io/name=vault
```

## Troubleshooting

### Application är OutOfSync

```bash
# Se vad som är annorlunda
kubectl describe application -n argocd bankvault

# Synka manuellt
kubectl patch application -n argocd bankvault -p '{"metadata":{"finalizers":["resources-finalizer.argocd.argoproj.io"]}}' --type merge
argocd app sync bankvault
```

### Helm values appliceras inte

```bash
# Verifiera att values-filerna finns i repo
ls Bankvault/values-*.yaml

# Kontrollera Application-specifikationen
kubectl get application -n argocd bankvault -o yaml | grep -A 10 "valueFiles"
```

### Repository access problem

ArgoCD kan inte nå Git-repo:
```bash
# Verifiera Git-repo är tillgänglig
kubectl get repository -n argocd

# Skapa deploy key i GitHub
# 1. SSH key: argocd repo add https://github.com/Meganselvidge/argo-hello.git
```

## Vanliga ArgoCD Commands

```bash
# Logga in
argocd login argocd-server.argocd:443 --insecure

# Lista alla applications
argocd app list

# Se application-status
argocd app get bankvault

# Synka application
argocd app sync bankvault

# Se synkhistorik
argocd app history bankvault

# Rollback till tidigare version
argocd app rollback bankvault 1

# Ange application-manuell sync
argocd app set bankvault --sync-policy none
argocd app set bankvault --sync-policy automated
```

## Best Practices

✅ **DO:**
- Lagra alla secrets i Git via sealed-secrets eller external-secrets
- Använd miljö-specifika values-filer (dev/prod)
- Aktivera automated sync med self-healing
- Regelbundet uppdatera Helm-chart versioner
- Använd branch-protection i GitHub

❌ **DON'T:**
- Ändra resurser manuellt på klustret
- Lagra plaintext-secrets i Git
- Inaktivera automatisk synkronisering utan god anledning
- Blanda dev och prod i samma Application

## Nästa Steg

1. **Applicera en av Applications** - Start med dev eller quick-start
2. **Verifiera i ArgoCD UI** - Se den automatiska synkroniseringen
3. **Testa GitOps workflow** - Ändra values-filer och se automatisk uppdatering
4. **Säkra Secrets** - Implementera sealed-secrets för hemliga värden
5. **Monitoring** - Integrera med Prometheus/Grafana för övervakning

## Resurser

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Application CRD](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/)
- [Helm Integration](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)
- [Bank-Vaults](https://github.com/banzaicloud/bank-vaults)

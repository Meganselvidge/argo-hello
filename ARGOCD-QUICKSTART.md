# ArgoCD + Bankvault - Snabbstart Guide (Svenska)

## Vad du precis fick

Du har nu ett komplett GitOps-setup för Vault med ArgoCD:

```
argocd/
├── bankvault-application.yaml       # Snabbtesta
├── bankvault-application-dev.yaml   # Development
├── bankvault-application-prod.yaml  # Production
└── README-ARGOCD.md                 # Detaljerad guide
```

## Snabbstart (30 sekunder)

### 1. Applicera en Application

Välj EN av dessa:

**Snabbtesta:**
```bash
kubectl apply -f argocd/bankvault-application.yaml
```

**Eller Development:**
```bash
kubectl apply -f argocd/bankvault-application-dev.yaml
```

**Eller Production:**
```bash
kubectl apply -f argocd/bankvault-application-prod.yaml
```

### 2. Se status i ArgoCD UI

```bash
# Port-forward till ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Öppna: https://localhost:8080
# Logga in (default user: admin, password hittas med:)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Du ser nu en "bankvault" tile som automatiskt installerar Vault!

### 3. Verifiera Vault är uppe

```bash
# Vänta tills alla pods är running
kubectl get pods -w

# Kontrollera Vault status
kubectl exec vault-0 -- vault status
```

## Hur det fungerar

```
┌──────────────────┐
│   GitHub Repo    │  ← Du committar ändringar här
└────────┬─────────┘
         │
         ↓
┌──────────────────────────────────────┐
│   ArgoCD (i argocd namespace)        │
│  - Kollar repo var 3:e sekund        │
│  - Ser ändringar i Bankvault/        │
│  - Applicerar Helm-chart automatiskt │
└────────┬─────────────────────────────┘
         │
         ↓
┌──────────────────────────────────────┐
│   Ditt Kubernetes Cluster            │
│  - Vault StatefulSet (3 repliker)   │
│  - Webhook Deployment               │
│  - Operator Deployment              │
│  - Alla resurser synkade             │
└──────────────────────────────────────┘
```

## Vad ArgoCD gör automatiskt

✅ **Installerar** - Skapar alla Vault-resurser  
✅ **Uppdaterar** - Om du ändrar values-filer  
✅ **Healdar** - Om någon manuellt ändrar något återställs det  
✅ **Prunar** - Tar bort resurser som inte längre finns i Git  

## GitOps Workflow

### Uppdatera Vault Size

1. **Redigera values-filen:**
```bash
# Öppna filen i din editor
vim Bankvault/values-prod.yaml

# Ändra:
vault:
  size: 5  # från 3 till 5
```

2. **Commit och push:**
```bash
git add Bankvault/values-prod.yaml
git commit -m "Scale Vault to 5 replicas"
git push
```

3. **ArgoCD gör resten automatiskt!**
   - Detekterar ändringen
   - Upgradar Helm release
   - Scaled upp Vault till 5 repliker

Du behöver inte köra helm upgrade manuellt!

## Miljö-konfigurationer

### values-dev.yaml (Development)
- 1 Vault-replik
- 1 Webhook-replik
- 1Gi lagring
- Installeras i `default` namespace

### values-prod.yaml (Production)
- 3 Vault-repliker (HA)
- 2 Webhook-repliker
- 5Gi lagring
- Installeras i `vault-infra` namespace

## Byt mellan Dev och Prod

### Från Dev till Prod
```bash
# Ta bort dev
kubectl delete application -n argocd bankvault-dev

# Applicera prod
kubectl apply -f argocd/bankvault-application-prod.yaml
```

## Vanliga Kommandon

```bash
# Se Application-status
kubectl get application -n argocd bankvault

# Se detaljerad info
kubectl describe application -n argocd bankvault

# Manuell synkronisering (om automation är disabled)
argocd app sync bankvault

# Se Vault pods
kubectl get pods -l app.kubernetes.io/name=vault

# Fjärr-ansluta till Vault UI
kubectl port-forward vault-0 8200:8200
# https://localhost:8200/ui

# Hämta root token
kubectl get secret bank-vaults -o jsonpath='{.data.vault-root}' | base64 -d
```

## Troubleshooting

### Application är "OutOfSync"
ArgoCD kan inte applicera ändringar:
```bash
# Manuell synkronisering
kubectl patch application -n argocd bankvault -p '{"status":{"operationState":null}}' --type merge

# Eller via ArgoCD CLI
argocd app sync bankvault --force
```

### Pods startar inte
```bash
# Se events
kubectl describe pod vault-0

# Se logs
kubectl logs vault-0

# Kontrollera resurser
kubectl top nodes
```

### Webhook injicerar inte secrets
```bash
# Verifiera webhook kör
kubectl get pods -l app.kubernetes.io/name=vault-secrets-webhook

# Se logs
kubectl logs -l app.kubernetes.io/name=vault-secrets-webhook

# Kontrollera webhook-konfiguration
kubectl get mutatingwebhookconfigurations
```

## Nästa Steg

1. ✅ **Applicera en Application** - Start med dev
2. ✅ **Testa GitOps** - Ändra values-filer och se automatisk uppdatering
3. ✅ **Integrera secrets** - Börja använda Vault-secrets i pods
4. ⭕ **Säkra sensitive data** - Använd sealed-secrets för gemens i Git
5. ⭕ **Monitoring** - Sätt upp Prometheus/Grafana

## Filer du committar till Git

```bash
# Alla dessa filer pushas till GitHub
git add argocd/
git add Bankvault/
git commit -m "Add ArgoCD Bankvault automation"
git push
```

ArgoCD hämtar sedan dessa automatiskt från GitHub och installerar dem!

---

**Behöver du mer hjälp?** Se `argocd/README-ARGOCD.md` för detaljerad dokumentation.

# vault-gitops

GitOps repo for deploying HashiCorp Vault across 3 AKS clusters (dev / staging / prod)
using a vendored Helm chart and ArgoCD.

## Structure

```
vault-gitops/
├── charts/vault/          Vendored HashiCorp Vault Helm chart (v0.28.0)
├── envs/
│   ├── values-dev.yaml    Dev Helm overrides
│   ├── values-staging.yaml
│   ├── values-prod.yaml
│   ├── dev/               Dev bootstrap secret
│   ├── staging/           Staging bootstrap secret (encrypt before committing)
│   └── prod/              Prod bootstrap secret (encrypt before committing)
├── jobs/
│   ├── init-unseal/       RBAC + init Job + unseal Job
│   └── bootstrap/         ConfigMap script + bootstrap Job
└── argocd/                One Application per cluster
```

## Prerequisites

- ArgoCD installed and connected to all 3 AKS clusters
- `kubectl apply` access to the ArgoCD namespace

## Setup

### 1. Replace placeholders in ArgoCD apps

Edit `argocd/app-*.yaml` and replace:
- `<your-org>` with your GitHub org/user
- `<AKS-*-API-SERVER>` with the actual AKS API server URL per cluster

Retrieve AKS API server URLs:
```bash
argocd cluster list
```

### 2. Encrypt staging/prod secrets

Before committing staging/prod bootstrap secrets, encrypt them:
```bash
# Using SealedSecrets
kubeseal --format yaml < envs/staging/bootstrap-secret.yaml \
  > envs/staging/bootstrap-secret-sealed.yaml

# Or using SOPS
sops --encrypt envs/prod/bootstrap-secret.yaml \
  > envs/prod/bootstrap-secret.enc.yaml
```

### 3. Register clusters and apply ArgoCD apps

```bash
argocd cluster add <aks-dev-context>
argocd cluster add <aks-staging-context>
argocd cluster add <aks-prod-context>

kubectl apply -f argocd/app-dev.yaml
kubectl apply -f argocd/app-staging.yaml
kubectl apply -f argocd/app-prod.yaml
```

### 4. Monitor sync

```bash
argocd app list
argocd app get vault-dev
argocd app get vault-staging
argocd app get vault-prod
```

## ArgoCD sync order (per cluster)

```
Sync phase
  ├── Vault Helm chart deployed (pods, services, injector)
  ├── RBAC (serviceaccount, role, rolebinding)
  ├── ConfigMap (bootstrap scripts)
  └── Bootstrap Secret

PostSync phase
  ├── Job: vault-init     → initializes Vault, saves keys to secret/vault-init-keys
  ├── Job: vault-unseal   → unseals all pods using saved keys
  └── Job: vault-bootstrap → enables KV v2, writes secrets, configures K8s auth, revokes root token
```

## Upgrading the Vault chart

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm pull hashicorp/vault --version <NEW_VERSION> --untar --untardir charts/
git add charts/vault/
git commit -m "chore: upgrade vault chart to v<NEW_VERSION>"
git push
```

ArgoCD auto-syncs and rolls out the upgrade.

## Security notes

| Concern | Mitigation |
|---|---|
| Unseal keys in K8s Secret | Back up externally, then delete secret/vault-init-keys after bootstrap |
| Root token | Revoked automatically at end of bootstrap job |
| Prod secrets in Git | Must be encrypted with SealedSecrets or SOPS |
| Vault UI | Disabled in prod (values-prod.yaml) |
| Audit logging | Enabled in staging and prod via ENABLE_AUDIT=true |
# vault

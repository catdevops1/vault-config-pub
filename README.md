# HashiCorp Vault + External Secrets Operator

Production-tested secrets management for bare-metal Kubernetes using HashiCorp Vault, External Secrets Operator, and AWS KMS auto-unseal. Fully GitOps managed via ArgoCD.

## Stack

- **HashiCorp Vault** v1.18.3 — secrets store
- **External Secrets Operator** v0.10.7 — syncs Vault secrets to Kubernetes
- **AWS KMS** — auto-unseal
- **Longhorn** — persistent storage for Vault data
- **ArgoCD** — GitOps deployment

## Architecture

```
Git repo (ExternalSecret manifests)
      │
      ▼ ArgoCD syncs
ExternalSecret CRDs in cluster
      │
      ▼ ESO reconciles every hour
Vault KV store (secret values)
      │
      ▼ ESO creates K8s secrets
App pods (unchanged — read secrets normally)
```

```
Vault Pod Restart Flow:
      │
      ▼
Vault calls AWS KMS
      │
      ▼
KMS decrypts master key
      │
      ▼
Vault unseals automatically
No human intervention required
```

## Repository Structure

```
vault/
├── namespace.yaml              # Vault namespace
├── pvc.yaml                    # Longhorn PVC for Vault data
├── helm-values.yaml            # Vault Helm chart values
└── README.md

external-secrets/
├── helm-values.yaml            # ESO Helm chart values
└── cluster-secret-store.yaml   # ClusterSecretStore → Vault

external-secrets-configs/
└── job-tracker/
    └── external-secret.yaml    # Example: postgres-secret + backend-secret

argocd/applications/
├── vault-app.yaml
└── external-secrets-app.yaml
```

## Vault Secret Paths

```
secret/job-tracker/postgres
  └── POSTGRES_PASSWORD

secret/job-tracker/backend
  └── DB_PASSWORD, JWT_SECRET
```

## How Auto-Unseal Works

Vault encrypts its master key using AWS KMS. On every pod restart Vault calls KMS to decrypt the master key automatically — no human intervention needed.

AWS credentials are stored in a manually bootstrapped K8s secret (`vault-aws-credentials`). This is the only secret not managed by Vault itself — the "secret zero".

## How ESO Authentication Works

ESO uses Vault's Kubernetes auth method:

```
ESO pod presents K8s service account JWT token
      │
      ▼
Vault verifies token with K8s API
      │
      ▼
Vault issues short-lived token (TTL: 1h)
      │
      ▼
ESO reads secrets from Vault KV store
      │
      ▼
ESO creates/refreshes K8s secrets in app namespaces
```

## Prerequisites

- Kubernetes cluster (v1.24+)
- Longhorn storage class
- ArgoCD
- AWS account with KMS key
- IAM user with `kms:Encrypt`, `kms:Decrypt`, `kms:DescribeKey` permissions

## Deployment

### 1. Create AWS KMS Key

```
AWS Console → KMS → Customer managed keys → Create key
Key type: Symmetric
Key usage: Encrypt and decrypt
Alias: vault-unseal
```

### 2. Create IAM User and Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:<YOUR-REGION>:<YOUR-ACCOUNT-ID>:key/<YOUR-KEY-ID>"
    }
  ]
}
```

### 3. Update helm-values.yaml

Replace placeholders in `vault/helm-values.yaml`:
```yaml
seal "awskms" {
  region     = "<YOUR-AWS-REGION>"
  kms_key_id = "<YOUR-KMS-KEY-ARN>"
}
```

### 4. Bootstrap (one-time manual steps)

```bash
# Create namespace
kubectl create namespace vault

# Create AWS credentials secret (secret zero)
kubectl create secret generic vault-aws-credentials \
  -n vault \
  --from-literal=AWS_ACCESS_KEY_ID=<your-key-id> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<your-secret-key>

# Apply ArgoCD apps
kubectl apply -f argocd/applications/vault-app.yaml
kubectl apply -f argocd/applications/external-secrets-app.yaml

# Wait for Vault pod
kubectl get pods -n vault -w

# Initialize Vault (one time only — save output securely in password manager)
kubectl exec -n vault vault-0 -- vault operator init
```

### 5. Configure Vault

```bash
# Login with root token
kubectl exec -it -n vault vault-0 -- vault login

# Enable KV v2 secrets engine
kubectl exec -n vault vault-0 -- vault secrets enable -path=secret kv-v2

# Enable Kubernetes auth
kubectl exec -n vault vault-0 -- vault auth enable kubernetes

# Configure Kubernetes auth
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/config \
  kubernetes_host="https://<YOUR-K8S-API-SERVER>:6443"

# Create ESO policy
kubectl exec -n vault vault-0 -- vault policy write external-secrets - << EOF
path "secret/data/*" {
  capabilities = ["read"]
}
path "secret/metadata/*" {
  capabilities = ["read"]
}
EOF

# Create Kubernetes auth role
kubectl exec -n vault vault-0 -- vault write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=external-secrets \
  ttl=1h
```

### 6. Apply ClusterSecretStore and ExternalSecrets

```bash
kubectl apply -f external-secrets/cluster-secret-store.yaml
kubectl apply -f external-secrets-configs/job-tracker/external-secret.yaml
```

### 7. Load secrets into Vault

```bash
kubectl exec -it -n vault vault-0 -- vault login

kubectl exec -n vault vault-0 -- vault kv put secret/job-tracker/postgres \
  POSTGRES_PASSWORD=<your-password>

kubectl exec -n vault vault-0 -- vault kv put secret/job-tracker/backend \
  DB_PASSWORD=<your-password> \
  JWT_SECRET=<your-jwt-secret>
```

## Verification

```bash
# Check Vault status
kubectl exec -n vault vault-0 -- vault status

# Check ESO ClusterSecretStore
kubectl get clustersecretstore vault-backend

# Check ExternalSecrets
kubectl get externalsecrets -A

# Check ArgoCD apps
kubectl get applications -n argocd
```

Expected output:
```
NAME                    SYNC STATUS   HEALTH STATUS
vault                   Synced        Healthy
external-secrets        Synced        Healthy
external-secrets-store  Synced        Healthy
external-secrets-config Synced        Healthy
```

## Updating a Secret

```bash
# Update value in Vault
kubectl exec -n vault vault-0 -- vault kv put secret/job-tracker/postgres \
  POSTGRES_PASSWORD='newpassword'

# ESO syncs automatically within 1 hour
# Force immediate sync
kubectl annotate externalsecret postgres-secret \
  -n job-tracker \
  force-sync=$(date +%s) --overwrite
```

## Recovery

If Vault becomes sealed (KMS unavailable):

```bash
# Manual unseal using recovery keys
kubectl exec -it -n vault vault-0 -- vault operator unseal
# Enter 3 of 5 recovery keys when prompted
```

## Key Design Decisions

- **Single-node Vault** — sufficient for homelab, HA can be added later
- **File storage backend** — backed by Longhorn 3-way replication across nodes
- **KMS auto-unseal** — cluster self-recovers after reboots without manual intervention
- **ESO over Vault Agent** — cleaner separation, no sidecar injection needed
- **creationPolicy: Owner** — ESO owns the K8s secrets, deleted when ExternalSecret is deleted

## Future Roadmap


- [ ] Vault database secrets engine for dynamic credentials
- [ ] Vault audit logging
- [ ] Vault backup to S3

# Phase 4: Zero Trust Secrets Management Walkthrough

**Objective:** Remove base64-encoded secrets from Git and etcd. Inject them dynamically from an external vault.

## 1. Install External Secrets Operator (ESO)

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

## 2. Configure the Secret Store

This example uses Google Secret Manager, but ESO supports AWS Secrets Manager, Azure Key Vault, and HashiCorp Vault.

### Step 2.1: Create a Service Account (GCP)
Ensure your GKE nodes or a specific Workload Identity ServiceAccount has permissions to access Secret Manager (`roles/secretmanager.secretAccessor`).

### Step 2.2: Define ClusterSecretStore
This resource tells ESO *how* to connect to your vault.

```yaml
# cluster-secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: gcp-secret-store
spec:
  provider:
    gcpsm:
      projectID: "your-gcp-project-id"
      auth:
        workloadIdentity:
          clusterLocation: "us-central1"
          clusterName: "your-cluster-name"
          serviceAccountRef:
            name: "external-secrets-sa"
            namespace: "external-secrets"
```

## 3. Sync a Secret

### Step 3.1: Create a Secret in GCP
Create a secret named `redis-password` in Google Secret Manager with the value `super-secure-password`.

### Step 3.2: Define ExternalSecret
This resource tells ESO *what* to fetch.

```yaml
# redis-external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: redis-credentials
  namespace: online-boutique
spec:
  refreshInterval: 1h             # Rotation frequency
  secretStoreRef:
    name: gcp-secret-store
    kind: ClusterSecretStore
  target:
    name: redis-secret            # The name of the K8s Secret to create
    creationPolicy: Owner
  data:
  - secretKey: password           # Key in K8s Secret
    remoteRef:
      key: redis-password         # Name in GCP Secret Manager
```

### Step 3.3: Verify Injection
```bash
kubectl get secret redis-secret -n online-boutique -o jsonpath="{.data.password}" | base64 -d
# Output should be: super-secure-password
```

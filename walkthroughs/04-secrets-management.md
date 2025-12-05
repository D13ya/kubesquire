# Phase 4: Zero Trust Secrets Management Walkthrough

**Objective:** Remove base64-encoded secrets from Git and etcd. Inject them dynamically from an external vault.

**Prerequisites:**
*   **Tools:** `helm`, `kubectl`.
*   **Cloud Provider:** Access to AWS Secrets Manager, Google Secret Manager, or Azure Key Vault.
*   **Permissions:** Ability to create IAM roles/Service Accounts in the cloud provider.

**Official Documentation:**
*   [External Secrets Operator Docs](https://external-secrets.io/latest/)
*   [Google Secret Manager Provider](https://external-secrets.io/latest/provider/google-secrets-manager/)
*   [AWS Secrets Manager Provider](https://external-secrets.io/latest/provider/aws-secrets-manager/)

## 1. Install External Secrets Operator (ESO)

**PowerShell:**
```powershell
# Ensure you are connected to the cluster
gcloud container clusters get-credentials zero-trust-cluster --zone us-central1-a

helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

**Bash:**
```bash
# Ensure you are connected to the cluster
gcloud container clusters get-credentials zero-trust-cluster --zone us-central1-a

helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

## 2. Configure the Secret Store

This example uses Google Secret Manager. For AWS or Azure, refer to the provider docs linked above.

### Step 2.1: Create a Service Account (GCP)
Ensure your GKE nodes or a specific Workload Identity ServiceAccount has permissions to access Secret Manager (`roles/secretmanager.secretAccessor`).

**Workload Identity Setup (PowerShell):**
Run this block to configure the identity binding:
```powershell
$PROJECT_ID = "kubesquire"
$GSA_NAME = "external-secrets-sa"
$GSA_EMAIL = "$GSA_NAME@$PROJECT_ID.iam.gserviceaccount.com"

# 1. Create GCP Service Account
gcloud iam service-accounts create $GSA_NAME --project $PROJECT_ID

# 2. Grant Secret Accessor Permission
gcloud projects add-iam-policy-binding $PROJECT_ID `
    --member "serviceAccount:$GSA_EMAIL" `
    --role "roles/secretmanager.secretAccessor"

# 3. Bind Workload Identity
gcloud iam service-accounts add-iam-policy-binding $GSA_EMAIL `
    --role roles/iam.workloadIdentityUser `
    --member "serviceAccount:${PROJECT_ID}.svc.id.goog[external-secrets/external-secrets]"

# 4. Annotate Kubernetes Service Account
kubectl annotate serviceaccount external-secrets `
    --namespace external-secrets `
    iam.gke.io/gcp-service-account=$GSA_EMAIL `
    --overwrite
```

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
      projectID: "kubesquire"
      auth:
        workloadIdentity:
          clusterLocation: "us-central1-a"
          clusterName: "zero-trust-cluster"
          serviceAccountRef:
            name: "external-secrets"
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

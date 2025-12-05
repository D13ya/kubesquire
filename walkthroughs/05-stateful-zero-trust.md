# Phase 5: Stateful Zero Trust (Databases) Walkthrough

**Objective:** Replace ephemeral, insecure databases with production-grade, encrypted, and managed instances using Operators.

## 1. Install CloudNativePG Operator

We will replace the default `redis` deployment with a secure Postgres cluster (as an example of stateful rigor, though the demo uses Redis primarily).

```bash
kubectl apply -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/releases/cnpg-1.22.0.yaml
```

## 2. Deploy a Secure Postgres Cluster

This manifest deploys a 3-node Postgres cluster with:
-   **TLS:** Enabled by default (auto-generated certificates).
-   **Bootstrap:** Initializes a database.
-   **Storage:** Uses a secure StorageClass (assumed to be encrypted at rest by the cloud provider).

```yaml
# secure-postgres.yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: secure-db
  namespace: online-boutique
spec:
  instances: 3
  
  # Storage Configuration (Ensure your StorageClass has encryption enabled)
  storage:
    size: 1Gi
    storageClass: standard-rwo 

  # Bootstrap method
  bootstrap:
    initdb:
      database: app
      owner: appuser
      secret:
        name: db-creds-source # Reference a secret created via ESO (Phase 4)

  # Backup Configuration (Zero Trust Data Recovery)
  backup:
    barmanObjectStore:
      destinationPath: s3://your-backup-bucket/
      endpointURL: https://storage.googleapis.com
      s3Credentials:
        inheritFromIAMRole: true # Use Workload Identity!
```

## 3. Verify Encryption

### Step 3.1: Check TLS
Connect to the pod and verify the connection is encrypted.

```bash
kubectl exec -it secure-db-1 -n online-boutique -- psql -U appuser app
# Inside psql:
# \conninfo
# Output should show: SSL connection (protocol: TLSv1.3, cipher: ...)
```

### Step 3.2: Verify Workload Identity
Ensure the pod can access the backup bucket without explicit access keys in the pod environment variables.

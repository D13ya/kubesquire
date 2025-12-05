# 8-Hour Zero Trust Sprint: Online Boutique (PowerShell Edition)

**Goal:** Deploy the Google Microservices Demo with "Security by Design" in one working day.
**Prerequisites:**
*   **Setup:** Complete `walkthroughs/00-prerequisites-and-setup.md` first.
*   **Environment:** Windows PowerShell + GKE Cluster.

---

## **Hour 1: The Secure Foundation (Templating)**

**Objective:** Download the app and wrap it with security policies.

1.  **Repo Setup (PowerShell):**
    ```powershell
    # Clone upstream repo to temp folder
    git clone https://github.com/GoogleCloudPlatform/microservices-demo.git temp-repo

    # Create your app structure
    New-Item -ItemType Directory -Force apps/online-boutique
    Copy-Item -Recurse temp-repo/kustomize apps/online-boutique/base

    # Cleanup
    Remove-Item -Recurse -Force temp-repo
    cd apps/online-boutique
    ```

2.  **Create Production Overlay:**
    ```powershell
    New-Item -ItemType Directory -Force overlays/production
    New-Item overlays/production/kustomization.yaml
    ```
    *   **Action:** Paste this into `overlays/production/kustomization.yaml`:
    ```yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources: [../../base]
    patches:
      - target: {kind: Deployment}
        patch: |-
          - op: add
            path: /spec/template/spec/containers/0/securityContext
            value: {readOnlyRootFilesystem: true, runAsNonRoot: true}
    ```

3.  **Commit & Push:** Push this folder to your Git repo.

---

## **Hour 2: GitOps Automation (ArgoCD)**

**Objective:** Let ArgoCD manage the deployment.

1.  **Install ArgoCD (Securely):**
    *   **Standard vs HA:** We use the standard `install.yaml` for this sprint. For production, use `ha/install.yaml` (High Availability) which runs multiple replicas of the API server and Redis.
    *   **Method:** We download the file instead of using `kubectl apply -k ...` (Kustomize remote build) to ensure we can inspect the code before applying it (Supply Chain Security).

    ```powershell
    # 1. Create namespace
    kubectl create ns argocd

    # 2. Download the manifest (Vendor it)
    # Use "ha/install.yaml" here if you want High Availability
    New-Item -ItemType Directory -Force -Path infrastructure
    Invoke-WebRequest -Uri "https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml" -OutFile "infrastructure/argocd-install.yaml"

    # 3. (Optional) Inspect the file for malicious changes
    # Get-Content infrastructure/argocd-install.yaml | Select-String "image:"

    # 4. Apply from your local file
    # We use --server-side to prevent "Bad Record MAC" errors common with large files on Windows/VPNs.
    kubectl apply -n argocd -f infrastructure/argocd-install.yaml --server-side
    ```

2.  **Connect App:**
    *   Create a file `argocd-app.yaml` pointing to **your** repo URL.
    *   `kubectl apply -f argocd-app.yaml`

3.  **Verify:**
    *   Get the admin password:
    ```powershell
    $secret = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
    [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($secret))
    ```

---

## **Hour 3: Identity & Encryption (Istio)**

**Objective:** Encrypt all traffic (mTLS).

1.  **Install Istio:**
    ```powershell
    istioctl install --set profile=minimal -y
    kubectl label ns online-boutique istio-injection=enabled
    # Restart pods to inject sidecars
    kubectl rollout restart deploy -n online-boutique
    ```

2.  **Enforce Strict mTLS:**
    *   Create `peer-auth.yaml`:
    ```yaml
    apiVersion: security.istio.io/v1beta1
    kind: PeerAuthentication
    metadata: {name: default, namespace: online-boutique}
    spec: {mtls: {mode: STRICT}}
    ```
    *   `kubectl apply -f peer-auth.yaml`

---

## **Hour 4: Access Control (AuthorizationPolicies)**

**Objective:** "Deny All" traffic by default.

1.  **Apply Deny-All:**
    *   Create `deny-all.yaml`:
    ```yaml
    apiVersion: security.istio.io/v1beta1
    kind: AuthorizationPolicy
    metadata: {name: deny-all, namespace: online-boutique}
    spec: {}
    ```
    *   `kubectl apply -f deny-all.yaml`
    *   **Result:** The app will stop working. This is good!

2.  **Allow Frontend:**
    *   Create `allow-frontend.yaml` to let the frontend talk to services.
    *   (See `walkthroughs/02-identity-segmentation.md` for the full policy list).

---

## **Hour 5: Secrets Management (ESO)**

**Objective:** Inject secrets from Google Secret Manager (No base64 in Git).

1.  **Setup Vault:**
    *   Create a secret in Google Secret Manager (e.g., `redis-password`).

2.  **Install ESO:**
    ```powershell
    helm repo add external-secrets https://charts.external-secrets.io
    helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
    ```

3.  **Bind Identity:**
    *   Follow `walkthroughs/04-secrets-management.md` to bind your GKE ServiceAccount to your GCP Service Account.
    *   **Verify:** `kubectl get secret redis-secret` should show the value from GCP.

---

## **Hour 6: Network Segmentation (Cilium)**

**Objective:** Stop data exfiltration (Egress).

1.  **Lock Down Egress:**
    *   Apply a `CiliumNetworkPolicy` that denies all egress except DNS (53) and Google APIs (443).
    *   **Test:** Exec into a pod and try `curl google.com`. It should hang/fail.

---

## **Hour 7: Policy as Code (Kyverno)**

**Objective:** Block insecure pods.

1.  **Install Kyverno (Securely):**
    ```powershell
    # 1. Download the manifest
    Invoke-WebRequest -Uri "https://github.com/kyverno/kyverno/releases/latest/download/install.yaml" -OutFile "infrastructure/kyverno-install.yaml"

    # 2. Apply from local
    kubectl create -f infrastructure/kyverno-install.yaml
    ```

2.  **Enforce Standards:**
    *   Apply a `ClusterPolicy` requiring `runAsNonRoot: true`.
    *   **Test:** Try to deploy a "bad" pod (root user). Kyverno should block it.

---

## **Hour 8: Verification & Observability**

**Objective:** Prove it works.

1.  **Hubble UI:**
    *   `cilium hubble ui`
    *   Look for **Red Lines** (Blocked Traffic).

2.  **Falco (Optional):**
    *   Install Falco.
    *   Exec into a pod and run `cat /etc/shadow`.
    *   Check logs for "Notice: Read sensitive file".

---

**Next Steps:**
You have completed the sprint! For detailed debugging, refer to the `walkthroughs/` folder.

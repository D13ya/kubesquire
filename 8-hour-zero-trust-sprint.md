# 8-Hour Zero Trust Sprint: Online Boutique (PowerShell Edition)

**Goal:** Deploy the Google Microservices Demo with "Security by Design" in one working day.
**Prerequisites:**
*   **Setup:** Complete `walkthroughs/00-prerequisites-and-setup.md` first.
*   **Environment:** Windows PowerShell + GKE Cluster.

---

## **0. Philosophy & Roles**

**The "Greenfield" Approach:**
We are building a **Secure by Design** environment, aligned with the **[CNCF Secure Defaults: Cloud Native 8](https://github.com/cncf/tag-security/blob/main/community/resources/security-whitepaper/secure-defaults-cloud-native-8.md)**. We do not deploy the app and *then* secure it. We build the secure enclosure first, then drop the app into it.

**Roles:**
*   **App Team (You, in Hour 1):** You provide the base application code. You care about features.
*   **Platform Team (You, in Hours 2-8):** You wrap that code in security policies (mTLS, NetworkPolicies) without changing the app code itself. You care about compliance and stability.

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
    *   **Action:** Create a file `infrastructure/argocd-app.yaml` pointing to **your** repo URL. This tells ArgoCD where to find the Kustomize overlay we built in Hour 1.
    *   **Important:** Ensure you have pushed your `apps/` folder to GitHub before running this, or ArgoCD will report a "Path not found" error.
    *   `kubectl apply -f infrastructure/argocd-app.yaml`

3.  **Verify:**
    *   Get the admin password:
    *   *Note: This retrieves the unique, auto-generated password for your specific installation. In production, you would replace this with SSO.*
    ```powershell
    $secret = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
    [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($secret))
    ```

---

## **Hour 3: Identity & Encryption (Istio)**

**Objective:** Encrypt all traffic (mTLS).

1.  **Install Istio & Enable Injection (GitOps Way):**
    *   **Install Istio:**
    ```powershell
    istioctl install --set profile=minimal -y
    ```
    *   **Enable Sidecar Injection:**
        Instead of manually labeling, we added a `namespace.yaml` to our Kustomize overlay in `apps/online-boutique/overlays/production/`.
        *   *Check:* `kubectl get ns online-boutique --show-labels` should show `istio-injection=enabled`.
    *   **Restart Pods:**
    ```powershell
    kubectl rollout restart deploy -n online-boutique
    ```

2.  **Enforce Strict mTLS (Mesh-Wide):**
    *   **Create Policy:**
        Create `apps/online-boutique/overlays/production/peer-auth.yaml`. We apply this to `istio-system` to enforce it for the entire cluster (Greenfield approach).
    ```yaml
    apiVersion: security.istio.io/v1beta1
    kind: PeerAuthentication
    metadata: {name: default, namespace: istio-system}
    spec: {mtls: {mode: STRICT}}
    ```
    *   **Update Kustomization:**
        Add `- peer-auth.yaml` to the `resources` list in `apps/online-boutique/overlays/production/kustomization.yaml`.
    *   **Push & Sync:**
        ```powershell
        git add .
        git commit -m "Enforce Strict mTLS"
        git push
        kubectl annotate application online-boutique -n argocd argocd.argoproj.io/refresh=hard --overwrite
        ```

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

2.  **Allow Traffic:**
    *   Create `allow-frontend.yaml` to let the frontend talk to services.
    *   Create `allow-internal.yaml` to allow backend services to communicate (e.g., Cart -> Redis).
    *   (See `walkthroughs/02-identity-segmentation.md` for the full policy list).

---

## **Hour 5: Secrets Management (ESO)**

**Objective:** Inject secrets from Google Secret Manager (No base64 in Git).

1.  **Prerequisite: Configure Pub/Sub (PowerShell):**
    Secret Manager needs a Pub/Sub topic for rotation notifications and permission to publish to it.
    ```powershell
    # 1. Create the topic
    gcloud pubsub topics create secret-rotation-topic

    # 2. Get your Project Number
    $PROJECT_NUMBER = gcloud projects list --filter="project_id=$(gcloud config get-value project)" --format="value(projectNumber)"

    # 3. Ensure Secret Manager Service Agent exists (Critical Step)
    gcloud beta services identity create --service=secretmanager.googleapis.com --project=$(gcloud config get-value project)

    # 4. Grant permission to Secret Manager Service Agent
    gcloud pubsub topics add-iam-policy-binding secret-rotation-topic `
        --member="serviceAccount:service-$PROJECT_NUMBER@gcp-sa-secretmanager.iam.gserviceaccount.com" `
        --role="roles/pubsub.publisher"
    ```
    *   **Note:** Step 3 is required to generate the `service-...@gcp-sa-secretmanager...` identity if it doesn't exist yet.

2.  **Setup Vault (Web Interface):**
    *   Navigate to **Security > Secret Manager** in the Google Cloud Console.
    *   Create a secret named `redis-password`.
    *   **Rotation & Notifications:**
        *   Enable **Rotation** and set a period (e.g., 30 days).
        *   *Important:* Secret Manager sends notifications to **Pub/Sub**; it does not automatically change the value without a configured rotator (Cloud Function).
        *   **Requirement:** Select the `secret-rotation-topic` you created above.

3.  **Install ESO:**
    ```powershell
    helm repo add external-secrets https://charts.external-secrets.io
    helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
    ```

4.  **Bind Identity:**
    *   Follow `walkthroughs/04-secrets-management.md` to bind your GKE ServiceAccount to your GCP Service Account.
    *   **Verify:** `kubectl get secret redis-secret` should show the value from GCP.

---

## **Hour 6: Network Segmentation (Cilium)**

**Objective:** Stop data exfiltration (Egress).

1.  **Lock Down Egress:**
    *   Apply a `CiliumNetworkPolicy` that denies all egress except:
        *   DNS (53)
        *   Google APIs (443)
        *   **Localhost (127.0.0.1/32):** Critical for Istio sidecar communication (See [Cilium Issue #18465](https://github.com/cilium/cilium/issues/18465)).
    *   **Test:** Exec into a pod and try `curl google.com`. It should hang/fail.

---

## **Hour 7: Policy as Code (Kyverno)**

**Objective:** Block insecure pods (CNCF Secure Software Factory "Admission Controller").

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

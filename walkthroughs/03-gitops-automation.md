# Phase 3: GitOps & Automation Walkthrough

**Objective:** Use ArgoCD to maintain the cluster state, ensuring that manual changes (drift) are automatically reverted.

**Prerequisites:**
*   **Tools:** `kubectl`, `argocd` CLI (optional but recommended).
*   **Cluster:** A running Kubernetes cluster.
*   **Git Repo:** A fork of the Online Boutique or your own configuration repo.

**Official Documentation:**
*   [ArgoCD Installation Guide](https://argo-cd.readthedocs.io/en/stable/getting_started/)
*   [ArgoCD Application Specification](https://argo-cd.readthedocs.io/en/stable/operator-manual/application.yaml/)
*   [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)

## 1. Install ArgoCD

### Step 1.1: Deploy ArgoCD
We install ArgoCD in its own namespace.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 1.2: Access the UI
For local testing, use port-forwarding. For production, configure an Ingress.

```bash
# Port forward to access the UI at https://localhost:8080
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 2. Define the Application

Create an `Application` manifest that points to your forked repository and the `overlays/production` path we created in Phase 1.

### Step 2.1: Create Application Manifest
*Note: Replace `https://github.com/YOUR_USERNAME/kubesquire.git` with your actual Git repository URL.*

```yaml
# online-boutique-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: online-boutique
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/YOUR_USERNAME/kubesquire.git' # <--- UPDATE THIS
    targetRevision: HEAD
    path: apps/online-boutique/overlays/production
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: online-boutique
  syncPolicy:
    automated:
      prune: true      # Delete resources not in Git
      selfHeal: true   # Revert manual changes immediately
    syncOptions:
      - CreateNamespace=true
```

### Step 2.2: Apply and Verify
```bash
kubectl apply -f online-boutique-app.yaml
```

## 3. Verify Drift Detection

1.  **Manual Change:** Try to manually delete a NetworkPolicy.
    ```bash
    kubectl delete networkpolicy restrict-egress -n online-boutique
    ```
2.  **Observe ArgoCD:** Watch the ArgoCD UI or logs. It should detect the "OutOfSync" state and immediately re-apply the policy because `selfHeal` is enabled.
    ```bash
    # Check status via CLI (if installed)
    argocd app get online-boutique
    ```
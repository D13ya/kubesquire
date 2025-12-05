# Phase 3: GitOps & Automation Walkthrough

**Objective:** Use ArgoCD to maintain the cluster state, ensuring that manual changes (drift) are automatically reverted.

## 1. Install ArgoCD

### Step 1.1: Deploy ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 1.2: Access the UI
```bash
# Port forward to access the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 2. Define the Application

Create an `Application` manifest that points to your forked repository and the `overlays/production` path we created in Phase 1.

### Step 2.1: Create Application Manifest
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
    repoURL: 'https://github.com/YOUR_USERNAME/kubesquire.git' # Replace with your repo
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

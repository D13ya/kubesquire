# Phase 0: Prerequisites & Environment Setup (Windows Edition)

**READ THIS FIRST**

You **MUST** install these tools to complete the sprint. Without them, commands like `istioctl`, `helm`, and `kustomize` will fail.

## 1. Install Required Tools (PowerShell)

Run these commands in your PowerShell terminal.

### Step 1: Install Core Tools

**1. Google Cloud SDK & Kustomize & Helm**
```powershell
winget install Google.CloudSDK
winget install Kubernetes.Kustomize
winget install Helm.Helm
```

**2. ArgoCD CLI**
The package ID is case-sensitive.
```powershell
winget install argoproj.argocd
```

**3. Istio CLI (Manual Install - Long Term Solution)**
*Since Istio does not have a Windows installer, we will create a dedicated "Tools" folder. This is a best practice for managing standalone binaries on Windows.*

1.  **Create a Tools Folder:**
    ```powershell
    New-Item -ItemType Directory -Force -Path "C:\Tools"
    ```

2.  **Add to PATH (One-time setup):**
    ```powershell
    $currentPath = [Environment]::GetEnvironmentVariable("Path", "User")
    if ($currentPath -notlike "*C:\Tools*") {
        [Environment]::SetEnvironmentVariable("Path", $currentPath + ";C:\Tools", "User")
        Write-Host "Added C:\Tools to PATH. Please restart your terminal."
    }
    ```

3.  **Install Istio:**
    *   Locate your downloaded `istioctl-*.zip`.
    *   Extract it and move `istioctl.exe` into `C:\Tools`.
    *   *Example command (adjust filename as needed):*
    ```powershell
    Expand-Archive -Path "$HOME\Downloads\istioctl-1.28.1-win-amd64.zip" -DestinationPath "temp-istio"
    Move-Item -Path "temp-istio\istioctl.exe" -Destination "C:\Tools\" -Force
    Remove-Item -Recurse -Force "temp-istio"
    ```

4.  **Verify:**
    Restart your terminal, then run:
    ```powershell
    istioctl version
    ```

*Note: You may need to restart your terminal after installing these for them to be recognized.*

### Step 2: Install kubectl
If you have `gcloud` installed, run:
```powershell
gcloud components install kubectl
```

## 2. Cloud & Cluster Setup

### Step 2.1: Login to Google Cloud
```powershell
gcloud auth login
# Follow the browser prompt
```

### Step 2.2: Create the Zero Trust Cluster
This creates a GKE cluster with "Workload Identity" enabled (required for the security steps).

```powershell
# Set your Project ID (Change 'kubesquire' to your actual ID if different)
$env:PROJECT_ID = "kubesquire"
gcloud config set project $env:PROJECT_ID

# Create the cluster
gcloud container clusters create zero-trust-cluster `
    --zone us-central1-a `
    --workload-pool="${env:PROJECT_ID}.svc.id.goog" `
    --enable-autoscaling --min-nodes=1 --max-nodes=4 `
    --num-nodes=1
```

### Step 2.3: Connect to the Cluster
```powershell
gcloud container clusters get-credentials zero-trust-cluster --zone us-central1-a
```

### Step 2.4: Verify Everything
Run this command. If you see version numbers and nodes, you are ready.
```powershell
Write-Host "--- Checking Tools ---"
kustomize version
istioctl version
helm version
argocd version --client
kubectl get nodes
```

## 3. Ready?
Once the verification step passes, go to `8-hour-zero-trust-sprint.md` and start **Hour 1**.


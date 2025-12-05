# Phase 1: The Secure Supply Chain Walkthrough

**Objective:** Establish a trusted pipeline where artifacts are verified, scanned, and templated before reaching the cluster.

**Prerequisites:**
*   **Tools:** `kpt`, `kustomize`, `docker`, `syft`, `trivy`, `cosign`.
*   **Access:** Read/Write access to an OCI Registry (e.g., GCR, Artifact Registry, Docker Hub).
*   **Cluster:** A running Kubernetes cluster (GKE/EKS/Kind) is not strictly required for this phase (it's mostly local/CI), but `kubectl` should be configured if you plan to apply.

**Official Documentation:**
*   [kpt Documentation](https://kpt.dev/book/02-concepts/01-packages)
*   [Kustomize Documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)
*   [Syft (SBOM) GitHub](https://github.com/anchore/syft)
*   [Trivy Documentation](https://aquasecurity.github.io/trivy/v0.18.3/)
*   [Cosign Documentation](https://docs.sigstore.dev/cosign/overview/)

## 1. Templating with Kustomize & kpt

We will use `kpt` to fetch the upstream Online Boutique and `Kustomize` to apply security overlays without modifying upstream code.

### Step 1.1: Fetch the Upstream Package
We need to get the base manifests from Google's repository.

**Option A: Using `kpt` (Recommended if you have Docker/WSL)**
```bash
mkdir -p apps/online-boutique
cd apps/online-boutique
kpt pkg get https://github.com/GoogleCloudPlatform/microservices-demo/kustomize@main base
```

**Option B: Using `git clone` (Windows PowerShell Alternative)**
If you cannot run `kpt`, you can manually clone the repo and copy the folder.
```powershell
# 1. Clone the repo to a temporary location
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git temp-repo

# 2. Create your app directory
New-Item -ItemType Directory -Force apps/online-boutique

# 3. Copy the 'kustomize' folder to 'apps/online-boutique/base'
Copy-Item -Recurse temp-repo/kustomize apps/online-boutique/base

# 4. Cleanup
Remove-Item -Recurse -Force temp-repo
cd apps/online-boutique
```

### Step 1.2: Create the Overlay Structure
Create a directory structure for your environment overlays (e.g., `production`).

**PowerShell:**
```powershell
New-Item -ItemType Directory -Force overlays/production
New-Item overlays/production/kustomization.yaml
```

**Bash:**
```bash
mkdir -p overlays/production
touch overlays/production/kustomization.yaml
```

### Step 1.3: Define the Kustomization
In `overlays/production/kustomization.yaml`, we reference the base and add our security configurations.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  # Ensure you have fetched or created these components. 
  # See Phase 2 for creating the network-policies component.
  # - ../../components/network-policies  
  # - ../../components/service-mesh-istio

# Common labels for all resources
commonLabels:
  env: production
  compliance: zero-trust

# Enforce read-only root filesystem globally (where possible)
patches:
  - target:
      kind: Deployment
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/securityContext
        value:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
```

## 2. Image Assurance (Trivy, Syft, Cosign)

These steps should be integrated into your CI pipeline (GitHub Actions / Cloud Build).

### Step 2.1: Generate SBOM with Syft
Generate a Software Bill of Materials for the frontend image.

*Note: Replace `gcr.io/google-samples/microservices-demo/frontend:v0.8.0` with your own registry image if you are rebuilding the app.*

```bash
# Generate SBOM in SPDX format
syft packages gcr.io/google-samples/microservices-demo/frontend:v0.8.0 -o spdx-json > frontend-sbom.spdx.json

# Upload SBOM to registry (if supported) or store as artifact
# Ensure you are authenticated to the registry (e.g., `gcloud auth configure-docker`)
cosign attach sbom --sbom frontend-sbom.spdx.json gcr.io/google-samples/microservices-demo/frontend:v0.8.0
```

### Step 2.2: Scan for Vulnerabilities with Trivy
Fail the build if critical vulnerabilities are found.

```bash
trivy image --severity CRITICAL --exit-code 1 gcr.io/google-samples/microservices-demo/frontend:v0.8.0
```

### Step 2.3: Sign the Image with Cosign
Sign the image to prove it passed your CI checks.

```bash
# Generate key pair (do this once and store in secrets)
cosign generate-key-pair

# Sign the image
# The 'cosign.key' is the private key generated above.
cosign sign --key cosign.key gcr.io/google-samples/microservices-demo/frontend:v0.8.0
```

## 3. Pre-admission Validation (Kyverno CLI)

Validate your rendered manifests against policies before applying.

**Prerequisite:** You need a set of Kyverno policies (e.g., Pod Security Standards) in a `policies/` folder.
*   [Kyverno Policies Library](https://kyverno.io/policies/)

```bash
# Render the manifests
kustomize build overlays/production > rendered.yaml

# Validate against Kyverno policies
kyverno apply policies/ --resource rendered.yaml
```

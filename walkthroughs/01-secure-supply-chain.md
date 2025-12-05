# Phase 1: The Secure Supply Chain Walkthrough

**Objective:** Establish a trusted pipeline where artifacts are verified, scanned, and templated before reaching the cluster.

## 1. Templating with Kustomize & kpt

We will use `kpt` to fetch the upstream Online Boutique and `Kustomize` to apply security overlays without modifying upstream code.

### Step 1.1: Fetch the Upstream Package
```bash
mkdir -p apps/online-boutique
cd apps/online-boutique

# Fetch the latest version of Online Boutique using kpt
kpt pkg get https://github.com/GoogleCloudPlatform/microservices-demo/kustomize@main base
```

### Step 1.2: Create the Overlay Structure
Create a directory structure for your environment overlays (e.g., `production`).

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
  - ../../components/network-policies  # If you fetched these or created them
  - ../../components/service-mesh-istio

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

```bash
# Generate SBOM in SPDX format
syft packages gcr.io/google-samples/microservices-demo/frontend:v0.8.0 -o spdx-json > frontend-sbom.spdx.json

# Upload SBOM to registry (if supported) or store as artifact
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
cosign sign --key cosign.key gcr.io/google-samples/microservices-demo/frontend:v0.8.0
```

## 3. Pre-admission Validation (Kyverno CLI)

Validate your rendered manifests against policies before applying.

```bash
# Render the manifests
kustomize build overlays/production > rendered.yaml

# Validate against Kyverno policies (assuming you have a policies folder)
kyverno apply policies/ --resource rendered.yaml
```

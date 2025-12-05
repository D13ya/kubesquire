# **Proposal: Zero Trust Enforcement in Kubernetes Security**

**Project:** Zero Trust Architecture Implementation for Google Microservices Demo (Online Boutique)

## **1\. Executive Summary**

This proposal outlines a strategy to deliver a reproducible Zero-Trust runtime for cloud Kubernetes1. Moving beyond simple proof-of-concepts, this project aims to deploy the [Google Microservices Demo](https://github.com/GoogleCloudPlatform/microservices-demo) using a "Security by Design" approach rather than bolting on security tools post-deployment2.

The architecture adheres to core Zero Trust tenets:

* **Segment-by-default:** Network Policy via Cilium3.

* **Policy-as-code:** Pod Security via Kyverno4.

* **Least-privilege:** RBAC and IAM Roles for Service Accounts (IRSA)5.

* **Identity-first:** Istio mTLS for rigorous workload identity66.

* **Runtime Assurance:** Continuous threat detection via Falco7.

## **1.1. Philosophy: "Greenfield" Zero Trust**

This project adopts a **"Greenfield" (Security by Design)** approach. Unlike "Brownfield" deployments where security is bolted on after the application is running, we enforce security controls *before* the first pod is deployed.

*   **Shift Left:** Security policies (NetworkPolicies, mTLS, RBAC) are defined in the Git repository alongside the application code.
*   **Deny-by-Default:** The environment is locked down from day one. Traffic is blocked unless explicitly allowed.
*   **Immutable Infrastructure:** No manual changes via `kubectl`. The Git repository is the single source of truth.

## **1.2. Operational Roles**

To ensure security does not hinder velocity, we define clear responsibilities:

| Role | Responsibility | Scope |
| :--- | :--- | :--- |
| **Application Team** (Developers) | Focus on business logic. Define the "Base" application manifests (Deployment, Service). Trigger backups and perform self-service recovery. | **Namespace-scoped:** Can deploy to their namespace but cannot alter cluster-wide policies. |
| **Platform/Security Team** (Admins) | Maintain the "Overlays" that inject security controls. Manage the Control Plane (Istio, Cilium, ArgoCD). Enforce guardrails via Policy-as-Code (Kyverno). | **Cluster-scoped:** Define the rules of the road (e.g., "No root users", "mTLS required"). |

## **2\. Technology Stack & Roles**

| Domain | Tool | Role & Configuration |
| :---- | :---- | :---- |
| **CNI & Network** | **Cilium** | L3/L4 Network Policies, Egress Filtering (FQDN), and Hubble Observability8.  |
| **Service Mesh** |  **Istio** | mTLS for encryption-in-transit and L7 Authorization Policies999.  |
| **GitOps** | **ArgoCD** | Automated drift detection and state synchronization10.  |
| **Secrets** | **External Secrets Operator** | On-demand secret injection and automated rotation from external vaults11.  |
| **Supply Chain** | **Trivy / Cosign / Syft** | Image scanning, cryptographic signing, and SBOM generation12.  |
| **Policy Engine** | **Kyverno** | Admission control to reject insecure, unsigned, or non-compliant workloads13.  |
| **Runtime Security** | **Falco** | Kernel-level threat detection (e.g., shell access alerts)14.  |
| **Infra Security** | **Kube-bench** | CIS Benchmark validation for Node and Cluster hardening. |

## ---

**3\. Implementation Phases**

Each phase below is framed as a long-term control, not a short-lived patch. Every control is baked into templates, enforced by policy/automation, and tied to evidence so regression is detectable.

### **Phase 1: The Secure Supply Chain (Design)**

*Objective: Ensure no untrusted artifacts enter the cluster.*

* **The Problem:** The default demo uses raw Kubernetes manifests, making it difficult to enforce security contexts globally15.

* **The Fix \- Templating (Kustomize/Helm/kpt):** Adopt a modular "Base + Overlay" strategy.
    *   *Approach:* Use **Kustomize** to maintain the base application manifests while applying security overlays (`components/network-policies`, `components/service-mesh-istio`) separately.
    *   *Advanced Approach:* Use **kpt** (Google's package manager) to fetch the Online Boutique package. `kpt` allows for "Configuration as Data," enabling you to programmatically inject security contexts and validate manifests against policies *before* they reach the cluster.
    *   *Alternative:* Use the official **Helm Chart** with `networkPolicies.create=true` and `serviceAccounts.annotations` for Workload Identity.
    *   *GitOps layout:* Create a repo structure such as `apps/online-boutique/base` and `environments/{dev,stage,prod}` overlays. Promotion happens via signed tags or PR merges that bump the overlay reference, keeping a clear audit trail for ArgoCD/Flux to sync.

* **Pre-admission validation:** Add **conftest/OPA** or **Kyverno CLI** checks in CI to lint rendered manifests (from kustomize/helm/kpt) against policy baselines before ArgoCD sees them. This ensures "verify before apply" and blocks insecure configs earlier in the pipeline.

* **The Fix \- Image Assurance:** Integrate **Trivy** for build-time scanning and **Cosign** (Sigstore) to sign images.
* **The Fix \- SBOM:** Generate a Software Bill of Materials (SBOM) using **Syft** during the build. Store it in the registry (OCI artifact). Use **Kyverno** to verify the presence of an SBOM and reject images with forbidden packages (e.g., Log4j vulnerable versions)17.

### **Phase 2: Identity-Based Segmentation (Core Zero Trust)**

*Objective: Establish strong identity and restrict communication.*

* **The Problem:** By default, the demo allows all services to communicate freely (mesh-wide trust)18.

* **The Fix \- Strict mTLS:** Enable Istio PeerAuthentication with mode: STRICT. This eliminates non-encrypted traffic19.
    *   *Real-World Option:* Use **Cloud Service Mesh** (Google's managed Istio). This offloads the maintenance of the control plane (`istiod`) and CA rotation to Google, allowing you to focus solely on defining `AuthorizationPolicies`.
    *   *Real-World Challenge:* Kubelet health probes fail because they are unencrypted.
    *   *Solution:* Use Istio's "rewrite app probes" feature or dedicated `grpc-health-probe` sidecars, and bake the choice into the overlays so health checks stay consistent across environments.

* **The Fix \- AuthorizationPolicies:** Define precise access rules.
  * *Trust boundaries:* Document the exact service pairs that can talk (e.g., frontend → cartservice, checkoutservice → paymentservice) and keep them as deny-by-default namespace policies with per-service allowlists to avoid gaps when onboarding new workloads.

* **The Fix \- Egress Filtering:** Use Cilium Network Policies to restrict outbound traffic.
  * *Mechanism:* Deny all internet access by default.
  * *Real-World Challenge:* Blocking egress breaks OpenTelemetry and Cloud Trace (used heavily by the demo).
  * *Solution:* Explicitly allow egress to Google Cloud APIs (`monitoring.googleapis.com`, `cloudtrace.googleapis.com`) and the Istio Control Plane via FQDN policies. Integrate egress allowlists into CI so adding a new external dependency requires a reviewed policy change, preventing silent sprawl while preserving observability.

### **Phase 3: GitOps & Automation**

*Objective: Prevent configuration drift and unauthorized changes.*

* **The Problem:** Manual kubectl apply commands allow for "drift" and unverified changes, breaking Zero Trust integrity22.

* **The Fix:** Implement **ArgoCD** or **Flux**. The "Template" becomes a Git repository23.

* **Outcome:** ArgoCD ensures the cluster state matches the secure Git template. If an attacker manually alters a NetworkPolicy, ArgoCD automatically reverts the change24.
* **Long-term signal:** Enable ArgoCD application health alerts and signed commits/tags; schedule weekly drift reports as artifacts to demonstrate continuous enforcement.

### **Phase 4: Zero Trust Secrets Management**

*Objective: Eliminate hardcoded secrets from the cluster state.*

* **The Problem:** Standard Kubernetes Secrets are base64-encoded strings stored in etcd. If etcd is compromised, all credentials are lost25.

* **The Fix:** Deploy the **External Secrets Operator (ESO)**26.

* **Mechanism:** ESO fetches secrets on-demand from a secure vault (e.g., AWS Secrets Manager, Google Secret Manager) and injects them into the pod only if the specific ServiceAccount is authorized via Workload Identity27272727.

* **The Fix \- Automated Rotation:** Configure ESO to poll for changes. When a secret is rotated in the external vault, ESO updates the Kubernetes Secret. Configure the application to watch for file changes (volume mount) or restart the pod (via Reloader) to pick up the new secret immediately.
* **Secrets lifecycle playbook:** Define rotation frequency targets (e.g., 90 days), break-glass steps for restoring access, and whether apps reload via sidecar reloader or controlled pod restarts so teams know how to react without downtime.

### **Phase 5: Stateful Zero Trust (Databases)**

*Objective: Secure the data at rest and in transit.*

* **The Problem:** The default demo uses simple, ephemeral deployments for Redis and PostgreSQL with no encryption at rest and static passwords.
* **The Fix:** Replace default manifests with **Operators** (e.g., **CloudNativePG** for Postgres, **Redis Operator**).
* **Zero Trust Integration:**
    *   **Encryption:** Enable TLS for all database connections (using cert-manager).
    *   **Identity:** Use **Workload Identity** for the Operator to access cloud backups (S3/GCS) without static keys.
    *   **Storage:** Enable **Encryption at Rest** on the PersistentVolumes (via StorageClass/KMS).
    *   **Recovery drills:** Schedule periodic backup/restore tests and encryption verification to prove controls work, not just exist.

### **Phase 6: Progressive Delivery**

*Objective: Automate deployment with security gates.*

* **The Problem:** Standard rolling updates replace pods blindly. If a new version introduces a security flaw, it immediately impacts production28.

* **The Fix:** Integrate **Argo Rollouts** or **Flagger**29.

* **Zero Trust Integration:** Implement a "Canary" strategy (e.g., 5% traffic). Use Prometheus and Falco metrics to gate the rollout. If Falco detects a security anomaly (e.g., "shell in container") in the canary, the rollout aborts automatically30.
  * *Guardrails:* Define rollback triggers (e.g., error budget burn rate, AuthZ deny spikes, Falco alerts) and timeouts so failed canaries halt quickly and revert.

### **Phase 7: Deep Observability (Verification)**

*Objective: Prove that policies are working ("Never Trust, Always Verify").*

* **The Fix:** Enable **Hubble** (via Cilium) or **Tetragon**31.

* **Outcome:** Generate dependency maps showing exactly which services attempted communication and which packets were dropped by Zero Trust policies. This provides the audit logs necessary for compliance32.
* **Long-term signal:** Retain flow logs and authz decisions with retention aligned to audit requirements; add recurring reviews of dropped flows to catch unintended dependencies before they become outages.

### **Phase 8: Infrastructure Hardening & Human Access**

*Objective: Secure the underlying platform and human interactions.*

* **The Problem:** Zero Trust often overlooks the Node OS and human administrators. A compromised node compromises all pods on it.
* **The Fix \- Node Hardening:** Run **Kube-bench** regularly to validate compliance with CIS Kubernetes Benchmarks. Use a minimal OS (e.g., Bottlerocket, COS) to reduce the attack surface.
* **The Fix \- Human Access:** Eliminate long-lived `kubeconfig` files. Use OIDC (e.g., Dex, Keycloak) for authentication. Implement Just-in-Time (JIT) access for emergency "break-glass" scenarios, where elevated permissions are granted temporarily and audited.
* **Long-term signal:** Automate CIS scans and OIDC/JIT access audits; publish recurring reports of role usage and node hardening drift to demonstrate sustained posture.

### **Phase 9: Operator Security & Auditing**

*Objective: Secure the tools that secure the cluster.*

* **The Problem:** Operators (like External Secrets or ArgoCD) often run with "God Mode" (cluster-admin) permissions. If an operator is compromised, the entire cluster is lost.
* **The Fix:** Audit and restrict Operator RBAC permissions.
    *   *Action:* Use **RoleBindings** (Namespace-scoped) instead of ClusterRoleBindings where possible.
    *   *Action:* Monitor Operator logs for anomalous behavior (e.g., accessing secrets it shouldn't).
    *   *Action:* Maintain baseline RBAC templates per operator (ArgoCD, ESO, Istio ingress) and schedule recurring audits to catch privilege creep.

## ---

**4\. Proposed Modifications to the Demo**

To transform the standard microservices demo into a "Real World" Zero Trust environment, the following modifications will be made33:

| Component | Default Demo State | Your Zero Trust Proposal Fix |
| :---- | :---- | :---- |
| **Network** | Open (Any-to-Any) | **Cilium NetworkPolicies** (L3/L4) to lock namespaces \+ **Istio AuthZ** (L7) to allow only specific gRPC methods. |
| **Identity** | ServiceAccount tokens | **Workload Identity** (GKE/EKS). Map K8s ServiceAccounts to Cloud IAM roles to eliminate long-lived keys. |
| **Secrets** | Kubernetes Secrets (Base64) | **External Secrets Operator** fetching from AWS Secrets Manager or Google Secret Manager. |
| **Runtime** | Root users allowed | **Kyverno Policies**: Enforce runAsNonRoot, readOnlyRootFilesystem, and allowPrivilegeEscalation: false. <br> *Note:* Mount `emptyDir` to `/tmp` for Node.js/Python services to prevent crashes. |

## **5\. Deliverables**

This project will generate the following artifacts34:

1. **Infrastructure Definitions:** Terraform/OpenTofu code for GKE clusters with Workload Identity enabled35.

2. **Kubernetes Manifests:** Kustomize overlays or Helm charts for the 11-microservice application with integrated security contexts.  
3. **Test Logs & Artifacts:** Evidence demonstrating:
   * End-to-end encryption (mTLS).
   * Blocked traffic logs (Micro-segmentation) and Egress denials.
   * Admission control rejections (Policy-as-Code).
   * Runtime threat alerts (Falco).
   * Generated SBOMs for all microservices.
   * CIS Benchmark report for the cluster.
   * Evidence collection matrix mapping each control to its dashboard or log query (e.g., Hubble flows for deny logs, ArgoCD audit trail for drift corrections, Falco alerts for runtime anomalies).

## **6\. Resources & References**

*   **Official Google Microservices Demo - Network Policies:**  
    [https://github.com/GoogleCloudPlatform/microservices-demo/tree/main/kustomize/components/network-policies](https://github.com/GoogleCloudPlatform/microservices-demo/tree/main/kustomize/components/network-policies)  
    *Reference for "segment-by-default" implementation.*

*   **Official Google Microservices Demo - Istio Service Mesh:**  
    [https://github.com/GoogleCloudPlatform/microservices-demo/tree/main/kustomize/components/service-mesh-istio](https://github.com/GoogleCloudPlatform/microservices-demo/tree/main/kustomize/components/service-mesh-istio)  
    *Reference for mTLS and L7 Authorization Policies.*

*   **Medium Article:** "Simplify the deployment of Online Boutique, with a Service Mesh, GitOps, and more!"  
    [https://medium.com/google-cloud/246119e46d53](https://medium.com/google-cloud/246119e46d53)  
    *Guide on using the official Helm chart with security features.*

*   **Google Cloud:** Deploy Online Boutique with kpt  
    [https://cloud.google.com/service-mesh/docs/onlineboutique-install-kpt](https://cloud.google.com/service-mesh/docs/onlineboutique-install-kpt)  
    *Official guide for using kpt to manage the demo application.*

*   **Google Cloud:** Provision Cloud Service Mesh Control Plane  
    [https://cloud.google.com/service-mesh/docs/onboarding/provision-control-plane](https://cloud.google.com/service-mesh/docs/onboarding/provision-control-plane)  
    *Guide for setting up the managed Istio control plane.*

*   **Chainguard:** Guide to Kubernetes Supply Chain Security36.

*   **Istio:** Security Policy Examples (Namespace isolation, Deny-by-default)37373737.

*   **Microsoft Azure:** Guide to CI/CD with Helm and GitOps38.

*   **Google Cloud:** Zero Trust Framework (Assume Breach, Least Privilege)39.

*   **Kong:** 5 Best Practices for Securing Microservices40.

*   **CNCF:** Data on Kubernetes Whitepaper (Database Patterns)  
    *Reference for securing stateful workloads (TLS, Encryption at Rest).*

*   **CNCF:** Operator White Paper  
    *Reference for securing Operators and understanding their lifecycle.*  

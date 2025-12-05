# Zero Trust Kubernetes Implementation Guide

Welcome to the **KubeSquire Zero Trust Implementation** repository. This project demonstrates how to deploy the Google Microservices Demo ("Online Boutique") using a "Security by Design" approach, adhering to CNCF Secure Defaults.

This README serves as your map to the documentation, explaining **what** to read, **when** to read it, and **how** the repository is structured.

---

## 🚀 Quick Start: The 8-Hour Sprint

**Target Audience:** Engineers who want to build the environment *now*.  
**Goal:** Go from zero to a fully secured cluster in one working day.

*   **[Start Here: 8-Hour Zero Trust Sprint](8-hour-zero-trust-sprint.md)**  
    This is the primary execution guide. It condenses all the theory into actionable PowerShell/CLI steps. It covers:
    *   Setting up the "Greenfield" environment.
    *   Deploying via ArgoCD (GitOps).
    *   Enforcing mTLS (Istio), Network Policies (Cilium), and Policy-as-Code (Kyverno).

---

## 📚 Detailed Walkthroughs (The "Deep Dive")

**Target Audience:** Engineers debugging issues or seeking to understand the *why* behind specific configurations.  
**Goal:** Step-by-step explanations of each security layer.

If you get stuck during the Sprint, or want to implement just *one* component (like Secrets Management) in your own cluster, refer to these modules:

1.  **[Phase 0: Prerequisites & Setup](walkthroughs/00-prerequisites-and-setup.md)**  
    *   *Read when:* You are setting up your Windows/WSL environment for the first time. Lists required tools (Helm, Kustomize, etc.).
2.  **[Phase 1: Secure Supply Chain](walkthroughs/01-secure-supply-chain.md)**  
    *   *Read when:* You want to understand how we template the application and validate manifests before deployment.
3.  **[Phase 2: Identity & Segmentation](walkthroughs/02-identity-segmentation.md)**  
    *   *Read when:* You need to configure Istio mTLS or AuthorizationPolicies.
4.  **[Phase 3: GitOps Automation](walkthroughs/03-gitops-automation.md)**  
    *   *Read when:* You are setting up ArgoCD or troubleshooting sync issues.
5.  **[Phase 4: Secrets Management](walkthroughs/04-secrets-management.md)**  
    *   *Read when:* You need to integrate External Secrets Operator (ESO) with a vault.
6.  **[Phase 5: Stateful Zero Trust](walkthroughs/05-stateful-zero-trust.md)**  
    *   *Read when:* You are dealing with persistent data and database security.
7.  **[Phase 6: Network Segmentation](walkthroughs/06-network-segmentation-pointer.md)**  
    *   *Read when:* You are configuring Cilium Network Policies or Egress filtering.
8.  **[Phase 7: Deep Observability](walkthroughs/07-deep-observability.md)**  
    *   *Read when:* You need to set up Hubble, Prometheus, or Grafana for visibility.
9.  **[Phase 8: Infrastructure Hardening](walkthroughs/08-infra-hardening.md)**  
    *   *Read when:* You are auditing the cluster nodes or API server configuration.
10. **[Phase 9: Operator Security](walkthroughs/09-operator-security.md)**  
    *   *Read when:* You are writing or securing Kubernetes Operators.

---

## 🏛️ Architecture & Theory (Whitepapers)

**Target Audience:** Architects, Security Leads, and Decision Makers.  
**Goal:** Understand the design philosophy, compliance alignment, and industry standards.

*   **[Zero Trust Enforcement in Kubernetes Security](Zero%20Trust%20Enforcement%20in%20Kubernetes%20Security.md)**  
    *   *Read when:* You need the high-level proposal, executive summary, or mapping to CNCF standards. This explains the "Greenfield" vs. "Brownfield" philosophy.
*   **[Data on Kubernetes (DoK) Whitepaper](data-on-kubernetes-whitepaper-databases.md)**  
    *   *Read when:* Designing stateful workloads on K8s.
*   **[Operator Whitepaper](Operator-WhitePaper_v1-0.md)**  
    *   *Read when:* Understanding the security implications of the Operator pattern.

---

## 📂 Repository Structure

*   **`apps/`**: Contains the application source code and Kustomize bases.
    *   `apps/online-boutique/base`: The upstream manifests.
    *   `apps/online-boutique/overlays`: The environment-specific configurations (Production, Dev).
*   **`infrastructure/`**: Contains the manifests for the security platform itself.
    *   `kyverno-install.yaml`, `falco-exception.yaml`, `argocd-install.yaml`, etc.
*   **`walkthroughs/`**: The detailed documentation modules listed above.

---

## 🛠️ Troubleshooting & Common Fixes

*   **Falco Pods Pending?**  
    See the [Sprint Guide (Hour 8)](8-hour-zero-trust-sprint.md#hour-8-verification--observability) for the CPU request patch.
*   **Kyverno Blocking Deployments?**  
    Ensure you have applied the `exceptionNamespace` patch described in [Hour 7](8-hour-zero-trust-sprint.md#hour-7-policy-as-code-kyverno).

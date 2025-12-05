# Presentation Guide: Zero Trust Enforcement in Kubernetes

**Title:** Zero Trust Enforcement in Kubernetes: A "Security by Design" Blueprint  
**Format:** Live Demo / Technical Lecture  
**Goal:** Demonstrate how to template and automate a secure-by-default microservices platform.

---

## 1. The "Why" & The Philosophy (Slides 1-3)

### **Concept: Security by Design vs. Bolted On**
*   **The Old Way (Bolted On):** Deploy the app first. Then install security tools. Then spend weeks fighting "false positives" and breaking the app trying to lock it down.
*   **Our Way (Greenfield/Security by Design):** The security "enclosure" (Policies, Mesh, Network Rules) is built *before* the app arrives. The app must conform to the environment, not vice versa.

### **Why the Google Microservices Demo?**
*   **Polyglot:** It uses Go, Java, Python, Node.js, and C#. This proves our security model works across different languages/runtimes.
*   **Real Complexity:** It uses gRPC and HTTP, requiring a real Service Mesh (Istio) to manage traffic.
*   **Insecure by Default:** The default upstream demo runs containers as `root`, has no network boundaries, and uses unencrypted traffic. It is the perfect "hostile" workload to tame.

---

## 2. The Workflow: App Team vs. Platform Team (Slide 4)

**Visual:** Open VS Code and show the folder structure.

### **The Separation of Concerns**
*   **The App Team (Developers):**
    *   **Where they live:** `apps/online-boutique/base`
    *   **Their Job:** They define the *Deployment* and *Service*. They care about features and image tags.
    *   **Evidence:** Show `apps/online-boutique/base/kustomization.yaml`. It's clean, simple, and has no security clutter.

*   **The Platform Team (Security/Infra):**
    *   **Where they live:** `apps/online-boutique/overlays/production`
    *   **Their Job:** They wrap the app in security controls *without modifying the app code*.
    *   **Evidence:** Show `apps/online-boutique/overlays/production/kustomization.yaml`.
    *   **Highlight:** Point out the `patches` section where we inject `runAsNonRoot: true`. This is the Platform team enforcing rules on the App team's code automatically.

---

## 3. The Artifacts & Evidence (The "Meat" of the Demo)

### **Layer 1: Identity & Segmentation (Istio & Cilium)**
*   **Concept:** "Assume Breach." If one pod is hacked, it shouldn't be able to talk to everything.
*   **Artifacts to Show:**
    1.  `apps/online-boutique/overlays/production/peer-auth.yaml`: **Strict mTLS**. No unencrypted traffic allowed.
    2.  `apps/online-boutique/overlays/production/deny-all.yaml`: **Default Deny**. The network is closed by default.
    3.  `apps/online-boutique/overlays/production/allow-internal.yaml`: **The Allowlist**. We explicitly define who can talk to whom.
*   **Live Evidence (Command):**
    ```bash
    # Show that mTLS is enforced mesh-wide
    kubectl get peerauthentication -A
    ```
*   **GCP Console Demo (Alternative):**
    1.  Go to **GKE Console** -> **Workloads**.
    2.  Click on any `online-boutique` workload (e.g., `frontend`).
    3.  Scroll to **Observability** or **Service Mesh** tab (if enabled).
    4.  **[INSERT SCREENSHOT: Show the "mTLS: Strict" badge or the Service Mesh topology graph showing lock icons on connections]**

### **Layer 2: Policy as Code (Kyverno)**
*   **Concept:** "Guardrails." Stop insecure configurations from ever entering the cluster.
*   **Artifacts to Show:**
    1.  `infrastructure/require-non-root.yaml`: The policy that bans root users.
    2.  `infrastructure/falco-exception.yaml`: The "Real World" nuance. We admit that *some* security tools (like Falco) need exceptions, and we manage them explicitly as code.
*   **Live Evidence (Command):**
    ```bash
    # Try to deploy a "Bad Pod" (Root user)
    kubectl apply -f infrastructure/bad-pod.yaml
    # EXPECTED OUTPUT: Error from server (Forbidden): ... validation error: Running as root is not allowed.
    ```
*   **GCP Console Demo (Alternative):**
    1.  Go to **GKE Console** -> **Workloads**.
    2.  Look for the `bad-pod` (it won't be there, or will be in error).
    3.  Go to **Kubernetes Engine** -> **Object Browser**.
    4.  Filter for `ClusterPolicy` to show the `require-non-root` policy is active.
    5.  **[INSERT SCREENSHOT: The terminal error message showing the Kyverno rejection]**

### **Layer 3: Runtime Security (Falco)**
*   **Concept:** "The Security Camera." If a policy is bypassed, we need to see it happening.
*   **Artifacts to Show:**
    1.  `infrastructure/falco-exception.yaml`: Show how we allowed Falco to run as privileged so it can monitor others.
*   **Live Evidence (Command):**
    ```bash
    # 1. Exec into a pod (Simulate an attacker getting a shell)
    kubectl exec -it -n online-boutique <any-pod-name> -- sh

    # 2. Trigger a rule
    cat /etc/shadow

    # 3. Show the alert (in a separate terminal)
    kubectl logs -l app.kubernetes.io/name=falco -n falco | Select-String "Sensitive"
    ```
*   **GCP Console Demo (Alternative):**
    1.  Go to **GKE Console** -> **Workloads**.
    2.  Click on a `falco` pod.
    3.  Click the **Logs** tab.
    4.  Filter logs for "Notice" or "Sensitive".
    5.  **[INSERT SCREENSHOT: The JSON log entry from Falco showing "Notice: Read sensitive file trusted=false"]**

### **Layer 4: Secrets Management (External Secrets Operator)**
*   **Concept:** "No Hardcoded Secrets." Base64 secrets in Git are a vulnerability. We fetch them on-demand.
*   **Artifacts to Show:**
    1.  `apps/online-boutique/overlays/production/cluster-secret-store.yaml`: The connection to the Vault (GCP Secret Manager/AWS Secrets Manager).
    2.  `apps/online-boutique/overlays/production/redis-external-secret.yaml`: The instruction to fetch the specific secret.
*   **Live Evidence (Command):**
    ```bash
    # Show the ExternalSecret status
    kubectl get externalsecrets -n online-boutique
    # EXPECTED OUTPUT: STATUS: SecretSynced, READY: True
    ```

### **Layer 5: Deep Observability (Hubble)**
*   **Concept:** "Never Trust, Always Verify." We need to see the dropped packets to prove our policies work.
*   **Live Evidence (Command):**
    ```bash
    # Open the Hubble UI
    cilium hubble ui
    ```
*   **GCP Console Demo (Alternative):**
    1.  If using **GKE Dataplane V2** (Cilium-based), go to **GKE Console** -> **Clusters**.
    2.  Select your cluster -> **Observability**.
    3.  **[INSERT SCREENSHOT: The Hubble Service Map showing red lines for blocked traffic]**

---

## 4. GitOps Automation (ArgoCD)
*   **Concept:** "Immutable Infrastructure." No one runs `kubectl apply` manually.
*   **Artifacts to Show:**
    1.  `infrastructure/argocd-app.yaml`: The bridge between Git and the Cluster.
*   **Talking Point:** If an attacker manually deletes a NetworkPolicy to open a backdoor, ArgoCD will see the "Drift" and immediately revert it, healing the security posture automatically.
*   **GCP Console Demo (Alternative):**
    1.  If using **Config Sync** (Google's GitOps tool), go to **GKE Console** -> **Config & Policy**.
    2.  Show the "Synced" status pointing to your GitHub repo.
    3.  **[INSERT SCREENSHOT: The ArgoCD UI Application view showing "Synced" and "Healthy" status]**

---

## 5. Future Work / Not Implemented (Honesty Section)

**"We built the Fortress, but we still need to secure the Supply Lines."**

1.  **Secure Supply Chain (Phase 1):**
    *   *Missing:* We are pulling images directly from Google.
    *   *Future:* Implement **Cosign** to sign images and **Trivy** to scan them in a CI pipeline before they reach ArgoCD.

2.  **Stateful Operators (Phase 5):**
    *   *Missing:* Redis is currently a simple Deployment.
    *   *Future:* Replace with **Redis Operator** to handle encrypted backups and automated failover.

3.  **Progressive Delivery (Phase 6):**
    *   *Missing:* We do rolling updates.
    *   *Future:* Implement **Argo Rollouts** to use Falco alerts as a "Gate." If Falco sees a shell during a canary deployment, automatically rollback.

4.  **Infrastructure Hardening (Phase 8):**
    *   *Missing:* Automated CIS Benchmarks.
    *   *Future:* Run **Kube-bench** as a CronJob to audit the nodes and API server configuration daily.

5.  **Operator Security (Phase 9):**
    *   *Missing:* Least-privilege RBAC for controllers.
    *   *Future:* Audit `ClusterRoleBindings` to ensure Operators like ArgoCD don't have unnecessary `cluster-admin` access.

---

## 6. Summary Checklist for the Demo

1.  [ ] **Open VS Code** to the root workspace.
2.  [ ] **Show the Folder Structure** (`apps` vs `infrastructure`).
3.  [ ] **Show the Kyverno Policy** (`require-non-root.yaml`).
4.  [ ] **Run the "Bad Pod" test** to prove admission control works.
5.  [ ] **Show the Falco Logs** (Live tail) and trigger a shell alert.
6.  [ ] **Show the ArgoCD UI** (if available) or the `argocd-app.yaml` file.

---

## 7. References & Further Reading

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

*   **CNCF:** Secure Defaults - Cloud Native 8  
    [https://github.com/cncf/tag-security/blob/main/community/resources/security-whitepaper/secure-defaults-cloud-native-8.md](https://github.com/cncf/tag-security/blob/main/community/resources/security-whitepaper/secure-defaults-cloud-native-8.md)

*   **Local Whitepaper:** Data on Kubernetes (Database Patterns)  
    [data-on-kubernetes-whitepaper-databases.md](data-on-kubernetes-whitepaper-databases.md)

*   **Local Whitepaper:** Operator Security  
    [Operator-WhitePaper_v1-0.md](Operator-WhitePaper_v1-0.md)


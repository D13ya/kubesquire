# Phase 8: Infrastructure Hardening & Human Access Walkthrough

**Objective:** Secure the underlying nodes and restrict human access to the cluster.

**Prerequisites:**
*   **Tools:** `kubectl`.
*   **Identity Provider:** Google, Okta, or Azure AD configured for OIDC.

**Official Documentation:**
*   [Kube-Bench GitHub](https://github.com/aquasecurity/kube-bench)
*   [Kubernetes OIDC Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)
*   [Kubectl OIDC Login](https://github.com/int128/kubelogin)

## 1. Node Hardening (CIS Benchmarks)

We use `kube-bench` to check if our nodes meet the Center for Internet Security (CIS) standards.

### Step 1.1: Run Kube-Bench
Deploy `kube-bench` as a Job.

```yaml
# kube-bench-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
  namespace: kube-system
spec:
  template:
    spec:
      hostPID: true
      containers:
      - name: kube-bench
        image: aquasec/kube-bench:latest
        command: ["kube-bench", "run", "--targets", "node"]
        volumeMounts:
        - name: var-lib-kubelet
          mountPath: /var/lib/kubelet
          readOnly: true
        - name: etc-systemd
          mountPath: /etc/systemd
          readOnly: true
        - name: etc-kubernetes
          mountPath: /etc/kubernetes
          readOnly: true
      restartPolicy: Never
      volumes:
      - name: var-lib-kubelet
        hostPath:
          path: "/var/lib/kubelet"
      - name: etc-systemd
        hostPath:
          path: "/etc/systemd"
      - name: etc-kubernetes
        hostPath:
          path: "/etc/kubernetes"
```

### Step 1.2: Analyze Results
```bash
kubectl logs job/kube-bench -n kube-system
# Look for [FAIL] or [WARN] and remediate the node configuration.
```

## 2. Human Access (OIDC & JIT)

*Note: This requires configuring your Identity Provider (Google, Okta, Azure AD).*

### Concept: No More Long-Lived Keys
Instead of giving developers a `kubeconfig` with a permanent certificate, configure the API server to trust an OIDC provider.

1.  **User** logs in via `kubectl oidc-login`.
2.  **IdP** issues a short-lived ID Token (e.g., 1 hour).
3.  **K8s API** validates the token and maps the user's groups to RBAC Roles.

### Concept: Just-in-Time (JIT) Access
For "Break Glass" scenarios (e.g., debugging a prod outage):
1.  Developer requests `cluster-admin` access via a portal (e.g., Sym, PagerDuty).
2.  Approval is granted.
3.  A temporary `RoleBinding` is created for that user for 1 hour.
4.  The binding is automatically deleted after the window expires.

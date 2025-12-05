# Phase 7: Deep Observability (Verification) Walkthrough

**Objective:** Visualize network flows and verify that your Zero Trust policies are actually dropping traffic.

**Prerequisites:**
*   **Tools:** `cilium` CLI, `hubble` CLI.
*   **Cluster:** Cilium installed with Hubble enabled.

**Official Documentation:**
*   [Hubble Observability](https://docs.cilium.io/en/stable/observability/hubble/intro/)
*   [Tetragon Documentation](https://tetragon.io/docs/)

## 1. Enable Hubble

If you installed Cilium in Phase 2, enable Hubble for visibility.

```bash
cilium hubble enable --ui
```

## 2. Verify Policy Enforcement

### Step 2.1: Access Hubble UI
```bash
cilium hubble ui
# Opens in browser at localhost:12000
```

### Step 2.2: Visualize Dropped Traffic
1.  Select the `online-boutique` namespace.
2.  Look for **Red Lines**. These represent dropped traffic.
3.  Click on a red flow to see *why* it was dropped (e.g., "Policy denied").

### Step 2.3: CLI Verification
You can also verify via CLI if you don't have UI access.

```bash
# Observe traffic from frontend
kubectl exec -it -n kube-system ds/cilium -- hubble observe --from-pod online-boutique/frontend-xyz

# Filter for dropped packets
kubectl exec -it -n kube-system ds/cilium -- hubble observe --verdict DROPPED
```

## 3. Runtime Security with Tetragon (Optional)

Tetragon provides kernel-level visibility and enforcement.

### Step 3.1: Install Tetragon
```bash
helm repo add cilium https://helm.cilium.io/
helm install tetragon cilium/tetragon -n kube-system
```

### Step 3.2: Detect Shell Access
Tetragon logs when a shell is spawned in a container (a common attack vector).

```bash
# Tail logs for process execution
kubectl logs -n kube-system -l app.kubernetes.io/name=tetragon -c export-stdout -f | grep "process_exec"
```

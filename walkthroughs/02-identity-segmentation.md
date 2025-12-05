# Phase 2: Identity-Based Segmentation Walkthrough

**Objective:** Enforce "Zero Trust" networking where every request is authenticated, authorized, and encrypted.

**Prerequisites:**
*   **Tools:** `istioctl`, `cilium` CLI, `kubectl`.
*   **Cluster:** A Kubernetes cluster (GKE/EKS/Kind).
*   **CNI:** Cilium must be installed as the CNI plugin. If using GKE, use Dataplane V2 or install Cilium manually.

**Official Documentation:**
*   [Istio Installation Guide](https://istio.io/latest/docs/setup/getting-started/)
*   [Istio Security Concepts](https://istio.io/latest/docs/concepts/security/)
*   [Cilium Network Policy](https://docs.cilium.io/en/stable/security/policy/)
*   [Cilium Egress Gateway](https://docs.cilium.io/en/stable/network/egress-gateway/)

## 1. Service Mesh (Istio) Setup

### Step 1.1: Install Istio
Install Istio with the `demo` profile (good for learning/testing) or `minimal` for production.
*   **Note:** The `demo` profile enables high levels of tracing and access logging, which is useful for debugging but consumes more resources. For production, use `minimal` and customize.

```bash
# Install Istio
istioctl install --set profile=demo -y

# Label the namespace for sidecar injection
kubectl label namespace online-boutique istio-injection=enabled
```

### Step 1.2: Enable Strict mTLS
Force all traffic in the `online-boutique` namespace to be encrypted. This prevents any non-mesh traffic from communicating with your services.

```yaml
# mtls-strict.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: online-boutique
spec:
  mtls:
    mode: STRICT
```

Apply it:
```bash
kubectl apply -f mtls-strict.yaml
```

## 2. Authorization Policies (L7 Access Control)

Define who can talk to whom. We start with a "Deny All" and then allow specific paths.

### Step 2.1: Deny All by Default
This policy ensures that even if mTLS is established, no requests are allowed unless explicitly permitted.

```yaml
# deny-all.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: online-boutique
spec:
  {} # Empty spec means deny all
```

### Step 2.2: Allow Frontend to CartService
Explicitly allow the `frontend` service account to call `cartservice` on specific gRPC paths.

```yaml
# allow-frontend-to-cart.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-cart
  namespace: online-boutique
spec:
  selector:
    matchLabels:
      app: cartservice
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/online-boutique/sa/frontend"]
    to:
    - operation:
        methods: ["POST", "GET"]
        paths: ["/hipstershop.CartService/*"] # gRPC paths
```

## 3. Network Policies (Cilium L3/L4 & Egress)

**Requirement:** Ensure `CiliumNetworkPolicy` CRDs are installed.

### Step 3.1: Restrict Egress (DNS & External APIs)
Block all internet access except for Google Cloud Monitoring and DNS. This prevents compromised pods from reaching out to C2 servers.

```yaml
# egress-policy.yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: restrict-egress
  namespace: online-boutique
spec:
  endpointSelector:
    matchLabels: {} # Select all pods in namespace
  egress:
  # Allow DNS (Required for resolving external names)
  - toEndpoints:
    - matchLabels:
        k8s-app: kube-dns
        io.kubernetes.pod.namespace: kube-system
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
  # Allow Google APIs (for Monitoring/Tracing)
  # Note: This requires Cilium DNS Proxy to be enabled
  - toFQDNs:
    - matchName: "monitoring.googleapis.com"
    - matchName: "cloudtrace.googleapis.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

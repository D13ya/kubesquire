# Phase 2: Identity-Based Segmentation Walkthrough

**Objective:** Enforce "Zero Trust" networking where every request is authenticated, authorized, and encrypted.

## 1. Service Mesh (Istio) Setup

### Step 1.1: Install Istio
Install Istio with the `demo` profile (good for learning) or `minimal` for production.

```bash
istioctl install --set profile=demo -y
```

### Step 1.2: Enable Strict mTLS
Force all traffic in the `online-boutique` namespace to be encrypted.

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

### Step 3.1: Restrict Egress (DNS & External APIs)
Block all internet access except for Google Cloud Monitoring and DNS.

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
  # Allow DNS
  - toEndpoints:
    - matchLabels:
        k8s-app: kube-dns
        io.kubernetes.pod.namespace: kube-system
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
  # Allow Google APIs (for Monitoring/Tracing)
  - toFQDNs:
    - matchName: "monitoring.googleapis.com"
    - matchName: "cloudtrace.googleapis.com"
    toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

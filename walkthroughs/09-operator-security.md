# Phase 9: Operator Security & Auditing Walkthrough

**Objective:** Ensure the tools managing your security (ArgoCD, ESO, Istio) are not themselves vulnerabilities.

## 1. Audit Operator Permissions

Many Operators request `cluster-admin` by default. We must verify if they actually need it.

### Step 1.1: Check ClusterRoleBindings
List who has `cluster-admin`.

```bash
kubectl get clusterrolebindings -o json | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .subjects[] | .name'
```
*Output should be minimal (e.g., `system:masters`, `cluster-admin`). If you see `argocd-server` or `external-secrets`, investigate.*

## 2. Restrict Operator Scope

Instead of giving an Operator power over the whole cluster, give it power only over specific namespaces.

### Example: Restricting External Secrets
If ESO only needs to manage secrets in `online-boutique`, do not give it a ClusterRole.

```yaml
# RoleBinding instead of ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: eso-secret-manager
  namespace: online-boutique
subjects:
- kind: ServiceAccount
  name: external-secrets
  namespace: external-secrets
roleRef:
  kind: Role
  name: secret-admin # A Role that allows create/update Secrets ONLY in this namespace
  apiGroup: rbac.authorization.k8s.io
```

## 3. Network Policies for Operators

Operators often don't need ingress access from the internet.

### Example: Lock Down ArgoCD Redis
ArgoCD's Redis cache should only be accessible by the ArgoCD Repo Server and Application Controller.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: argocd-redis-allow-internal
  namespace: argocd
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: argocd-redis
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: argocd-server
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: argocd-application-controller
```

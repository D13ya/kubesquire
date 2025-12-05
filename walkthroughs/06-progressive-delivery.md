# Phase 6: Progressive Delivery Walkthrough

**Objective:** Automate safe deployments using Canary releases gated by real-time metrics.

## 1. Install Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

## 2. Convert Deployment to Rollout

We will convert the `frontend` Deployment to a `Rollout`.

### Step 2.1: Define the Rollout
```yaml
# frontend-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: frontend
  namespace: online-boutique
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: server
        image: gcr.io/google-samples/microservices-demo/frontend:v0.8.0
        ports:
        - containerPort: 8080
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 1m}
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 50
      - pause: {duration: 10s}
      - setWeight: 100
```

## 3. Define Analysis Template (The Gate)

This template queries Prometheus to ensure the new version isn't throwing errors.

```yaml
# analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: online-boutique
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.99
    provider:
      prometheus:
        address: http://prometheus-server.monitoring.svc.cluster.local:9090
        query: |
          sum(irate(istio_requests_total{reporter="source",destination_service=~"{{args.service-name}}",response_code!~"5.*"}[1m])) / 
          sum(irate(istio_requests_total{reporter="source",destination_service=~"{{args.service-name}}"}[1m]))
```

## 4. Trigger a Rollout

1.  Update the image in `frontend-rollout.yaml` to a new version.
2.  Apply the change.
3.  Watch the rollout:
    ```bash
    kubectl argo rollouts get rollout frontend -n online-boutique --watch
    ```
    You will see it pause at 20%, run the analysis, and if successful, proceed to 100%.

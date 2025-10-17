# Canary Deployment with Argo Rollouts and Istio
Implementing automated rollbacks with Argo Rollouts and Istio for canary deployments involves integrating Argo Rollouts' advanced deployment capabilities with Istio's traffic management and adding automated analysis to trigger rollbacks based on metrics. Below is a detailed guide to set this up, building on the myapp canary deployment example provided earlier. This guide assumes you have a Kubernetes cluster with Istio installed and the myapp service, Gateway, VirtualService, and DestinationRule already configured.

**Overview of Automated Rollbacks with Argo Rollouts and Istio**
Argo Rollouts is a Kubernetes controller that extends the Deployment resource with advanced strategies like canary deployments. It integrates with Istio for traffic splitting and supports AnalysisTemplates to monitor metrics and automate promotion or rollback. Automated rollbacks occur when the canary version fails to meet predefined success criteria (e.g., high error rates or latency), reverting traffic to the stable version.

The workflow involves:
- 1. Replacing the Kubernetes Deployment with an Argo Rollout resource.
- 2. Configuring Istio for traffic splitting between stable and canary versions.
- 3. Defining an AnalysisTemplate to monitor metrics (e.g., via Prometheus).
- 4. Setting up the Rollout to trigger automated promotion or rollback based on analysis results.
 
**Prerequisites**
- **Kubernetes Cluster:** Running with Istio installed (e.g., version 1.18+).
- **Istio Gateway and VirtualService:** As defined in the previous myapp-gateway.yaml and myapp-virtualservice.yaml.
- **DestinationRule:** From the previous example, defining v1 and v2 subsets.
- **Argo Rollouts Installed:**
```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```
- **Prometheus:** Installed for metrics collection (Istio’s Prometheus or another instance).
- **Kiali/Grafana:** Optional for visualization.
- **myapp Service:** Deployed with v1 (stable) and v2 (canary) versions, as in the previous example.

**Step-by-Step Implementation**
**Step 1: Replace Deployment with Argo Rollout**
Replace the Kubernetes Deployments (myapp-v1.yaml and myapp-v2.yaml) with a single Argo Rollout resource. The Rollout manages both stable and canary ReplicaSets, and Istio handles traffic splitting.
`myapp-rollout.yaml:`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 3
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: myapp
  strategy:
    canary:
      canaryService: myapp-canary # Service for canary pods
      stableService: myapp # Service for stable pods
      trafficRouting:
        istio:
          virtualService:
            name: myapp-virtualservice # References the VirtualService
            routes:
            - primary # Route name in VirtualService
          destinationRule:
            name: myapp-destinationrule # References the DestinationRule
            canarySubsetName: v2 # Matches subset in DestinationRule
            stableSubsetName: v1 # Matches subset in DestinationRule
      steps:
      - setWeight: 10 # Start with 10% traffic to canary
      - pause: { duration: 300 } # Pause for 5 minutes to analyze
      - setWeight: 50 # Increase to 50% if analysis passes
      - pause: { duration: 300 }
      - setWeight: 100 # Fully promote to 100% if analysis passes
      analysis:
        templates:
        - templateName: apdex-analysis # References AnalysisTemplate
  template:
    metadata:
      labels:
        app: myapp
        version: v2 # Canary version (v2)
    spec:
      containers:
      - name: myapp
        image: jamaldevsecops/myapp:v2 # Canary image
        ports:
        - containerPort: 80
```

**Notes:**
- `canaryService` and `stableService`: Define services for canary and stable pods. The `myapp` service already exists; create a `myapp-canary` service.
- `trafficRouting.istio`: Configures Istio integration, referencing the VirtualService and DestinationRule.
- `steps`: Defines the canary traffic increments (10%, 50%, 100%) with pauses for analysis.
- `analysis.templates`: References an AnalysisTemplate for automated checks.
- Create Canary Service (`myapp-canary-service.yaml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-canary
  namespace: default
spec:
  selector:
    app: myapp
  ports:
  - port: 8080
    targetPort: 80
```
Apply the manifests:
```bash
kubectl apply -f myapp-canary-service.yaml
kubectl apply -f myapp-rollout.yaml
```

**Step 2: Update Istio Resources**  
Ensure the **DestinationRule** and **VirtualService** are configured to work with Argo Rollouts. The previous `myapp-destinationrule.yaml` is already correct, as it defines `v1` and `v2` subsets. Update the VirtualService to reference the stable and canary services or subsets.
**Updated `myapp-virtualservice.yaml` (using subset-based routing):**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-virtualservice
  namespace: default
spec:
  hosts:
  - "myapp.apsissolutions.com"
  gateways:
  - myapp-gateway
  http:
  - name: primary # Referenced in Rollout
    route:
    - destination:
        host: myapp.default.svc.cluster.local
        subset: v1 # Stable subset
      weight: 100
    - destination:
        host: myapp.default.svc.cluster.local
        subset: v2 # Canary subset
      weight: 0
```
Apply:
```bash
kubectl apply -f myapp-virtualservice.yaml
```

**Step 3: Define AnalysisTemplate for Automated Rollbacks**  
Create an **AnalysisTemplate** to monitor the canary’s performance using Prometheus metrics (e.g., HTTP error rate). If the canary fails the success criteria, Argo Rollouts automatically rolls back by reverting traffic to the stable version (v1).
`apdex-analysis.yaml:`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: apdex-analysis
  namespace: default
spec:
  metrics:
  - name: error-rate
    interval: 60s # Check every 60 seconds
    successCondition: result[0] <= 0.05 # Success if error rate <= 5%
    failureLimit: 3 # Fail after 3 consecutive failures
    provider:
      prometheus:
        address: http://prometheus.istio-system.svc.cluster.local:9090
        query: |
          sum(rate(istio_requests_total{
            destination_service="myapp.default.svc.cluster.local",
            destination_version="v2",
            response_code=~"5.*"
          }[5m]))
          /
          sum(rate(istio_requests_total{
            destination_service="myapp.default.svc.cluster.local",
            destination_version="v2"
          }[5m]))
```
**Notes:**
- metrics: Defines the metric to check (here, HTTP 5xx error rate for v2).
- successCondition: The canary is considered successful if the error rate is ≤5%.
- failureLimit: Rollback occurs after 3 failed checks.
- prometheus.query: Calculates the error rate for the v2 subset using Istio’s metrics.
- Adjust the Prometheus address to match your setup.
Apply:
```bash
kubectl apply -f apdex-analysis.yaml
```

**Step 4: Trigger and Monitor the Canary Deployment**    
1. **Trigger the Rollout:**
   The Rollout starts when you apply the myapp-rollout.yaml or update the image:
   ```bash
   kubectl argo rollouts set image myapp myapp=myapp:2.0 -n default
   ```
2. **Monitor the Rollout:**  
   Use the Argo Rollouts CLI to track progress:
   ```bash
   kubectl argo rollouts get rollout myapp -n default --watch
   ```
   Example output (during canary):
   ```text
   Name:            myapp
    Namespace:       default
    Status:          ॥ Paused
    Strategy:        Canary
    Step:            1/3
    SetWeight:       10
    ActualWeight:    10
    Images:          myapp:1.0 (stable), myapp:2.0 (canary)
    Replicas:
      Desired:       3
      Current:       4
      Updated:       1
      Ready:         4
      Available:     4
    ```
3. **Check AnalysisRun:**  
   Argo creates an `AnalysisRun` to evaluate the canary. View it:
   ```bash
   kubectl get analysisrun -n default
   ```
   If the error rate exceeds 5% for three intervals, the Rollout aborts, and traffic reverts to 100% v1.
   
4. **Promote or Rollback:**  
   - If the analysis succeeds, Argo promotes the canary (v2) to stable, scaling down v1.
   - If the analysis fails, Argo rolls back by setting setWeight: 0 for v2 and 100 for v1 in the VirtualService.



**Step 5: Visualize and Validate**  
- Kiali: Check traffic distribution at http://<kiali-url> to confirm 10% traffic to v2.
- Prometheus: Query the error rate:
```promql
sum(rate(istio_requests_total{destination_service="myapp.default.svc.cluster.local", destination_version="v2", response_code=~"5.*"}[5m])) / sum(rate(istio_requests_total{destination_service="myapp.default.svc.cluster.local", destination_version="v2"}[5m]))
```
- Grafana: Use dashboards to monitor latency and errors.
- Argo Rollouts Dashboard:
```bash
kubectl argo rollouts dashboard -n default
```
Access at http://localhost:3100/rollouts.

**Step 6: Rollback or Complete**   
- Automatic Rollback: If the error rate exceeds 5% for three intervals, Argo Rollouts automatically reverts traffic to v1 (stable) and scales down the v2 ReplicaSet.
- Manual Promotion: If you want to manually promote the canary:
```bash
kubectl argo rollouts promote myapp -n default
```
- Completion: Once v2 is fully promoted (100% traffic), the v1 ReplicaSet is scaled down, and v2 becomes the stable version.

**Example Workflow**  
1. Deploy v1 (stable) with the Rollout, serving 100% traffic.
2. Update the Rollout to use myapp:2.0 (v2 canary).
3. Argo Rollouts:
  - Creates a v2 ReplicaSet.
  - Updates the VirtualService to route 10% traffic to v2.
  - Runs the AnalysisTemplate to check the error rate.
4. If the error rate is ≤5%, Argo proceeds to 50% traffic, then 100%.
5. If the error rate >5% for three intervals, Argo reverts to 100% v1 and scales down v2.
6. Once v2 is stable, v1 is removed.

**Best Practices**  
- **Define Clear Metrics:** Use specific metrics (e.g., error rate, latency, Apdex score) for analysis. Avoid overly sensitive thresholds to prevent false positives.
- **Short Intervals:** Set short interval (e.g., 60s) for quick feedback during analysis.
- **Failure Limits:** Configure failureLimit to balance sensitivity and stability (e.g., 3 failures).
- **Notifications:** Integrate with Slack or email for deployment events using Argo Rollouts notifications.
- Testing: Use traffic mirroring to test v2 without user impact before the canary.
- **GitOps:** Use ArgoCD with Argo Rollouts for Git-based manifest management.
- **Resource Optimization:** Enable dynamicStableScale: true in the Rollout to scale down the stable ReplicaSet as canary traffic increases, reducing resource usage.

**Troubleshooting**
- **Analysis Fails:** Check AnalysisRun logs (kubectl describe analysisrun -n default) for Prometheus query issues.
- **Traffic Not Splitting:** Verify VirtualService weights in Kiali or kubectl describe virtualservice myapp-virtualservice.
- **Rollout Stuck:** Check Rollout status (kubectl argo rollouts get rollout myapp) for pause conditions or errors.
- **Prometheus Unreachable:** Ensure the Prometheus address is correct and accessible.


This setup enables automated rollbacks by integrating Argo Rollouts with Istio and Prometheus, ensuring safe canary deployments with minimal manual intervention.














# Canary Deployment with Istio

A ***Canary deployment*** is a strategy for rolling out new versions of an application gradually to a small subset of users before deploying it to the entire user base. This minimizes risk by allowing you to test the new version in production with real traffic while monitoring its behavior. Istio, a service mesh, provides powerful tools to manage canary deployments by controlling traffic routing and splitting.
Below is a full workflow for implementing a canary deployment using Istio, including prerequisites, steps, examples, and monitoring.

## Canary Deployment Workflow with Istio
**1. Prerequisites**
  - **Kubernetes Cluster**: A running Kubernetes cluster (e.g., Minikube, GKE, EKS).
  - **Istio Installed**: Istio service mesh installed on the cluster (version 1.18 or later recommended for stability).
      - Install Istio using istioctl install or Helm.
      - Enable Istio injection for the namespace (e.g., kubectl label namespace default istio-injection=enabled).
  - **Sample Application**: A microservice-based application deployed in Kubernetes with Istio sidecars.
      - For this example, we’ll use a simple "Bookinfo" application or a custom app with two versions (v1 and v2).
  - **Monitoring Tools**: Tools like Prometheus, Grafana, or Kiali for observing traffic and performance.
  - **kubectl and istioctl**: Command-line tools for interacting with Kubernetes and Istio.

**2. Workflow Steps**
1. **Deploy the Application Versions**
    - Deploy both the stable (v1) and canary (v2) versions of your application in the same Kubernetes namespace.
    - Ensure each version is labeled distinctly (e.g., app: myapp, version: v1 and app: myapp, version: v2).
2. **Define Istio Resources**
    - Create a DestinationRule to define subsets for each version based on labels.
    - Create a VirtualService to control traffic routing, initially directing 100% of traffic to v1.
3. **Route a Small Percentage of Traffic to the Canary (v2)**
    - Update the VirtualService to split traffic (e.g., 90% to v1, 10% to v2).
    - Use weighted routing to control the traffic split.
4. **Monitor the Canary**
    - Use Istio’s telemetry (Prometheus, Grafana, Kiali) to monitor metrics like request success rate, latency, and errors for the canary version.
    - Validate the canary’s behavior with logs or external monitoring tools.
5. **Gradually Increase Traffic to the Canary**
    - If the canary is stable, incrementally increase traffic to v2 (e.g., 50% v1, 50% v2, then 10% v1, 90% v2).
    - Update the VirtualService for each traffic shift.
6. **Roll Back or Complete the Deployment**
    - If issues arise, roll back by routing 100% traffic to v1.
    - If the canary is successful, route 100% traffic to v2 and clean up v1 resources.
7. **Clean Up**
    - Remove the old version (v1) from the cluster once v2 is fully deployed and stable.

-----
## Example: Canary Deployment with Istio**
Let’s walk through a canary deployment for a sample application called myapp with two versions: v1 (stable) and v2 (canary).

**Step 1: Deploy Application Versions**
Assume `myapp` is a simple web service. Deploy both versions in Kubernetes.

**v1 Deployment** (`myapp-v1.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: myapp
        image: jamaldevsecops/myapp:v1
        ports:
        - containerPort: 80
```

**v2 Deployment** (`myapp-v2.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: myapp
        image: jamaldevsecops/myapp:v2
        ports:
        - containerPort: 80
```

**Service Definition (`myapp-service.yaml`):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
spec:
  selector:
    app: myapp
  ports:
  - port: 8080
    targetPort: 80
```

**Apply the manifests:**
```bash
kubectl apply -f myapp-v1.yaml
kubectl apply -f myapp-v2.yaml
kubectl apply -f myapp-service.yaml
```

**Step 2: Define Istio Resources** 
**DestinationRule (`myapp-destinationrule.yaml`):**
Define subsets for v1 and v2 based on their `version` labels.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: myapp-destinationrule
  namespace: default
spec:
  host: myapp.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**Istio Gateway**
The Gateway defines how external traffic enters the Istio service mesh.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: myapp-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway # Targets the default Istio ingress gateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: apsis-tls # Kubernetes secret with TLS cert
    hosts:
    - "myapp.apsissolutions.com" # Replace with your domain or use '*' for all hosts
```

**Create TLS Secret**  
Ensure you have: * CA_chain.crt – Your full certificates chain which includes 'Your main cert of the domain' > 'Any intermediate cert(s) > 'Root cert' in order. * apsissolutions.com.key – Your private key
Then create the secret in the default namespace:  
```bash
kubectl create secret tls apsis-tls \
  --cert=/home/ops/2025/CA_chain.crt \
  --key=/home/ops/2025/apsissolutions.com.key \
  -n default
```

**VirtualService (`myapp-virtualservice.yaml`):**
Initially route 100% traffic to v1.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-virtualservice
  namespace: default
spec:
  hosts:
  - myapp.apsissolutions.com # Matches the Gateway's host
  gateways:
  - myapp-gateway # References the Gateway
  http:
  - route:
    - destination:
        host: myapp.default.svc.cluster.local
        subset: v1
      weight: 100
```

Apply the Istio resources:
```bash
kubectl apply -f myapp-destinationrule.yaml
kubectl apply -f myapp-virtualservice.yaml
```

**Step 3: Route Traffic to Canary**
Update the VirtualService to split traffic, sending 10% to v2 (canary) and 90% to v1.
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-virtualservice
  namespace: default
spec:
  hosts:
  - myapp.apsissolutions.com
  gateways:
  - myapp-gateway 
  http:
  - route:
    - destination:
        host: myapp.default.svc.cluster.local
        subset: v1
      weight: 90
    - destination:
        host: myapp.default.svc.cluster.local
        subset: v2
      weight: 10
```
Apply the updated VirtualService:
```bash
kubectl apply -f myapp-virtualservice.yaml
```

**Step 4: Monitor the Canary**
Use Istio’s observability tools to monitor the canary:
  - **Kiali:** Visualize traffic flow and check request distribution between v1 and v2.
  - **Prometheus:** Query metrics like istio_requests_total to verify traffic split. Example query: sum(rate(istio_requests_total{destination_service="myapp.default.svc.cluster.local"}[5m])) by (destination_version)
  - **Grafana:** Use dashboards to monitor latency, error rates, and throughput.
  - **Logs:** Check pod logs for v2 to identify errors or anomalies.

Example: If v2 shows higher error rates (e.g., 5xx responses), investigate before increasing traffic.

**Step 5: Gradually Increase Traffic**
If v2 is stable, update the VirtualService to shift more traffic (e.g., 50% v1, 50% v2):

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-virtualservice
  namespace: default
spec:
  hosts:
  - myapp.apsissolutions.com
  gateways:
  - myapp-gateway 
  http:
  - route:
    - destination:
        host: myapp.default.svc.cluster.local
        subset: v1
      weight: 50
    - destination:
        host: myapp.default.svc.cluster.local
        subset: v2
      weight: 50
```
Apply the update:
```bash
kubectl apply -f myapp-virtualservice.yaml
```

If stable, route 100% traffic to v2:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-virtualservice
  namespace: default
spec:
  hosts:
  - myapp.apsissolutions.com
  gateways:
  - myapp-gateway 
  http:
  - route:
    - destination:
        host: myapp.default.svc.cluster.local
        subset: v2
      weight: 100
```
Apply:
```bash
kubectl apply -f myapp-virtualservice.yaml
```

**Step 6: Roll Back or Complete**
  - Rollback: If v2 fails (e.g., high error rates), revert to 100% v1 by applying the original VirtualService (100% to v1).
  - Complete: If v2 is successful, proceed to clean up.

**Step 7: Clean Up**
Once v2 is fully deployed and stable, delete the v1 deployment:
```bash
kubectl delete -f myapp-v1.yaml
```
Optionally, update the DestinationRule to remove the v1 subset.

**Additional Features with Istio**
1. **Traffic Mirroring (Shadowing):**
    - Send a copy of production traffic to v2 without serving it to users.
    - Example VirtualService for mirroring:
    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: myapp-virtualservice
      namespace: default
    spec:
      hosts:
      - myapp.apsissolutions.com
      gateways:
      - myapp-gateway 
      http:
      - route:
        - destination:
            host: myapp.default.svc.cluster.local
            subset: v1
          weight: 100
        mirror:
          host: myapp.default.svc.cluster.local
          subset: v2
    ```

2. **Canary by Header or User:**
    - Route traffic to v2 for specific users (e.g., based on headers).
    - Example: Route requests with header x-canary: true to v2:
    ```yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: myapp-virtualservice
      namespace: default
    spec:
      hosts:
      - myapp.apsissolutions.com
      gateways:
      - myapp-gateway 
      http:
      - match:
        - headers:
            x-canary:
              exact: "true"
        route:
        - destination:
            host: myapp.default.svc.cluster.local
            subset: v2
      - route:
        - destination:
            host: myapp.default.svc.cluster.local
            subset: v1
    ```
3. **Timeout and Retries:**
    - Add fault tolerance to the canary with Istio’s timeout and retry policies in the VirtualService.

**Monitoring Example**
To verify the traffic split, use Prometheus to check the request distribution:

```bash
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090
```
In Prometheus UI, query:
```text
sum(rate(istio_requests_total{destination_service="myapp.default.svc.cluster.local"}[5m])) by (destination_version)
```
Expected output: ~90% of requests to version:v1, ~10% to version:v2.

**Best Practices**
- Small Traffic Increments: Start with 1-10% traffic to the canary to minimize impact.
- Automated Rollbacks: Use tools like Argo Rollouts with Istio for automated canary analysis and rollback.
- Clear Metrics: Define success criteria (e.g., <1% error rate, <200ms latency) before increasing traffic.
- Testing: Test the canary with synthetic traffic or mirroring before exposing it to users.
- Kiali for Visualization: Use Kiali to visualize traffic flow and detect anomalies.

This workflow provides a robust way to perform canary deployments with Istio, leveraging its traffic management capabilities. If you need specific tweaks (e.g., header-based routing, automated rollouts), let me know!

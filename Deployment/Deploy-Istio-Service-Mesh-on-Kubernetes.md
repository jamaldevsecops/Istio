# 🚢 Istio Service Mesh Deployment on Kubernetes

## 🧩 Prerequisites
- ✅ Running **Kubernetes** cluster (kind / Minikube / GKE / EKS / AKS)
- ✅ `kubectl` connected to the cluster
- 🧠 Resources: ~4 vCPU / 8 GB RAM (demo profile)
- ⛑️ Optional: `helm` (for the alternate method)

Check connectivity:
```bash
kubectl cluster-info
```

---

## ⚙️ A) Install via `istioctl` (Recommended)

### 1) ⬇️ Install `istioctl`
**macOS/Linux**
```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH="$PWD/bin:$PATH"
istioctl version
```

### 2) 🧪 Pre-check the cluster (optional but helpful)
```bash
istioctl x precheck
```

### 3) 🏷️ Create control-plane namespace
```bash
kubectl create namespace istio-system
```

### 4) 🚀 Install the Istio control plane
- **Learning / sandbox (demo):**
```bash
istioctl install --set profile=demo -y
```
- **Lean / closer to prod (default):**
```bash
istioctl install --set profile=default -y
```
Example Output:
```
        |\
        | \
        |  \
        |   \
      /||    \
     / ||     \
    /  ||      \
   /   ||       \
  /    ||        \
 /     ||         \
/______||__________\
____________________
  \__       _____/
     \_____/

✔ Istio core installed ⛵️
✔ Istiod installed 🧠
✔ Egress gateways installed 🛫
✔ Ingress gateways installed 🛬
✔ Installation complete
```

Check components:
```bash
kubectl -n istio-system get pods
```
Example Output: 
```
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-5dcfcd4f76-726qz    1/1     Running   0          4h1m
istio-ingressgateway-54b44bc89d-zpfs6   1/1     Running   0          4h1m
istiod-567d49697-g4nbr                  1/1     Running   0          4h1m
```
You should see `istiod` and (for demo) `istio-ingressgateway`.

### 5) 🧱 Create **bookinfo** namespace
```bash
kubectl create namespace bookinfo
```

### 6) 🧩 Enable **automatic sidecar injection** for `bookinfo`
```bash
kubectl label namespace bookinfo istio-injection=enabled --overwrite
kubectl get namespace bookinfo --show-labels
```
Expect to see `istio-injection=enabled`.

### 🚪 Install the Kubernetes Gateway API CRDs
The Kubernetes Gateway API CRDs do not come installed by default on most Kubernetes clusters, so make sure they are installed before using the Gateway API.

Install the Gateway API CRDs, if they are not already present:
```bash
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
{ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.4.0" | kubectl apply -f -; }
```

### 7) 📘 (Optional) Deploy Bookinfo sample app
From the extracted Istio folder:
```bash
kubectl apply -n bookinfo -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -n bookinfo -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get pods -n bookinfo
```
Example Output: 
```
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-766844796b-4trsg       2/2     Running   0          5m9s
productpage-v1-54bb874995-tdqmr   2/2     Running   0          5m9s
ratings-v1-5dc79b6bcd-v6nqp       2/2     Running   0          5m9s
reviews-v1-598b896c9d-vf2nt       2/2     Running   0          5m9s
reviews-v2-556d6457d-fvscl        2/2     Running   0          5m9s
reviews-v3-564544b4d6-26p48       2/2     Running   0          5m9s
```

### 8) 🌐 Get ingress address & test
```bash
kubectl -n istio-system get svc istio-ingressgateway
```
Example Output:
```
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.104.115.93   192.168.20.161   15021:30216/TCP,80:32377/TCP,443:32008/TCP,31400:32347/TCP,15443:31302/TCP   4h37m
```
- **Cloud**: use `EXTERNAL-IP` → `http://<EXTERNAL-IP>/productpage`
- **Minikube**: run `minikube tunnel`, then open `http://localhost/productpage`
- **kind (no LB)**: port-forward:
```bash
kubectl -n istio-system port-forward svc/istio-ingressgateway 8080:80
# open http://localhost:8080/productpage
```

### 9) 🔍 Verify install
```bash
istioctl version
```
Example Output: 
```

```
client version: 1.28.0
control plane version: 1.28.0
data plane version: 1.28.0 (8 proxies)
---

## 🧰 B) Alternate Install via **Helm**

### 1) 📦 Add charts repo
```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
kubectl create namespace istio-system
```

### 2) 🧱 Base (CRDs)
```bash
helm install istio-base istio/base -n istio-system
```

### 3) 🧠 Control plane (istiod)
```bash
helm install istiod istio/istiod -n istio-system
# Example tweak:
# helm install istiod istio/istiod -n istio-system --set values.global.proxy.logLevel=warning
```

### 4) 🌉 Ingress gateway
```bash
helm install istio-ingress istio/gateway -n istio-system
```

### 5) 🧱 Create & label bookinfo
```bash
kubectl create namespace bookinfo
kubectl label namespace bookinfo istio-injection=enabled --overwrite
```

### 6) ✅ Verify
```bash
kubectl -n istio-system get pods
```

---

## 🧭 Common Next Steps & Best Practices
- 🔒 **mTLS / Zero-trust**: Start permissive → enforce with `PeerAuthentication` + `DestinationRule`.
- 📈 **Observability**: Add Kiali / Prometheus / Grafana / Jaeger (demo has some wiring; prod should be explicit).
- 🔁 **Safe upgrades** with **revisions**:
  ```bash
  istioctl install --revision v123x
  kubectl label ns bookinfo istio.io/rev=v123x --overwrite
  ```
- 🌀 **Ambient mode** (sidecar-less): evaluate if it fits your infra & policies before adopting.
- ⚖️ **Resource tuning**: Set requests/limits + HPA for `istiod` & gateways to match traffic.

---

## 🧼 Uninstall / Cleanup

### If installed with `istioctl`
```bash
istioctl uninstall --purge -y
kubectl delete namespace istio-system
kubectl delete namespace bookinfo
```

### If installed with **Helm**
```bash
helm uninstall istio-ingress -n istio-system
helm uninstall istiod -n istio-system
helm uninstall istio-base -n istio-system
kubectl delete namespace istio-system
kubectl delete namespace bookinfo
```

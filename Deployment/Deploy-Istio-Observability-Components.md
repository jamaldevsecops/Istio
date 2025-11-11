# 📊 Deploy Istio Observability Components

## 🧩 Prerequisites

Before proceeding, ensure:
- ✅ Istio is installed and running (per [previous setup guide](./istio_installation_guide.md))
- ✅ `kubectl` configured for your cluster
- ✅ Istio ingress gateway (`istio-ingressgateway`) is active in `istio-system`
- ✅ DNS or `/etc/hosts` entries for local domain access

Example `/etc/hosts` entries (replace `<INGRESS-IP>` accordingly):
```
<INGRESS-IP> grafanax.apsis.localnet
<INGRESS-IP> kialix.apsis.localnet
<INGRESS-IP> jaegerx.apsis.localnet
<INGRESS-IP> prometheusx.apsis.localnet
```

---

## ⚙️ 1) Deploy Istio Addons (Observability Tools)

Istio provides default manifests for observability addons in the Istio package.

```bash
cd istio-*
kubectl apply -f samples/addons
```

Check deployment status:
```bash
kubectl get pods -n istio-system
```

You should see pods for:
- `kiali`
- `prometheus`
- `grafana`
- `jaeger`

---

## 🌐 2) Expose Addons via Istio Ingress Gateway

We’ll create **Gateway** and **VirtualService** resources for each tool, using your domain names.

Save the following manifest as `observability-gateway.yaml`:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: observability-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "grafanax.apsis.localnet"
    - "kialix.apsis.localnet"
    - "jaegerx.apsis.localnet"
    - "prometheusx.apsis.localnet"
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: grafana-vs
  namespace: istio-system
spec:
  hosts:
  - "grafanax.apsis.localnet"
  gateways:
  - observability-gateway
  http:
  - route:
    - destination:
        host: grafana.istio-system.svc.cluster.local
        port:
          number: 3000
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: kiali-vs
  namespace: istio-system
spec:
  hosts:
  - "kialix.apsis.localnet"
  gateways:
  - observability-gateway
  http:
  - route:
    - destination:
        host: kiali.istio-system.svc.cluster.local
        port:
          number: 20001
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: jaeger-vs
  namespace: istio-system
spec:
  hosts:
  - "jaegerx.apsis.localnet"
  gateways:
  - observability-gateway
  http:
  - route:
    - destination:
        host: tracing.istio-system.svc.cluster.local
        port:
          number: 80
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: prometheus-vs
  namespace: istio-system
spec:
  hosts:
  - "prometheusx.apsis.localnet"
  gateways:
  - observability-gateway
  http:
  - route:
    - destination:
        host: prometheus.istio-system.svc.cluster.local
        port:
          number: 9090
```

Apply it:
```bash
kubectl apply -f observability-gateway.yaml
```

---

## 🔍 3) Verify Gateway and Routes

Confirm Gateway and VirtualServices:
```bash
kubectl get gateway,virtualservice -n istio-system
```

Ensure all components have `HOSTS` set correctly.

---

## 🌎 4) Access the Dashboards

Once your `/etc/hosts` entries are configured, open:

| Tool | URL | Default Port |
|------|-----|---------------|
| 📈 **Grafana** | [http://grafanax.apsis.localnet](http://grafanax.apsis.localnet) | 3000 |
| 🧭 **Kiali** | [http://kialix.apsis.localnet](http://kialix.apsis.localnet) | 20001 |
| 🕵️ **Jaeger** | [http://jaegerx.apsis.localnet](http://jaegerx.apsis.localnet) | 80 |
| 📊 **Prometheus** | [http://prometheusx.apsis.localnet](http://prometheusx.apsis.localnet) | 9090 |

---

## 🧠 5) (Optional) Secure with HTTPS / TLS

If desired, extend the `Gateway` above with a TLS server and provide certificates (e.g., via cert-manager or self-signed):

```yaml
servers:
- port:
    number: 443
    name: https
    protocol: HTTPS
  tls:
    mode: SIMPLE
    credentialName: apsis-cert
  hosts:
  - "*.apsis.localnet"
```

Then update your VirtualServices to use `443` instead of `80`.

---

## 🧼 6) Cleanup — Remove Observability Components

To remove all observability components and gateway configuration:

```bash
# Remove the custom ingress routes
kubectl delete -f observability-gateway.yaml -n istio-system

# Remove Istio addon deployments (Grafana, Kiali, Prometheus, Jaeger)
kubectl delete -f samples/addons -n istio-system

# Optionally remove any leftover services or pods
kubectl delete svc grafana kiali prometheus tracing -n istio-system --ignore-not-found
kubectl delete deploy grafana kiali prometheus tracing -n istio-system --ignore-not-found
```

If you also want to remove all Istio resources entirely (including control plane):

```bash
istioctl uninstall --purge -y
kubectl delete namespace istio-system
```

---

## ✅ Summary

You now have Istio observability dashboards exposed via friendly domain names:

- `grafanax.apsis.localnet`
- `kialix.apsis.localnet`
- `jaegerx.apsis.localnet`
- `prometheusx.apsis.localnet`

All routed securely through your Istio ingress gateway 🎯

# 🧪 RKE2 + Traefik Gateway API  
## Production-Clean, Repeatable Lab Guide

> **Goal**  
> Set up an RKE2 cluster **without default ingress**, deploy **Traefik using Gateway API**, and expose a sample app using **Gateway + HTTPRoute**.

---

## 🧱 Lab Assumptions

- OS: Linux (Ubuntu / Rocky / CentOS)
- Root or sudo access
- Single-node RKE2 (works the same for multi-node)
- No existing ingress controllers

---

## 🧹 Phase 0 — Clean State (VERY IMPORTANT)

```bash
systemctl stop rke2-server || true
rm -rf /etc/rancher/rke2
rm -rf /var/lib/rancher/rke2
rm -rf ~/.kube
```

Reboot recommended.

---

## 🚀 Phase 1 — Install RKE2 Server (Ingress Disabled)

```bash
curl -sfL https://get.rke2.io | sh -
```

```yaml
# /etc/rancher/rke2/config.yaml
ingress:
  disable:
    - rke2-ingress-nginx
```

```bash
systemctl enable rke2-server
systemctl start rke2-server
```

```bash
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml
export PATH=$PATH:/var/lib/rancher/rke2/bin
```

```bash
kubectl get nodes
kubectl get pods -A
```

---

## 🌐 Phase 2 — Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml
```

```bash
kubectl get crd | grep gateway
```

---

## 🚦 Phase 3 — Install Traefik (Gateway API Mode)

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
```

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

```bash
helm install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --set providers.kubernetesGateway.enabled=true \
  --set rbac.enabled=true
```

---

## 🧭 Phase 4 — Gateway API Objects

## GatewayClass
### gatewayclass.yaml
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: traefik
spec:
  controllerName: traefik.io/gateway-controller
```

## Gateway
### gateway.yaml
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: traefik-gateway
spec:
  gatewayClassName: traefik
  listeners:
    - name: web
      protocol: HTTP
      port: 80
```

---

## 📦 Phase 5 — Sample App (whoami)
### deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
```

### service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  selector:
    app: whoami
  ports:
    - port: 80
      targetPort: 80
```

---

## 🧩 Phase 6 — HTTPRoute
### httproute.yaml

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: whoami-route
spec:
  parentRefs:
    - name: traefik-gateway
  hostnames:
    - whoami.local
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: whoami
          port: 80
```

---

## 🧹 Cleanup

```bash
helm uninstall traefik -n traefik
kubectl delete gateway traefik-gateway
kubectl delete gatewayclass traefik
kubectl delete httproute whoami-route
kubectl delete deploy whoami
kubectl delete svc whoami
```

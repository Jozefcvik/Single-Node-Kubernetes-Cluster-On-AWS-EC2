
# 🚀 Helm

## ✅ Step 1: Install Helm on your Ubuntu VM

Run this on your **k8s-master node**:
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## ✅ Step 2: Verify installation
```bash
helm version
```
Expected output (example):
```bash
version.BuildInfo{Version:"v3.x.x", ...}
```

---

# 🚀 Ingress with NodePort

## ✅ Step 1: Add Helm repo
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

## ✅ Step 2: Install Ingress with NodePort
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --set controller.service.type=NodePort
```
## 🔍 Step 3: Check the NodePort
```bash
kubectl get svc -n default
```
(or if installed in its own namespace)
```bash
kubectl get svc -A | grep ingress
```
You’ll see something like:
```bash
ingress-nginx-controller   NodePort   10.x.x.x   <none>   80:30080/TCP,443:30443/TCP
```

👉 Important ports:
- 30080 → HTTP
- 30443 → HTTPS

## 🌐 Step 4: Access it

First get your node IP:
```bash
hostname -I
```
Then access:
```bash
http://<NODE-IP>:30080
```

## ✅ Step 5: Test with a sample app
**Create a simple nginx deployment**
```bash
kubectl create deployment demo --image=nginx
kubectl expose deployment demo --port=80
```
**Create an Ingress resource**
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo
            port:
              number: 80
```
Apply:
```bash
kubectl apply -f ingress.yaml
```

## 🧪 Step 6: Test DNS (important)

Edit your local '/etc/hosts':
```bash
<NODE-IP> demo.local
```
Then open:
```bash
http://demo.local:30080
```
### 🚀 Keycloak + PostgreSQL (StatefulSet) using Helm


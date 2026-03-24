
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

---

# 🚀 Keycloak + PostgreSQL (StatefulSet) using Helm

## 🚀 Step 1: Install PostgreSQL with Helm (StatefulSet)
### ✅ 1. PersistentVolume
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pg-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data/postgresql
```
### ✅ 2. StatefulSet
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: database
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres

  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: keycloak
        - name: POSTGRES_USER
          value: keycloak
        - name: POSTGRES_PASSWORD
          value: supersecretpassword
        volumeMounts:
        - name: pgdata
          mountPath: /var/lib/postgresql/data

  volumeClaimTemplates:
  - metadata:
      name: pgdata
    spec:
      storageClassName: ""
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
```
### ✅ 3. Headless Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: database
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```
### 🚀 Apply in correct order
```bash
kubectl apply -f pv.yaml
kubectl apply -f service.yaml
kubectl apply -f statefulset.yaml
```
### 🔍 Verify
```bash
kubectl get pv
kubectl get pvc -n database
kubectl get pods -n database
```
You want:
```bash
PVC → Bound
Pod → Running
```

## 🚀 Step 2: Install Keycloak with external PostgreSQL
### 1️⃣ Add Keycloak Helm repo
```bash
helm repo add codecentric https://codecentric.github.io/helm-charts
helm repo update
```
### 2️⃣ Create keycloak-values.yaml
```bash
replicaCount: 1   # single-node cluster (use 2+ in real prod)

postgresql:
  enabled: false

externalDatabase:
  host: postgres-postgresql.database.svc.cluster.local
  user: keycloak
  password: supersecretpassword
  database: keycloak
  port: 5432

proxyAddressForwarding: true

extraEnv: |
  - name: KC_PROXY
    value: edge
  - name: KC_HTTP_RELATIVE_PATH
    value: /auth

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: keycloak.local
      paths:
        - path: /auth
          pathType: Prefix
```
### 3️⃣ Install Keycloak
```bash
kubectl create namespace keycloak

helm install keycloak codecentric/keycloak \
  -n keycloak \
  -f keycloak-values.yaml
```
## 🌐 Step 3: Configure access
**Update** /etc/hosts
```bash
172.16.10.109 keycloak.local
```
**Access Keycloak**
```bash
http://keycloak.local/auth
```
## 🔍 Step 4: Verify everything
```bash
kubectl get pods -n keycloak
kubectl get ingress -n keycloak
kubectl describe ingress -n keycloak
```


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
### ✅ 2. Headless Service
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
### ✅ 3. StatefulSet
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
      volumeName: pg-pv
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
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
### ✅ 1. Deployment
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
        - name: keycloak
          image: quay.io/keycloak/keycloak:26.3.3
          args: ["start-dev"]
          ports:
            - containerPort: 8080
          env:
            - name: KC_DB
              value: postgres
            - name: KC_DB_URL_HOST
              value: postgres.database.svc.cluster.local
            - name: KC_DB_URL_PORT
              value: "5432"
            - name: KC_DB_URL_DATABASE
              value: keycloak
            - name: KC_DB_USERNAME
              value: keycloak
            - name: KC_DB_PASSWORD
              value: supersecretpassword
            - name: KC_BOOTSTRAP_ADMIN_USERNAME
              value: admin
            - name: KC_BOOTSTRAP_ADMIN_PASSWORD
              value: admin123
```
### ✅ 2. Service
```bash
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: keycloak
spec:
  selector:
    app: keycloak
  ports:
    - port: 80
      targetPort: 8080
```
### ✅ 3. Ingress
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: keycloak
  namespace: keycloak
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: keycloak.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: keycloak
                port:
                  number: 80
```
**🚀 Apply in correct order**
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```
**Verify everything is running**

Before logging in, make sure all components are healthy:
```bash
kubectl get pods -n keycloak
kubectl get pods -n database
```

You want:
- Keycloak pod → Running
- Postgres pod → Running

Check logs if needed:
```bash
kubectl logs -n keycloak deploy/keycloak
```
Look for:
```bash
Keycloak started in ...
```
### ✅ 4. Access Keycloak
Add to your '/etc/hosts':
```bash
127.0.0.1 keycloak.local
```
Then open:
```bash
http://keycloak.local:<ingress-http-port>
```
### ✅ 5. First Login (Admin Console)

**1. Open Admin Console**
Go to:
```bash
http://keycloak.local:<ingress-http-port>/admin
```
Login with:
```bash
Username: admin
Password: admin123
```
(from your env vars)

**2. Create a new permanent admin user**
Go to:
```bash
Users → Add user
```

Fill in:
- Username: admin2 (or anything you prefer)
- Click **Create**

**3. Set password**
- Go to **Credentials** tab
- Set password
- Turn OFF “Temporary”
- Save

**4. Assign admin role**

This is the important part 👇
- Go to Role Mappings
- Click Assign role
- Filter by: realm-management
- Add:
  - realm-admin
 
**5. Log out and log back in**

Logout, then login with your new user:
```bash
admin2 / your-password
```

**6. Delete temporary admin user**

Now remove the bootstrap user:
- Go to **Users**
- Find admin
- Delete it

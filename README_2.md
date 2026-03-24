
### Helm

### ✅ Step 1: Install Helm on your Ubuntu VM

Run this on your **k8s-master node**:
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### ✅ Step 2: Verify installation
```bash
helm version
```
Expected output (example):
```bash
version.BuildInfo{Version:"v3.x.x", ...}
```

##

### Keycloak + PostgreSQL (StatefulSet) using Helm


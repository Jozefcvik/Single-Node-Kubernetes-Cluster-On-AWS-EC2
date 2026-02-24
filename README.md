# Single Node Kubernetes on AWS EC2 (t3.small) 🚀

This guide explains step-by-step how to deploy a **single-node Kubernetes cluster** on:

- **AWS EC2 t3.small**
- **Ubuntu 24.04**
- **kubeadm + containerd**
- **Calico CNI**
- **NGINX Ingress Controller (NodePort)**
- **External NGINX Reverse Proxy (on the host)**
- **Let's Encrypt HTTPS**
- ❌ No AWS Load Balancer (to avoid extra cost)

This setup is ideal for:
- Personal projects
- Portfolio demos
- Dev / staging environments
- Learning Kubernetes

---

### 📌 Why Choose EC2 t3.small?

| Instance | vCPU | RAM | Why / Why Not |
|-----------|------|------|---------------|
| t3.micro | 2 | 1 GB | ❌ Too little memory for Kubernetes + Ingress |
| t3.small | 2 | 2 GB | ✅ Good balance for single-node K8s |
| t3.medium | 2 | 4 GB | ✅ Better performance but higher cost |

#### Why **t3.small** is a good choice:

- 2 GB RAM is enough for:
  - kube-apiserver
  - etcd
  - controller-manager
  - scheduler
  - Calico
  - Ingress Controller
  - Your workloads
- Burstable CPU performance
- Low cost (~$15/month region dependent)
- Good for small production workloads

---

### 🏗 Architecture Overview

<div align="center" style="background-color:#f6f8fa; padding:10px; border-radius:5px; display:inline-block; text-align:left;">
<pre>
Internet
|
DNS (Route53 or other)
|
EC2 Public IP
|
NGINX Reverse Proxy (Host Machine)
|
NodePort (NGINX Ingress Controller)
|
Ingress Rules
|
Kubernetes Services
|
Pods
</pre>
</div>

✔ No AWS Load Balancer  
✔ Lower cost  
✔ Full control  
✔ Let's Encrypt SSL  

---

### 1️⃣ Create AWS EC2 Instance

#### Instance Configuration

- AMI: **Ubuntu Server 24.04 LTS**
- Instance type: `t3.small`
- Storage: 20 GB (recommended)
- Security Group:
  - 22 (SSH)
  - 80 (HTTP)
  - 443 (HTTPS)
  - 30000-32767 (NodePort range) or specific chosen port

---

### 2️⃣ Install Single Node Kubernetes (kubeadm + containerd + Calico)

SSH into your EC2 instance.

#### Update system

```bash
sudo apt update && sudo apt upgrade -y
```

#### Disable swap (required)

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

#### Install containerd

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

#### Enable required kernel modules

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

#### Add Kubernetes repo

```bash
sudo apt install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Install:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### Initialize Cluster

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Configure kubectl:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### Allow scheduling on control plane

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

#### Install Calico CNI

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.2/manifests/calico.yaml
```
Wait until all pods are Running.

---

### 3️⃣ Install NGINX Ingress Controller (NodePort mode)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
```

Patch service to NodePort:

```bash
kubectl patch svc ingress-nginx-controller \
  -n ingress-nginx \
  -p '{"spec": {"type": "NodePort"}}'
```

Check assigned NodePort:

```bash
kubectl get svc -n ingress-nginx
```

Example output:

```code
80: 32080
443: 32443
```

---

### 4️⃣ Install NGINX Reverse Proxy on EC2 Host

Install nginx:

```bash
sudo apt install nginx -y
```

Create config:

```bash
sudo nano /etc/nginx/sites-available/k8s
```

Example config:

```bash
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:32080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/k8s /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

Now traffic flows:

<div align="center" style="background-color:#f6f8fa; padding:10px; border-radius:5px; display:inline-block; text-align:left;">
<pre>
Internet → Nginx → NodePort → Ingress → Service → Pod
</pre>
</div>

---

### 5️⃣ Enable Let's Encrypt SSL

Install Certbot:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Request certificate:

```bash
sudo certbot --nginx -d yourdomain.com
```

Certbot automatically:
- Adds SSL config
- Enables HTTPS
- Sets auto-renewal

Test renewal:

```bash
sudo certbot renew --dry-run
```

---

### ✅ Final Result

You now have:
- Single node Kubernetes cluster
- Calico networking
- NGINX Ingress via NodePort
- Reverse proxy on host
- Free Let's Encrypt HTTPS
- No AWS Load Balancer cost

---

### 💰 Cost Optimization

✔ No AWS ELB (~$20/month saved)
✔ Single t3.small instance
✔ Only pay for EC2 + storage

---

### ⚠️ Production Considerations

This setup is good for:
- Small production workloads
- Personal apps
- Demos
- Learning

For high availability production consider:
- Multi-node cluster
- External etcd
- AWS Load Balancer
- Managed Kubernetes (EKS)

---

### 📝 License

MIT License

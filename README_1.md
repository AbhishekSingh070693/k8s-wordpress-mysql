# Kubernetes WordPress Project

> **WordPress + MySQL deployed on Kubernetes (K8s v1.29) — 3 AWS EC2 t3.small instances | NFS shared storage | Kubernetes Dashboard | Calico CNI | containerd runtime | Ubuntu 24.04 LTS**

---

## 🌐 Live Endpoints

| Service | URL |
|---------|-----|
| WordPress Site | `http://3.95.224.129:30007` |
| Kubernetes Dashboard | `https://3.95.224.129:30001` |

---

## 🏗️ Infrastructure Overview

This project deploys a production-grade WordPress website backed by a MySQL database, running inside a Kubernetes cluster on AWS EC2. Three dedicated machines with clearly separated responsibilities follow real-world DevOps practices.

```
┌─────────────────────────────────────────────────────────────┐
│                    AWS VPC — us-east-1c                      │
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐    │
│  │ Master node  │   │ Worker node  │   │  NFS Server  │    │
│  │              │──▶│              │──▶│              │    │
│  │ Control plane│   │ MySQL + WP   │   │ /var/nfs/wp  │    │
│  │ 172.31.89.189│   │ 172.31.83.196│   │172.31.85.249 │    │
│  └──────────────┘   └──────────────┘   └──────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### The Three Machines

| Machine | Role | Public IP | Private IP | Key Software |
|---------|------|-----------|------------|--------------|
| Master node | Kubernetes control plane | 44.210.137.150 | 172.31.89.189 | kubeadm, kubectl, etcd, API server |
| Worker node | Runs all application pods | 3.95.224.129 | 172.31.83.196 | kubelet, containerd, nfs-common |
| NFS server | Dedicated persistent file storage | 44.202.125.220 | 172.31.85.249 | nfs-kernel-server |

---

## 📁 Project Structure

```
k8s-wordpress-mysql/
├── dashboard-admin.yaml          # Kubernetes Dashboard admin user + RBAC
├── mysql/
│   ├── mysql-secret.yaml         # MySQL passwords (base64 encoded)
│   ├── mysql-pv.yaml             # PersistentVolume (hostPath local disk)
│   ├── mysql-pvc.yaml            # PersistentVolumeClaim (5Gi RWO)
│   ├── mysql-deployment.yaml     # MySQL 5.7 Deployment
│   └── mysql-service.yaml        # Headless ClusterIP service (port 3306)
└── wordpress/
    ├── wp-configmap.yaml         # DB_HOST, DB_NAME, DB_USER config
    ├── wp-pv.yaml                # PersistentVolume (NFS backed)
    ├── wp-pvc.yaml               # PersistentVolumeClaim (5Gi RWX)
    ├── wp-deployment.yaml        # WordPress:latest Deployment
    └── wp-service.yaml           # NodePort service (port 30007)
```

---

## ⚙️ Tech Stack

| Component | Version/Detail |
|-----------|---------------|
| Kubernetes | v1.29.15 |
| Container Runtime | containerd v2.2.3 |
| CNI Plugin | Calico v3.25.0 |
| OS | Ubuntu 24.04 LTS |
| MySQL | 5.7 |
| WordPress | latest |
| Storage | NFS (ReadWriteMany) + hostPath (ReadWriteOnce) |
| Dashboard | v2.7.0 |

---

## 🚀 Deployment Guide

### Prerequisites

- 3 AWS EC2 t3.small instances (Master, Worker, NFS)
- All instances in the same VPC and Security Group
- Ubuntu 24.04 LTS on all instances

### Step 1 — Master Node Setup

```bash
# Update system
sudo apt-get update && sudo apt-get upgrade

# Disable swap (required by Kubernetes)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Enable kernel modules
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure network parameters
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

# Install containerd
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update && sudo apt install -y containerd.io
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd

# Install Kubernetes tools
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Initialize cluster
sudo kubeadm init
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Calico CNI
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

### Step 2 — Worker Node Setup

Run Steps 1–6 from Master Node Setup (same commands), then:

```bash
# Join the cluster (use token from master's kubeadm init output)
sudo kubeadm join 172.31.89.189:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>

# Regenerate join command on master if expired:
# kubeadm token create --print-join-command

# Install NFS client (CRITICAL — required for WordPress NFS volume)
sudo apt install -y nfs-common
```

### Step 3 — NFS Server Setup

```bash
# Install NFS server
sudo apt update
sudo apt install -y nfs-kernel-server nfs-common

# Create and configure export directory
sudo mkdir -p /var/nfs/wordpress
sudo chown -R nobody:nogroup /var/nfs/wordpress
sudo chmod 777 /var/nfs/wordpress

# Configure exports
sudo nano /etc/exports
# Add this line (no # at start):
# /var/nfs/wordpress *(rw,sync,no_subtree_check,no_root_squash)

# Apply and enable
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server

# Verify export is active
sudo exportfs -v
```

### Step 4 — Test NFS from Worker Node

```bash
# Run on worker node — must succeed before deploying YAML files
sudo mount -t nfs 172.31.85.249:/var/nfs/wordpress /mnt
ls /mnt       # should return empty with no error
sudo umount /mnt
```

### Step 5 — AWS Security Group Rules

Add these inbound rules to your Security Group (all 3 instances share the same group):

| Type | Protocol | Port | Source | Purpose |
|------|----------|------|--------|---------|
| Custom TCP | TCP | 6443 | 172.31.0.0/16 | Kubernetes API server |
| Custom TCP | TCP | 10250 | 172.31.0.0/16 | Kubelet API |
| Custom TCP | TCP | 179 | 172.31.0.0/16 | Calico BGP |
| Custom Protocol | 4 (IP-in-IP) | All | 172.31.0.0/16 | Calico pod traffic |
| NFS | TCP | 2049 | 172.31.0.0/16 | NFS data transfer |
| Custom UDP | UDP | 2049 | 172.31.0.0/16 | NFS UDP |
| Custom TCP | TCP | 111 | 172.31.0.0/16 | RPC portmapper |
| Custom TCP | TCP | 30007 | 0.0.0.0/0 | WordPress NodePort |
| Custom TCP | TCP | 30001 | 0.0.0.0/0 | Dashboard NodePort |
| HTTP | TCP | 80 | 0.0.0.0/0 | HTTP traffic |
| HTTPS | TCP | 443 | 0.0.0.0/0 | HTTPS traffic |
| SSH | TCP | 22 | 0.0.0.0/0 | SSH access |

### Step 6 — Deploy MySQL Stack

```bash
kubectl apply -f mysql/mysql-secret.yaml
kubectl apply -f mysql/mysql-pv.yaml
kubectl apply -f mysql/mysql-pvc.yaml
kubectl apply -f mysql/mysql-deployment.yaml
kubectl apply -f mysql/mysql-service.yaml

# Verify
kubectl get pods -l app=mysql
kubectl get pvc mysql-pvc
```

### Step 7 — Deploy WordPress Stack

```bash
kubectl apply -f wordpress/wp-configmap.yaml
kubectl apply -f wordpress/wp-pv.yaml
kubectl apply -f wordpress/wp-pvc.yaml
kubectl apply -f wordpress/wp-deployment.yaml
kubectl apply -f wordpress/wp-service.yaml

# Verify
kubectl get pods -l app=wordpress
kubectl get pvc wp-pvc
kubectl get svc wordpress-service
```

### Step 8 — Deploy Kubernetes Dashboard

```bash
# Deploy dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# Create admin user
kubectl apply -f dashboard-admin.yaml

# Expose via NodePort
kubectl patch svc kubernetes-dashboard -n kubernetes-dashboard \
  -p '{"spec":{"type":"NodePort","ports":[{"port":443,"targetPort":8443,"nodePort":30001}]}}'

# Generate login token
kubectl -n kubernetes-dashboard create token admin-user
```

Access at `https://3.95.224.129:30001` — select **Token** and paste the output above.

---

## 📦 YAML Files — What Each One Does

### MySQL Stack

| File | Kind | Purpose |
|------|------|---------|
| `mysql-secret.yaml` | Secret | Stores `MYSQL_ROOT_PASSWORD` and `MYSQL_PASSWORD` as base64 |
| `mysql-pv.yaml` | PersistentVolume | 5Gi hostPath storage at `/mnt/mysql-data` on worker node |
| `mysql-pvc.yaml` | PersistentVolumeClaim | Claims mysql-pv with ReadWriteOnce access |
| `mysql-deployment.yaml` | Deployment | Runs mysql:5.7, mounts PVC at `/var/lib/mysql` |
| `mysql-service.yaml` | Service | Headless ClusterIP (None) on port 3306 — DNS name `mysql-service` |

### WordPress Stack

| File | Kind | Purpose |
|------|------|---------|
| `wp-configmap.yaml` | ConfigMap | Stores `WORDPRESS_DB_HOST`, `DB_NAME`, `DB_USER` |
| `wp-pv.yaml` | PersistentVolume | 5Gi NFS storage pointing to `172.31.85.249:/var/nfs/wordpress` |
| `wp-pvc.yaml` | PersistentVolumeClaim | Claims wp-pv with ReadWriteMany access |
| `wp-deployment.yaml` | Deployment | Runs wordpress:latest, mounts PVC at `/var/www/html` |
| `wp-service.yaml` | Service | NodePort on 30007 — exposes WordPress publicly |

---

## 🔗 YAML File Connections

```
mysql-secret.yaml
  └── MYSQL_ROOT_PASSWORD ──────────────▶ mysql-deployment.yaml (secretKeyRef)
  └── MYSQL_PASSWORD ───────────────────▶ wp-deployment.yaml (secretKeyRef)

mysql-pv.yaml (storage:5Gi + RWO)
  └── binds to ────────────────────────▶ mysql-pvc.yaml (storage:5Gi + RWO)
      └── claimName: mysql-pvc ────────▶ mysql-deployment.yaml

mysql-deployment.yaml
  └── labels: app:mysql ───────────────▶ mysql-service.yaml (selector)
  └── MYSQL_DATABASE: wordpress ───────▶ wp-configmap.yaml (DB_NAME)

mysql-service.yaml
  └── name: mysql-service ─────────────▶ wp-configmap.yaml (WORDPRESS_DB_HOST)

wp-pv.yaml (server:172.31.85.249 + RWX)
  └── binds to ────────────────────────▶ wp-pvc.yaml (storage:5Gi + RWX)
      └── claimName: wp-pvc ───────────▶ wp-deployment.yaml

wp-configmap.yaml
  └── name: wordpress-config ──────────▶ wp-deployment.yaml (envFrom configMapRef)

wp-deployment.yaml
  └── labels: app:wordpress ───────────▶ wp-service.yaml (selector)
```

---

## 🌊 Traffic Flow

When a user visits `http://3.95.224.129:30007`:

```
Browser
  └──▶ AWS Security Group (port 30007 allowed)
        └──▶ kube-proxy intercepts :30007
              └──▶ wordpress-service (NodePort)
                    └──▶ WordPress pod :80 (Apache)
                          ├──▶ mysql-service:3306 (CoreDNS resolves)
                          │     └──▶ MySQL pod (queries wordpress database)
                          └──▶ /var/www/html (NFS mount)
                                └──▶ NFS server 172.31.85.249:/var/nfs/wordpress
```

---

## 🔑 Credentials

> ⚠️ **For production use, replace all default credentials and use external secret management (AWS Secrets Manager / HashiCorp Vault)**

| Credential | Value | Used By |
|-----------|-------|---------|
| MySQL root password | `mypassword` | MySQL container |
| WordPress DB user | `wordpress` | WordPress → MySQL connection |
| WordPress DB password | `wordpresspass` | WordPress → MySQL connection |
| WordPress DB name | `wordpress` | Both MySQL and WordPress |

---

## 🖥️ Kubernetes Dashboard

- **URL:** `https://3.95.224.129:30001`
- **Login:** Token (generate with `kubectl -n kubernetes-dashboard create token admin-user`)
- **Browser warning:** Expected — self-signed certificate, click Advanced → Proceed
- **What you can see:** All pods, deployments, services, PVCs, secrets, logs across all namespaces

---

## 🔍 Useful Commands

```bash
# Check all pods
kubectl get pods -A

# Check all resources in default namespace
kubectl get all

# Check pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# Describe pod (events + details)
kubectl describe pod <pod-name>

# Check Calico network
kubectl get pods -n kube-system | grep calico

# Test DNS inside cluster
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup mysql-service

# Scale WordPress
kubectl scale deployment wordpress --replicas=3

# Restart Calico
kubectl rollout restart daemonset/calico-node -n kube-system

# Get dashboard token
kubectl -n kubernetes-dashboard create token admin-user
```

---

## 📋 Deployment Order (Important)

Apply files in this exact order — each file depends on the previous:

| Order | File | Why |
|-------|------|-----|
| 1 | mysql-secret.yaml | Credentials must exist before deployment references them |
| 2 | mysql-pv.yaml | Storage must exist before it can be claimed |
| 3 | mysql-pvc.yaml | Claim must exist before deployment mounts it |
| 4 | mysql-deployment.yaml | MySQL must run before WordPress connects |
| 5 | mysql-service.yaml | DNS name must exist before WordPress uses it |
| 6 | wp-configmap.yaml | Config must exist before deployment loads it |
| 7 | wp-pv.yaml | NFS storage must exist before it can be claimed |
| 8 | wp-pvc.yaml | Claim must exist before deployment mounts it |
| 9 | wp-deployment.yaml | Deploy WordPress after all dependencies ready |
| 10 | wp-service.yaml | Expose WordPress publicly last |

---

## 📸 Project Screenshots

### 3 EC2 Instances Running
All 3 instances (Master-node, Worker-node, NFS-server) running on AWS EC2 t3.small in us-east-1c.

### Both Nodes Ready
```
NAME               STATUS   ROLES           AGE    VERSION
ip-172-31-83-196   Ready    <none>          125m   v1.29.15
ip-172-31-89-189   Ready    control-plane   128m   v1.29.15
```

### WordPress Installation Screen
WordPress accessible at `http://3.95.224.129:30007`

### Kubernetes Dashboard
All workloads green — 2 Deployments, 2 Pods, 2 ReplicaSets all Running.

### MySQL Database Verified
```sql
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| wordpress          |
+--------------------+
```

---

## 🏛️ Architecture Decisions

| Decision | Reasoning |
|----------|-----------|
| MySQL uses hostPath, not NFS | Databases need fast local disk — NFS latency degrades query performance |
| WordPress uses NFS, not hostPath | Files need ReadWriteMany for scaling — NFS supports multiple pods mounting simultaneously |
| MySQL service is headless (clusterIP:None) | Direct pod connection = lower latency. No load balancer needed for single-instance DB |
| WordPress service is NodePort | Must be accessible from public internet. NodePort opens port on worker node's public IP |
| 3 separate EC2 instances | Separation of concerns — compute, orchestration, and storage on dedicated machines |
| Calico for CNI | BGP routing, network policies, production-grade — industry standard |

---

## ⚠️ Production Considerations

- Replace `base64` encoded secrets with AWS Secrets Manager or HashiCorp Vault
- Use a LoadBalancer service type or Ingress controller instead of NodePort
- Add TLS/SSL certificate for HTTPS on WordPress (Let's Encrypt + cert-manager)
- Set up etcd backups for disaster recovery
- Use MySQL with replication for high availability
- Set resource limits and requests on all containers
- Enable Kubernetes network policies to restrict pod-to-pod traffic
- Restrict NFS server access to worker node IP only (not `*` in exports)

---

*WordPress on Kubernetes | AWS EC2 | NFS Storage | Kubernetes Dashboard | Calico CNI*

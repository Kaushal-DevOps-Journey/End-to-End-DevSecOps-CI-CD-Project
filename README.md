# End-to-End-DevSecOps-CI-CD-Project

# 🚀 Kubernetes Cluster Setup with kubeadm (v1.30 + Docker)

This guide explains how to set up a Kubernetes cluster using **kubeadm**, with **Docker** as the container runtime (via `cri-dockerd`), and deploy **Calico CNI** for networking + **NGINX Ingress Controller**.

---

## 📌 Prerequisites

- **Ubuntu 20.04 / 22.04 LTS** (Master + Worker nodes)
- **At least 2 vCPUs & 2GB RAM** (recommended for master node)
- **Passwordless sudo** enabled
- Internet access from all nodes

---

## ⚡ Steps

### 1. Disable Swap (on all nodes)
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

### 2. Update System Packages (on all nodes)
```bash
sudo apt-get update && sudo apt-get upgrade -y
```

---

### 3. Install Docker (on all nodes)
```bash
sudo apt install -y docker.io
sudo systemctl enable docker --now
sudo chmod 666 /var/run/docker.sock
```

---

### 4. Install cri-dockerd (on all nodes)

```bash
# Download latest release
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.15/cri-dockerd-0.3.15.amd64.tgz

# Extract and move binary
tar xvf cri-dockerd-0.3.15.amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/bin/
rm -rf cri-dockerd cri-dockerd-0.3.15.amd64.tgz

# Create systemd service unit
cat <<EOF | sudo tee /etc/systemd/system/cri-docker.service
[Unit]
Description=CRI Interface for Docker
Documentation=https://docs.mirantis.com
After=network.target docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint=unix:///var/run/cri-dockerd.sock
Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF

# Create systemd socket unit
cat <<EOF | sudo tee /etc/systemd/system/cri-docker.socket
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=/var/run/cri-dockerd.sock
SocketMode=0660

[Install]
WantedBy=sockets.target
EOF

# Reload systemd and start cri-dockerd
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service cri-docker.socket
sudo systemctl start cri-docker.service cri-docker.socket
```

Verify:
```bash
systemctl status cri-docker.service
```

---

### 5. Install Kubernetes Components (on all nodes)
```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.30.* kubeadm=1.30.* kubectl=1.30.*
sudo apt-mark hold kubelet kubeadm kubectl
```

---

### 6. Initialize Master Node
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16   --cri-socket=unix:///var/run/cri-dockerd.sock
```

Copy the `kubeadm join` command printed — you’ll need it for workers.

---

### 7. Configure kubectl (on master only)
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

### 8. Install Calico CNI (on master only)
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

### 9. Install NGINX Ingress Controller (on master only)
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.6/deploy/static/provider/baremetal/deploy.yaml
```

---

### 10. Join Worker Nodes
Run the `kubeadm join` command printed in Step 6 on each worker node. Example:
```bash
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN>   --discovery-token-ca-cert-hash sha256:<HASH>   --cri-socket=unix:///var/run/cri-dockerd.sock
```

---

## ✅ Verification

1. Check nodes:
```bash
kubectl get nodes
```

Expected:
```
NAME               STATUS   ROLES           AGE   VERSION
master-node        Ready    control-plane   5m    v1.30.0
worker-node1       Ready    <none>          3m    v1.30.0
worker-node2       Ready    <none>          2m    v1.30.0
```

---

## 🎯 Conclusion
You now have a **Kubernetes 1.30 cluster** running with **Docker (via cri-dockerd)**, **Calico CNI**, and **NGINX Ingress** — ready to deploy workloads.

# End-to-End DevSecOps CI/CD Project 🚀

This project demonstrates an **End-to-End DevSecOps CI/CD pipeline** using Jenkins, SonarQube, Docker, Kubernetes, Nexus, Prometheus, Grafana, and Trivy.  
It covers the entire software delivery lifecycle — from code build, testing, security scanning, artifact management, to deployment and monitoring.  

---

## 🔥 Features

- Jenkins Declarative Pipeline  
- Maven Build & Test Execution  
- SonarQube Code Quality Analysis  
- Trivy Vulnerability Scanning  
- Docker Image Build & Push to DockerHub  
- Nexus Artifact Repository Integration  
- Kubernetes Deployment & Service Exposure  
- Prometheus & Grafana Monitoring  

---


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
## ✅ Kubernetes cluster security scan (Run On Master)
https://github.com/Shopify/kubeaudit/releases  --> Select Kubeaudit link according to architecture
wget <link>
tar -xvzf kubeaudit_0.22.2_linux_amd64.tar.gz   ---> extract zipped file
sudo mv kubeaudit /usr/local/bin
kubeaudit all 
 


## 🎯 Conclusion
You now have a **Kubernetes 1.30 cluster** running with **Docker (via cri-dockerd)**, **Calico CNI**, and **NGINX Ingress** — ready to deploy workloads.


## ⚡ SetUp SonarQube

```bash
#!/bin/bash

# Update package manager repositories
sudo apt-get update

# Install necessary dependencies
sudo apt-get install -y ca-certificates curl

# Create directory for Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker's GPG key
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

# Ensure proper permissions for the key
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository to Apt sources
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package manager repositories
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

Save this script in a file, for example, `install_docker.sh`, and make it executable using:

```bash
chmod +x install_docker.sh
```

Then, you can run the script using:

```bash
./install_docker.sh
```

```bash
sudo chmod 666 /var/run/docker.sock
```
## ⚡ Create Sonarqube Docker container
To run SonarQube in a Docker container with the provided command, you can follow these steps:

1. Open your terminal or command prompt.

2. Run the following command:

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

This command will download the `sonarqube:lts-community` Docker image from Docker Hub if it's not already available locally. Then, it will create a container named "sonar" from this image, running it in detached mode (`-d` flag) and mapping port 9000 on the host machine to port 9000 in the container (`-p 9000:9000` flag).

3. Access SonarQube by opening a web browser and navigating to `http://<VM-IP>:9000`.

This will start the SonarQube server, and you should be able to access it using the provided URL. If you're running Docker on a remote server or a different port, replace `localhost` with the appropriate hostname or IP address and adjust the port accordingly.

## ⚡ SetUp Nexus

```bash
#!/bin/bash

# Update package manager repositories
sudo apt-get update

# Install necessary dependencies
sudo apt-get install -y ca-certificates curl

# Create directory for Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker's GPG key
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

# Ensure proper permissions for the key
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository to Apt sources
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package manager repositories
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin 
```

Save this script in a file, for example, `install_docker.sh`, and make it executable using:

```bash
chmod +x install_docker.sh
```

Then, you can run the script using:

```bash
./install_docker.sh
```

## ⚡ Create Nexus using docker container

To create a Docker container running Nexus 3 and exposing it on port 8081, you can use the following command:

```bash
docker run -d --name nexus -p 8081:8081 sonatype/nexus3:latest
```

This command does the following:

- `-d`: Detaches the container and runs it in the background.
- `--name nexus`: Specifies the name of the container as "nexus".
- `-p 8081:8081`: Maps port 8081 on the host to port 8081 on the container, allowing access to Nexus through port 8081.
- `sonatype/nexus3:latest`: Specifies the Docker image to use for the container, in this case, the latest version of Nexus 3 from the Sonatype repository.

After running this command, Nexus will be accessible on your host machine at http://IP:8081.

## ⚡ Get Nexus initial password
Your provided commands are correct for accessing the Nexus password stored in the container. Here's a breakdown of the steps:

1. **Get Container ID**: You need to find out the ID of the Nexus container. You can do this by running:

    ```bash
    docker ps
    ```

    This command lists all running containers along with their IDs, among other information.

2. **Access Container's Bash Shell**: Once you have the container ID, you can execute the `docker exec` command to access the container's bash shell:

    ```bash
    docker exec -it <container_ID> /bin/bash
    ```

    Replace `<container_ID>` with the actual ID of the Nexus container.

3. **Navigate to Nexus Directory**: Inside the container's bash shell, navigate to the directory where Nexus stores its configuration:

    ```bash
    cd sonatype-work/nexus3
    ```

4. **View Admin Password**: Finally, you can view the admin password by displaying the contents of the `admin.password` file:

    ```bash
    cat admin.password
    ```

5. **Exit the Container Shell**: Once you have retrieved the password, you can exit the container's bash shell:

    ```bash
    exit
    ```

This process allows you to access the Nexus admin password stored within the container. Make sure to keep this password secure, as it grants administrative access to your Nexus instance.


## 📌 Jenkins Plugins Used & Their Usage

| Plugin                              | Purpose                                                             |
| ----------------------------------- | ------------------------------------------------------------------- |
| **Eclipse Temurin Installer**       | Installs Java (Temurin JDK) required for Maven builds               |
| **Pipeline Maven Integration**      | Provides Maven build steps and caching inside pipelines             |
| **Config File Provider**            | Supplies Maven `settings.xml` and other config files to jobs        |
| **SonarQube Scanner**               | Runs static code analysis & integrates with SonarQube server        |
| **Kubernetes CLI**                  | Executes `kubectl` commands from Jenkins pipelines                  |
| **Kubernetes**                      | Allows Jenkins agents/pods to be provisioned dynamically inside K8s |
| **Docker**                          | Provides Docker client support inside Jenkins                       |
| **Docker Pipeline**                 | Enables `docker.build` and `docker.push` steps in pipelines         |
| **Stage View**                      | Provides a visual UI for Jenkins pipeline stages                    |
| **Kubernetes Client API**           | Required by Kubernetes-related plugins to interact with cluster     |
| **Kubernetes Credentials Provider** | Securely stores kubeconfig and cluster access credentials           |
| **Prometheus Metrics**              | Exposes Jenkins metrics to Prometheus for monitoring                |


---


## ⚙️ Jenkins Setup

### 1️⃣ Install Jenkins
```bash
# Update system
sudo apt update -y

# Install Java (Temurin JDK)
sudo apt install openjdk-17-jdk -y

# Add Jenkins repo key
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Add Jenkins apt repository
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update -y
sudo apt install jenkins -y

# Start and enable service
sudo systemctl enable jenkins
sudo systemctl start jenkins
2️⃣ Unlock Jenkins
bash
Copy code
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Open in browser: http://<server-public-ip>:8080

Paste the above password into Jenkins setup wizard

Install Suggested Plugins

3️⃣ Install Required Plugins
Go to Manage Jenkins → Plugins → Available Plugins and install:
✅ Eclipse Temurin Installer
✅ Pipeline Maven Integration
✅ Config File Provider
✅ SonarQube Scanner
✅ Kubernetes CLI
✅ Kubernetes
✅ Docker
✅ Docker Pipeline
✅ Stage View
✅ Kubernetes Client API
✅ Kubernetes Credentials Provider
✅ Prometheus Metrics

4️⃣ Configure Tools
Manage Jenkins → Tools

Add JDK via Temurin Installer

Add Maven

Add SonarQube Scanner

Verify Docker is installed

5️⃣ Add Credentials
Go to Manage Jenkins → Credentials and add:

GitHub → Personal Access Token

DockerHub → Username & Password

Kubernetes → Kubeconfig file

SonarQube → Authentication Token

6️⃣ Configure SonarQube in Jenkins
Go to Manage Jenkins → System → SonarQube Servers

Add:

Name: SonarQube

Server URL: http://<sonarqube-ip>:9000

Token: Generated from SonarQube

7️⃣ Enable Prometheus Metrics
Go to Manage Jenkins → Configure System

Enable Prometheus Plugin

Metrics exposed at:
http://<jenkins-ip>:8080/prometheus
8️⃣ Test Jenkins Setup
Create a Pipeline Job

Connect to GitHub repo

Add Jenkinsfile

Run Pipeline 🚀

---
### 📊 Monitoring
Prometheus scrapes Jenkins, Kubernetes, and Node Exporter metrics

Grafana visualizes dashboards for CI/CD and cluster workloads

---

### 📸 Live Screenshots
https://www.linkedin.com/feed/update/urn:li:activity:7370916028333113345/

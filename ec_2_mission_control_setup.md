# ☁️ Prepare an EC2 Instance for a Mission-Control Install Based on Kind

## 📋 Prerequisites

Provision an **Ubuntu 24.04 LTS EC2 instance** with:

- **Instance Type**: `t3.2xlarge` (minimum 8 vCPUs, 32 GB RAM recommended)
- **Disk**: 100 GB root volume (no secondary volume required)
- **Networking**: Only **SSH port (22)** needs to be open for access
- **Local Requirements**:
  - Your private SSH key (e.g., `dieter-key.pem`)
  - A terminal with `ssh` and `curl` installed

Connect to the instance:

```bash
ssh -i "dieter-key.pem" ubuntu@ec2-3-122-56-88.eu-central-1.compute.amazonaws.com
```

---

## 🐳 Step 1: Install Docker

Install Docker from the official Docker repository:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Optional: run Docker without sudo
sudo usermod -aG docker $USER
newgrp docker
```

---

## 📦 Step 2: Install Kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

---

## 📦 Step 3: Install `kubectl`

```bash
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

---

## 📦 Step 4: Install Helm

```bash
curl https://baltocdn.com/helm/signing.asc | sudo gpg --dearmor -o /usr/share/keyrings/helm.gpg

echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | \
  sudo tee /etc/apt/sources.list.d/helm-stable-debian.list > /dev/null

sudo apt update
sudo apt install -y helm
```

---

## 📦 Step 5: Install the KOTS Plugin for `kubectl`

```bash
curl https://kots.io/install | sudo bash
```

---

## ✅ Step 6: Verify Installed Tools

```bash
docker --version
kind --version
kubectl version --client
helm version
kubectl kots version
```

---

## 🧱 Step 7: Create and Launch the Kind Cluster

Create a working directory and config file:

```bash
mkdir mc
cd mc

nano kind.yaml
```

Paste the following into `kind.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: 0.0.0.0
  apiServerPort: 45451
nodes:
- role: control-plane
  image: kindest/node:v1.32.2
  labels:
    mission-control.datastax.com/role: platform
  extraPortMappings:
    - containerPort: 30880
      hostPort: 30880
      listenAddress: "0.0.0.0"
      protocol: tcp
    - containerPort: 30081
      hostPort: 30081
      listenAddress: "0.0.0.0"
      protocol: tcp
    - containerPort: 30001
      hostPort: 30001
      listenAddress: "0.0.0.0"
      protocol: tcp
- role: worker
  image: kindest/node:v1.32.2
  labels:
    mission-control.datastax.com/role: platform
- role: worker
  image: kindest/node:v1.32.2
  labels:
    mission-control.datastax.com/role: platform
- role: worker
  image: kindest/node:v1.32.2
  labels:
    mission-control.datastax.com/role: platform
```

Create the Kind cluster:

```bash
kind create cluster --config kind.yaml
```

Verify the cluster:

```bash
kubectl get nodes
```

---

## 🔐 Step 8: Create SSH Tunnel (From Local Machine)

Run this on your **local machine** to forward UI ports from the EC2 instance:

```bash
ssh -i dieter-key.pem \
  -L 8800:localhost:8800 \
  -L 30880:localhost:30880 \
  ubuntu@3.122.56.88
```

- Access **KOTS Admin Console** at: [http://localhost:8800](http://localhost:8800) later
- Access **Mission Control UI** at: [http://localhost:30880](http://localhost:30880) later

---

## 🚀 Step 9: Continue With Mission Control Setup

Proceed with the next steps here:\
👉 [Mission Control Local Setup – Install Cert-Manager](https://github.com/difli/mission-control-local-setup#2-install-cert-manager)

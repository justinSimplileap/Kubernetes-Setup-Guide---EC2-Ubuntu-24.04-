# Kubernetes Setup Guide - EC2 (Ubuntu 24.04)

Complete guide for setting up a Kubernetes cluster from scratch on AWS EC2 using containerd, nerdctl, and buildkit (without Docker).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation Steps](#installation-steps)
- [CI/CD Setup](#cicd-setup)
- [Troubleshooting](#troubleshooting)
- [Verification](#verification)

## Prerequisites

- AWS EC2 instance running Ubuntu 24.04
- Root or sudo access
- Security group allowing necessary ports:
  - 6443 (Kubernetes API)
  - 2379-2380 (etcd)
  - 10250 (Kubelet API)
  - 10251 (kube-scheduler)
  - 10252 (kube-controller-manager)
  - 30000-32767 (NodePort Services)

## Installation Steps

### 1. System Preparation

Update the system and install required packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget apt-transport-https ca-certificates gnupg lsb-release
```

### 2. Install containerd

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

### ⚠️ CRITICAL: Configure SystemdCgroup

The default configuration sets `SystemdCgroup = false`, which will cause Kubernetes to fail. You **must** change this to `true`:

```bash
# Update SystemdCgroup to true
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Verify the change
sudo cat /etc/containerd/config.toml | grep SystemdCgroup
# Output should show: SystemdCgroup = true
```

Restart and enable containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 3. Install NerdCTL (Docker replacement)

```bash
# Download nerdctl
curl -LO https://github.com/containerd/nerdctl/releases/download/v1.7.5/nerdctl-1.7.5-linux-amd64.tar.gz

# Extract and move to bin
tar -xvf nerdctl-1.7.5-linux-amd64.tar.gz
sudo mv nerdctl /usr/local/bin/
```

### 4. Install BuildKit (for nerdctl build)

```bash
curl -LO https://github.com/moby/buildkit/releases/download/v0.12.5/buildkit-v0.12.5.linux-amd64.tar.gz

# Extract and move binaries
sudo tar -C /usr/local/bin -xzf buildkit-v0.12.5.linux-amd64.tar.gz --strip-components=1 buildkit-v0.12.5.linux-amd64/bin/buildctl buildkit-v0.12.5.linux-amd64/bin/buildkitd

# Start BuildKit
sudo nohup buildkitd > /dev/null 2>&1 &
```

### 5. Add Kubernetes Repository & Install Tools

> **Note:** The old Google-hosted repository (`packages.cloud.google.com`) has been deprecated. Use the new official repository.

```bash
sudo mkdir -p /etc/apt/keyrings

# Download the GPG key for Kubernetes v1.31
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the Kubernetes repository
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update and install Kubernetes tools
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**For different Kubernetes versions:** Replace `v1.31` with your desired version (e.g., `v1.30`, `v1.29`, `v1.32`).

### 6. Enable IP Forwarding

Kubernetes requires IP forwarding for pod networking:

```bash
# Enable IP forwarding immediately
sudo sysctl -w net.ipv4.ip_forward=1

# Make it persistent across reboots
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Verify it's set to 1
cat /proc/sys/net/ipv4/ip_forward
```

### 7. Disable Swap (Required for Kubernetes)

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### 8. Initialize Kubernetes Cluster (Master Node)

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

> **Important:** Save the `kubeadm join` command from the output - you'll need it to add worker nodes later.

Example output:
```
kubeadm join 172.31.13.236:6443 --token xxxxx.xxxxxxxxxx \
    --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxx
```

### 9. Set Up kubectl Access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 10. Install Calico (Pod Network Add-On)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Wait 30-60 seconds for Calico to initialize.

### 11. Verify Cluster

```bash
kubectl get nodes
kubectl get pods -A
```

Expected output:
```
NAME               STATUS   ROLES           AGE   VERSION
ip-172-31-13-236   Ready    control-plane   5m    v1.31.13
```

All pods in `kube-system` namespace should show `Running` status.

## CI/CD Setup

### GitHub Actions Configuration

#### Required Secrets

Add these secrets to your GitHub repository (Settings → Secrets and variables → Actions):

| Secret Name | Description | Example |
|------------|-------------|---------|
| `EC2_HOST` | EC2 Public IP | `54.123.45.67` |
| `EC2_USER` | SSH Username | `ubuntu` |
| `EC2_SSH_KEY` | Private SSH Key | `-----BEGIN RSA PRIVATE KEY-----...` |
| `GH_PAT` | GitHub Personal Access Token | `ghp_xxxxxxxxxxxxx` |

#### Workflow File

Create `.github/workflows/deploy-k8s.yml`:

```yaml
name: Deploy to Kubernetes

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
      
      - name: Set up SSH
        uses: webfactory/ssh-agent@v0.7.0
        with:
          ssh-private-key: ${{ secrets.EC2_SSH_KEY }}
      
      - name: Deploy to Kubernetes EC2
        run: |
          ssh -o StrictHostKeyChecking=no ${{ secrets.EC2_USER }}@${{ secrets.EC2_HOST }} << 'EOF'
            set -e
            
            # Navigate to project directory
            cd /home/ubuntu/your-project-path
            
            # Pull latest changes
            echo "Pulling latest changes"
            if [ -d ".git" ]; then
              git fetch origin main
              git reset --hard origin/main
            else
              rm -rf ./*
              git clone https://${{ secrets.GH_PAT }}@github.com/your-org/your-repo.git .
            fi
            
            # Build container image
            echo "Building container image"
            sudo nerdctl build -t your-app:latest .
            
            # Deploy to Kubernetes
            echo "Applying Kubernetes manifests"
            sudo kubectl apply -f k8s/deployment.yaml
            sudo kubectl apply -f k8s/service.yaml
            
            # Verify deployment
            sudo kubectl rollout status deployment/your-app
          EOF
```

## Manual Deployment

### Build and Deploy Application

```bash
# Navigate to your project
cd /path/to/your/project

# Build container image with nerdctl
sudo nerdctl build -t your-app:latest .

# Apply Kubernetes manifests
sudo kubectl apply -f k8s/deployment.yaml
sudo kubectl apply -f k8s/service.yaml

# Check deployment status
kubectl get deployments
kubectl get pods
kubectl get services
```

### Access Your Application

```bash
# Get service details
kubectl get svc

# Access via NodePort
curl http://<EC2-PUBLIC-IP>:<NodePort>
```

## Troubleshooting

### Issue: kubeadm init fails with IP forwarding error

**Error:**
```
[ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
```

**Solution:**
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

### Issue: etcd or API server connection refused

**Error:**
```
dial tcp 127.0.0.1:2379: connect: connection refused
```

**Root Cause:** `SystemdCgroup = false` in containerd configuration

**Solution:**
```bash
# Fix containerd configuration
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# Reset and reinitialize cluster
sudo kubeadm reset -f
sudo kubeadm init --pod-network-cidr=192.168.0.0/16

# Reconfigure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Issue: Node shows "NotReady" status

**Solution:** Install the Calico network plugin:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

Wait 30-60 seconds and check again:
```bash
kubectl get nodes
```

### Issue: Old Kubernetes repository errors (404 Not Found)

**Error:**
```
E: The repository 'https://apt.kubernetes.io kubernetes-xenial Release' does not have a Release file.
```

**Solution:** The Google-hosted repository is deprecated. Remove it and use the new repository:
```bash
sudo rm /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
```

### Issue: Pods stuck in "Pending" or "ContainerCreating"

**Check pod details:**
```bash
kubectl describe pod <pod-name> -n <namespace>
```

**Common causes:**
- Network plugin not installed (install Calico)
- Insufficient resources
- Image pull errors

### Check Logs

```bash
# Kubelet logs
sudo journalctl -u kubelet -n 50

# Container logs
sudo crictl logs <container-id>

# Pod logs
kubectl logs <pod-name> -n <namespace>
```

## Verification

### Verify All Components

```bash
# Check node status
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Check all resources
kubectl get all -A

# Check component health
kubectl get cs
```

### Expected Output

All system pods should be in `Running` state:
- calico-kube-controllers
- calico-node
- coredns (2 replicas)
- etcd
- kube-apiserver
- kube-controller-manager
- kube-proxy
- kube-scheduler

## Useful Commands

```bash
# View cluster info
kubectl cluster-info

# Get all resources
kubectl get all -A

# Describe a resource
kubectl describe <resource-type> <resource-name>

# View logs
kubectl logs <pod-name>

# Execute command in pod
kubectl exec -it <pod-name> -- /bin/bash

# Delete a resource
kubectl delete <resource-type> <resource-name>

# Apply configuration
kubectl apply -f <file.yaml>

# Get service endpoints
kubectl get endpoints

# Scale deployment
kubectl scale deployment <deployment-name> --replicas=3
```

## Architecture

This setup uses:
- **containerd** - Container runtime (replaces Docker)
- **nerdctl** - Docker-compatible CLI for containerd
- **buildkit** - Build toolkit for creating container images
- **kubeadm** - Kubernetes cluster bootstrapping tool
- **kubectl** - Kubernetes command-line tool
- **Calico** - Pod networking and network policy

## Security Considerations

1. **Secure SSH access** - Use key-based authentication only
2. **Firewall rules** - Restrict access to necessary ports only
3. **Update regularly** - Keep system and Kubernetes components updated
4. **RBAC** - Implement Role-Based Access Control
5. **Network policies** - Use Calico network policies to restrict pod communication
6. **Secrets management** - Use Kubernetes secrets for sensitive data

## Additional Resources

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [containerd Documentation](https://containerd.io/)
- [Calico Documentation](https://docs.projectcalico.org/)
- [nerdctl GitHub](https://github.com/containerd/nerdctl)
- [BuildKit GitHub](https://github.com/moby/buildkit)

## License

This guide is provided as-is for educational and production use.

## Contributing

Feel free to submit issues or pull requests to improve this guide.

---

**Last Updated:** November 2025  
**Kubernetes Version:** v1.31.13  
**Ubuntu Version:** 24.04 LTS

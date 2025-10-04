# Environment Setup

This section helps you verify your local setup and start a simple Kubernetes cluster using **kind** (Kubernetes in Docker).

## 1. Verify Installed Versions

Run the following commands to check if the required tools are installed:

```bash
# Check kubectl version
kubectl version --client

# Check kind version
kind version

# Check Docker version
docker --version

# On Linux / WSL
systemctl status docker

# Create cluster using Kind
kind create cluster --name cka-cluster

# Verify cluster
# List clusters managed by kind
kind get clusters

# Get nodes in the cluster
kubectl get nodes

# Check cluster info
kubectl cluster-info

# Delete Cluster
kind delete cluster --name cka-cluster

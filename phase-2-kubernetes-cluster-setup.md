# Phase 2: Kubernetes Cluster Setup

## Learning Objectives
- Install container runtime (containerd)
- Install Kubernetes components (kubeadm, kubelet, kubectl)
- Initialize Kubernetes control plane
- Install Container Network Interface (CNI) plugin
- Join worker nodes to the cluster
- Verify cluster functionality

## Prerequisites
- Completed Phase 1: Infrastructure Foundation
- All three nodes accessible via SSH
- Static IP addresses configured
- Basic system hardening applied

## Understanding Kubernetes Architecture

Before installation, let's understand what we're building:

```
┌─────────────────────────────────────────────────────────────┐
│                    k8s-control (192.168.1.10)              │
│  ┌─────────────┐ ┌──────────────┐ ┌─────────────────────┐   │
│  │   etcd      │ │ kube-apiserver│ │  kube-controller-   │   │
│  │             │ │              │ │     manager         │   │
│  └─────────────┘ └──────────────┘ └─────────────────────┘   │
│  ┌─────────────┐ ┌──────────────┐ ┌─────────────────────┐   │
│  │kube-scheduler│ │   kubelet    │ │    kube-proxy       │   │
│  │             │ │              │ │                     │   │
│  └─────────────┘ └──────────────┘ └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                                │
                                │ Cluster Network
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
┌───────▼──────┐        ┌───────▼──────┐        ┌───────▼──────┐
│ k8s-worker-1 │        │ k8s-worker-2 │        │   (future)   │
│ ┌──────────┐ │        │ ┌──────────┐ │        │              │
│ │ kubelet  │ │        │ │ kubelet  │ │        │              │
│ └──────────┘ │        │ └──────────┘ │        │              │
│ ┌──────────┐ │        │ ┌──────────┐ │        │              │
│ │kube-proxy│ │        │ │kube-proxy│ │        │              │
│ └──────────┘ │        │ └──────────┘ │        │              │
└──────────────┘        └──────────────┘        └──────────────┘
```

## Step 1: Install Container Runtime (All Nodes)

**Why containerd?**
- Default runtime for Kubernetes 1.24+
- Lighter weight than Docker
- CRI (Container Runtime Interface) compliant
- Production-ready and stable

### Install containerd

```bash
# Run these commands on ALL nodes (k8s-control, k8s-worker-1, k8s-worker-2)

# Add Docker's official GPG key and repository
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index and install containerd
sudo apt-get update
sudo apt-get install -y containerd.io
```

### Configure containerd

```bash
# Generate default configuration
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (required for Kubernetes)
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

# Restart and enable containerd
sudo systemctl restart containerd
sudo systemctl enable containerd

# Verify containerd is running
sudo systemctl status containerd
```

### Verify Container Runtime

```bash
# Test containerd
sudo ctr version

# Check if containerd can pull images
sudo ctr images pull docker.io/library/hello-world:latest
sudo ctr run --rm docker.io/library/hello-world:latest hello-world
```

## Step 2: Install Kubernetes Components (All Nodes)

### Add Kubernetes Repository

```bash
# Run on ALL nodes

# Add Kubernetes signing key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package index
sudo apt-get update
```

### Install Kubernetes Tools

```bash
# Install specific version for consistency
sudo apt-get install -y kubelet=1.28.2-1.1 kubeadm=1.28.2-1.1 kubectl=1.28.2-1.1

# Prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet service
sudo systemctl enable kubelet
```

**Why hold packages?**
- Prevents automatic updates that could break cluster compatibility
- Allows controlled cluster upgrades
- Ensures all nodes run same Kubernetes version

### Configure kubectl (Control Plane Only)

```bash
# Run this only on k8s-control

# Create kubectl configuration directory
mkdir -p $HOME/.kube

# We'll copy admin.conf here after cluster initialization
```

## Step 3: Initialize Control Plane (k8s-control Only)

### Create kubeadm Configuration

```bash
# Run on k8s-control only

# Create kubeadm config file
cat <<EOF | sudo tee /etc/kubernetes/kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.28.2
controlPlaneEndpoint: "k8s-control:6443"
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.244.0.0/16"
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.1.10"
  bindPort: 6443
nodeRegistration:
  criSocket: "unix:///var/run/containerd/containerd.sock"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```

**Configuration Explanation:**
- `controlPlaneEndpoint`: DNS name for API server (enables HA later)
- `serviceSubnet`: Internal cluster service network
- `podSubnet`: Network for pod-to-pod communication
- `criSocket`: Container runtime interface socket

### Initialize the Cluster

```bash
# Initialize cluster (this takes 2-5 minutes)
sudo kubeadm init --config=/etc/kubernetes/kubeadm-config.yaml

# IMPORTANT: Save the join command output!
# It will look like:
# kubeadm join k8s-control:6443 --token abc123.xyz456 \
#   --discovery-token-ca-cert-hash sha256:abcdef123456...
```

### Configure kubectl for Regular User

```bash
# Copy admin configuration
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Test kubectl access
kubectl get nodes
kubectl get pods -A
```

You should see the control plane node in "NotReady" state - this is expected before CNI installation.

## Step 4: Install Container Network Interface (CNI)

**Why do we need CNI?**
- Kubernetes doesn't provide networking by default
- CNI plugin handles pod-to-pod communication
- Required for nodes to be "Ready"

### Install Flannel CNI

```bash
# Run on k8s-control only

# Download and apply Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Wait for Flannel pods to be ready
kubectl wait --for=condition=ready pod -l app=flannel -n kube-flannel-system --timeout=300s

# Verify CNI installation
kubectl get pods -n kube-flannel-system
kubectl get nodes
```

**Alternative: Calico CNI (optional)**
```bash
# If you prefer Calico over Flannel
# kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/tigera-operator.yaml
# kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/custom-resources.yaml
```

## Step 5: Configure Firewall for Kubernetes

### Control Plane Firewall Rules

```bash
# Run on k8s-control

# Kubernetes API server
sudo ufw allow 6443/tcp

# etcd server client API
sudo ufw allow 2379:2380/tcp

# Kubelet API
sudo ufw allow 10250/tcp

# kube-scheduler
sudo ufw allow 10259/tcp

# kube-controller-manager
sudo ufw allow 10257/tcp

# Flannel VXLAN
sudo ufw allow 8472/udp

# Check firewall status
sudo ufw status numbered
```

### Worker Node Firewall Rules

```bash
# Run on k8s-worker-1 and k8s-worker-2

# Kubelet API
sudo ufw allow 10250/tcp

# NodePort Services
sudo ufw allow 30000:32767/tcp

# Flannel VXLAN
sudo ufw allow 8472/udp

# Check firewall status
sudo ufw status numbered
```

## Step 6: Join Worker Nodes

### Get Join Command (if lost)

```bash
# Run on k8s-control if you need to regenerate the join command
kubeadm token create --print-join-command
```

### Join Worker Nodes

```bash
# Run on k8s-worker-1 and k8s-worker-2
# Use the join command from Step 3 or regenerated above

sudo kubeadm join k8s-control:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### Verify Node Join

```bash
# Run on k8s-control
kubectl get nodes

# You should see all three nodes
# NAME           STATUS   ROLES           AGE   VERSION
# k8s-control    Ready    control-plane   5m    v1.28.2
# k8s-worker-1   Ready    <none>          2m    v1.28.2
# k8s-worker-2   Ready    <none>          2m    v1.28.2
```

## Step 7: Verification and Testing

### Check Cluster Health

```bash
# Check all system pods
kubectl get pods -A

# Check node details
kubectl describe nodes

# Check cluster info
kubectl cluster-info

# Check component status
kubectl get componentstatuses
```

### Deploy Test Application

```bash
# Create a test deployment
kubectl create deployment nginx-test --image=nginx

# Scale to 3 replicas
kubectl scale deployment nginx-test --replicas=3

# Check pod distribution
kubectl get pods -o wide

# Expose the deployment
kubectl expose deployment nginx-test --port=80 --type=NodePort

# Get service details
kubectl get services nginx-test
```

### Test Pod Communication

```bash
# Get pod IPs
kubectl get pods -o wide

# Test pod-to-pod communication
kubectl exec nginx-test-<pod-id> -- curl <another-pod-ip>

# Test service discovery
kubectl exec nginx-test-<pod-id> -- nslookup nginx-test
```

### Clean Up Test Resources

```bash
# Remove test deployment and service
kubectl delete deployment nginx-test
kubectl delete service nginx-test
```

## Step 8: Optional Control Plane Configuration

### Label Worker Nodes

```bash
# Add labels to identify node roles
kubectl label node k8s-worker-1 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker-2 node-role.kubernetes.io/worker=worker

# Verify labels
kubectl get nodes --show-labels
```

### Configure kubectl on Worker Nodes (Optional)

```bash
# Run on k8s-control
sudo cat /etc/kubernetes/admin.conf

# Copy the content and create ~/.kube/config on worker nodes
# This allows running kubectl commands from worker nodes
```

## Troubleshooting

### Common Issues

**Nodes stuck in NotReady state:**
```bash
# Check kubelet logs
sudo journalctl -xeu kubelet

# Check CNI installation
kubectl get pods -n kube-flannel-system

# Restart kubelet if needed
sudo systemctl restart kubelet
```

**Pod networking issues:**
```bash
# Check Flannel daemonset
kubectl describe ds kube-flannel-ds -n kube-flannel-system

# Check CNI configuration
sudo ls -la /etc/cni/net.d/
```

**Control plane components not starting:**
```bash
# Check static pod manifests
sudo ls -la /etc/kubernetes/manifests/

# Check kubelet configuration
sudo systemctl status kubelet
```

### Reset Cluster (if needed)

```bash
# If you need to start over
sudo kubeadm reset
sudo rm -rf /etc/kubernetes/
sudo rm -rf ~/.kube/
sudo systemctl restart kubelet containerd
```

## Security Considerations

### RBAC is Enabled by Default

Kubernetes 1.28+ has Role-Based Access Control (RBAC) enabled by default, which means:
- Service accounts have minimal permissions
- Custom roles needed for applications
- Admin access requires explicit configuration

### Network Policies (Future)

Once you understand basic Kubernetes operations, consider implementing:
- Network policies to control pod-to-pod communication
- Pod security policies/standards
- Resource quotas and limits

## Next Steps

With your Kubernetes cluster running, you're ready for **Phase 3: Essential Cluster Components**. You should now have:

- ✅ Three-node Kubernetes cluster (1 control plane, 2 workers)
- ✅ Container runtime (containerd) configured
- ✅ CNI networking (Flannel) operational
- ✅ All nodes in "Ready" state
- ✅ kubectl configured and functional
- ✅ Basic pod deployment and communication tested

**Key Commands for Daily Use:**
```bash
# Check cluster status
kubectl get nodes
kubectl get pods -A

# View cluster events
kubectl get events --sort-by='.lastTimestamp'

# Check resource usage
kubectl top nodes  # requires metrics-server (Phase 3)
```

**Important Files Created:**
- `/etc/kubernetes/admin.conf` - Cluster admin credentials
- `/etc/kubernetes/kubeadm-config.yaml` - Cluster configuration
- `~/.kube/config` - kubectl configuration
# Setting Up Kubernetes Across Multiple Networks

## Prerequisites

Required on all nodes:
```bash
# Update and install dependencies
apt-get update && apt-get upgrade -y
apt-get install -y curl apt-transport-https ca-certificates software-properties-common

# Install containerd
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
systemctl restart containerd

# Disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Load required modules
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# Configure network parameters
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

## Network Configuration

Create a network configuration file for Calico:

```yaml
# calico-config.yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Enable cross-subnet overlay mode
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLAN
      natOutgoing: true
      nodeSelector: all()
```

## Control Plane Setup (Primary Network)

```bash
# Initialize control plane
kubeadm init --control-plane-endpoint "LOAD_BALANCER_IP:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16 \
  --service-cidr=10.96.0.0/12

# Save the join commands output for workers

# Install Calico CNI
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/tigera-operator.yaml
kubectl create -f calico-config.yaml
```

## Worker Node Setup (All Networks)

For each worker node:
```bash
# Use the join command from control plane setup
kubeadm join LOAD_BALANCER_IP:6443 \
  --token  \
  --discovery-token-ca-cert-hash sha256:
```

## Network Labels and Taints

Label nodes according to their network:
```bash
# Label nodes
kubectl label nodes  network=network1
kubectl label nodes  network=network2
kubectl label nodes  network=network3

# Optional: Add network-specific taints
kubectl taint nodes -l network=network1 network=network1:NoSchedule
kubectl taint nodes -l network=network2 network=network2:NoSchedule
kubectl taint nodes -l network=network3 network=network3:NoSchedule
```

## Example Network Policy

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: network-isolation
spec:
  podSelector:
    matchLabels:
      network: network1
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          network: network1
  egress:
  - to:
    - podSelector:
        matchLabels:
          network: network1
```

## Example Deployment with Network Affinity

```yaml
# deployment-network1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-network1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      network: network1
  template:
    metadata:
      labels:
        app: myapp
        network: network1
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: network
                operator: In
                values:
                - network1
      tolerations:
      - key: "network"
        operator: "Equal"
        value: "network1"
        effect: "NoSchedule"
      containers:
      - name: myapp
        image: nginx:latest
```

## High Availability Setup

1. Load Balancer Configuration (HAProxy example):
```conf
frontend kubernetes
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-control-plane

backend kubernetes-control-plane
    mode tcp
    balance roundrobin
    option tcp-check
    server control1 CONTROL1_IP:6443 check fall 3 rise 2
    server control2 CONTROL2_IP:6443 check fall 3 rise 2
    server control3 CONTROL3_IP:6443 check fall 3 rise 2
```

2. Set up additional control plane nodes:
```bash
kubeadm join LOAD_BALANCER_IP:6443 \
  --token  \
  --discovery-token-ca-cert-hash sha256: \
  --control-plane \
  --certificate-key 
```

## Monitoring Setup

1. Install Prometheus and Grafana:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack
```

## Troubleshooting Commands

```bash
# Check node status
kubectl get nodes -o wide

# Check pod networking
kubectl get pods -o wide -A

# Check Calico status
kubectl get pods -n calico-system

# View logs
kubectl logs -n calico-system 

# Test cross-network connectivity
kubectl exec -it  -- ping 
```

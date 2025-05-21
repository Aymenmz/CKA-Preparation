# Solution Guide: Cluster Setup with kubeadm

## Task 1: Cluster Setup with kubeadm

**Scenario**: TechNova needs to set up a new Kubernetes cluster for their development team.

**Task**: Initialize a Kubernetes control plane using kubeadm, configure it for high availability, and join a worker node to the cluster.

## Solution Steps

### 1. Prerequisites Check

First, ensure all nodes meet the requirements:

```bash
# On all nodes (control plane and worker)
# Check system resources
free -m
nproc

# Verify Docker/container runtime is installed
systemctl status docker

# Disable swap
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Enable required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Set required sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 2. Install kubeadm, kubelet, and kubectl

```bash
# On all nodes
# Update package index
sudo apt-get update

# Install required packages
sudo apt-get install -y apt-transport-https ca-certificates curl

# Add Kubernetes apt repository
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package index again
sudo apt-get update

# Install kubeadm, kubelet, and kubectl
sudo apt-get install -y kubelet kubeadm kubectl

# Pin their versions
sudo apt-mark hold kubelet kubeadm kubectl
```

### 3. Initialize the Control Plane

```bash
# On the first control plane node
# Create a kubeadm configuration file for HA setup
cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.28.0
controlPlaneEndpoint: "control-plane-endpoint:6443"  # Replace with your load balancer address
networking:
  podSubnet: 192.168.0.0/16  # For Calico CNI
apiServer:
  certSANs:
  - "control-plane-endpoint"  # Replace with your load balancer hostname
  - "control-plane-1"  # Replace with your control plane node hostnames
  - "control-plane-2"
  - "control-plane-3"
EOF

# Initialize the first control plane node
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs

# Save the output commands for joining other control plane nodes and worker nodes
```

### 4. Set Up kubectl Access

```bash
# On the first control plane node
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5. Install a CNI Plugin (Calico)

```bash
# On the first control plane node
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Verify all system pods are running
kubectl get pods -n kube-system
```

### 6. Join Additional Control Plane Nodes (for HA)

```bash
# On additional control plane nodes
# Use the join command with the --control-plane flag from the kubeadm init output
sudo kubeadm join control-plane-endpoint:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <certificate-key>
```

### 7. Join Worker Nodes

```bash
# On worker nodes
# Use the join command from the kubeadm init output
sudo kubeadm join control-plane-endpoint:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

### 8. Verify Cluster Status

```bash
# On the control plane node
kubectl get nodes
kubectl get pods --all-namespaces
```

### 9. Configure High Availability for etcd (if not using external etcd)

The kubeadm setup automatically configures a stacked etcd cluster when multiple control plane nodes are added.

### 10. Test Cluster Functionality

```bash
# Create a test deployment
kubectl create deployment nginx --image=nginx

# Expose the deployment
kubectl expose deployment nginx --port=80 --type=NodePort

# Check if the service is accessible
kubectl get svc nginx
```

## Verification

A successful solution will show:

1. Multiple nodes in the cluster with correct roles:
   ```
   kubectl get nodes
   NAME            STATUS   ROLES           AGE   VERSION
   control-plane-1 Ready    control-plane   10m   v1.28.0
   control-plane-2 Ready    control-plane   8m    v1.28.0
   control-plane-3 Ready    control-plane   6m    v1.28.0
   worker-1        Ready    <none>          4m    v1.28.0
   ```

2. All system pods running correctly:
   ```
   kubectl get pods -n kube-system
   ```

3. Successful test deployment:
   ```
   kubectl get deployment nginx
   NAME    READY   UP-TO-DATE   AVAILABLE   AGE
   nginx   1/1     1            1           2m
   ```

## Common Issues and Troubleshooting

1. **Node Not Joining**: Check network connectivity, firewall rules, and token validity
2. **Pods Not Starting**: Verify CNI installation and network configuration
3. **API Server Unavailable**: Check certificates and control plane endpoint configuration
4. **etcd Issues**: Verify etcd health with `kubectl get pods -n kube-system | grep etcd`

## Notes for CKA Exam

- In the actual exam, you might be asked to set up a single control plane node rather than a full HA setup
- Focus on getting the basic cluster functionality working first before optimizing
- Remember to check node status and system pods to verify your setup
- The exam environment will have internet access for downloading necessary components

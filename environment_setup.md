# Kubernetes Environment Setup for CKA Practice Exam

This document outlines the setup process for creating a realistic Kubernetes environment for CKA exam practice. Based on research and best practices, we'll use Minikube as our primary tool to create a local Kubernetes cluster that mimics the exam environment.

## Environment Requirements

To successfully complete the CKA practice exam, you'll need:

1. A system with at least:
   - 4 CPU cores
   - 8GB RAM
   - 20GB free disk space

2. The following tools installed:
   - Docker or a compatible container runtime
   - kubectl (Kubernetes command-line tool)
   - Minikube (for local Kubernetes cluster)

## Setup Instructions

### 1. Install Required Tools

#### Install Docker
```bash
# For Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
```

#### Install kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

#### Install Minikube
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
```

### 2. Create Multi-Node Kubernetes Cluster

For a realistic CKA exam environment, we'll create a multi-node cluster using Minikube:

```bash
# Start a 3-node Minikube cluster
minikube start --nodes=3 --memory=2048 --cpus=2 --kubernetes-version=v1.28.0 --driver=docker

# Verify nodes are running
kubectl get nodes
```

### 3. Install Additional Components

To support all exam tasks, we need to install additional components:

#### Install Helm
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

#### Install Metrics Server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### Configure Storage Classes
```bash
# Create a default storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: k8s.io/minikube-hostpath
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF

# Create a high-performance storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-performance
provisioner: k8s.io/minikube-hostpath
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  type: ssd
EOF
```

### 4. Configure RBAC for Practice Tasks

```bash
# Create namespaces for practice
kubectl create namespace development
kubectl create namespace production
kubectl create namespace monitoring

# Create a development role and rolebinding
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer
rules:
- apiGroups: ["", "apps", "extensions"]
  resources: ["pods", "deployments", "services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
EOF

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 5. Install Ingress Controller

```bash
# Enable the Ingress addon in Minikube
minikube addons enable ingress

# Verify Ingress controller is running
kubectl get pods -n ingress-nginx
```

### 6. Setup Gateway API

```bash
# Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.8.0/standard-install.yaml

# Install a Gateway implementation
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
helm install nginx-gateway nginx-stable/nginx-gateway --namespace nginx-gateway --create-namespace
```

### 7. Create Sample Applications for Practice

```bash
# Create a sample deployment
kubectl create deployment nginx --image=nginx --replicas=3 -n development

# Create a service
kubectl expose deployment nginx --port=80 --type=ClusterIP -n development

# Create a ConfigMap
kubectl create configmap app-config --from-literal=APP_ENV=development -n development

# Create a Secret
kubectl create secret generic app-secret --from-literal=DB_PASSWORD=password123 -n development
```

### 8. Setup Troubleshooting Scenarios

```bash
# Create a deployment with resource issues
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-issue
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: resource-issue
  template:
    metadata:
      labels:
        app: resource-issue
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            memory: "2Gi"
            cpu: "2"
EOF

# Create a deployment with image pull issues
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: image-issue
  namespace: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: image-issue
  template:
    metadata:
      labels:
        app: image-issue
    spec:
      containers:
      - name: nginx
        image: nginx:nonexistent
EOF
```

## Alternative Setup: Using Kind (Kubernetes in Docker)

If Minikube doesn't work well in your environment, you can use Kind as an alternative:

```bash
# Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create a multi-node Kind cluster
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

kind create cluster --config kind-config.yaml --name cka-practice
```

## Verification

To verify your environment is correctly set up:

```bash
# Check nodes
kubectl get nodes

# Check core components
kubectl get pods -n kube-system

# Check storage classes
kubectl get storageclass

# Check RBAC configuration
kubectl get roles,rolebindings -n development

# Check ingress controller
kubectl get pods -n ingress-nginx
```

## Resetting the Environment

If you need to reset your environment between practice sessions:

```bash
# For Minikube
minikube delete
# Then follow the setup instructions again

# For Kind
kind delete cluster --name cka-practice
# Then follow the Kind setup instructions again
```

## Recommended Workflow

1. Set up the environment following the instructions above
2. Work through the practice exam tasks one by one
3. Reset the environment between major practice sessions to ensure a clean state
4. Use the Kubernetes documentation (https://kubernetes.io/docs/) as you would in the real exam

This environment setup provides a realistic simulation of what you'll encounter in the actual CKA exam, allowing you to practice all the required skills and scenarios.

# Solution Guide: Helm and Kustomize Implementation

## Task 3: Helm and Kustomize Implementation

**Scenario**: TechNova wants to standardize application deployment using Helm and Kustomize.

**Task**: Use Helm to install a monitoring stack (Prometheus) and use Kustomize to customize a provided application deployment for different environments (dev, staging).

## Solution Steps

### Part 1: Installing Prometheus with Helm

#### 1. Add the Prometheus Helm Repository

```bash
# Add the Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update the Helm repositories
helm repo update
```

#### 2. Create a Namespace for Monitoring

```bash
# Create a namespace for the monitoring stack
kubectl create namespace monitoring
```

#### 3. Install Prometheus using Helm

```bash
# Install Prometheus with default values
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.enabled=true \
  --set alertmanager.enabled=true
```

#### 4. Verify the Installation

```bash
# Check if all pods are running
kubectl get pods -n monitoring

# Check the services created
kubectl get svc -n monitoring

# Check the Prometheus deployment
kubectl get deploy -n monitoring
```

### Part 2: Using Kustomize for Application Customization

#### 1. Create Base Application Structure

```bash
# Create directories for the kustomize structure
mkdir -p kustomize-demo/base
mkdir -p kustomize-demo/overlays/dev
mkdir -p kustomize-demo/overlays/staging
```

#### 2. Create Base Application Resources

```bash
# Create a deployment.yaml in the base directory
cat <<EOF > kustomize-demo/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.19
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF

# Create a service.yaml in the base directory
cat <<EOF > kustomize-demo/base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
EOF

# Create a configmap.yaml in the base directory
cat <<EOF > kustomize-demo/base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  app.properties: |
    environment=base
    log.level=INFO
EOF

# Create a kustomization.yaml in the base directory
cat <<EOF > kustomize-demo/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
- configmap.yaml
EOF
```

#### 3. Create Dev Environment Overlay

```bash
# Create a kustomization.yaml for the dev overlay
cat <<EOF > kustomize-demo/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: dev
resources:
- ../../base
patchesStrategicMerge:
- deployment-patch.yaml
- configmap-patch.yaml
EOF

# Create a deployment patch for dev
cat <<EOF > kustomize-demo/overlays/dev/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
EOF

# Create a configmap patch for dev
cat <<EOF > kustomize-demo/overlays/dev/configmap-patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  app.properties: |
    environment=development
    log.level=DEBUG
EOF
```

#### 4. Create Staging Environment Overlay

```bash
# Create a kustomization.yaml for the staging overlay
cat <<EOF > kustomize-demo/overlays/staging/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: staging
resources:
- ../../base
patchesStrategicMerge:
- deployment-patch.yaml
- configmap-patch.yaml
EOF

# Create a deployment patch for staging
cat <<EOF > kustomize-demo/overlays/staging/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: webapp
        resources:
          requests:
            memory: "128Mi"
            cpu: "200m"
          limits:
            memory: "256Mi"
            cpu: "400m"
EOF

# Create a configmap patch for staging
cat <<EOF > kustomize-demo/overlays/staging/configmap-patch.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  app.properties: |
    environment=staging
    log.level=WARN
EOF
```

#### 5. Create Namespaces for Dev and Staging

```bash
# Create namespaces
kubectl create namespace dev
kubectl create namespace staging
```

#### 6. Apply the Kustomized Configurations

```bash
# Apply the dev configuration
kubectl apply -k kustomize-demo/overlays/dev

# Apply the staging configuration
kubectl apply -k kustomize-demo/overlays/staging
```

#### 7. Verify the Deployments

```bash
# Check the dev deployment
kubectl get all -n dev

# Check the staging deployment
kubectl get all -n staging

# Compare the configmaps
kubectl get configmap webapp-config -n dev -o yaml
kubectl get configmap webapp-config -n staging -o yaml
```

## Verification

A successful solution will show:

1. Prometheus stack installed and running in the monitoring namespace:
   ```
   kubectl get pods -n monitoring
   ```

2. Different configurations applied to dev and staging environments:
   ```
   # Dev should have 2 replicas
   kubectl get deployment webapp -n dev -o jsonpath='{.spec.replicas}'
   # Output: 2

   # Staging should have 3 replicas
   kubectl get deployment webapp -n staging -o jsonpath='{.spec.replicas}'
   # Output: 3
   ```

3. Different ConfigMap values in each environment:
   ```
   # Check dev environment setting
   kubectl get configmap webapp-config -n dev -o jsonpath='{.data.app\.properties}' | grep environment
   # Output: environment=development

   # Check staging environment setting
   kubectl get configmap webapp-config -n staging -o jsonpath='{.data.app\.properties}' | grep environment
   # Output: environment=staging
   ```

## Common Issues and Troubleshooting

1. **Helm Repository Issues**: If you can't add the Helm repository, check your internet connection or try a different repository mirror.

2. **Prometheus Resource Requirements**: The Prometheus stack requires significant resources. If pods are pending, check if your cluster has enough resources.

3. **Kustomize Patch Errors**: Ensure your patches correctly target the resources they're meant to modify. The resource name and kind must match exactly.

4. **Namespace Issues**: Verify that namespaces exist before applying resources to them.

5. **YAML Indentation**: YAML is sensitive to indentation. If you get syntax errors, check your indentation.

## Notes for CKA Exam

- In the CKA exam, you might be asked to use Helm to install a specific chart or to use Kustomize to customize existing manifests.
- Focus on understanding the basic commands and concepts rather than memorizing all options.
- Remember that Helm uses a client-server architecture, with the client (helm CLI) and the server-side component (Tiller) in older versions, but Helm v3 removed the server-side component.
- Kustomize is built into kubectl (kubectl apply -k), so you don't need to install it separately.
- Practice using both tools to become comfortable with their syntax and capabilities.

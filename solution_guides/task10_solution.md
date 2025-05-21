# Solution Guide: Gateway API and Ingress Setup

## Task 10: Gateway API and Ingress Setup

**Scenario**: TechNova wants to expose multiple services through a single entry point.

**Task**: Configure an Ingress controller and create Ingress resources to route external traffic to different services based on paths and hostnames. Then implement the same using Gateway API resources.

## Solution Steps

### Part 1: Setting Up Ingress Controller and Resources

#### 1. Install the Nginx Ingress Controller

```bash
# Create a namespace for the ingress controller
kubectl create namespace ingress-nginx

# Install the Nginx Ingress Controller using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.replicaCount=2

# Alternatively, install using manifests
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml
```

#### 2. Verify the Ingress Controller Installation

```bash
# Check if the ingress controller pods are running
kubectl get pods -n ingress-nginx

# Check the services
kubectl get svc -n ingress-nginx
```

#### 3. Deploy Sample Applications

```bash
# Create namespaces for the applications
kubectl create namespace app1
kubectl create namespace app2

# Deploy the first application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  namespace: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: app1
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: init-html
        image: busybox
        command: ["/bin/sh", "-c", "echo '<h1>App 1</h1>' > /html/index.html"]
        volumeMounts:
        - name: html-volume
          mountPath: /html
      volumes:
      - name: html-volume
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: app1
  namespace: app1
spec:
  selector:
    app: app1
  ports:
  - port: 80
    targetPort: 80
EOF

# Deploy the second application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
  namespace: app2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: init-html
        image: busybox
        command: ["/bin/sh", "-c", "echo '<h1>App 2</h1>' > /html/index.html"]
        volumeMounts:
        - name: html-volume
          mountPath: /html
      volumes:
      - name: html-volume
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: app2
  namespace: app2
spec:
  selector:
    app: app2
  ports:
  - port: 80
    targetPort: 80
EOF
```

#### 4. Create Ingress Resources for Path-Based Routing

```bash
# Create an Ingress resource for path-based routing
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80
EOF
```

#### 5. Create Ingress Resources for Host-Based Routing

```bash
# Create an Ingress resource for host-based routing
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1
            port:
              number: 80
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2
            port:
              number: 80
EOF
```

#### 6. Verify the Ingress Resources

```bash
# Check the Ingress resources
kubectl get ingress --all-namespaces

# Describe the Ingress resources
kubectl describe ingress path-based-ingress
kubectl describe ingress host-based-ingress

# Get the external IP or hostname of the Ingress controller
INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Ingress IP: $INGRESS_IP"

# If IP is not available, get the hostname
if [ -z "$INGRESS_IP" ]; then
  INGRESS_HOSTNAME=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
  echo "Ingress Hostname: $INGRESS_HOSTNAME"
fi
```

#### 7. Test the Ingress Resources

```bash
# Test path-based routing
curl http://$INGRESS_IP/app1
curl http://$INGRESS_IP/app2

# Test host-based routing (add entries to /etc/hosts if testing locally)
# echo "$INGRESS_IP app1.example.com app2.example.com" | sudo tee -a /etc/hosts
curl -H "Host: app1.example.com" http://$INGRESS_IP
curl -H "Host: app2.example.com" http://$INGRESS_IP
```

### Part 2: Setting Up Gateway API Resources

#### 1. Install the Gateway API CRDs

```bash
# Install the Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v0.8.0/standard-install.yaml
```

#### 2. Install a Gateway API Implementation

```bash
# For this example, we'll use Contour as a Gateway API implementation
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml

# Alternatively, you can use Nginx Gateway
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
helm install nginx-gateway nginx-stable/nginx-gateway --namespace nginx-gateway --create-namespace
```

#### 3. Create a Gateway Resource

```bash
# Create a Gateway resource
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: technova-gateway
  namespace: default
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
EOF
```

#### 4. Create HTTPRoute Resources for Path-Based Routing

```bash
# Create an HTTPRoute resource for path-based routing
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: path-based-route
  namespace: default
spec:
  parentRefs:
  - name: technova-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /app1
    backendRefs:
    - name: app1
      namespace: app1
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /app2
    backendRefs:
    - name: app2
      namespace: app2
      port: 80
EOF
```

#### 5. Create HTTPRoute Resources for Host-Based Routing

```bash
# Create an HTTPRoute resource for host-based routing
cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: host-based-route
  namespace: default
spec:
  parentRefs:
  - name: technova-gateway
  hostnames:
  - "app1.example.com"
  - "app2.example.com"
  rules:
  - matches:
    - headers:
      - name: Host
        value: app1.example.com
    backendRefs:
    - name: app1
      namespace: app1
      port: 80
  - matches:
    - headers:
      - name: Host
        value: app2.example.com
    backendRefs:
    - name: app2
      namespace: app2
      port: 80
EOF
```

#### 6. Verify the Gateway API Resources

```bash
# Check the Gateway resource
kubectl get gateway

# Check the HTTPRoute resources
kubectl get httproute

# Describe the Gateway resource
kubectl describe gateway technova-gateway

# Describe the HTTPRoute resources
kubectl describe httproute path-based-route
kubectl describe httproute host-based-route

# Get the external IP or hostname of the Gateway
GATEWAY_IP=$(kubectl get gateway technova-gateway -o jsonpath='{.status.addresses[0].value}')
echo "Gateway IP: $GATEWAY_IP"
```

#### 7. Test the Gateway API Resources

```bash
# Test path-based routing
curl http://$GATEWAY_IP/app1
curl http://$GATEWAY_IP/app2

# Test host-based routing (add entries to /etc/hosts if testing locally)
# echo "$GATEWAY_IP app1.example.com app2.example.com" | sudo tee -a /etc/hosts
curl -H "Host: app1.example.com" http://$GATEWAY_IP
curl -H "Host: app2.example.com" http://$GATEWAY_IP
```

## Verification

A successful solution will show:

1. The Ingress controller is running:
   ```
   kubectl get pods -n ingress-nginx
   ```

2. The Ingress resources are created:
   ```
   kubectl get ingress --all-namespaces
   ```

3. The Gateway API resources are created:
   ```
   kubectl get gateway
   kubectl get httproute
   ```

4. Path-based routing works correctly:
   ```
   curl http://$INGRESS_IP/app1  # Should return "App 1"
   curl http://$INGRESS_IP/app2  # Should return "App 2"
   ```

5. Host-based routing works correctly:
   ```
   curl -H "Host: app1.example.com" http://$INGRESS_IP  # Should return "App 1"
   curl -H "Host: app2.example.com" http://$INGRESS_IP  # Should return "App 2"
   ```

## Common Issues and Troubleshooting

1. **Ingress Controller Not Ready**: Check the logs of the Ingress controller pods:
   ```
   kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
   ```

2. **Ingress Rules Not Working**: Verify the Ingress resource configuration:
   ```
   kubectl describe ingress <ingress-name>
   ```

3. **Gateway API Resources Not Working**: Check if the Gateway API implementation is correctly installed and the Gateway resource is ready:
   ```
   kubectl describe gateway technova-gateway
   ```

4. **Service Not Found**: Ensure the services exist in the correct namespaces:
   ```
   kubectl get svc -n app1
   kubectl get svc -n app2
   ```

5. **Path Rewriting Issues**: Check the annotations on the Ingress resource for path rewriting.

## Notes for CKA Exam

- In the CKA exam, you might be asked to set up Ingress resources for routing traffic to different services.
- Understand the difference between path-based and host-based routing.
- Be familiar with the Ingress resource structure and common annotations.
- Know how to verify that Ingress resources are working correctly.
- The Gateway API is newer and might not be extensively tested in the CKA exam, but understanding its basic concepts is valuable.
- Remember that the Gateway API provides a more expressive and extensible model for routing compared to Ingress.
- Be prepared to troubleshoot issues with Ingress and Gateway resources.

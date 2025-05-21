# Solution Guide: Service Configuration

## Task 9: Service Configuration

**Scenario**: Different applications require different types of service exposure.

**Task**: Create ClusterIP, NodePort, and LoadBalancer services for different applications and verify connectivity between them.

## Solution Steps

### 1. Create Namespaces for Different Applications

```bash
# Create namespaces for different applications
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database
```

### 2. Deploy Applications in Each Namespace

```bash
# Deploy frontend application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF

# Deploy backend application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF

# Deploy database application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          value: "password123"
EOF
```

### 3. Create a ClusterIP Service for the Database

```bash
# Create a ClusterIP service for the database
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: database
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
EOF
```

### 4. Create a ClusterIP Service for the Backend

```bash
# Create a ClusterIP service for the backend
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

### 5. Create a NodePort Service for the Backend (Alternative Access)

```bash
# Create a NodePort service for the backend
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
  namespace: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort
EOF
```

### 6. Create a LoadBalancer Service for the Frontend

```bash
# Create a LoadBalancer service for the frontend
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
EOF
```

### 7. Verify the Services

```bash
# Check all services across namespaces
kubectl get services --all-namespaces

# Check the ClusterIP service for the database
kubectl get service database -n database
kubectl describe service database -n database

# Check the ClusterIP service for the backend
kubectl get service backend -n backend
kubectl describe service backend -n backend

# Check the NodePort service for the backend
kubectl get service backend-nodeport -n backend
kubectl describe service backend-nodeport -n backend

# Check the LoadBalancer service for the frontend
kubectl get service frontend -n frontend
kubectl describe service frontend -n frontend
```

### 8. Test Connectivity Between Services

```bash
# Create a temporary pod for testing
kubectl run test-pod --image=busybox -n frontend -- sleep 3600

# Wait for the pod to be ready
kubectl wait --for=condition=Ready pod/test-pod -n frontend --timeout=60s

# Test connectivity to the backend service
kubectl exec -n frontend test-pod -- wget -O- http://backend.backend.svc.cluster.local

# Test connectivity to the database service
kubectl exec -n frontend test-pod -- nc -zv database.database.svc.cluster.local 5432

# Test connectivity to the backend NodePort service (from within the cluster)
kubectl exec -n frontend test-pod -- wget -O- http://backend-nodeport.backend.svc.cluster.local

# Get the node IP to test NodePort from outside
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "Access backend via NodePort at: $NODE_IP:30080"

# For LoadBalancer, get the external IP (if available in your environment)
EXTERNAL_IP=$(kubectl get service frontend -n frontend -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Access frontend via LoadBalancer at: $EXTERNAL_IP:80"
```

### 9. Create a Headless Service for the Database (Optional)

```bash
# Create a headless service for the database
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: database-headless
  namespace: database
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None
EOF

# Test the headless service
kubectl exec -n frontend test-pod -- nslookup database-headless.database.svc.cluster.local
```

### 10. Create a Service with Multiple Ports (Optional)

```bash
# Create a deployment with multiple ports
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-port-app
  namespace: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multi-port-app
  template:
    metadata:
      labels:
        app: multi-port-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        - containerPort: 443
EOF

# Create a service with multiple ports
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: multi-port-service
  namespace: frontend
spec:
  selector:
    app: multi-port-app
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
  type: ClusterIP
EOF

# Test the multi-port service
kubectl exec -n frontend test-pod -- wget -O- http://multi-port-service.frontend.svc.cluster.local
```

### 11. Clean Up the Test Pod

```bash
# Delete the test pod
kubectl delete pod test-pod -n frontend
```

## Verification

A successful solution will show:

1. All services are created and running:
   ```
   kubectl get services --all-namespaces | grep -E 'frontend|backend|database'
   ```

2. The ClusterIP services are accessible within the cluster:
   ```
   kubectl exec -n frontend test-pod -- wget -O- http://backend.backend.svc.cluster.local
   kubectl exec -n frontend test-pod -- nc -zv database.database.svc.cluster.local 5432
   ```

3. The NodePort service is accessible via the node IP:
   ```
   # From a node or external machine with access to the node
   curl http://<node-ip>:30080
   ```

4. The LoadBalancer service has an external IP (if supported by your environment):
   ```
   kubectl get service frontend -n frontend
   ```

## Common Issues and Troubleshooting

1. **Service Not Routing Traffic**: Verify the service selector matches the pod labels:
   ```
   kubectl get pods -n <namespace> --show-labels
   kubectl describe service <service-name> -n <namespace>
   ```

2. **NodePort Service Not Accessible**: Check if the port is open in any firewall rules and that the node is reachable.

3. **LoadBalancer Pending**: In some environments (like Minikube), LoadBalancer services may remain in "pending" state. Use `minikube tunnel` or a similar command to simulate a LoadBalancer.

4. **DNS Resolution Issues**: If services can't be reached by name, check CoreDNS:
   ```
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   kubectl logs -n kube-system -l k8s-app=kube-dns
   ```

5. **Connection Refused**: Ensure the target pods are running and the application is listening on the expected port:
   ```
   kubectl get pods -n <namespace>
   kubectl exec -n <namespace> <pod-name> -- netstat -tulpn
   ```

## Notes for CKA Exam

- In the CKA exam, you might be asked to create different types of services for various applications.
- Understand the differences between ClusterIP, NodePort, and LoadBalancer service types.
- Be familiar with service DNS naming: `<service-name>.<namespace>.svc.cluster.local`
- Know how to test connectivity between services using temporary pods.
- Remember that ClusterIP services are only accessible within the cluster, NodePort services expose ports on all nodes, and LoadBalancer services create external load balancers (if supported by the environment).
- Understand headless services (services with `clusterIP: None`) and when to use them.
- Be able to create services with multiple ports and named ports.

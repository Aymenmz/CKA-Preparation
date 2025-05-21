# Solution Guide: Network Policy Implementation

## Task 8: Network Policy Implementation

**Scenario**: TechNova needs to secure communication between microservices.

**Task**: Create network policies that allow specific communication paths between pods in different namespaces while blocking unauthorized traffic.

## Solution Steps

### 1. Create the Required Namespaces

```bash
# Create namespaces if they don't exist
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database
```

### 2. Deploy Sample Applications in Each Namespace

```bash
# Deploy frontend application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend
  labels:
    app: frontend
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
        image: nginx
        ports:
        - containerPort: 80
---
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
EOF

# Deploy backend application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: backend
  labels:
    app: backend
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
        image: nginx
        ports:
        - containerPort: 80
---
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
EOF

# Deploy database application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: database
  labels:
    app: database
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
        env:
        - name: POSTGRES_PASSWORD
          value: "password"
        ports:
        - containerPort: 5432
---
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
EOF
```

### 3. Create Default Deny Network Policies for Each Namespace

```bash
# Default deny for frontend namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: frontend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Default deny for backend namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

# Default deny for database namespace
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: database
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

### 4. Create Network Policy for Frontend to Backend Communication

```bash
# Allow frontend to backend communication
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
      podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
EOF

# Allow frontend egress to backend
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-egress-to-backend
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: backend
      podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 80
EOF
```

### 5. Create Network Policy for Backend to Database Communication

```bash
# Allow backend to database communication
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: backend
      podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 5432
EOF

# Allow backend egress to database
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-egress-to-database
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: database
      podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
EOF
```

### 6. Allow DNS Access for All Pods

```bash
# Allow DNS egress for frontend
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: frontend
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF

# Allow DNS egress for backend
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF

# Allow DNS egress for database
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: database
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
```

### 7. Test the Network Policies

```bash
# Get a frontend pod name
FRONTEND_POD=$(kubectl get pod -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}')

# Get a backend pod name
BACKEND_POD=$(kubectl get pod -n backend -l app=backend -o jsonpath='{.items[0].metadata.name}')

# Get a database pod name
DATABASE_POD=$(kubectl get pod -n database -l app=database -o jsonpath='{.items[0].metadata.name}')

# Test frontend to backend connectivity (should work)
kubectl exec -n frontend $FRONTEND_POD -- curl -s backend.backend.svc.cluster.local

# Test frontend to database connectivity (should fail)
kubectl exec -n frontend $FRONTEND_POD -- curl -s --connect-timeout 5 database.database.svc.cluster.local:5432

# Test backend to database connectivity (should work)
kubectl exec -n backend $BACKEND_POD -- curl -s database.database.svc.cluster.local:5432

# Test database to backend connectivity (should fail)
kubectl exec -n database $DATABASE_POD -- curl -s --connect-timeout 5 backend.backend.svc.cluster.local
```

## Verification

A successful solution will show:

1. All network policies are created:
   ```
   kubectl get networkpolicy -A
   ```

2. Communication tests show the expected results:
   - Frontend can communicate with backend
   - Backend can communicate with database
   - Frontend cannot directly communicate with database
   - Database cannot communicate with backend or frontend

## Common Issues and Troubleshooting

1. **DNS Resolution Issues**: Ensure DNS egress policies are correctly configured
2. **Namespace Labels**: Verify that namespace labels are correctly set for namespaceSelector
3. **Pod Labels**: Ensure pod labels match those specified in the network policies
4. **CNI Plugin**: Confirm that your cluster's CNI plugin supports NetworkPolicy (e.g., Calico, Weave, Cilium)

## Notes for CKA Exam

- In the CKA exam, you might be asked to implement a subset of these policies
- Focus on understanding the syntax and structure of NetworkPolicy resources
- Remember that both ingress and egress rules may be needed for complete communication paths
- Always test your network policies to verify they work as expected
- Label your namespaces if they aren't already labeled for namespaceSelector to work

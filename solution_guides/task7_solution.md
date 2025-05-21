# Solution Guide: Resource Management and Scheduling

## Task 7: Resource Management and Scheduling

**Scenario**: TechNova has applications with specific resource requirements and node preferences.

**Task**: Deploy pods with resource requests/limits and configure node affinity to ensure pods run on appropriate nodes based on labels.

## Solution Steps

### 1. Check and Label the Nodes

```bash
# Check available nodes
kubectl get nodes

# Add labels to nodes for scheduling purposes
kubectl label node <worker-node-1> disktype=ssd
kubectl label node <worker-node-2> disktype=hdd
kubectl label node <worker-node-1> environment=production
kubectl label node <worker-node-2> environment=development

# Verify the labels
kubectl get nodes --show-labels
```

### 2. Create a Namespace for the Applications

```bash
# Create a namespace
kubectl create namespace resource-demo
```

### 3. Deploy a Pod with Resource Requests and Limits

```bash
# Create a pod with resource requests and limits
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo-pod
  namespace: resource-demo
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
EOF
```

### 4. Deploy a Pod with Node Affinity for SSD Nodes

```bash
# Create a pod with node affinity for SSD nodes
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
  namespace: resource-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
EOF
```

### 5. Deploy a Pod with Node Affinity for Production Environment

```bash
# Create a pod with node affinity for production environment
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: production-pod
  namespace: resource-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "512Mi"
        cpu: "1000m"
EOF
```

### 6. Deploy a Pod with Preferred Node Affinity

```bash
# Create a pod with preferred node affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: preferred-node-pod
  namespace: resource-demo
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 20
        preference:
          matchExpressions:
          - key: environment
            operator: In
            values:
            - production
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
EOF
```

### 7. Deploy a Pod with Both Node Affinity and Anti-Affinity

```bash
# Create a pod with both node affinity and anti-affinity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: complex-affinity-pod
  namespace: resource-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: environment
            operator: In
            values:
            - production
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - nginx
          topologyKey: "kubernetes.io/hostname"
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "256Mi"
        cpu: "500m"
EOF
```

### 8. Create a Deployment with Resource Requests/Limits and Node Affinity

```bash
# Create a deployment with resource requests/limits and node affinity
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-deployment
  namespace: resource-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resource-app
  template:
    metadata:
      labels:
        app: resource-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
EOF
```

### 9. Verify Pod Placement

```bash
# Check where the pods are scheduled
kubectl get pods -n resource-demo -o wide

# Describe pods to see the events and node assignment
kubectl describe pod ssd-pod -n resource-demo
kubectl describe pod production-pod -n resource-demo
kubectl describe pod preferred-node-pod -n resource-demo
kubectl describe pod complex-affinity-pod -n resource-demo

# Check the deployment pods
kubectl get pods -l app=resource-app -n resource-demo -o wide
```

## Verification

A successful solution will show:

1. Pods are scheduled on nodes according to their affinity rules:
   ```
   kubectl get pods -n resource-demo -o wide
   ```

2. The `ssd-pod` is scheduled on a node with the `disktype=ssd` label:
   ```
   kubectl describe pod ssd-pod -n resource-demo | grep Node:
   Node:         worker-node-1/10.0.0.1
   ```

3. The `production-pod` is scheduled on a node with the `environment=production` label:
   ```
   kubectl describe pod production-pod -n resource-demo | grep Node:
   Node:         worker-node-1/10.0.0.1
   ```

4. The deployment pods are scheduled on nodes with the `disktype=ssd` label:
   ```
   kubectl get pods -l app=resource-app -n resource-demo -o wide
   ```

## Common Issues and Troubleshooting

1. **Pod Stuck in Pending State**: This could be due to:
   - No nodes matching the affinity rules
   - Insufficient resources on matching nodes
   - Check events with `kubectl describe pod <pod-name> -n resource-demo`

2. **Node Labels Not Applied**: Verify labels with:
   ```
   kubectl get nodes --show-labels
   ```

3. **Resource Constraints**: If the cluster is resource-constrained, pods might not schedule even with correct affinity. Check node capacity:
   ```
   kubectl describe nodes | grep -A 5 Capacity
   ```

4. **Affinity Syntax Errors**: Ensure the affinity configuration is correct. Common issues include:
   - Incorrect operator (In, NotIn, Exists, DoesNotExist)
   - Typos in key or value names
   - Missing required fields

5. **Pod Anti-Affinity Issues**: If using pod anti-affinity, ensure the `topologyKey` exists on your nodes.

## Notes for CKA Exam

- In the CKA exam, you might be asked to schedule pods on specific nodes based on labels or other criteria.
- Understand the difference between `requiredDuringSchedulingIgnoredDuringExecution` (hard requirement) and `preferredDuringSchedulingIgnoredDuringExecution` (soft preference).
- Be familiar with resource requests and limits and how they affect scheduling.
- Know how to check node capacity and allocatable resources.
- Remember that node affinity is used to constrain which nodes a pod can be scheduled on, while pod affinity/anti-affinity is used to influence pod placement relative to other pods.
- Be able to troubleshoot scheduling issues by checking events and node conditions.

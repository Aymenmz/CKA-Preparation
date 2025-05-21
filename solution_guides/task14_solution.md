# Solution Guide: Cluster Networking Troubleshooting

## Task 14: Cluster Networking Troubleshooting

**Scenario**: Network connectivity issues are affecting communication between services in the cluster.

**Task**: Diagnose and resolve networking problems, which may include DNS resolution failures, network policy issues, or service connectivity problems.

## Solution Steps

### 1. Identify the Networking Issues

```bash
# Create a test namespace for troubleshooting
kubectl create namespace network-troubleshooting

# Deploy test pods for network diagnostics
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: network-test
  namespace: network-troubleshooting
spec:
  containers:
  - name: network-test
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
EOF

# Wait for the pod to be ready
kubectl wait --for=condition=Ready pod/network-test -n network-troubleshooting --timeout=60s
```

### 2. Check DNS Resolution

```bash
# Test DNS resolution for kubernetes service
kubectl exec -n network-troubleshooting network-test -- nslookup kubernetes.default.svc.cluster.local

# Test DNS resolution for a specific service
kubectl exec -n network-troubleshooting network-test -- nslookup <service-name>.<namespace>.svc.cluster.local

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Check CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml
```

### 3. Check Network Policies

```bash
# List all network policies
kubectl get networkpolicies --all-namespaces

# Examine a specific network policy
kubectl describe networkpolicy <policy-name> -n <namespace>

# Test if network policies are blocking traffic
# Create a test pod in the target namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: target-test
  namespace: <target-namespace>
  labels:
    app: target-test
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
  name: target-test
  namespace: <target-namespace>
spec:
  selector:
    app: target-test
  ports:
  - port: 80
    targetPort: 80
EOF

# Test connectivity to the target pod
kubectl exec -n network-troubleshooting network-test -- curl -s --connect-timeout 5 target-test.<target-namespace>.svc.cluster.local
```

### 4. Check Service Connectivity

```bash
# List all services
kubectl get services --all-namespaces

# Examine a specific service
kubectl describe service <service-name> -n <namespace>

# Check if the service has endpoints
kubectl get endpoints <service-name> -n <namespace>

# Check if the service selector matches pod labels
kubectl get pods -n <namespace> -l <selector-key>=<selector-value>

# Test connectivity to the service
kubectl exec -n network-troubleshooting network-test -- curl -s --connect-timeout 5 <service-name>.<namespace>.svc.cluster.local
```

### 5. Check Pod Network Connectivity

```bash
# Check if pods can communicate with each other
kubectl exec -n network-troubleshooting network-test -- ping -c 3 <pod-ip>

# Check if pods can reach external services
kubectl exec -n network-troubleshooting network-test -- ping -c 3 8.8.8.8

# Check if pods can resolve external domains
kubectl exec -n network-troubleshooting network-test -- nslookup google.com

# Check the pod's network interface
kubectl exec -n network-troubleshooting network-test -- ip addr

# Check the pod's routing table
kubectl exec -n network-troubleshooting network-test -- ip route

# Check the pod's iptables rules
kubectl exec -n network-troubleshooting network-test -- iptables -L
```

### 6. Check Node Network Configuration

```bash
# Check node network interfaces
kubectl get nodes -o wide

# SSH to a node (if possible) and check network configuration
# ssh <node-ip>
# ip addr
# ip route
# iptables -L

# Check if kube-proxy is running on all nodes
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Check kube-proxy logs
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

### 7. Check CNI Plugin Status

```bash
# Check CNI plugin pods (e.g., Calico, Flannel, Weave)
kubectl get pods -n kube-system -l k8s-app=calico-node
# or
kubectl get pods -n kube-system -l app=flannel
# or
kubectl get pods -n kube-system -l name=weave-net

# Check CNI plugin logs
kubectl logs -n kube-system -l k8s-app=calico-node
# or
kubectl logs -n kube-system -l app=flannel
# or
kubectl logs -n kube-system -l name=weave-net

# Check CNI configuration on a node (if possible)
# ssh <node-ip>
# ls -la /etc/cni/net.d/
# cat /etc/cni/net.d/10-calico.conflist
```

## Specific Troubleshooting Scenarios

### Scenario A: DNS Resolution Failures

```bash
# Identify DNS issues
kubectl exec -n network-troubleshooting network-test -- nslookup kubernetes.default.svc.cluster.local
# If this fails, check CoreDNS

# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
# NAME                       READY   STATUS    RESTARTS   AGE
# coredns-66bff467f8-8p7vx   0/1     Error     5          1h
# coredns-66bff467f8-v8nbb   0/1     Error     5          1h

# Check CoreDNS logs
kubectl logs -n kube-system coredns-66bff467f8-8p7vx

# If CoreDNS pods are not running, check the deployment
kubectl describe deployment coredns -n kube-system

# Fix CoreDNS deployment if needed
kubectl edit deployment coredns -n kube-system
# Correct any issues in the deployment spec

# Alternatively, recreate CoreDNS pods
kubectl delete pod -n kube-system -l k8s-app=kube-dns

# Verify DNS resolution after fix
kubectl exec -n network-troubleshooting network-test -- nslookup kubernetes.default.svc.cluster.local
```

### Scenario B: Network Policy Blocking Traffic

```bash
# Identify network policy issues
kubectl get networkpolicies --all-namespaces
# NAMESPACE   NAME           POD-SELECTOR    AGE
# app         deny-all       app=web-app     1h
# app         allow-internal app=web-app     1h

# Examine the network policies
kubectl describe networkpolicy deny-all -n app
kubectl describe networkpolicy allow-internal -n app

# Test connectivity to a service in the app namespace
kubectl exec -n network-troubleshooting network-test -- curl -s --connect-timeout 5 web-app.app.svc.cluster.local
# If this fails, the network policy might be blocking traffic

# Modify or create a network policy to allow traffic
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-network-test
  namespace: app
spec:
  podSelector:
    matchLabels:
      app: web-app
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: network-troubleshooting
EOF

# Label the network-troubleshooting namespace
kubectl label namespace network-troubleshooting name=network-troubleshooting

# Test connectivity again
kubectl exec -n network-troubleshooting network-test -- curl -s --connect-timeout 5 web-app.app.svc.cluster.local
```

### Scenario C: Service Endpoint Issues

```bash
# Identify service endpoint issues
kubectl get service web-app -n app
# NAME     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# web-app  ClusterIP   10.96.45.67     <none>        80/TCP    1h

kubectl get endpoints web-app -n app
# NAME     ENDPOINTS   AGE
# web-app  <none>      1h

# Check if the service selector matches pod labels
kubectl describe service web-app -n app
# Selector:  app=web-app

kubectl get pods -n app -l app=web-app
# No resources found in app namespace.

# Check if pods exist with different labels
kubectl get pods -n app --show-labels
# NAME                      READY   STATUS    RESTARTS   AGE   LABELS
# web-app-66bff467f8-8p7vx  1/1     Running   0          1h    app=webapp

# Fix the service selector or pod labels
kubectl patch service web-app -n app -p '{"spec":{"selector":{"app":"webapp"}}}'
# or
kubectl label pods -n app -l app=webapp app=web-app --overwrite

# Verify endpoints after fix
kubectl get endpoints web-app -n app
# NAME     ENDPOINTS         AGE
# web-app  10.244.1.2:80     1h

# Test connectivity
kubectl exec -n network-troubleshooting network-test -- curl -s --connect-timeout 5 web-app.app.svc.cluster.local
```

### Scenario D: CNI Plugin Issues

```bash
# Identify CNI plugin issues
kubectl get pods -n kube-system -l k8s-app=calico-node
# NAME                READY   STATUS    RESTARTS   AGE
# calico-node-5t7zx   0/1     Error     5          1h
# calico-node-8p7vx   0/1     Error     5          1h

# Check CNI plugin logs
kubectl logs -n kube-system calico-node-5t7zx

# Check CNI configuration on nodes (if possible)
# ssh <node-ip>
# ls -la /etc/cni/net.d/
# cat /etc/cni/net.d/10-calico.conflist

# Restart CNI plugin pods
kubectl delete pod -n kube-system -l k8s-app=calico-node

# Verify CNI plugin status after restart
kubectl get pods -n kube-system -l k8s-app=calico-node
# NAME                READY   STATUS    RESTARTS   AGE
# calico-node-6u8zy   1/1     Running   0          1m
# calico-node-9q8wx   1/1     Running   0          1m

# Test pod connectivity
kubectl exec -n network-troubleshooting network-test -- ping -c 3 <pod-ip>
```

## Verification

A successful solution will show:

1. DNS resolution works correctly:
   ```
   kubectl exec -n network-troubleshooting network-test -- nslookup kubernetes.default.svc.cluster.local
   Server:    10.96.0.10
   Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

   Name:      kubernetes.default.svc.cluster.local
   Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local
   ```

2. Network policies allow necessary traffic:
   ```
   kubectl exec -n network-troubleshooting network-test -- curl -s web-app.app.svc.cluster.local
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   ...
   ```

3. Services have endpoints:
   ```
   kubectl get endpoints web-app -n app
   NAME     ENDPOINTS         AGE
   web-app  10.244.1.2:80     1h
   ```

4. Pods can communicate with each other:
   ```
   kubectl exec -n network-troubleshooting network-test -- ping -c 3 10.244.1.2
   PING 10.244.1.2 (10.244.1.2): 56 data bytes
   64 bytes from 10.244.1.2: seq=0 ttl=64 time=0.069 ms
   64 bytes from 10.244.1.2: seq=1 ttl=64 time=0.074 ms
   64 bytes from 10.244.1.2: seq=2 ttl=64 time=0.076 ms
   ```

5. CNI plugin is running correctly:
   ```
   kubectl get pods -n kube-system -l k8s-app=calico-node
   NAME                READY   STATUS    RESTARTS   AGE
   calico-node-6u8zy   1/1     Running   0          10m
   calico-node-9q8wx   1/1     Running   0          10m
   ```

## Common Issues and Troubleshooting

1. **DNS Resolution Failures**:
   - Check CoreDNS pods and logs
   - Verify CoreDNS configuration
   - Check if kube-dns service is running
   - Check if pod DNS configuration is correct

2. **Network Policy Issues**:
   - Check if network policies are blocking traffic
   - Verify network policy selectors match pod labels
   - Test with temporary policies to isolate issues

3. **Service Connectivity Issues**:
   - Check if service has endpoints
   - Verify service selector matches pod labels
   - Check if pods are running and ready
   - Test direct pod-to-pod communication

4. **CNI Plugin Issues**:
   - Check CNI plugin pods and logs
   - Verify CNI configuration on nodes
   - Restart CNI plugin pods if necessary
   - Check node network interfaces and routing

5. **Node Network Issues**:
   - Check node network configuration
   - Verify kube-proxy is running on all nodes
   - Check if nodes can communicate with each other
   - Check if nodes can reach the control plane

## Notes for CKA Exam

- In the CKA exam, you might be asked to troubleshoot various networking issues.
- Focus on using diagnostic tools like `nslookup`, `ping`, `curl`, and `traceroute`.
- Be familiar with common networking components in Kubernetes: CoreDNS, kube-proxy, and CNI plugins.
- Know how to check and modify network policies.
- Understand service and endpoint relationships.
- Be methodical in your troubleshooting approach: identify the issue, understand the root cause, apply a fix, and verify the solution.

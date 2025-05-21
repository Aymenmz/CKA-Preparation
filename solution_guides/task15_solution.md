# Solution Guide: Control Plane Troubleshooting

## Task 15: Control Plane Troubleshooting

**Scenario**: The Kubernetes control plane is experiencing issues affecting cluster operations.

**Task**: Diagnose and resolve problems with control plane components such as etcd, kube-apiserver, kube-scheduler, or kube-controller-manager.

## Solution Steps

### 1. Check Control Plane Component Status

```bash
# Check the status of control plane pods
kubectl get pods -n kube-system

# Focus on control plane components
kubectl get pods -n kube-system -l tier=control-plane

# Check the status of static pods on the control plane node
# (If you have SSH access to the control plane node)
ssh <control-plane-node>
ls -la /etc/kubernetes/manifests/
```

### 2. Check Control Plane Component Logs

```bash
# Check kube-apiserver logs
kubectl logs -n kube-system kube-apiserver-<control-plane-node-name>

# Check kube-controller-manager logs
kubectl logs -n kube-system kube-controller-manager-<control-plane-node-name>

# Check kube-scheduler logs
kubectl logs -n kube-system kube-scheduler-<control-plane-node-name>

# Check etcd logs
kubectl logs -n kube-system etcd-<control-plane-node-name>
```

### 3. Check Control Plane Component Health

```bash
# Check API server health
curl -k https://localhost:6443/healthz

# Check etcd health
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check etcd member list
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

### 4. Common Issues and Solutions

#### Issue 1: API Server Not Starting

```bash
# Check API server manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Check for syntax errors or misconfigurations
# Common issues include:
# - Incorrect certificate paths
# - Incorrect etcd endpoints
# - Incorrect service cluster IP range

# Fix the manifest file
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Make necessary corrections

# Kubelet will automatically restart the static pod
# Wait a moment and check if the API server is running
kubectl get pods -n kube-system -l component=kube-apiserver
```

#### Issue 2: etcd Cluster Issues

```bash
# Check etcd manifest
cat /etc/kubernetes/manifests/etcd.yaml

# Check etcd data directory
ls -la /var/lib/etcd

# Check etcd data directory permissions
ls -la /var/lib/etcd/member

# If etcd data is corrupted, you may need to restore from backup
# Assuming you have an etcd backup:
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot restore /path/to/etcd-backup.db \
  --data-dir=/var/lib/etcd-backup

# Update etcd manifest to use the restored data directory
vi /etc/kubernetes/manifests/etcd.yaml
# Change the volume mount path or move the restored data
```

#### Issue 3: Controller Manager or Scheduler Issues

```bash
# Check controller manager manifest
cat /etc/kubernetes/manifests/kube-controller-manager.yaml

# Check scheduler manifest
cat /etc/kubernetes/manifests/kube-scheduler.yaml

# Common issues include:
# - Incorrect leader election settings
# - Incorrect kubeconfig paths
# - Incorrect certificate paths

# Fix the manifest files
vi /etc/kubernetes/manifests/kube-controller-manager.yaml
vi /etc/kubernetes/manifests/kube-scheduler.yaml
# Make necessary corrections

# Kubelet will automatically restart the static pods
```

#### Issue 4: Certificate Issues

```bash
# Check certificate expiration
kubeadm certs check-expiration

# If certificates are expired, renew them
kubeadm certs renew all

# Restart control plane components after certificate renewal
# Kubelet will automatically restart static pods if manifests are unchanged
# You can force a restart by moving the manifests out and back in:
mkdir -p /tmp/kube-manifests
mv /etc/kubernetes/manifests/*.yaml /tmp/kube-manifests/
sleep 60
mv /tmp/kube-manifests/*.yaml /etc/kubernetes/manifests/
```

#### Issue 5: Kubelet Issues on Control Plane Node

```bash
# Check kubelet status
systemctl status kubelet

# Check kubelet logs
journalctl -u kubelet

# If kubelet is not running, start it
systemctl start kubelet

# If kubelet has configuration issues, check its configuration
cat /var/lib/kubelet/config.yaml

# Check kubelet client certificate
ls -la /var/lib/kubelet/pki/

# Restart kubelet after fixing issues
systemctl restart kubelet
```

### 5. Verify the Fix

```bash
# Check if all control plane components are running
kubectl get pods -n kube-system -l tier=control-plane

# Check the overall cluster status
kubectl get nodes
kubectl get componentstatuses

# Test API server functionality
kubectl get namespaces

# Test controller functionality by creating a deployment
kubectl create deployment nginx --image=nginx

# Test scheduler functionality by checking if pods are scheduled
kubectl get pods
```

## Specific Troubleshooting Scenarios

### Scenario A: API Server Certificate Issues

```bash
# Identify certificate issues in API server logs
kubectl logs -n kube-system kube-apiserver-<control-plane-node-name>
# Error: failed to run Kubelet: unable to load client CA file /etc/kubernetes/pki/ca.crt: open /etc/kubernetes/pki/ca.crt: no such file or directory

# Check if the certificate file exists
ls -la /etc/kubernetes/pki/ca.crt

# If the certificate is missing, you may need to restore it from backup
# or recreate it using kubeadm

# If using kubeadm, you can try to renew certificates
kubeadm certs renew all

# Check the API server manifest for correct certificate paths
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Ensure all certificate paths are correct:
# - --client-ca-file=/etc/kubernetes/pki/ca.crt
# - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
# - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
# etc.

# After fixing, kubelet will automatically restart the API server
# Wait a moment and check if the API server is running
kubectl get pods -n kube-system -l component=kube-apiserver
```

### Scenario B: etcd Data Corruption

```bash
# Identify etcd data corruption in etcd logs
kubectl logs -n kube-system etcd-<control-plane-node-name>
# Error: database file is corrupt

# Check etcd data directory
ls -la /var/lib/etcd

# If you have an etcd backup, restore it
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot restore /path/to/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# Update etcd manifest to use the restored data directory
vi /etc/kubernetes/manifests/etcd.yaml
# Change the hostPath for the data volume:
# volumes:
# - hostPath:
#     path: /var/lib/etcd-restored
#     type: DirectoryOrCreate
#   name: etcd-data

# Kubelet will automatically restart etcd with the new data directory
# Wait a moment and check if etcd is running
kubectl get pods -n kube-system -l component=etcd
```

### Scenario C: Scheduler Configuration Issues

```bash
# Identify scheduler issues in scheduler logs
kubectl logs -n kube-system kube-scheduler-<control-plane-node-name>
# Error: unable to read client config file: invalid configuration: no configuration has been provided

# Check scheduler manifest
cat /etc/kubernetes/manifests/kube-scheduler.yaml

# Check if the kubeconfig file exists
ls -la /etc/kubernetes/scheduler.conf

# Fix the scheduler manifest
vi /etc/kubernetes/manifests/kube-scheduler.yaml
# Ensure the kubeconfig path is correct:
# - --kubeconfig=/etc/kubernetes/scheduler.conf

# Kubelet will automatically restart the scheduler
# Wait a moment and check if the scheduler is running
kubectl get pods -n kube-system -l component=kube-scheduler
```

### Scenario D: Controller Manager Leader Election Issues

```bash
# Identify controller manager issues in logs
kubectl logs -n kube-system kube-controller-manager-<control-plane-node-name>
# Error: unable to elect a leader

# Check controller manager manifest
cat /etc/kubernetes/manifests/kube-controller-manager.yaml

# Check if the lease lock is working properly
kubectl get lease -n kube-system

# Fix the controller manager manifest
vi /etc/kubernetes/manifests/kube-controller-manager.yaml
# Ensure leader election settings are correct:
# - --leader-elect=true
# - --leader-elect-lease-duration=15s
# - --leader-elect-renew-deadline=10s
# - --leader-elect-retry-period=2s

# Kubelet will automatically restart the controller manager
# Wait a moment and check if the controller manager is running
kubectl get pods -n kube-system -l component=kube-controller-manager
```

## Verification

A successful solution will show:

1. All control plane components are running:
   ```
   kubectl get pods -n kube-system -l tier=control-plane
   NAME                                      READY   STATUS    RESTARTS   AGE
   etcd-control-plane                        1/1     Running   0          10m
   kube-apiserver-control-plane              1/1     Running   0          10m
   kube-controller-manager-control-plane     1/1     Running   0          10m
   kube-scheduler-control-plane              1/1     Running   0          10m
   ```

2. The cluster is functioning properly:
   ```
   kubectl get nodes
   NAME            STATUS   ROLES           AGE   VERSION
   control-plane   Ready    control-plane   1h    v1.28.0
   worker-1        Ready    <none>          1h    v1.28.0
   worker-2        Ready    <none>          1h    v1.28.0
   ```

3. API server is responding to requests:
   ```
   kubectl get namespaces
   NAME              STATUS   AGE
   default           Active   1h
   kube-node-lease   Active   1h
   kube-public       Active   1h
   kube-system       Active   1h
   ```

4. etcd is healthy:
   ```
   ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key \
     endpoint health
   https://127.0.0.1:2379 is healthy: successfully committed proposal: took = 9.867ms
   ```

## Common Issues and Troubleshooting

1. **API Server Not Starting**:
   - Check certificate paths and permissions
   - Verify etcd endpoints
   - Check service cluster IP range
   - Look for syntax errors in the manifest

2. **etcd Issues**:
   - Check data directory permissions
   - Look for data corruption
   - Verify certificate paths
   - Check etcd member list for quorum issues

3. **Controller Manager or Scheduler Issues**:
   - Check kubeconfig paths
   - Verify certificate paths
   - Check leader election settings
   - Look for syntax errors in the manifests

4. **Certificate Issues**:
   - Check certificate expiration
   - Verify certificate paths
   - Ensure certificates are properly signed
   - Renew certificates if needed

5. **Kubelet Issues**:
   - Check kubelet status and logs
   - Verify kubelet configuration
   - Check kubelet client certificate
   - Ensure kubelet can access the API server

## Notes for CKA Exam

- In the CKA exam, you might be asked to troubleshoot control plane issues on a cluster.
- Focus on checking logs and manifest files for common issues.
- Be familiar with the structure of control plane component manifests.
- Know how to check and interpret component logs.
- Understand the relationships between control plane components.
- Be methodical in your troubleshooting approach: identify the issue, understand the root cause, apply a fix, and verify the solution.
- Remember that most control plane components run as static pods managed by kubelet, so fixing their manifests is often the solution.

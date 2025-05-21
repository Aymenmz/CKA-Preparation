# Solution Guide: Troubleshooting Node Issues

## Task 12: Node Troubleshooting

**Scenario**: A worker node in the TechNova cluster is reporting NotReady status.

**Task**: Diagnose and fix issues with the worker node, which may include kubelet problems, certificate issues, or networking configuration.

## Solution Steps

### 1. Identify the Problem Node

```bash
# Check the status of all nodes
kubectl get nodes

# Get more detailed information about the problematic node
kubectl describe node <node-name>
```

Look for error messages, conditions, and events that might indicate the cause of the NotReady status.

### 2. Common Issues and Their Solutions

#### Issue 1: Kubelet Service Not Running

```bash
# SSH into the problematic node
ssh <node-name>

# Check kubelet service status
systemctl status kubelet

# If kubelet is not running, start it
sudo systemctl start kubelet

# Enable kubelet to start on boot
sudo systemctl enable kubelet

# Check kubelet logs for errors
sudo journalctl -u kubelet -n 100 --no-pager
```

#### Issue 2: Kubelet Configuration Issues

```bash
# Check kubelet configuration
sudo cat /var/lib/kubelet/config.yaml

# Verify kubelet arguments
ps aux | grep kubelet

# Check for configuration errors in kubelet logs
sudo journalctl -u kubelet | grep -i error
```

#### Issue 3: Certificate Issues

```bash
# Check certificate expiration
sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# If certificates are expired or invalid, renew them
sudo kubeadm alpha certs renew all

# Or manually recreate kubelet certificates
sudo rm -rf /var/lib/kubelet/pki
sudo systemctl restart kubelet
```

#### Issue 4: Network Connectivity Issues

```bash
# Check network plugin status
ls -la /etc/cni/net.d/

# Verify network connectivity to control plane
ping <control-plane-ip>

# Check if required ports are open
sudo netstat -tulpn | grep kubelet

# Restart network service if needed
sudo systemctl restart networking
```

#### Issue 5: Container Runtime Issues

```bash
# Check container runtime status (Docker/containerd)
sudo systemctl status docker  # If using Docker
sudo systemctl status containerd  # If using containerd

# Restart container runtime if needed
sudo systemctl restart docker  # If using Docker
sudo systemctl restart containerd  # If using containerd

# Check container runtime logs
sudo journalctl -u docker -n 100 --no-pager  # If using Docker
sudo journalctl -u containerd -n 100 --no-pager  # If using containerd
```

#### Issue 6: Resource Constraints

```bash
# Check system resources
free -m
df -h
top -n 1

# Clear disk space if needed
sudo du -sh /var/log/* | sort -hr
sudo journalctl --vacuum-time=1d
```

#### Issue 7: Kubelet API Endpoint Issues

```bash
# Check if kubelet API is accessible
curl -k https://localhost:10250/healthz

# Verify kubelet can reach the API server
sudo cat /etc/kubernetes/kubelet.conf
```

### 3. Specific Troubleshooting Scenarios

#### Scenario A: Kubelet Service Failing Due to Misconfiguration

```bash
# Edit kubelet configuration
sudo vi /var/lib/kubelet/config.yaml

# Fix any syntax errors or incorrect settings

# Restart kubelet
sudo systemctl restart kubelet

# Check status
sudo systemctl status kubelet
```

#### Scenario B: Node with Tainted Network Plugin

```bash
# Remove problematic CNI configuration
sudo rm /etc/cni/net.d/*

# Apply correct CNI configuration
sudo mkdir -p /etc/cni/net.d/
sudo cp /path/to/correct/cni-config.json /etc/cni/net.d/

# Restart kubelet
sudo systemctl restart kubelet
```

#### Scenario C: Node with Pressure Conditions

```bash
# Check node conditions
kubectl describe node <node-name> | grep Conditions -A 10

# Address memory pressure
sudo sync
sudo echo 3 > /proc/sys/vm/drop_caches

# Address disk pressure
sudo find /var/log -type f -name "*.log" -exec truncate -s 0 {} \;
sudo docker system prune -af  # If using Docker
```

### 4. Verify the Fix

```bash
# Check node status again
kubectl get nodes

# Verify node is Ready
kubectl describe node <node-name> | grep "Ready"

# Check if pods are being scheduled on the node
kubectl get pods --all-namespaces -o wide | grep <node-name>
```

## Verification

A successful solution will show:

1. The node status has changed from NotReady to Ready:
   ```
   kubectl get nodes
   NAME         STATUS   ROLES    AGE   VERSION
   master       Ready    master   1d    v1.28.0
   worker-1     Ready    <none>   1d    v1.28.0
   ```

2. Pods are running on the node:
   ```
   kubectl get pods --all-namespaces -o wide | grep worker-1
   ```

3. No error events for the node:
   ```
   kubectl describe node worker-1 | grep -A 10 Events
   ```

## Common Issues and Troubleshooting

1. **Persistent NotReady Status**: If the node remains NotReady after fixes, try rebooting the node
2. **Kubelet Crashes**: Check for compatibility issues between kubelet and container runtime versions
3. **Network Plugin Errors**: Ensure the correct CNI plugin is installed and configured
4. **API Server Communication**: Verify the node can reach the API server on port 6443

## Notes for CKA Exam

- In the CKA exam, you might be given specific error messages or symptoms to diagnose
- Focus on checking logs and status first before making changes
- Remember the common locations for Kubernetes configuration files
- Use `kubectl describe node` to get detailed information about node status
- Document your troubleshooting steps as you go, in case you need to revert changes

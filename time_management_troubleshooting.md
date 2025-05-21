# CKA Practice Exam - Time Management and Troubleshooting Tips

## Time Management Strategies

### Before the Exam

1. **Practice with a Timer**: Get comfortable working under time pressure by practicing with a timer set to 2 hours.

2. **Know the Documentation**: Familiarize yourself with the Kubernetes documentation structure so you can quickly find information during the exam.

3. **Master kubectl Commands**: Practice common kubectl commands until they become second nature, reducing the time spent looking up syntax.

4. **Create Aliases and Shortcuts**: Set up helpful aliases and kubectl autocompletion to save time:
   ```bash
   alias k=kubectl
   source <(kubectl completion bash)
   complete -o default -F __start_kubectl k
   ```

### During the Exam

1. **Quick Assessment**: Spend the first 5 minutes scanning all questions to identify:
   - Easy tasks you can complete quickly
   - Complex tasks that will require more time
   - Tasks from your strongest domains

2. **Strategic Task Order**:
   - Start with tasks you're confident in to build momentum
   - Tackle high-point tasks early when your mind is fresh
   - Don't spend more than 10-15 minutes on any single task initially

3. **Time Allocation**:
   - Allocate approximately 8 minutes per task on average
   - Reserve 15-20 minutes at the end for review
   - Set mental checkpoints (e.g., "By 1 hour, I should have completed 7-8 tasks")

4. **Use Imperative Commands**: Save time by using imperative kubectl commands when appropriate:
   ```bash
   # Instead of writing YAML from scratch
   kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > nginx-deployment.yaml
   ```

5. **Flag and Move On**: If you get stuck on a task:
   - Make a note of it
   - Move on to the next task
   - Return to it if time permits

6. **Partial Points**: Remember that partial solutions can earn partial points. If you can't complete a task fully, implement what you can.

## Common Troubleshooting Approaches

### General Troubleshooting Framework

1. **Identify**: Clearly define what's not working
2. **Gather Information**: Collect relevant data about the issue
3. **Analyze**: Determine the most likely cause
4. **Resolve**: Implement a solution
5. **Verify**: Confirm the issue is resolved

### Cluster-Level Issues

1. **Check Node Status**:
   ```bash
   kubectl get nodes
   kubectl describe node <node-name>
   ```

2. **Verify Control Plane Components**:
   ```bash
   kubectl get pods -n kube-system
   kubectl logs -n kube-system <pod-name>
   ```

3. **Check API Server Availability**:
   ```bash
   kubectl get --raw /healthz
   kubectl get --raw /readyz
   ```

4. **Examine etcd**:
   ```bash
   kubectl -n kube-system logs etcd-<master-node-name>
   ```

### Pod and Application Issues

1. **Check Pod Status**:
   ```bash
   kubectl get pods -o wide
   kubectl describe pod <pod-name>
   ```

2. **Examine Pod Logs**:
   ```bash
   kubectl logs <pod-name>
   kubectl logs <pod-name> -c <container-name>  # For multi-container pods
   kubectl logs <pod-name> --previous  # For crashed containers
   ```

3. **Debug with Temporary Pods**:
   ```bash
   kubectl run debug --image=busybox --rm -it -- sh
   ```

4. **Check Events**:
   ```bash
   kubectl get events --sort-by='.lastTimestamp'
   ```

### Networking Issues

1. **Verify Service Endpoints**:
   ```bash
   kubectl get endpoints <service-name>
   ```

2. **Test Network Connectivity**:
   ```bash
   # From a debug pod
   kubectl run debug --image=busybox --rm -it -- sh
   # Then inside the pod
   wget -O- <service-name>:<port>
   nslookup <service-name>
   ```

3. **Check Network Policies**:
   ```bash
   kubectl get networkpolicies
   kubectl describe networkpolicy <policy-name>
   ```

4. **Examine DNS Configuration**:
   ```bash
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   kubectl logs -n kube-system -l k8s-app=kube-dns
   ```

### Storage Issues

1. **Check PV and PVC Status**:
   ```bash
   kubectl get pv,pvc
   kubectl describe pv <pv-name>
   kubectl describe pvc <pvc-name>
   ```

2. **Verify StorageClass**:
   ```bash
   kubectl get storageclass
   kubectl describe storageclass <storageclass-name>
   ```

3. **Check Pod Volume Mounts**:
   ```bash
   kubectl describe pod <pod-name> | grep -A 10 Volumes
   ```

### Common Error Patterns and Solutions

1. **ImagePullBackOff**:
   - Verify image name and tag
   - Check registry access and credentials
   - Use `kubectl describe pod` to see detailed error messages

2. **CrashLoopBackOff**:
   - Check container logs with `kubectl logs`
   - Verify command and arguments
   - Check resource constraints

3. **Pending Pods**:
   - Check node resources with `kubectl describe node`
   - Verify PVC binding if pod uses persistent storage
   - Check for node affinity/taints that might prevent scheduling

4. **Service Connectivity Issues**:
   - Verify service selector matches pod labels
   - Check endpoints with `kubectl get endpoints`
   - Test connectivity from within the cluster

5. **Permission Denied Errors**:
   - Check RBAC configuration
   - Verify service account permissions
   - Use `kubectl auth can-i` to test permissions

Remember that systematic troubleshooting is key. Start with the symptoms, gather information, form a hypothesis, test it, and verify the solution works.

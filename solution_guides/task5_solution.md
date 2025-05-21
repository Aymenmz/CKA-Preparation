# Solution Guide: Deployment Management

## Task 5: Deployment Management

**Scenario**: TechNova is updating their web application and needs to ensure zero downtime.

**Task**: Create a deployment with multiple replicas, perform a rolling update to a new version, and then roll back to the previous version when issues are detected.

## Solution Steps

### 1. Create a Deployment with Multiple Replicas

```bash
# Create a namespace for the application
kubectl create namespace webapp

# Create a deployment with 3 replicas
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: webapp
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.20
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 15
EOF

# Create a service to expose the deployment
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp
  namespace: webapp
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

### 2. Verify the Deployment

```bash
# Check if the deployment was created successfully
kubectl get deployment -n webapp

# Check if all pods are running
kubectl get pods -n webapp

# Check the deployment details
kubectl describe deployment webapp -n webapp

# Check the service
kubectl get service -n webapp
```

### 3. Perform a Rolling Update

```bash
# Update the deployment to a new version
kubectl set image deployment/webapp webapp=nginx:1.21 -n webapp

# Alternatively, you can edit the deployment
# kubectl edit deployment webapp -n webapp
# Then change the image from nginx:1.20 to nginx:1.21

# Monitor the rolling update
kubectl rollout status deployment/webapp -n webapp
```

### 4. Verify the Update

```bash
# Check the deployment to confirm the new image is being used
kubectl describe deployment webapp -n webapp | grep Image

# Check the pods to see if they're using the new image
kubectl get pods -n webapp -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

### 5. Simulate an Issue with the New Version

Let's assume the new version has issues and we need to roll back:

```bash
# For demonstration purposes, let's update to a non-existent image to simulate a problem
kubectl set image deployment/webapp webapp=nginx:1.21-broken -n webapp

# Check the status of the rollout
kubectl rollout status deployment/webapp -n webapp

# You should see some pods failing to start
kubectl get pods -n webapp
```

### 6. Roll Back to the Previous Version

```bash
# Roll back to the previous version
kubectl rollout undo deployment/webapp -n webapp

# Check the status of the rollback
kubectl rollout status deployment/webapp -n webapp

# Verify the rollback
kubectl describe deployment webapp -n webapp | grep Image

# Check the pods to confirm they're using the previous image
kubectl get pods -n webapp -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
```

### 7. View Rollout History

```bash
# View the rollout history
kubectl rollout history deployment/webapp -n webapp

# View details of a specific revision
kubectl rollout history deployment/webapp -n webapp --revision=2
```

### 8. Roll Back to a Specific Revision (Optional)

```bash
# Roll back to a specific revision
kubectl rollout undo deployment/webapp -n webapp --to-revision=1
```

## Verification

A successful solution will show:

1. The deployment is created with 3 replicas:
   ```
   kubectl get deployment webapp -n webapp
   NAME     READY   UP-TO-DATE   AVAILABLE   AGE
   webapp   3/3     3            3           5m
   ```

2. The rolling update is performed successfully:
   ```
   kubectl rollout status deployment/webapp -n webapp
   deployment "webapp" successfully rolled out
   ```

3. After simulating an issue, some pods may be in an error state:
   ```
   kubectl get pods -n webapp
   NAME                      READY   STATUS             RESTARTS   AGE
   webapp-7d9fd8b5b5-2xqnz   1/1     Running           0          2m
   webapp-7d9fd8b5b5-5hm2l   1/1     Running           0          2m
   webapp-84b87c7d6d-jkl2p   0/1     ImagePullBackOff  0          30s
   ```

4. After rollback, all pods are running with the previous version:
   ```
   kubectl get pods -n webapp -o custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
   NAME                      IMAGE
   webapp-7d9fd8b5b5-2xqnz   nginx:1.20
   webapp-7d9fd8b5b5-5hm2l   nginx:1.20
   webapp-7d9fd8b5b5-9kl3p   nginx:1.20
   ```

## Common Issues and Troubleshooting

1. **Pods Stuck in Pending State**: Check for resource constraints or node issues.

2. **ImagePullBackOff**: Verify the image name and tag are correct, and that the image exists in the registry.

3. **Readiness Probe Failures**: Ensure the application is properly responding to the readiness probe.

4. **Rollout Stuck**: Check for issues with the new pods starting up or the old pods terminating.

5. **Service Not Routing Traffic**: Verify the service selector matches the pod labels.

## Notes for CKA Exam

- In the CKA exam, you might be asked to perform rolling updates and rollbacks as part of larger scenarios.
- Focus on understanding the deployment update strategies (RollingUpdate vs. Recreate).
- Be familiar with the kubectl commands for managing deployments and rollouts.
- Remember that rollbacks are performed to the previous revision by default, but you can specify a revision.
- Understand how to check the status of a rollout and view the rollout history.
- Know how to troubleshoot common issues with deployments and updates.

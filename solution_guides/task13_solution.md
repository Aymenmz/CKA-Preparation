# Solution Guide: Pod and Deployment Troubleshooting

## Task 13: Pod and Deployment Troubleshooting

**Scenario**: A critical application deployment is failing to start properly.

**Task**: Investigate and resolve issues with pods that are in CrashLoopBackOff or Error states, which may include configuration problems, resource constraints, or image issues.

## Solution Steps

### 1. Identify the Problematic Pods

```bash
# Check all pods across all namespaces to find problematic ones
kubectl get pods --all-namespaces

# Focus on the application namespace
kubectl get pods -n application
```

### 2. Examine the Pod Details

```bash
# Get detailed information about the problematic pod
kubectl describe pod <pod-name> -n application

# Check the pod's events section for error messages
kubectl describe pod <pod-name> -n application | grep -A 20 Events:
```

### 3. Check Pod Logs

```bash
# Check the logs of the problematic pod
kubectl logs <pod-name> -n application

# If the pod has multiple containers, specify the container
kubectl logs <pod-name> -c <container-name> -n application

# If the pod is restarting, check the previous container's logs
kubectl logs <pod-name> -n application --previous
```

### 4. Common Issues and Solutions

#### Issue 1: Image Pull Errors (ImagePullBackOff)

```bash
# Check the image name and repository
kubectl describe pod <pod-name> -n application | grep Image:

# Correct the image name in the deployment
kubectl edit deployment <deployment-name> -n application
# Update the image name to a valid one

# Alternatively, patch the deployment
kubectl patch deployment <deployment-name> -n application --patch '{"spec": {"template": {"spec": {"containers": [{"name": "<container-name>", "image": "correct-image:tag"}]}}}}'
```

#### Issue 2: Application Crashes (CrashLoopBackOff)

```bash
# Check the application logs for error messages
kubectl logs <pod-name> -n application

# If the application is failing due to missing configuration, create or update ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: application
data:
  config.properties: |
    app.mode=production
    app.log.level=info
EOF

# Update the deployment to use the ConfigMap
kubectl edit deployment <deployment-name> -n application
# Add the ConfigMap volume and volume mount
```

#### Issue 3: Resource Constraints

```bash
# Check if the pod is being terminated due to resource constraints
kubectl describe pod <pod-name> -n application | grep -A 10 "Last State"

# Check node resource usage
kubectl describe node <node-name>

# Update the deployment with appropriate resource requests and limits
kubectl edit deployment <deployment-name> -n application
# Adjust the resources section:
# resources:
#   requests:
#     memory: "128Mi"
#     cpu: "100m"
#   limits:
#     memory: "256Mi"
#     cpu: "200m"
```

#### Issue 4: Liveness/Readiness Probe Failures

```bash
# Check if probes are failing
kubectl describe pod <pod-name> -n application | grep -A 10 "Liveness\|Readiness"

# Update the deployment with correct probe configuration
kubectl edit deployment <deployment-name> -n application
# Adjust the probe configuration:
# livenessProbe:
#   httpGet:
#     path: /health
#     port: 8080
#   initialDelaySeconds: 30
#   periodSeconds: 10
```

#### Issue 5: Volume Mount Issues

```bash
# Check if there are volume mount issues
kubectl describe pod <pod-name> -n application | grep -A 20 "Volumes"

# Check if the PersistentVolumeClaim exists and is bound
kubectl get pvc -n application

# Create a missing PVC if needed
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: application
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

#### Issue 6: Environment Variable Issues

```bash
# Check environment variables
kubectl describe pod <pod-name> -n application | grep -A 20 "Environment"

# Update the deployment with correct environment variables
kubectl edit deployment <deployment-name> -n application
# Adjust the env section:
# env:
# - name: DATABASE_URL
#   value: "postgres://user:password@db:5432/app"
```

#### Issue 7: Command or Arguments Issues

```bash
# Check the command and arguments
kubectl describe pod <pod-name> -n application | grep -A 10 "Command\|Args"

# Update the deployment with correct command and arguments
kubectl edit deployment <deployment-name> -n application
# Adjust the command and args sections:
# command: ["/bin/sh"]
# args: ["-c", "while true; do echo hello; sleep 10; done"]
```

### 5. Verify the Fix

```bash
# Check if the pod is now running
kubectl get pod <pod-name> -n application

# If using a deployment, check if the new pods are created and running
kubectl get deployment <deployment-name> -n application

# Check the logs to ensure the application is running correctly
kubectl logs <pod-name> -n application
```

### 6. Scale the Deployment (if needed)

```bash
# Scale the deployment to the desired number of replicas
kubectl scale deployment <deployment-name> -n application --replicas=3

# Verify the scaling
kubectl get deployment <deployment-name> -n application
```

## Specific Troubleshooting Scenarios

### Scenario A: Application Failing Due to Missing Configuration

```bash
# Identify the deployment with issues
kubectl get deployments -n application
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# web-app      0/3     3            0           10m

# Check the pods
kubectl get pods -n application
# NAME                      READY   STATUS             RESTARTS   AGE
# web-app-7d9fd8b5b5-2xqnz  0/1     CrashLoopBackOff   5          10m
# web-app-7d9fd8b5b5-5hm2l  0/1     CrashLoopBackOff   5          10m
# web-app-7d9fd8b5b5-9kl3p  0/1     CrashLoopBackOff   5          10m

# Check the logs
kubectl logs web-app-7d9fd8b5b5-2xqnz -n application
# Error: Configuration file not found at /app/config.json

# Create the missing ConfigMap
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
  namespace: application
data:
  config.json: |
    {
      "database": {
        "host": "db.application.svc.cluster.local",
        "port": 5432,
        "user": "app",
        "password": "password"
      },
      "server": {
        "port": 8080,
        "logLevel": "info"
      }
    }
EOF

# Update the deployment to use the ConfigMap
kubectl patch deployment web-app -n application --patch '
{
  "spec": {
    "template": {
      "spec": {
        "volumes": [
          {
            "name": "config-volume",
            "configMap": {
              "name": "web-app-config"
            }
          }
        ],
        "containers": [
          {
            "name": "web-app",
            "volumeMounts": [
              {
                "name": "config-volume",
                "mountPath": "/app"
              }
            ]
          }
        ]
      }
    }
  }
}'

# Verify the fix
kubectl get pods -n application
# NAME                      READY   STATUS    RESTARTS   AGE
# web-app-8f9fd7c6c6-3yqmz  1/1     Running   0          1m
# web-app-8f9fd7c6c6-6in3k  1/1     Running   0          1m
# web-app-8f9fd7c6c6-8lm4q  1/1     Running   0          1m
```

### Scenario B: Application Failing Due to Resource Constraints

```bash
# Identify the deployment with issues
kubectl get deployments -n application
# NAME         READY   UP-TO-DATE   AVAILABLE   AGE
# api-server   0/2     2            0           5m

# Check the pods
kubectl get pods -n application
# NAME                        READY   STATUS       RESTARTS   AGE
# api-server-6d8fd7b4b4-1xpny 0/1     OOMKilled    3          5m
# api-server-6d8fd7b4b4-7jm2k 0/1     OOMKilled    3          5m

# Check the pod details
kubectl describe pod api-server-6d8fd7b4b4-1xpny -n application
# ...
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137
#   Started:      Wed, 21 May 2025 10:10:00 +0000
#   Finished:     Wed, 21 May 2025 10:10:30 +0000
# ...

# Update the deployment with increased memory limits
kubectl patch deployment api-server -n application --patch '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "api-server",
            "resources": {
              "requests": {
                "memory": "256Mi",
                "cpu": "200m"
              },
              "limits": {
                "memory": "512Mi",
                "cpu": "500m"
              }
            }
          }
        ]
      }
    }
  }
}'

# Verify the fix
kubectl get pods -n application
# NAME                        READY   STATUS    RESTARTS   AGE
# api-server-7e9fd8c7c7-2zqnz 1/1     Running   0          1m
# api-server-7e9fd8c7c7-8km3p 1/1     Running   0          1m
```

## Verification

A successful solution will show:

1. All pods are running:
   ```
   kubectl get pods -n application
   NAME                      READY   STATUS    RESTARTS   AGE
   web-app-8f9fd7c6c6-3yqmz  1/1     Running   0          10m
   web-app-8f9fd7c6c6-6in3k  1/1     Running   0          10m
   web-app-8f9fd7c6c6-8lm4q  1/1     Running   0          10m
   api-server-7e9fd8c7c7-2zqnz 1/1   Running   0          5m
   api-server-7e9fd8c7c7-8km3p 1/1   Running   0          5m
   ```

2. Deployments show the correct number of available replicas:
   ```
   kubectl get deployments -n application
   NAME         READY   UP-TO-DATE   AVAILABLE   AGE
   web-app      3/3     3            3           20m
   api-server   2/2     2            2           15m
   ```

3. Pod logs show the application is running correctly:
   ```
   kubectl logs web-app-8f9fd7c6c6-3yqmz -n application
   Server started on port 8080
   Connected to database at db.application.svc.cluster.local:5432
   ```

## Common Issues and Troubleshooting

1. **ImagePullBackOff**: Check image name, registry access, and credentials.

2. **CrashLoopBackOff**: Check application logs, configuration, and environment variables.

3. **OOMKilled**: Increase memory limits or optimize the application.

4. **ContainerCreating**: Check volume mounts, PVCs, and node resources.

5. **Error**: Check the pod events and logs for specific error messages.

## Notes for CKA Exam

- In the CKA exam, you might be asked to troubleshoot various pod and deployment issues.
- Focus on using `kubectl describe` and `kubectl logs` to identify the root cause.
- Be familiar with common pod failure modes and their solutions.
- Know how to update deployments using `kubectl edit`, `kubectl patch`, or by applying a new YAML file.
- Remember to verify your fixes by checking pod status and logs.
- Be methodical in your troubleshooting approach: identify the issue, understand the root cause, apply a fix, and verify the solution.

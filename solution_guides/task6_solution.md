# Solution Guide: ConfigMaps and Secrets

## Task 6: ConfigMaps and Secrets

**Scenario**: An application requires configuration data and sensitive information.

**Task**: Create ConfigMaps and Secrets for an application, then mount them appropriately in a pod. Update the ConfigMap and verify the changes are reflected in the running pod.

## Solution Steps

### 1. Create a Namespace for the Application

```bash
# Create a namespace
kubectl create namespace config-demo
```

### 2. Create a ConfigMap with Application Configuration

```bash
# Create a ConfigMap with literal values
kubectl create configmap app-config \
  --namespace=config-demo \
  --from-literal=APP_ENV=production \
  --from-literal=APP_DEBUG=false \
  --from-literal=APP_LOG_LEVEL=info

# Alternatively, create a ConfigMap from a file
cat <<EOF > app-config.properties
APP_ENV=production
APP_DEBUG=false
APP_LOG_LEVEL=info
DATABASE_URL=db.example.com
DATABASE_PORT=5432
EOF

kubectl create configmap app-config-file \
  --namespace=config-demo \
  --from-file=app-config.properties
```

### 3. Create a Secret with Sensitive Information

```bash
# Create a Secret with literal values
kubectl create secret generic app-secrets \
  --namespace=config-demo \
  --from-literal=DB_USERNAME=admin \
  --from-literal=DB_PASSWORD=s3cr3t!

# Alternatively, create a Secret from files
echo -n "admin" > ./username.txt
echo -n "s3cr3t!" > ./password.txt

kubectl create secret generic app-secrets-file \
  --namespace=config-demo \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt

# Clean up temporary files
rm ./username.txt ./password.txt
```

### 4. Create a Pod that Uses the ConfigMap and Secret

```bash
# Create a pod that mounts the ConfigMap and Secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-pod
  namespace: config-demo
spec:
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh", "-c", "while true; do echo 'App is running'; sleep 30; done"]
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    - name: APP_DEBUG
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_DEBUG
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DB_USERNAME
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DB_PASSWORD
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: secrets-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config-file
  - name: secrets-volume
    secret:
      secretName: app-secrets-file
EOF
```

### 5. Verify the ConfigMap and Secret are Mounted Correctly

```bash
# Wait for the pod to be running
kubectl wait --for=condition=Ready pod/config-demo-pod -n config-demo --timeout=60s

# Check environment variables in the pod
kubectl exec -n config-demo config-demo-pod -- env | grep APP_
kubectl exec -n config-demo config-demo-pod -- env | grep DB_

# Check mounted ConfigMap as a volume
kubectl exec -n config-demo config-demo-pod -- ls -la /etc/config
kubectl exec -n config-demo config-demo-pod -- cat /etc/config/app-config.properties

# Check mounted Secret as a volume
kubectl exec -n config-demo config-demo-pod -- ls -la /etc/secrets
kubectl exec -n config-demo config-demo-pod -- cat /etc/secrets/username
kubectl exec -n config-demo config-demo-pod -- cat /etc/secrets/password
```

### 6. Update the ConfigMap

```bash
# Update the ConfigMap
kubectl edit configmap app-config -n config-demo
# Change APP_LOG_LEVEL from "info" to "debug"

# Alternatively, update using patch
kubectl patch configmap app-config -n config-demo --patch '{"data":{"APP_LOG_LEVEL":"debug"}}'

# For the file-based ConfigMap, create an updated file
cat <<EOF > app-config-updated.properties
APP_ENV=production
APP_DEBUG=true
APP_LOG_LEVEL=debug
DATABASE_URL=db.example.com
DATABASE_PORT=5432
EOF

# Apply the update
kubectl create configmap app-config-file \
  --namespace=config-demo \
  --from-file=app-config.properties=./app-config-updated.properties \
  -o yaml --dry-run=client | kubectl replace -f -

# Clean up temporary file
rm ./app-config-updated.properties
```

### 7. Verify the ConfigMap Update

```bash
# Check the updated ConfigMap
kubectl get configmap app-config -n config-demo -o yaml

# For environment variables, the pod needs to be restarted to see changes
kubectl delete pod config-demo-pod -n config-demo

# Recreate the pod
# (Use the same pod definition as in step 4)

# For volume mounts, changes will be reflected automatically after some time
# Wait a moment for the ConfigMap volume to update
sleep 30

# Check the updated file
kubectl exec -n config-demo config-demo-pod -- cat /etc/config/app-config.properties
```

### 8. Create a Deployment for Better Update Handling (Optional)

```bash
# Create a deployment instead of a single pod for better update handling
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-demo
  namespace: config-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: config-demo
  template:
    metadata:
      labels:
        app: config-demo
    spec:
      containers:
      - name: app
        image: busybox
        command: ["/bin/sh", "-c", "while true; do echo 'App is running'; sleep 30; done"]
        env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        - name: APP_DEBUG
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_DEBUG
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: app-config-file
EOF
```

## Verification

A successful solution will show:

1. The ConfigMap and Secret are created:
   ```
   kubectl get configmap,secret -n config-demo
   ```

2. The pod is running with the correct environment variables:
   ```
   kubectl exec -n config-demo config-demo-pod -- env | grep APP_
   APP_ENV=production
   APP_DEBUG=false
   ```

3. The ConfigMap is mounted as a volume:
   ```
   kubectl exec -n config-demo config-demo-pod -- cat /etc/config/app-config.properties
   APP_ENV=production
   APP_DEBUG=false
   APP_LOG_LEVEL=info
   DATABASE_URL=db.example.com
   DATABASE_PORT=5432
   ```

4. The Secret is mounted as a volume:
   ```
   kubectl exec -n config-demo config-demo-pod -- cat /etc/secrets/username
   admin
   ```

5. After updating the ConfigMap, the changes are reflected:
   ```
   kubectl exec -n config-demo config-demo-pod -- cat /etc/config/app-config.properties
   APP_ENV=production
   APP_DEBUG=true
   APP_LOG_LEVEL=debug
   DATABASE_URL=db.example.com
   DATABASE_PORT=5432
   ```

## Common Issues and Troubleshooting

1. **ConfigMap or Secret Not Found**: Verify the names match exactly in the pod specification.

2. **Volume Mount Issues**: Check the path and permissions for the mounted volumes.

3. **Environment Variables Not Updated**: Remember that environment variables from ConfigMaps are only set when the pod starts. To see changes, you need to restart the pod.

4. **Volume Updates Delay**: ConfigMap volume updates may take up to a minute to propagate to the pod.

5. **Secret Data Encoding**: Remember that Secret data is base64-encoded in the YAML, but automatically decoded when mounted.

## Notes for CKA Exam

- In the CKA exam, you might be asked to create ConfigMaps and Secrets from various sources (literals, files, etc.).
- Understand the difference between using ConfigMaps/Secrets as environment variables versus volume mounts.
- Remember that environment variables from ConfigMaps/Secrets are only set at pod startup, while volume mounts can be updated.
- Be familiar with the different ways to create and update ConfigMaps and Secrets.
- Know how to verify that ConfigMaps and Secrets are correctly mounted and accessible in pods.

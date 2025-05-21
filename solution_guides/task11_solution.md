# Solution Guide: Dynamic Volume Provisioning

## Task 11: Dynamic Volume Provisioning

**Scenario**: Applications need persistent storage with specific performance characteristics.

**Task**: Create storage classes for different performance tiers, then create PVCs that use these storage classes and mount them in pods.

## Solution Steps

### 1. Create Storage Classes for Different Performance Tiers

```bash
# Create a standard storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
EOF

# Create a high-performance storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-performance
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
parameters:
  type: ssd
EOF

# Create a high-capacity storage class
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-capacity
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
parameters:
  type: hdd
  fsType: xfs
EOF
```

### 2. Create a Namespace for the Storage Demo

```bash
# Create a namespace
kubectl create namespace storage-demo
```

### 3. Create Persistent Volume Claims Using Different Storage Classes

```bash
# Create a PVC using the standard storage class
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: standard-pvc
  namespace: storage-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard
EOF

# Create a PVC using the high-performance storage class
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: high-performance-pvc
  namespace: storage-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: high-performance
EOF

# Create a PVC using the high-capacity storage class
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: high-capacity-pvc
  namespace: storage-demo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: high-capacity
EOF
```

### 4. Create Pods That Mount the PVCs

```bash
# Create a pod using the standard PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: standard-pod
  namespace: storage-demo
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: standard-volume
  volumes:
  - name: standard-volume
    persistentVolumeClaim:
      claimName: standard-pvc
EOF

# Create a pod using the high-performance PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: high-performance-pod
  namespace: storage-demo
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: high-performance-volume
  volumes:
  - name: high-performance-volume
    persistentVolumeClaim:
      claimName: high-performance-pvc
EOF

# Create a pod using the high-capacity PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: high-capacity-pod
  namespace: storage-demo
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: "/usr/share/nginx/html"
      name: high-capacity-volume
  volumes:
  - name: high-capacity-volume
    persistentVolumeClaim:
      claimName: high-capacity-pvc
EOF
```

### 5. Create a StatefulSet with Persistent Storage

```bash
# Create a StatefulSet with persistent storage
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: storage-demo
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "standard"
      resources:
        requests:
          storage: 1Gi
EOF

# Create a headless service for the StatefulSet
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: storage-demo
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
EOF
```

### 6. Verify the Storage Resources

```bash
# Check the storage classes
kubectl get storageclass

# Check the PVCs
kubectl get pvc -n storage-demo

# Check the pods
kubectl get pods -n storage-demo

# Check the StatefulSet
kubectl get statefulset -n storage-demo

# Check the PVs that were dynamically provisioned
kubectl get pv
```

### 7. Test Writing to the Volumes

```bash
# Write data to the standard volume
kubectl exec -n storage-demo standard-pod -- sh -c "echo 'Hello from standard storage' > /usr/share/nginx/html/index.html"

# Write data to the high-performance volume
kubectl exec -n storage-demo high-performance-pod -- sh -c "echo 'Hello from high-performance storage' > /usr/share/nginx/html/index.html"

# Write data to the high-capacity volume
kubectl exec -n storage-demo high-capacity-pod -- sh -c "echo 'Hello from high-capacity storage' > /usr/share/nginx/html/index.html"

# Write data to the StatefulSet volumes
for i in 0 1 2; do
  kubectl exec -n storage-demo web-$i -- sh -c "echo 'Hello from StatefulSet pod $i' > /usr/share/nginx/html/index.html"
done
```

### 8. Verify the Data Persistence

```bash
# Read data from the standard volume
kubectl exec -n storage-demo standard-pod -- cat /usr/share/nginx/html/index.html

# Read data from the high-performance volume
kubectl exec -n storage-demo high-performance-pod -- cat /usr/share/nginx/html/index.html

# Read data from the high-capacity volume
kubectl exec -n storage-demo high-capacity-pod -- cat /usr/share/nginx/html/index.html

# Read data from the StatefulSet volumes
for i in 0 1 2; do
  kubectl exec -n storage-demo web-$i -- cat /usr/share/nginx/html/index.html
done
```

### 9. Demonstrate Volume Expansion (if supported)

```bash
# Check if the storage class supports volume expansion
kubectl get storageclass standard -o jsonpath='{.allowVolumeExpansion}'

# If supported, expand a PVC
kubectl patch pvc standard-pvc -n storage-demo -p '{"spec": {"resources": {"requests": {"storage": "2Gi"}}}}'

# Check the PVC status
kubectl get pvc standard-pvc -n storage-demo
```

## Verification

A successful solution will show:

1. Storage classes are created:
   ```
   kubectl get storageclass
   NAME                        PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
   standard (default)          kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  10m
   high-performance            kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  10m
   high-capacity               kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  10m
   ```

2. PVCs are created and bound:
   ```
   kubectl get pvc -n storage-demo
   NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
   standard-pvc          Bound    pvc-12345678-1234-1234-1234-123456789012   1Gi        RWO            standard           10m
   high-performance-pvc  Bound    pvc-23456789-2345-2345-2345-234567890123   1Gi        RWO            high-performance   10m
   high-capacity-pvc     Bound    pvc-34567890-3456-3456-3456-345678901234   5Gi        RWO            high-capacity      10m
   ```

3. Pods are running with mounted volumes:
   ```
   kubectl get pods -n storage-demo
   NAME                   READY   STATUS    RESTARTS   AGE
   standard-pod           1/1     Running   0          9m
   high-performance-pod   1/1     Running   0          9m
   high-capacity-pod      1/1     Running   0          9m
   web-0                  1/1     Running   0          8m
   web-1                  1/1     Running   0          8m
   web-2                  1/1     Running   0          8m
   ```

4. Data is successfully written and read from the volumes:
   ```
   kubectl exec -n storage-demo standard-pod -- cat /usr/share/nginx/html/index.html
   Hello from standard storage
   ```

## Common Issues and Troubleshooting

1. **PVCs Stuck in Pending State**: This could be due to:
   - No storage provisioner available for the storage class
   - No matching PVs available
   - Volume binding mode set to WaitForFirstConsumer but no pod using the PVC

2. **Pod Stuck in ContainerCreating State**: Check if the PVC is bound:
   ```
   kubectl describe pod <pod-name> -n storage-demo
   ```

3. **Storage Class Not Found**: Verify the storage class name in the PVC:
   ```
   kubectl get storageclass
   kubectl describe pvc <pvc-name> -n storage-demo
   ```

4. **Volume Expansion Not Working**: Check if the storage class supports volume expansion:
   ```
   kubectl get storageclass <storage-class-name> -o jsonpath='{.allowVolumeExpansion}'
   ```

5. **StatefulSet Pods Not Creating PVCs**: Verify the volumeClaimTemplates section in the StatefulSet:
   ```
   kubectl describe statefulset web -n storage-demo
   ```

## Notes for CKA Exam

- In the CKA exam, you might be asked to create storage classes, PVCs, and pods that use persistent storage.
- Understand the difference between static and dynamic provisioning.
- Be familiar with the different volume binding modes (Immediate vs. WaitForFirstConsumer).
- Know how to check the status of PVCs and troubleshoot common issues.
- Understand how StatefulSets use volumeClaimTemplates to create PVCs for each replica.
- Remember that the actual provisioner used in your environment may vary (e.g., AWS EBS, GCE PD, Azure Disk).
- Be aware that some features like volume expansion may not be available in all environments.

# Solution Guide: CRD and Operator Management

## Task 4: CRD and Operator Management

**Scenario**: TechNova is implementing a database service that requires a custom operator.

**Task**: Install a provided database operator, create a Custom Resource Definition, and deploy an instance of the database using the custom resource.

## Solution Steps

### 1. Install the Database Operator

For this task, we'll use a simple database operator. In a real scenario, you might use operators like PostgreSQL, MySQL, or MongoDB operators.

```bash
# Create a namespace for the operator
kubectl create namespace database-system

# Apply the operator CRDs and resources
# First, let's create the CRD
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.db.example.com
spec:
  group: db.example.com
  names:
    kind: Database
    listKind: DatabaseList
    plural: databases
    singular: database
    shortNames:
    - db
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
                enum: ["postgres", "mysql", "mongodb"]
              version:
                type: string
              storageSize:
                type: string
              replicas:
                type: integer
                minimum: 1
              user:
                type: string
              password:
                type: string
            required: ["engine", "version", "storageSize"]
    additionalPrinterColumns:
    - name: Engine
      type: string
      jsonPath: .spec.engine
    - name: Version
      type: string
      jsonPath: .spec.version
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
EOF

# Now, let's create the service account, role, and role binding for the operator
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database-operator
  namespace: database-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: database-operator
rules:
- apiGroups: [""]
  resources: ["pods", "services", "persistentvolumeclaims", "secrets", "configmaps"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["*"]
- apiGroups: ["db.example.com"]
  resources: ["databases"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: database-operator
subjects:
- kind: ServiceAccount
  name: database-operator
  namespace: database-system
roleRef:
  kind: ClusterRole
  name: database-operator
  apiGroup: rbac.authorization.k8s.io
EOF

# Finally, deploy the operator itself
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database-operator
  namespace: database-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database-operator
  template:
    metadata:
      labels:
        app: database-operator
    spec:
      serviceAccountName: database-operator
      containers:
      - name: operator
        image: busybox
        command: ["/bin/sh", "-c", "while true; do echo 'Database operator running...'; sleep 30; done"]
EOF
```

### 2. Verify the Operator Installation

```bash
# Check if the CRD is installed
kubectl get crd databases.db.example.com

# Check if the operator pod is running
kubectl get pods -n database-system
```

### 3. Create a Database Instance Using the Custom Resource

```bash
# Create a namespace for the database
kubectl create namespace application

# Create a secret for database credentials
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: application
type: Opaque
data:
  username: YWRtaW4=  # "admin" in base64
  password: cGFzc3dvcmQxMjM=  # "password123" in base64
EOF

# Create a database instance using the custom resource
cat <<EOF | kubectl apply -f -
apiVersion: db.example.com/v1
kind: Database
metadata:
  name: production-db
  namespace: application
spec:
  engine: postgres
  version: "13.4"
  storageSize: "10Gi"
  replicas: 3
  user: admin
  password: password123
EOF
```

### 4. Verify the Database Instance

```bash
# Check if the database custom resource was created
kubectl get databases -n application

# Get more details about the database instance
kubectl describe database production-db -n application
```

### 5. Simulate the Operator Controller Logic

In a real operator, the controller would watch for Database resources and create the necessary Kubernetes objects. Since we're using a simplified example, let's manually create what the operator would typically create:

```bash
# Create a StatefulSet for the database (this would normally be done by the operator)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: production-db
  namespace: application
  labels:
    app: production-db
    db.example.com/name: production-db
spec:
  serviceName: production-db
  replicas: 3
  selector:
    matchLabels:
      app: production-db
  template:
    metadata:
      labels:
        app: production-db
    spec:
      containers:
      - name: postgres
        image: postgres:13.4
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: production-db
  namespace: application
  labels:
    app: production-db
    db.example.com/name: production-db
spec:
  selector:
    app: production-db
  ports:
  - port: 5432
    targetPort: 5432
    name: postgres
  clusterIP: None
EOF
```

### 6. Verify the Created Resources

```bash
# Check if the StatefulSet was created
kubectl get statefulset -n application

# Check if the pods are being created
kubectl get pods -n application

# Check if the service was created
kubectl get service -n application
```

## Verification

A successful solution will show:

1. The CRD is properly installed:
   ```
   kubectl get crd databases.db.example.com
   ```

2. The operator deployment is running:
   ```
   kubectl get pods -n database-system
   ```

3. The database custom resource is created:
   ```
   kubectl get databases -n application
   ```

4. The StatefulSet and Service are created:
   ```
   kubectl get statefulset,service -n application
   ```

5. The database pods are running (or creating):
   ```
   kubectl get pods -n application
   ```

## Common Issues and Troubleshooting

1. **CRD Not Registered**: If the CRD isn't showing up, check for syntax errors in your CRD definition.

2. **RBAC Issues**: If the operator can't create resources, verify the ClusterRole and ClusterRoleBinding are correctly set up.

3. **Operator Pod Not Running**: Check the operator pod logs for errors.

4. **Custom Resource Not Accepted**: Ensure your custom resource matches the schema defined in the CRD.

5. **StatefulSet Pods Not Starting**: Check for issues with PersistentVolumeClaims or resource constraints.

## Notes for CKA Exam

- In the CKA exam, you might be asked to work with existing operators rather than creating one from scratch.
- Focus on understanding how to install CRDs, create custom resources, and verify their status.
- Remember that operators extend Kubernetes functionality by automating complex application management.
- The exam might test your ability to troubleshoot operator-related issues.
- Be familiar with the basic structure of CRDs and custom resources.
- Understand the relationship between CRDs, custom resources, and the operator pattern.

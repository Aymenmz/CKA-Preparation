# Solution Guide: RBAC Configuration

## Task 2: RBAC Configuration

**Scenario**: A new team of developers needs access to specific resources in the cluster.

**Task**: Create a new role and rolebinding that allows users in the "developers" group to create, list, and update deployments, services, and pods in the "development" namespace, but only view resources in other namespaces.

## Solution Steps

### 1. Create the Development Namespace (if not exists)

```bash
kubectl create namespace development
```

### 2. Create Role for Development Namespace

```bash
cat <<EOF > developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-role
rules:
- apiGroups: ["", "apps", "extensions"]
  resources: ["pods", "deployments", "services"]
  verbs: ["create", "get", "list", "update", "patch"]
EOF

kubectl apply -f developer-role.yaml
```

### 3. Create RoleBinding for Development Namespace

```bash
cat <<EOF > developer-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-rolebinding
  namespace: development
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f developer-rolebinding.yaml
```

### 4. Create ClusterRole for View-Only Access

```bash
cat <<EOF > view-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-reader
rules:
- apiGroups: ["", "apps", "extensions"]
  resources: ["pods", "deployments", "services"]
  verbs: ["get", "list", "watch"]
EOF

kubectl apply -f view-clusterrole.yaml
```

### 5. Create ClusterRoleBinding for View-Only Access

```bash
cat <<EOF > view-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developers-view-only
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f view-clusterrolebinding.yaml
```

### 6. Verify the RBAC Configuration

```bash
# Check the role in the development namespace
kubectl get role -n development
kubectl describe role developer-role -n development

# Check the rolebinding in the development namespace
kubectl get rolebinding -n development
kubectl describe rolebinding developer-rolebinding -n development

# Check the clusterrole
kubectl get clusterrole namespace-reader
kubectl describe clusterrole namespace-reader

# Check the clusterrolebinding
kubectl get clusterrolebinding developers-view-only
kubectl describe clusterrolebinding developers-view-only
```

### 7. Test the Configuration (Optional)

To test the configuration, you can create a test user that belongs to the "developers" group:

```bash
# Create a private key for the test user
openssl genrsa -out developer.key 2048

# Create a certificate signing request
openssl req -new -key developer.key -out developer.csr -subj "/CN=developer/O=developers"

# Sign the CSR with the Kubernetes CA
# Note: In a real environment, you would use the cluster's CA. For testing, you can use:
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: developer-csr
spec:
  request: $(cat developer.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
  groups:
  - system:authenticated
EOF

# Approve the CSR
kubectl certificate approve developer-csr

# Get the signed certificate
kubectl get csr developer-csr -o jsonpath='{.status.certificate}' | base64 --decode > developer.crt

# Create a kubeconfig for the developer
KUBECONFIG_FILE=developer-kubeconfig
kubectl config set-cluster $(kubectl config view -o jsonpath='{.clusters[0].name}') \
  --server=$(kubectl config view -o jsonpath='{.clusters[0].cluster.server}') \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --kubeconfig=$KUBECONFIG_FILE

kubectl config set-credentials developer \
  --client-certificate=developer.crt \
  --client-key=developer.key \
  --kubeconfig=$KUBECONFIG_FILE

kubectl config set-context developer-context \
  --cluster=$(kubectl config view -o jsonpath='{.clusters[0].name}') \
  --user=developer \
  --kubeconfig=$KUBECONFIG_FILE

kubectl config use-context developer-context --kubeconfig=$KUBECONFIG_FILE
```

Now test the permissions:

```bash
# Test creating resources in development namespace (should work)
kubectl --kubeconfig=$KUBECONFIG_FILE create deployment nginx --image=nginx -n development

# Test viewing resources in other namespaces (should work)
kubectl --kubeconfig=$KUBECONFIG_FILE get pods -n kube-system

# Test creating resources in other namespaces (should fail)
kubectl --kubeconfig=$KUBECONFIG_FILE create deployment nginx --image=nginx -n default
```

## Verification

A successful solution will show:

1. The role and rolebinding exist in the development namespace:
   ```
   kubectl get role,rolebinding -n development
   ```

2. The clusterrole and clusterrolebinding exist:
   ```
   kubectl get clusterrole namespace-reader
   kubectl get clusterrolebinding developers-view-only
   ```

3. When testing with a user in the developers group:
   - Can create, list, and update resources in the development namespace
   - Can only view resources in other namespaces

## Common Issues and Troubleshooting

1. **Permission Denied Errors**: Check that the role and rolebinding are correctly configured
2. **Group Membership Issues**: Ensure the user is correctly associated with the developers group
3. **Namespace Issues**: Verify the namespace exists and is correctly specified in the role and rolebinding
4. **API Group Problems**: Make sure the correct API groups are specified for the resources

## Notes for CKA Exam

- In the CKA exam, you might not need to create test users or test the configuration
- Focus on correctly creating the RBAC resources as specified
- Double-check the apiGroups, resources, and verbs in your role definitions
- Remember that ClusterRoles and ClusterRoleBindings are not namespaced, while Roles and RoleBindings are namespaced

# CKA Practice Exam Tasks

## Cluster Architecture, Installation & Configuration (25%)

### Task 1: Cluster Setup with kubeadm
**Scenario**: TechNova needs to set up a new Kubernetes cluster for their development team.
**Task**: Initialize a Kubernetes control plane using kubeadm, configure it for high availability, and join a worker node to the cluster.
**Points**: 15

### Task 2: RBAC Configuration
**Scenario**: A new team of developers needs access to specific resources in the cluster.
**Task**: Create a new role and rolebinding that allows users in the "developers" group to create, list, and update deployments, services, and pods in the "development" namespace, but only view resources in other namespaces.
**Points**: 10

### Task 3: Helm and Kustomize Implementation
**Scenario**: TechNova wants to standardize application deployment using Helm and Kustomize.
**Task**: Use Helm to install a monitoring stack (Prometheus) and use Kustomize to customize a provided application deployment for different environments (dev, staging).
**Points**: 10

### Task 4: CRD and Operator Management
**Scenario**: TechNova is implementing a database service that requires a custom operator.
**Task**: Install a provided database operator, create a Custom Resource Definition, and deploy an instance of the database using the custom resource.
**Points**: 10

## Workloads & Scheduling (15%)

### Task 5: Deployment Management
**Scenario**: TechNova is updating their web application and needs to ensure zero downtime.
**Task**: Create a deployment with multiple replicas, perform a rolling update to a new version, and then roll back to the previous version when issues are detected.
**Points**: 8

### Task 6: ConfigMaps and Secrets
**Scenario**: An application requires configuration data and sensitive information.
**Task**: Create ConfigMaps and Secrets for an application, then mount them appropriately in a pod. Update the ConfigMap and verify the changes are reflected in the running pod.
**Points**: 7

### Task 7: Resource Management and Scheduling
**Scenario**: TechNova has applications with specific resource requirements and node preferences.
**Task**: Deploy pods with resource requests/limits and configure node affinity to ensure pods run on appropriate nodes based on labels.
**Points**: 7

## Services & Networking (20%)

### Task 8: Network Policy Implementation
**Scenario**: TechNova needs to secure communication between microservices.
**Task**: Create network policies that allow specific communication paths between pods in different namespaces while blocking unauthorized traffic.
**Points**: 8

### Task 9: Service Configuration
**Scenario**: Different applications require different types of service exposure.
**Task**: Create ClusterIP, NodePort, and LoadBalancer services for different applications and verify connectivity between them.
**Points**: 7

### Task 10: Gateway API and Ingress Setup
**Scenario**: TechNova wants to expose multiple services through a single entry point.
**Task**: Configure an Ingress controller and create Ingress resources to route external traffic to different services based on paths and hostnames. Then implement the same using Gateway API resources.
**Points**: 10

## Storage (10%)

### Task 11: Dynamic Volume Provisioning
**Scenario**: Applications need persistent storage with specific performance characteristics.
**Task**: Create storage classes for different performance tiers, then create PVCs that use these storage classes and mount them in pods.
**Points**: 10

## Troubleshooting (30%)

### Task 12: Node Troubleshooting
**Scenario**: A worker node in the TechNova cluster is reporting NotReady status.
**Task**: Diagnose and fix issues with the worker node, which may include kubelet problems, certificate issues, or networking configuration.
**Points**: 12

### Task 13: Pod and Deployment Troubleshooting
**Scenario**: A critical application deployment is failing to start properly.
**Task**: Investigate and resolve issues with pods that are in CrashLoopBackOff or Error states, which may include configuration problems, resource constraints, or image issues.
**Points**: 10

### Task 14: Networking Troubleshooting
**Scenario**: Services are unable to communicate with each other as expected.
**Task**: Diagnose and fix networking issues between services, which may include DNS problems, network policy conflicts, or service configuration errors.
**Points**: 10

### Task 15: Control Plane Troubleshooting
**Scenario**: The Kubernetes API server is experiencing intermittent availability issues.
**Task**: Investigate and resolve issues with control plane components, which may include etcd problems, API server configuration, or certificate expiration.
**Points**: 13

## Scoring

- Total available points: 147
- Passing score: 97 points (66%)
- Time limit: 2 hours

## Instructions for Candidates

1. Read each task carefully before beginning
2. Manage your time effectively - don't spend too long on any single task
3. Use the Kubernetes documentation as needed
4. Verify your solutions work as expected before moving to the next task
5. If you get stuck on a task, move on and return to it later if time permits

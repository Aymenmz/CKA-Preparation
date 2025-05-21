# CKA Exam Requirements Analysis

## Exam Overview
The Certified Kubernetes Administrator (CKA) is a program created by the Cloud Native Computing Foundation (CNCF) in collaboration with The Linux Foundation. It is designed to certify that individuals have the skills, knowledge, and competency to perform the responsibilities of Kubernetes administrators. The exam is an online, proctored, performance-based test that requires solving multiple issues from a command line within a 2-hour timeframe.

## Exam Structure
- **Format**: Online, proctored, performance-based test
- **Duration**: 2 hours
- **Cost**: $445 (includes one free retake)
- **Passing Score**: Not explicitly stated in the sources, but typically requires demonstrating proficiency across all domains
- **Validity**: 3 years

## Exam Domains and Weights (2025)
The CKA exam focuses on the following domains with their respective weights:

1. **Cluster Architecture, Installation & Configuration** (25%)
   - Manage role-based access control (RBAC)
   - Prepare underlying infrastructure for installing a Kubernetes cluster
   - Create and manage Kubernetes clusters using kubeadm
   - Manage the lifecycle of Kubernetes clusters
   - Implement and configure a highly-available control plane
   - Use Helm and Kustomize to install cluster components
   - Understand extension interfaces (CNI, CSI, CRI, etc.)
   - Understand CRDs, install and configure operators

2. **Workloads & Scheduling** (15%)
   - Understand application deployments and how to perform rolling update and rollbacks
   - Use ConfigMaps and Secrets to configure applications
   - Configure workload autoscaling
   - Understand the primitives used to create robust, self-healing, application deployments
   - Configure Pod admission and scheduling (limits, node affinity, etc.)

3. **Services & Networking** (20%)
   - Understand connectivity between Pods
   - Define and enforce Network Policies
   - Use ClusterIP, NodePort, LoadBalancer service types and endpoints
   - Use the Gateway API to manage Ingress traffic
   - Know how to use Ingress controllers and Ingress resources
   - Understand and use CoreDNS

4. **Storage** (10%)
   - Implement storage classes and dynamic volume provisioning
   - Configure volume types, access modes and reclaim policies
   - Manage persistent volumes and persistent volume claims

5. **Troubleshooting** (30%)
   - Troubleshoot clusters and nodes
   - Troubleshoot cluster components
   - Monitor cluster and application resource usage
   - Manage and evaluate container output streams
   - Troubleshoot services and networking

## Recent Updates (2025)
According to the Linux Foundation, the CKA exam underwent updates that were released on February 18, 2025. While the domain structure remains unchanged, there have been updates to the competencies within each domain. Key additions include:

- Helm and Kustomize for cluster component installation
- Gateway API for managing Ingress traffic
- Enhanced focus on troubleshooting network services
- Dynamic storage provisioning

## Expected Skills
A Certified Kubernetes Administrator (CKA) is expected to:

1. Demonstrate their ability to do basic installation as well as configuring and managing production-grade Kubernetes clusters.
2. Understand key concepts such as Kubernetes networking, storage, security, maintenance, logging and monitoring, application lifecycle, troubleshooting, and API object primitives.
3. Establish basic use-cases for end users.

## Exam Environment
- The exam is conducted in a command-line environment
- Candidates are expected to solve multiple practical tasks
- Access to official Kubernetes documentation is permitted during the exam
- The exam environment uses a recent version of Kubernetes aligned with the CNCF's support policy

## Preparation Resources
The Linux Foundation and CNCF provide various resources for exam preparation:
- Curriculum overview
- Candidate handbook
- Exam tips
- Training courses (Kubernetes Fundamentals e-learning, Kubernetes Administration instructor-led)

This analysis provides a comprehensive understanding of the CKA exam requirements as of May 2025, which will serve as the foundation for designing a realistic practice exam environment.

# CKA Practice Exam Design

## Overview
This practice exam is designed to simulate the real Certified Kubernetes Administrator (CKA) exam experience. It follows the same domain distribution, difficulty level, and time constraints as the official exam. The scenarios and tasks are crafted to test your knowledge and skills across all required competencies updated for 2025.

## Exam Structure
- **Duration**: 2 hours
- **Number of Questions**: 15-20 performance-based tasks
- **Environment**: Command-line based Kubernetes clusters
- **Passing Score**: 66% (for practice purposes)
- **Resources Allowed**: Access to Kubernetes documentation (https://kubernetes.io/docs/)

## Domain Distribution

### 1. Cluster Architecture, Installation & Configuration (25%)
- **Tasks**: 4-5 tasks
- **Focus Areas**:
  - RBAC configuration
  - Cluster setup with kubeadm
  - High-availability control plane implementation
  - Helm and Kustomize usage
  - CRD and operator management

### 2. Workloads & Scheduling (15%)
- **Tasks**: 2-3 tasks
- **Focus Areas**:
  - Deployment management with rolling updates
  - ConfigMaps and Secrets implementation
  - Workload autoscaling
  - Pod scheduling and resource constraints

### 3. Services & Networking (20%)
- **Tasks**: 3-4 tasks
- **Focus Areas**:
  - Network policy implementation
  - Service type configuration
  - Gateway API and Ingress setup
  - CoreDNS configuration

### 4. Storage (10%)
- **Tasks**: 1-2 tasks
- **Focus Areas**:
  - Storage class implementation
  - PV and PVC management
  - Dynamic volume provisioning

### 5. Troubleshooting (30%)
- **Tasks**: 5-6 tasks
- **Focus Areas**:
  - Cluster and node troubleshooting
  - Application failure diagnosis
  - Network connectivity issues
  - Resource usage monitoring

## Scenario Design

The practice exam will be built around a fictional organization "TechNova" that is expanding its Kubernetes infrastructure to support various applications. As a Kubernetes administrator, you'll be tasked with setting up, configuring, and troubleshooting different aspects of their Kubernetes environment.

### Scenario Context

TechNova is a growing technology company that provides SaaS solutions to its clients. They have recently decided to migrate their applications to Kubernetes to improve scalability, reliability, and deployment efficiency. As their Kubernetes administrator, you are responsible for ensuring the infrastructure meets their requirements and resolves any issues that arise.

## Task Design Principles

1. **Authenticity**: Tasks mirror real-world scenarios and challenges faced by Kubernetes administrators.
2. **Comprehensiveness**: Coverage of all domains and competencies in the CKA curriculum.
3. **Progressive Difficulty**: Tasks range from basic to advanced, allowing for skill assessment across different levels.
4. **Independence**: Each task can be completed independently to prevent cascading failures.
5. **Clear Instructions**: Tasks include clear objectives and success criteria.

## Detailed Task Examples

Below are examples of tasks that will be included in the practice exam:

### Cluster Architecture Task Example
**Task**: Configure RBAC for a new team member who needs to manage deployments and services in the "production" namespace but should only have read access to other namespaces.

### Workloads & Scheduling Task Example
**Task**: Deploy an application that requires specific node affinity rules and resource limits, then perform a rolling update without downtime.

### Services & Networking Task Example
**Task**: Implement network policies to restrict communication between pods in different namespaces, allowing only specific paths.

### Storage Task Example
**Task**: Create a storage class for dynamic provisioning and configure a stateful application to use persistent volumes with this storage class.

### Troubleshooting Task Example
**Task**: Diagnose and fix a non-functioning service that should connect to a backend deployment but is failing to route traffic correctly.

## Scoring Methodology

Each task will be assigned points based on its complexity and the domain weight:
- Simple tasks: 5-7 points
- Moderate tasks: 8-12 points
- Complex tasks: 13-15 points

The total possible score will be 100 points, with a passing threshold of 66 points.

## Environment Requirements

The practice exam will require:
1. A multi-node Kubernetes cluster (at least 3 nodes)
2. Access to kubectl command-line tool
3. Ability to create and modify Kubernetes resources
4. Access to Kubernetes documentation
5. Pre-configured scenarios that can be reset between attempts

## Preparation Instructions

Before taking the practice exam:
1. Ensure familiarity with kubectl commands
2. Review the Kubernetes documentation structure
3. Practice time management strategies
4. Set up a quiet, uninterrupted environment
5. Have necessary tools ready (terminal, browser for documentation)

This practice exam design aims to provide a comprehensive and realistic simulation of the actual CKA exam, helping candidates identify strengths and areas for improvement before taking the official certification.

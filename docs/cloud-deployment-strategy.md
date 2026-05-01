---
layout: default
title: Cloud Deployment Strategy
nav_order: 5
---

# Cloud Deployment Strategy for VulnerableApp

## Overview

This document outlines a standardized approach for deploying VulnerableApp to major cloud providers (AWS, GCP, and Azure) using modern containerization and Infrastructure as Code (IaC) practices. The strategy emphasizes automation, scalability, and consistency across different cloud environments.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Containerization Strategy](#containerization-strategy)
3. [Cloud Provider Deployments](#cloud-provider-deployments)
4. [Kubernetes Orchestration](#kubernetes-orchestration)
5. [Infrastructure as Code](#infrastructure-as-code)
6. [Deployment Workflow](#deployment-workflow)
7. [Monitoring and Maintenance](#monitoring-and-maintenance)

## Architecture Overview

### Target Deployment Architecture

```
┌─────────────────────────────────────────────┐
│        Cloud Provider (AWS/GCP/Azure)       │
├─────────────────────────────────────────────┤
│  ┌──────────────────────────────────────┐   │
│  │   Kubernetes Cluster (EKS/GKE/AKS)   │   │
│  │  ┌──────────────────────────────────┐│   │
│  │  │  VulnerableApp Pod               ││   │
│  │  │  ├─ Spring Boot Application      ││   │
│  │  │  ├─ ReactJS Frontend             ││   │
│  │  │  └─ Health Checks                ││   │
│  │  └──────────────────────────────────┘│   │
│  │  ┌──────────────────────────────────┐│   │
│  │  │  Load Balancer / Ingress         ││   │
│  │  │  └─ TLS Termination              ││   │
│  │  └──────────────────────────────────┘│   │
│  │  ┌──────────────────────────────────┐│   │
│  │  │  Storage & Persistence Layer     ││   │
│  │  │  ├─ ConfigMaps (Configuration)   ││   │
│  │  │  ├─ Secrets (Credentials)        ││   │
│  │  │  └─ PersistentVolumes (if needed)││   │
│  │  └──────────────────────────────────┘│   │
│  └──────────────────────────────────────┘   │
│  ┌──────────────────────────────────────┐   │
│  │   Monitoring Stack                   │   │
│  │   ├─ Prometheus                      │   │
│  │   ├─ Grafana                         │   │
│  │   └─ Cloud Provider Logging          │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### Key Design Principles

- **Cloud-Agnostic**: Use Kubernetes as the abstraction layer to minimize cloud provider lock-in
- **Infrastructure as Code**: All infrastructure defined in Terraform/Helm for reproducibility
- **Containerization**: Docker for consistent application delivery across environments
- **High Availability**: Multi-node Kubernetes clusters with auto-scaling
- **Security**: Network policies, RBAC, secrets management, and regular security scanning
- **Observability**: Comprehensive logging, monitoring, and tracing

## Containerization Strategy

### Docker Image Building

#### Dockerfile Structure

```dockerfile
# Multi-stage build for optimization
FROM openjdk:11-jre-slim as builder
WORKDIR /app
COPY . .
RUN ./gradlew clean build -x test

FROM openjdk:11-jre-slim
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8080/VulnerabilityDefinitions || exit 1

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Image Registry Strategy

- **Registry Choice**: Docker Hub or cloud provider registries (Amazon ECR, Google Artifact Registry, Azure Container Registry)
- **Image Tagging**: `vulnerableapp:<version>-<timestamp>` or use semantic versioning
- **Image Scanning**: Implement automated vulnerability scanning on push
- **Image Cleanup**: Implement lifecycle policies for old image cleanup

### Build Pipeline

- Use GitHub Actions, GitLab CI, or cloud provider CI/CD for automated builds
- Build and push on every release/main branch commit
- Tag images with commit SHA and version numbers
- Run security scans (Trivy, Snyk) before registry push

## Cloud Provider Deployments

### AWS (Elastic Kubernetes Service - EKS)

#### Infrastructure Components

1. **VPC & Networking**
   - Create VPC with public and private subnets
   - NAT Gateway for private subnet internet access
   - Security groups for ingress/egress rules

2. **EKS Cluster**
   - Multi-AZ deployment across 2-3 availability zones
   - Managed node groups with auto-scaling (min: 2, max: 10)
   - Using t3.medium instances (adjustable based on load)
   - Enable RBAC and network policies

3. **Load Balancing**
   - AWS ALB (Application Load Balancer) for ingress
   - SSL/TLS termination via AWS Certificate Manager

4. **Storage**
   - EBS volumes for persistent storage (if needed)
   - AWS Secrets Manager for sensitive data

5. **Monitoring**
   - CloudWatch for logs and metrics
   - Prometheus in-cluster for Kubernetes metrics
   - VPC Flow Logs for network monitoring

#### Deployment Steps

```bash
# Create EKS cluster using Terraform (see IaC section)
terraform apply -var-file=aws.tfvars

# Configure kubectl context
aws eks update-kubeconfig --name vulnerable-app --region us-east-1

# Deploy using Helm
helm install vulnerable-app ./helm-charts/vulnerable-app -f values-aws.yaml
```

### GCP (Google Kubernetes Engine - GKE)

#### Infrastructure Components

1. **Network & Firewall**
   - Custom VPC with subnets for GKE
   - Cloud NAT for outbound NAT
   - Firewall rules for ingress/egress

2. **GKE Cluster**
   - Regional cluster across 3 zones
   - Workload Identity enabled for secure service authentication
   - Network Policy enabled
   - Auto Node Repair and Auto Node Upgrade enabled

3. **Load Balancing**
   - Google Cloud Load Balancer (L7)
   - Cloud CDN for static content caching
   - Google-managed SSL certificates

4. **Storage**
   - GCP Persistent Disks
   - Secret Manager for credentials

5. **Monitoring**
   - Cloud Logging (Stackdriver)
   - Cloud Monitoring
   - Cloud Trace for distributed tracing

#### Deployment Steps

```bash
# Create GKE cluster using Terraform
terraform apply -var-file=gcp.tfvars

# Get credentials
gcloud container clusters get-credentials vulnerable-app --region us-central1

# Deploy using Helm
helm install vulnerable-app ./helm-charts/vulnerable-app -f values-gcp.yaml
```

### Azure (Azure Kubernetes Service - AKS)

#### Infrastructure Components

1. **Virtual Network & Subnets**
   - Azure VNet with multiple subnets
   - Network Security Groups for firewall rules
   - Azure Network Policies (Calico)

2. **AKS Cluster**
   - Multi-zone availability sets
   - System and user node pools
   - Azure AD integration for authentication
   - RBAC enabled

3. **Load Balancing**
   - Azure Load Balancer / Application Gateway
   - Application Gateway for advanced routing
   - Azure Front Door for global load balancing

4. **Storage**
   - Azure Managed Disks
   - Azure Key Vault for secrets management

5. **Monitoring**
   - Azure Monitor
   - Container Insights
   - Azure Log Analytics

#### Deployment Steps

```bash
# Create AKS cluster using Terraform
terraform apply -var-file=azure.tfvars

# Configure kubectl
az aks get-credentials --resource-group vulnerable-app-rg --name vulnerable-app

# Deploy using Helm
helm install vulnerable-app ./helm-charts/vulnerable-app -f values-azure.yaml
```

## Kubernetes Orchestration

### Helm Charts

#### Chart Structure

```
helm-charts/vulnerable-app/
├── Chart.yaml
├── values.yaml
├── values-aws.yaml
├── values-gcp.yaml
├── values-azure.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secrets.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   └── servicemonitor.yaml
└── README.md
```

#### Key Kubernetes Resources

1. **Deployment**
   - Replicas: 3 (high availability)
   - Resource requests/limits defined
   - Liveness and readiness probes
   - Pod Disruption Budget

2. **Service**
   - Type: ClusterIP (internal) with Ingress for external access
   - Port mapping for port 8080

3. **Ingress**
   - TLS enabled
   - Host-based routing (optional)
   - Ingress controller: nginx or cloud provider native

4. **ConfigMap**
   - Application configuration
   - Environment-specific settings

5. **Secrets**
   - Database credentials
   - API keys
   - TLS certificates
   - Consider using external secret management (Vault, cloud provider secrets)

6. **Horizontal Pod Autoscaler (HPA)**
   - CPU-based scaling (70% threshold)
   - Memory-based scaling (80% threshold)
   - Min replicas: 2, Max replicas: 10

7. **Pod Disruption Budget**
   - Ensures availability during cluster maintenance
   - Min available: 1

#### Example Deployment Values

```yaml
# values.yaml
replicaCount: 3

image:
  repository: vulnerableapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: vulnerableapp.example.com
      paths:
        - path: /
          pathType: Prefix

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

## Infrastructure as Code

### Terraform Structure

#### Directory Layout

```
terraform/
├── main.tf
├── vpc.tf
├── kubernetes.tf
├── monitoring.tf
├── variables.tf
├── outputs.tf
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   ├── prod.tfvars
│   ├── aws/
│   │   ├── main.tf
│   │   └── provider.tf
│   ├── gcp/
│   │   ├── main.tf
│   │   └── provider.tf
│   └── azure/
│       ├── main.tf
│       └── provider.tf
└── modules/
    ├── vpc/
    ├── kubernetes/
    └── monitoring/
```

### Key Terraform Resources

#### AWS Example

```hcl
# Create EKS cluster
resource "aws_eks_cluster" "vulnerable_app" {
  name            = "vulnerable-app"
  role_arn        = aws_iam_role.eks_cluster_role.arn
  vpc_config {
    subnet_ids              = aws_subnet.private[*].id
    endpoint_private_access = true
    endpoint_public_access  = true
  }
}

# Node group
resource "aws_eks_node_group" "vulnerable_app" {
  cluster_name    = aws_eks_cluster.vulnerable_app.name
  node_group_name = "vulnerable-app-ng"
  node_role_arn   = aws_iam_role.eks_node_role.arn
  subnet_ids      = aws_subnet.private[*].id

  scaling_config {
    desired_size = 3
    max_size     = 10
    min_size     = 2
  }
}
```

#### GCP Example

```hcl
# Create GKE cluster
resource "google_container_cluster" "vulnerable_app" {
  name     = "vulnerable-app"
  location = "us-central1"

  initial_node_count = 3
  
  node_config {
    machine_type = "n1-standard-2"
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}
```

#### Azure Example

```hcl
# Create AKS cluster
resource "azurerm_kubernetes_cluster" "vulnerable_app" {
  name                = "vulnerable-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "vulnerable-app"

  default_node_pool {
    name       = "default"
    node_count = 3
    vm_size    = "Standard_B2s"
  }
}
```

## Deployment Workflow

### Continuous Deployment Pipeline

#### GitHub Actions Workflow Example

```yaml
name: Deploy to Cloud

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'dev'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker image
        run: docker build -t vulnerableapp:${{ github.sha }} .
      
      - name: Push to registry
        run: |
          docker tag vulnerableapp:${{ github.sha }} ${{ secrets.REGISTRY }}/vulnerableapp:latest
          docker push ${{ secrets.REGISTRY }}/vulnerableapp:latest

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure cloud credentials
        run: |
          # Cloud-specific authentication
          
      - name: Deploy with Helm
        run: |
          helm upgrade --install vulnerable-app ./helm-charts/vulnerable-app \
            --values ./helm-charts/vulnerable-app/values-${{ github.event.inputs.environment }}.yaml
```

### Blue-Green Deployment

For production environments, consider implementing blue-green deployments:

1. Deploy new version to "green" environment
2. Run smoke tests
3. Switch traffic to green
4. Keep blue running for quick rollback

### Canary Deployments

For gradual rollouts:

1. Deploy to 10% of replicas
2. Monitor metrics
3. Gradually increase traffic
4. Complete rollout or rollback based on health

## Monitoring and Maintenance

### Logging

- **Centralized Logging**: Aggregate logs from all containers
  - AWS: CloudWatch Logs
  - GCP: Cloud Logging
  - Azure: Log Analytics
- **Log Aggregation**: Consider ELK stack or Splunk
- **Log Retention**: Define retention policies (30-90 days)

### Metrics & Observability

- **Prometheus**: Scrape application and Kubernetes metrics
- **Grafana**: Visualization dashboards
- **Key Metrics**:
  - HTTP request latency (p50, p95, p99)
  - Error rate
  - CPU and memory usage
  - Pod restart count

### Alerting

Define alerts for:
- Pod crash loops
- High CPU/memory usage
- Request error rates > threshold
- Deployment failures

### Security Considerations

1. **Network Policies**: Restrict pod-to-pod communication
2. **RBAC**: Implement least privilege access
3. **Image Scanning**: Scan container images for vulnerabilities
4. **Secrets Management**: Use cloud provider secret management
5. **Regular Patching**: Keep Kubernetes and node OS updated
6. **Pod Security**: Enable Pod Security Standards/Policies
7. **Audit Logging**: Enable Kubernetes audit logs

### Backup & Disaster Recovery

1. **Persistent Data Backups**: Regular snapshots of PersistentVolumes
2. **Helm Release Backups**: Export Helm releases regularly
3. **Terraform State**: Backup remote state (S3, GCS, Azure Storage)
4. **RTO/RPO Targets**: Define Recovery Time Objective and Recovery Point Objective
5. **Disaster Recovery Drills**: Test recovery procedures quarterly

### Updates & Maintenance

- **Kubernetes Upgrades**: Plan for quarterly minor version upgrades
- **Node OS Updates**: Automated patching with scheduled maintenance windows
- **Application Updates**: Use GitOps (ArgoCD, Flux) for automated deployments
- **Dependency Updates**: Regular scanning and updates via Dependabot

## Cost Optimization

1. **Right-sizing**: Monitor usage and adjust instance types
2. **Reserved Instances**: Purchase reserved capacity for base load
3. **Spot Instances**: Use for non-critical workloads
4. **Autoscaling**: Aggressive downscaling during off-peak hours
5. **Network Costs**: Use cloud provider's data transfer optimization
6. **Storage Lifecycle**: Archive old logs and backups

## Getting Started

1. **Development**: Start with local Docker and kind/minikube
2. **Staging**: Single-node Kubernetes cluster in cloud provider
3. **Production**: Multi-node, multi-zone cluster with full monitoring
4. **Gradual Migration**: Migrate workloads incrementally

## References & Resources

- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Google GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)
- [Azure AKS Documentation](https://learn.microsoft.com/en-us/azure/aks/)
- [Helm Documentation](https://helm.sh/docs/)
- [Terraform Documentation](https://www.terraform.io/docs)
- [Cloud Native Security Best Practices](https://www.cncf.io/blog/2020/11/17/cloud-native-security-best-practices-for-kubernetes/)

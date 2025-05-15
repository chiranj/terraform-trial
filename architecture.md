# EKS Deployment Architecture with GitLab CI/CD

## Introduction

This document outlines a GitLab CI/CD-based deployment architecture for Amazon EKS clusters using Infrastructure as Code (IaC) with Terraform and Kubernetes resources via Helm charts. The solution enables teams to quickly provision production-ready EKS clusters with standardized add-ons while maintaining flexibility for customization.

The architecture follows a modular approach where users can specify their infrastructure requirements in a simple configuration file, while the heavy lifting is performed by centralized, reusable pipeline templates. This reduces duplication of code, ensures consistent deployments across teams, and implements organizational best practices by default.

Key components include:
- User-configurable EKS infrastructure deployment via Terraform
- Automated installation of essential add-ons via Helm charts
- Standardized ingress with Traefik
- Comprehensive observability with Prometheus, Grafana, X-ray, and CloudWatch
- Advanced scaling with Karpenter and KEDA
- Enhanced security with cert-manager, ACM, and AWS Secrets Provider

## User Deployment Workflow

The deployment process is designed to be straightforward for end users while maintaining robust security controls and standardization. Users only need to focus on their specific requirements while the underlying complexities are abstracted away.

```mermaid
flowchart TD
    A[User] -->|1. Create new repo from template| B[Repository with default .gitlab-ci.yml]
    A -->|2. Configure| C[Update parameters]
    D[Account ID, Subnets, Cluster name, IAM Role] -->|Set in| C
    E[Add-ons selection] -->|Enable/Disable in| C
    E -->|Example options| F[traefik=true, external-dns=true, prometheus=false]
    
    C -->|3. Commit & Push| G[Trigger GitLab Pipeline]
    G -->|Runs on user's| H[EC2 GitLab Runners in user AWS account]
    H -->|4. Import| I[Main Pipeline Templates Repository]
    
    I -->|5. Execute| J[Core Infrastructure Pipeline]
    J -->|Validate| K[Verify IAM permissions]
    J -->|Deploy with Terraform| L[EKS control plane & node groups]
    J -->|Create| M[IRSA Roles for add-ons]
    
    L -->|6. Trigger| N[Child Pipeline for Add-ons]
    M -->|Pass parameters| N
    N -->|Import| O[Add-ons .gitlab-ci.yml]
    O -->|Install selected| P[Helm charts for enabled add-ons]
    P -->|Configure with| Q[Environment variables]
    Q -->|Examples| R[Cluster name, IRSA ARN, Region]
```

## Ingress Architecture

The ingress architecture leverages AWS Route 53 for DNS routing, Network Load Balancer for efficient traffic management, and Traefik as the ingress controller within the Kubernetes cluster.

```mermaid
flowchart LR
    A[External Traffic] -->|DNS Routing| B[Route 53]
    B -->|Forward to| C[AWS Network Load Balancer]
    C -->|TLS Termination & Load Balancing| D[Traefik Ingress Controller]
    D -->|Routing based on Host/Path| E[Kubernetes Services]
    E -->|Backend Selection| F[Kubernetes Pods]
    
    subgraph "AWS Infrastructure"
    B
    C
    end
    
    subgraph "EKS Cluster"
    D
    E
    F
    end
    
    G[GitLab Pipeline] -.->|Deploy & Configure| D
    H[Cert-Manager] -.->|TLS Certificates| D
    I[External-DNS] -.->|Automate DNS records| B
```

## Observability Architecture

The observability stack provides comprehensive monitoring, logging, metrics collection, and tracing capabilities across the EKS cluster.

```mermaid
flowchart TD
    subgraph "EKS Cluster"
        A[Application Pods]
        B[AWS Distro for OpenTelemetry Collector]
        C[Prometheus]
        D[FluentBit]
        
        A -->|Metrics| B
        A -->|Metrics| C
        A -->|Logs| D
        A -->|Traces| B
    end
    
    subgraph "AWS Managed Services"
        E[Amazon Managed Grafana]
        F[AWS X-Ray]
        G[Amazon CloudWatch]
        H[Amazon S3 for Long-Term Storage]
        
        E -->|Visualization & Alerting Policies| I[PagerDuty/Slack]
        E -->|Custom & Pre-packaged Dashboards| J[DevOps Teams]
    end
    
    B -->|Export Metrics| C
    B -->|Export Traces| F
    B -->|Export Logs| G
    C -->|Export Metrics| E
    C -->|Archive Metrics| H
    D -->|Stream Logs| G
    G -->|Logs Analysis| E
    
    K[GitLab Pipeline] -.->|Deploy & Configure| B
    K -.->|Deploy & Configure| C
    K -.->|Deploy & Configure| D
    
    L[Terraform] -.->|Provision| E
    L -.->|Provision| F
    L -.->|Provision| G
    L -.->|Provision| H
```

## Scaling Architecture

The scaling architecture uses Karpenter for efficient node provisioning and KEDA for event-driven autoscaling of pods.

```mermaid
flowchart TD
    subgraph "EKS Cluster"
        A[Pending Pods]
        B[Karpenter Controller]
        C[KEDA Operator]
        D[KEDA ScaledObjects]
        E[HPA/VPA]
        F[Application Pods]
        
        A -->|Trigger| B
        D -->|Control| E
        E -->|Scale| F
    end
    
    subgraph "AWS Infrastructure"
        G[EC2 Instance Fleet]
        H[EC2 Spot Instances]
        I[EC2 On-Demand Instances]
        J[AWS Event Sources]
        K[CloudWatch Metrics]
        L[SQS Queues]
        
        G -->|Includes| H
        G -->|Includes| I
    end
    
    B -->|Provision Just-in-Time| G
    J -->|Events| C
    K -->|Metrics| C
    L -->|Queue Length| C
    C -->|Create/Update| D
    
    M[GitLab Pipeline] -.->|Deploy & Configure| B
    M -.->|Deploy & Configure| C
    
    N[Terraform] -.->|IAM Roles & Policies| B
    N -.->|IAM Roles & Policies| C
```

## Security Architecture

The security architecture implements robust certificate management and secure secret handling.

```mermaid
flowchart TD
    subgraph "EKS Cluster"
        A[Cert-Manager]
        B[AWS Secrets & Configuration Provider]
        C[Application Pods]
        D[Traefik Ingress Routes]
        
        A -->|Inject Certificates| D
        B -->|Mount Secrets| C
        D -->|TLS Termination| C
    end
    
    subgraph "AWS Infrastructure"
        E[AWS Certificate Manager]
        F[AWS Secrets Manager]
        G[AWS Systems Manager Parameter Store]
        H[IAM OIDC Provider]
        
        E -->|Issue Public Certificates| A
        F -->|Secure Storage| B
        G -->|Configuration Storage| B
    end
    
    H -->|Trust Relationship| I[Service Account]
    I -->|Credentials| B
    I -->|Credentials| A
    
    J[GitLab Pipeline] -.->|Deploy & Configure| A
    J -.->|Deploy & Configure| B
    
    K[Terraform] -.->|Provision IRSA| H
    K -.->|Provision| E
    K -.->|Provision| F
    K -.->|Provision| G
```

## Deployment & Operations

The entire infrastructure and application deployment is orchestrated through GitLab CI/CD pipelines:

1. **Infrastructure Pipeline**: Provisions and configures the EKS cluster and supporting AWS resources using Terraform
2. **Add-ons Pipeline**: Installs and configures Kubernetes add-ons using Helm charts
3. **Application Pipeline**: Deploys and configures user applications on the EKS cluster

Operational tasks are also automated:
- Cluster upgrades are handled via GitLab CI/CD with blue/green deployment strategy
- Certificate rotation is automated via cert-manager
- Secret rotation is automated via AWS Secrets Manager
- Configuration changes are applied via GitOps workflow

The modular design allows teams to customize their infrastructure while maintaining organizational standards and security best practices.

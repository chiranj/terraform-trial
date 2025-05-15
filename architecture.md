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

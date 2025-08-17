# Enterprise AWS EKS Clusters Deployments
One of the most critical infrastructure components to manage is Amazon Elastic Kubernetes Service (EKS). Let's see how to streamline EKS cluster provisioning while following enterprise best practices.

## Architecture Overview

The module follows a modular architecture with clear separation of concerns:

```
eks-cluster/
├── modules/
│   ├── control-plane/      # EKS control plane
│   ├── managed-workers/    # Managed node groups
│   ├── ingress-controller/ # ALB integration
│   ├── autoscaler/         # Cluster autoscaling
│   └── tags/               # VPC tagging
└── use-cases/
    ├── managed-workers/
    └── support-services/
```

This design allows for flexble deployment of components needed for a specific use case.

## Multi-Environment EKS Deployment

This is a typical enterprise scenario where we need to deploy EKS clusters across development, staging, and production environments.

### Requirements

- **Development**: Basic cluster for testing and development
- **Staging**: Production-like environment with monitoring
- **Production**: High-availability cluster with advanced observability, security, and autoscaling

### Infrastructure Implementation

#### 1. Basic EKS Cluster with Managed Workers

Here's how we provision the EKS cluster:

```hcl
# main.tf
terraform {
  required_version = ">= 1.8.4"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.48.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.26.0"
    }
  }
}

# VPC with proper EKS tagging
module "vpc" {
  source     = "modules/vpc"
  vpc_name   = var.vpc_name
  aws_region = var.aws_region
  
  custom_tags                 = module.tags.vpc_eks_tags
  public_subnet_tags          = module.tags.vpc_public_subnet_tags
  private_subnet_tags         = module.tags.vpc_private_subnet_tags
  database_subnet_tags        = module.tags.vpc_database_subnet_tags
  
  cidr_block       = "10.0.0.0/18"
  nat_gateways = 1
}

# EKS VPC tags
module "tags" {
  source = "modules/tags"
  cluster_names = [var.cluster_name]
}

# EKS Control Plane
module "cluster" {
  source       = "eks/modules/control-plane"
  cluster_name = var.cluster_name
  
  vpc_id                       = module.vpc.vpc_id
  control_plane_subnet_ids = local.usable_subnet_ids
  
  version           = var.kubernetes_version
  config_path         = var.kubectl_config_path
  public_access      = var.endpoint_public_access
  public_access_cidrs = var.endpoint_public_access_cidrs
  cluster_log_types   = ["api"]
  
  # CNI optimization for high-density workloads
  cni_prefix_delegation = var.cni_prefix_delegation
  cni_warm_ip_target          = var.cni_warm_ip_target
  cni_minimum_ip_target       = var.cni_minimum_ip_target
  
  # EKS Add-ons
  eks_addons_on = var.eks_addons_on
  eks_addons        = var.eks_addons
}

# Managed Worker Node
module "workers" {
  source       = "eks/modules/workers"
  cluster_name = module.cluster.cluster_name
  
  node_group_configurations = {
    group1 = {
      desired_size  = 1
      min_size     = 1
      max_size     = 4
      instance_type = ["t3.medium"]
      subnet_ids   = [local.usable_subnet_ids[0]]
    }
  }
  
  cluster_keypair_name = var.cluster_keypair_name
}
```

#### 2. Advanced Production Configuration

```hcl
# variables.tf - Production configurations
variable "addons" {
  description = "EKS addons for production"
  type        = any
  default = {
    coredns = {
      resolve_conflicts = "OVERWRITE"
      addon_version     = "v1.11.0-eksbuild.3"
    }
    kube-proxy = {
      resolve_conflicts = "OVERWRITE"
      addon_version     = "v1.30.1-eksbuild.1"
    }
    vpc-cni = {
      resolve_conflicts = "OVERWRITE"
      addon_version     = "v1.17.1-eksbuild.1"
    }
    aws-ebs-csi-driver = {
      resolve_conflicts = "OVERWRITE"
      addon_version     = "v1.27.2-eksbuild.1"
    }
  }
}

variable "prefix_delegation" {
  description = "Enable prefix delegation for increased pod density"
  type        = bool
  default     = true
}
```

### Security by Design
```hcl
# Security group with restricted access
resource "aws_security_group_rule" "inbound_api_access" {
  count             = length(var.private_access_cidrs) > 0 ? 1 : 0
  type              = "ingress"
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  cidr_blocks       = var.endpoint_private_cidrs
  security_group_id = aws_security_group.eks.id
}

# IAM roles with least privilege
resource "aws_iam_role" "eks" {
  name                 = "${var.cluster_name}-cluster"
  assume_role_policy   = data.aws_iam_policy_document.allow_eks_to_assume_role.json
  permissions_boundary = var.cluster_iam_role_permissions_boundary
}
```

### Production Deployment

#### Step 1: Environment Configuration

```bash
# Development environment
cat > environments/dev/terraform.tfvars <<EOF
aws_region             = "us-west-2"
cluster_name      = "dev-eks-cluster"
vpc_name              = "dev-vpc"
kubernetes_version    = "1.32"
endpoint_public_access = true
node_group_configurations = {
  dev_nodes = {
    desired_size  = 1
    min_size     = 1
    max_size     = 2
    instance_type = ["t3.small"]
  }
}
EOF

# Production environment  
cat > environments/prod/terraform.tfvars <<EOF
aws_region             = "us-west-2"
cluster_name      = "prod-eks-cluster"
vpc_name              = "prod-vpc"
kubernetes_version    = "1.29"
endpoint_public_access = false
cni_prefix_delegation = true
eks_addons_on     = true
node_group_configurations = {
  system_nodes = {
    desired_size  = 3
    min_size     = 3
    max_size     = 10
    instance_type = ["m5.large"]
    subnet_ids   = local.private_subnets
  }
  app_nodes = {
    desired_size  = 3
    min_size     = 3
    max_size     = 20
    instance_type = ["m5.xlarge", "m5.2xlarge"]
    subnet_ids   = local.private_subnets
  }
}
EOF
```

#### Step 2: Deployment Pipeline Integration

```yaml
# .github/workflows/eks-deploy.yml
name: EKS Infrastructure Deploy
on:
  push:
    branches: [main]
    paths: ['infrastructure/eks/**']

jobs:
  terraform:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [dev, staging, prod]
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.0
        
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2
        
    - name: Terraform Init
      run: |
        cd infrastructure/eks
        terraform init -backend-config="environments/${{ matrix.environment }}/backend.conf"
        
    - name: Terraform Plan
      run: |
        cd infrastructure/eks
        terraform plan -var-file="environments/${{ matrix.environment }}/terraform.tfvars"
        
    - name: Terraform Apply
      if: matrix.environment == 'dev' || github.ref == 'refs/heads/main'
      run: |
        cd infrastructure/eks
        terraform apply -auto-approve -var-file="environments/${{ matrix.environment }}/terraform.tfvars"
```

#### Step 3: Post-Deployment Configuration

```bash
# Configure kubectl
EKS_ARN=$(terraform output -raw cluster_arn)
aws eks update-kubeconfig --region us-west-2 --name $(terraform output -raw cluster_name)

# Verify cluster health
kubectl get nodes
kubectl get pods -A

# Deploy monitoring stack
kubectl apply -f manifests/monitoring/
```

### Advanced Features for Enterprise Environments

#### 1. Multi-Tenancy Support

```hcl
# Dedicated node groups for different workloads
node_group_configurations = {
  system_critical = {
    desired_size = 3
    min_size     = 3
    max_size     = 5
    instance_type = ["m5.large"]
    
    # Taints for dedicated workloads
    taints = [{
      key    = "node-role.kubernetes.io/system"
      value  = "true"
      effect = "NO_SCHEDULE"
    }]
    
    labels = {
      "node-role.kubernetes.io/system" = "true"
    }
  }
  
  application_workloads = {
    desired_size = 2
    min_size     = 2
    max_size     = 20
    instance_type = ["m5.xlarge", "c5.xlarge"]
  }
}
```

#### 2. Cost Optimization with Spot Instances

```hcl
# Mixed instance types with Spot instances
launch_template_config = {
  image_id      = data.aws_ami.eks_worker.id
  instance_type = "m5.large"
  
  # Spot instance configuration
  instance_market_options {
    market_type = "spot"
    spot_options {
      spot_instance_type = "one-time"
      max_price         = "0.10"
    }
  }
}
```

#### 3. GitOps Integration

```bash
# Install ArgoCD on the cluster
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Configure application deployment
cat > applications/app-of-apps.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/k8s-apps
    targetRevision: HEAD
    path: applications
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

### Monitoring and Observability

#### CloudWatch Integration

```hcl
# Enable logging
enabled_cluster_log_types = [
  "api",
  "audit", 
  "authenticator",
  "controllerManager",
  "scheduler"
]

# CloudWatch log group configuration
cloudwatch_log_group_retention_in_days = 30
cloudwatch_log_group_kms_key_id        = aws_kms_key.logs.arn
```

#### Prometheus and Grafana Setup

```bash
# Install monitoring stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.service.type=LoadBalancer
```

### Disaster Recovery and Backup

#### EKS Cluster Backup Strategy

```hcl
# Backup IAM policy for Velero
resource "aws_iam_policy" "velero_backup" {
  name_prefix = "${var.cluster_name}-velero-backup"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "ec2:DescribeVolumes",
        "ec2:DescribeSnapshots", 
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot"
      ]
      Resource = "*"
    }]
  })
}
```

### Performance Tuning and Best Practices

#### 1. Node Optimization

```hcl
# Custom launch template for performance tuning
resource "aws_launch_template" "optimized_nodes" {
  name_prefix   = "${var.cluster_name}-optimized"
  image_id      = data.aws_ami.eks_optimized.id
  instance_type = "c5.xlarge"
  
  # Performance optimizations
  user_data = base64encode(templatefile("${path.module}/user-data/optimized.sh", {
    cluster_name = var.cluster_name
  }))
  
  # Enhanced networking
  network_interfaces {
    associate_public_ip_address = false
    security_groups            = [aws_security_group.worker_nodes.id]
    
    # Enable enhanced networking
    interface_type = "efa"
  }
  
  # Instance metadata service v2
  metadata_options {
    http_endpoint = "enabled"
    http_tokens   = "required"
    http_put_response_hop_limit = 2
  }
}
```

#### 2. CNI Configuration for High Density

```hcl
# Optimize VPC CNI for maximum pod density
vpc_cni_prefix_delegation = true
vpc_cni_warm_ip_target          = 1
vpc_cni_minimum_ip_target       = 1
```

### Security Hardening

#### 1. Network Policies

```yaml
# Default deny network policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

#### 2. Pod Security Standards

```yaml
# Pod Security Policy
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: eks-restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
```

## Conclusion
This deployment pattern enables teams to focus on delivering value, while the infrastructure layer remains secure and scalable. It establishes a strong foundation for deploying Amazon EKS clusters across diverse environments. The primary benefits include:

- **Modular Architecture**: Deploy only the required components tailored to specific use cases.
- **Integrated Security**: Adherence to AWS security best practices and hardening.
- **Operational Excellence**: Monitoring, logging, and automation capabilities.
- **Cost Optimization**: Efficient resource utilization through Spot instance support and right-sizing.
- **CI/CD**: GitOps workflows facilitated by automated deployment pipelines.

By abstracting the complexities of EKS cluster management, this approach ensures the flexibility required by enterprise workloads, accommodating everything from simple development clusters to multi-tenant production environments.



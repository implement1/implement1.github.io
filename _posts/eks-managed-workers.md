Production Amazon EKS Implementation: A Complete Guide to Enterprise Kubernetes on AWS

  Introduction

  Deploying production-ready Kubernetes infrastructure requires careful consideration of security, scalability, operational excellence, and cost optimization. Amazon Elastic
  Kubernetes Service (EKS) provides a managed Kubernetes control plane while giving you full control over worker nodes and applications. This comprehensive guide demonstrates
  how to implement enterprise-grade EKS clusters using Infrastructure as Code, showcasing real production patterns and best practices.

  Understanding EKS Architecture in Production

  Amazon EKS separates the Kubernetes control plane from worker nodes, allowing AWS to manage the control plane while you maintain control over your worker infrastructure. This
   separation enables you to focus on application deployment and scaling while AWS handles cluster management, security patches, and high availability.

  Key Production Benefits

  - Managed Control Plane: AWS handles etcd, API server, and controller manager
  - Multi-AZ High Availability: Control plane automatically distributed across AZs
  - Integrated Security: Native integration with IAM, VPC security groups, and AWS security services
  - Flexible Worker Options: Choice between managed node groups, self-managed nodes, or Fargate

  Production Architecture Overview

  Our production implementation showcases a secure, scalable EKS architecture:

  ┌─────────────────────────────────────────────────────────────────┐
  │                  Production VPC (10.0.0.0/18)                  │
  ├─────────────────────────────────────────────────────────────────┤
  │  ┌─────────────────┐    ┌─────────────────┐    ┌──────────────┐ │
  │  │  Public Subnet  │    │ Private Subnet  │    │Private Subnet│ │
  │  │   AZ-1a         │    │   AZ-1a         │    │   AZ-1b      │ │
  │  │  (NAT Gateway)  │    │ (EKS Workers)   │    │(EKS Workers) │ │
  │  └─────────────────┘    └─────────────────┘    └──────────────┘ │
  │           │                       │                       │     │
  │           ▼                       ▼                       ▼     │
  │    ┌─────────────┐         ┌─────────────┐         ┌─────────┐  │
  │    │     ALB     │         │ EKS Worker  │         │EKS Worker│ │
  │    │   Ingress   │◄────────┤   Node      │◄────────┤  Node   │ │
  │    │ Controller  │         │             │         │         │ │
  │    └─────────────┘         └─────────────┘         └─────────┘  │
  └─────────────────────────────────────────────────────────────────┘
                │
                ▼
       ┌─────────────────┐
       │  EKS Control    │
       │    Plane        │
       │ (AWS Managed)   │
       └─────────────────┘

  Core Infrastructure Implementation

  1. VPC Foundation with EKS-Specific Tagging

  EKS requires specific VPC and subnet tagging for proper resource discovery:

  module "vpc_app" {
    source     = "git::git@github.com:implement1/terraform-aws-vpc.git//modules/vpc-app"
    vpc_name   = var.vpc_name
    aws_region = var.aws_region

    # EKS-specific tags applied to VPC and subnets
    custom_tags = module.vpc_tags.vpc_eks_tags
    
    public_subnet_custom_tags              = module.vpc_tags.vpc_public_subnet_eks_tags
    private_app_subnet_custom_tags         = module.vpc_tags.vpc_private_app_subnet_eks_tags
    private_persistence_subnet_custom_tags = module.vpc_tags.vpc_private_persistence_subnet_eks_tags

    cidr_block = "10.0.0.0/18"  # /18 provides 16,384 IPs
    num_nat_gateways = 1        # Single NAT for cost optimization
  }

  module "vpc_tags" {
    source = "git::git@github.com:implement1/terraform-aws-eks.git//modules/eks-vpc-tags"
    eks_cluster_names = [var.eks_cluster_name]
  }

  Key VPC Design Considerations:
  - CIDR Block: /18 provides sufficient IP space for large-scale deployments
  - EKS Tagging: Automatic tagging ensures proper ALB and service discovery
  - Multi-AZ Distribution: Subnets span multiple availability zones for high availability
  - Cost Optimization: Single NAT Gateway for non-production, multiple for production

  2. EKS Control Plane Configuration

  The control plane module provides enterprise-grade cluster management:

  module "eks_cluster" {
    source       = "git::git@github.com:implement1/terraform-aws-eks.git//modules/eks-cluster-control-plane"
    cluster_name = var.eks_cluster_name

    vpc_id                       = module.vpc_app.vpc_id
    vpc_control_plane_subnet_ids = local.usable_subnet_ids

    kubernetes_version  = var.kubernetes_version
    kubectl_config_path = var.kubectl_config_path

    # Security configuration
    endpoint_public_access       = var.endpoint_public_access
    endpoint_public_access_cidrs = var.endpoint_public_access_cidrs
    enabled_cluster_log_types    = ["api", "audit", "authenticator", "controllerManager", "scheduler"]

    # CNI optimization for high pod density
    vpc_cni_enable_prefix_delegation = var.vpc_cni_enable_prefix_delegation
    vpc_cni_warm_ip_target           = var.vpc_cni_warm_ip_target
    vpc_cni_minimum_ip_target        = var.vpc_cni_minimum_ip_target

    # EKS Add-ons for enhanced functionality
    enable_eks_addons = var.enable_eks_addons
    eks_addons = var.eks_addons
  }

  Production Control Plane Features:
  - Comprehensive Logging: All control plane logs sent to CloudWatch for troubleshooting
  - CNI Prefix Delegation: Increases pod density per node by up to 10x
  - EKS Add-ons: Managed versions of core cluster components
  - Security: Private endpoint access with controlled public access

  3. Managed Worker Node Groups

  The managed workers module showcases enterprise worker node management:

  module "eks_workers" {
    source       = "git::git@github.com:implement1/terraform-aws-eks.git//modules/eks-cluster-managed-workers"
    cluster_name = module.eks_cluster.eks_cluster_name

    # Multi-AZ node group configuration
    node_group_configurations = {
      group1 = {
        desired_size = 1
        min_size     = 1
        max_size     = 4
        instance_types = ["t3.medium"]
        subnet_ids     = [local.usable_subnet_ids[0]]
        capacity_type  = "ON_DEMAND"  # or "SPOT" for cost optimization

        # Node labeling for workload scheduling
        labels = {
          "node.kubernetes.io/instance-type" = "t3.medium"
          "topology.kubernetes.io/zone" = local.usable_subnet_azs[0]
        }
      }

      group2 = {
        desired_size = 1
        min_size     = 1
        max_size     = 4
        instance_types = ["t3.medium"]
        subnet_ids     = [local.usable_subnet_ids[1]]
        capacity_type  = "ON_DEMAND"

        labels = {
          "node.kubernetes.io/instance-type" = "t3.medium"
          "topology.kubernetes.io/zone" = local.usable_subnet_azs[1]
        }
      }
    }

    cluster_instance_keypair_name = var.cluster_instance_keypair_name
  }

  Managed Node Group Advantages:
  - Automatic Updates: AWS manages AMI updates and security patches
  - Graceful Scaling: Built-in support for draining nodes during scale-down
  - Security Groups: Automatic security group configuration
  - Multi-AZ Distribution: Enhanced availability through zone distribution

  Advanced Production Patterns

  1. Dynamic Kubernetes Provider Configuration

  The implementation handles the chicken-and-egg problem of configuring the Kubernetes provider:

  # Dynamic provider configuration using data sources
  provider "kubernetes" {
    host                   = data.aws_eks_cluster.cluster.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)

    # Short-lived token authentication
    exec {
      api_version = "client.authentication.k8s.io/v1alpha1"
      command     = "aws"
      args = ["eks", "get-token", "--cluster-name", module.eks_cluster.eks_cluster_name]
    }
  }

  # Data source to avoid provider configuration dependency issues
  data "aws_eks_cluster" "cluster" {
    name = module.eks_cluster.eks_cluster_name
  }

  Configuration Benefits:
  - Token Refresh: Automatic token refresh prevents authentication failures
  - Bootstrap Solution: Resolves circular dependency between cluster and provider
  - Security: Short-lived tokens instead of static credentials

  2. Launch Template Customization

  For advanced worker node customization:

  resource "aws_launch_template" "template" {
    count         = var.use_launch_template ? 1 : 0
    name_prefix   = var.eks_cluster_name
    image_id      = var.launch_template_ami_id
    instance_type = "t3.micro"

    # Custom user data for node initialization
    user_data = base64encode(templatefile(
      "${path.module}/user-data/user_data.sh",
      {
        eks_cluster_name          = var.eks_cluster_name
        eks_endpoint              = module.eks_cluster.eks_cluster_endpoint
        eks_certificate_authority = module.eks_cluster.eks_cluster_certificate_authority
      }
    ))

    # Security hardening
    network_interfaces {
      associate_public_ip_address = false
    }

    metadata_options {
      http_endpoint = "enabled"
      http_tokens   = "required"  # IMDSv2 enforcement
    }
  }

  Launch Template Benefits:
  - Custom AMIs: Support for hardened or customized AMIs
  - Advanced Configuration: Boot scripts and specialized configurations
  - Security: IMDSv2 enforcement and network isolation

  3. Cluster Autoscaler Integration

  Automatic scaling based on workload demands:

  # Data source to discover autoscaling groups created by managed node groups
  data "aws_autoscaling_groups" "autoscaling_workers" {
    filter {
      name   = "key"
      values = ["k8s.io/cluster-autoscaler/${module.eks_cluster.eks_cluster_name}"]
    }
    depends_on = [module.eks_workers]
  }

  # IAM policy for cluster autoscaler
  module "k8s_cluster_autoscaler_iam_policy" {
    source              = "git::git@github.com:implement1/terraform-aws-eks.git//modules/eks-k8s-cluster-autoscaler-iam-policy"
    name_prefix         = module.eks_cluster.eks_cluster_name
    eks_worker_asg_arns = data.aws_autoscaling_groups.autoscaling_workers.arns
  }

  # Attach policy to worker nodes
  resource "aws_iam_role_policy_attachment" "worker_k8s_cluster_autoscaler_policy_attachment" {
    role       = module.eks_workers.eks_worker_iam_role_name
    policy_arn = module.k8s_cluster_autoscaler_iam_policy.k8s_cluster_autoscaler_policy_arn
  }

  Autoscaling Benefits:
  - Resource Optimization: Automatic scaling based on pending pods
  - Cost Efficiency: Scale down unused nodes to reduce costs
  - Application Availability: Ensure sufficient capacity for workloads

  Production Security Implementation

  1. Network Security Architecture

  # Private subnet deployment
  locals {
    usable_subnet_ids = module.vpc_app.private_app_subnet_ids

    # Ensure multi-AZ distribution
    usable_subnet_azs = [
      for subnet_id in local.usable_subnet_ids :
      data.aws_subnet.subnets[subnet_id].availability_zone
    ]
  }

  # Security group configuration through managed workers
  module "eks_workers" {
    # ... other configuration
    
    # Controlled SSH access
    allow_ssh_from_security_groups = var.enable_worker_ssh ? [
      aws_security_group.bastion[0].id
    ] : []
  }

  2. IAM Roles and Policies

  Managed node groups automatically configure required IAM roles:

  # Automatic IAM role configuration in managed workers module
  resource "aws_iam_role_policy_attachment" "worker_AmazonEKSWorkerNodePolicy" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
    role       = aws_iam_role.workers.name
  }

  resource "aws_iam_role_policy_attachment" "worker_AmazonEKS_CNI_Policy" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
    role       = aws_iam_role.workers.name
  }

  resource "aws_iam_role_policy_attachment" "worker_AmazonEC2ContainerRegistryReadOnly" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
    role       = aws_iam_role.workers.name
  }

  3. Pod-Level Security with Service Accounts

  # Example: Service account with IAM role for AWS service access
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: aws-load-balancer-controller
    namespace: kube-system
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/AmazonEKSLoadBalancerControllerRole

  Monitoring and Observability

  1. Control Plane Logging

  # Comprehensive logging configuration
  enabled_cluster_log_types = [
    "api",             # API server logs
    "audit",           # Kubernetes audit logs  
    "authenticator",   # IAM authenticator logs
    "controllerManager", # Controller manager logs
    "scheduler"        # Scheduler logs
  ]

  2. Worker Node Monitoring

  # CloudWatch agent for worker nodes
  module "cloudwatch_agent" {
    source = "git::git@github.com:implement1/terraform-aws-eks.git//modules/eks-cloudwatch-agent"
    
    cluster_name = module.eks_cluster.eks_cluster_name
    
    # Custom metrics configuration
    cloudwatch_config = {
      metrics = {
        namespace = "EKS/Cluster"
        metrics_collected = {
          cpu = {
            measurement = ["cpu_usage_idle", "cpu_usage_iowait"]
            metrics_collection_interval = 60
          }
          disk = {
            measurement = ["used_percent"]
            metrics_collection_interval = 60
            resources = ["*"]
          }
          mem = {
            measurement = ["mem_used_percent"]
            metrics_collection_interval = 60
          }
        }
      }
    }
  }

  3. Application-Level Monitoring

  # Prometheus and Grafana deployment
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: prometheus-config
  data:
    prometheus.yml: |
      global:
        scrape_interval: 15s
      scrape_configs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true

  Operational Excellence

  1. Blue-Green Deployments for Node Groups

  The managed workers module supports blue-green deployments:

  # Production update strategy
  node_group_configurations = {
    # Current production group
    production_v1 = {
      desired_size = 3
      min_size     = 1
      max_size     = 6
      instance_types = ["t3.medium"]
      subnet_ids = local.usable_subnet_ids
    }

    # New version for blue-green deployment
    production_v2 = {
      desired_size = 0  # Start scaled down
      min_size     = 0
      max_size     = 6
      instance_types = ["t3.large"]  # Upgraded instance type
      subnet_ids = local.usable_subnet_ids
    }
  }

  Update Process:
  1. Scale up new node group (production_v2)
  2. Cordon and drain old nodes
  3. Migrate workloads to new nodes
  4. Scale down old node group (production_v1)

  2. Disaster Recovery and Backup

  # Multi-region EKS setup for disaster recovery
  module "primary_cluster" {
    source = "./modules/eks-cluster"
    
    cluster_name = "production-primary"
    aws_region   = "us-west-2"
    # ... configuration
  }

  module "disaster_recovery_cluster" {
    source = "./modules/eks-cluster"
    providers = {
      aws = aws.disaster_recovery
    }

    cluster_name = "production-dr"
    aws_region   = "us-east-1"
    # ... configuration
  }

  3. Cost Optimization Strategies

  # Mixed instance types and spot instances
  node_group_configurations = {
    spot_group = {
      desired_size = 2
      min_size     = 0
      max_size     = 10
      instance_types = ["t3.medium", "t3.large", "t3.xlarge"]
      capacity_type = "SPOT"

      # Spot instance configuration
      tags = {
        "k8s.io/cluster-autoscaler/node-template/label/node-type" = "spot"
      }
    }

    on_demand_group = {
      desired_size = 1
      min_size     = 1
      max_size     = 3
      instance_types = ["t3.medium"]
      capacity_type = "ON_DEMAND"

      # Critical workloads on on-demand instances
      tags = {
        "k8s.io/cluster-autoscaler/node-template/label/node-type" = "on-demand"
      }
    }
  }

  Advanced Add-ons and Integrations

  1. AWS Load Balancer Controller

  # ALB Ingress Controller for production workloads
  module "alb_ingress_controller" {
    source = "git::git@github.com:implement1/terraform-aws-eks.git//modules/eks-alb-ingress-controller"

    cluster_name = module.eks_cluster.eks_cluster_name

    # IAM role for service account
    iam_role_for_service_accounts_eks_oidc_provider_arn = module.eks_cluster.eks_oidc_provider_arn

    # VPC configuration
    vpc_id = module.vpc_app.vpc_id
  }

  2. External DNS Integration

  # Automatic DNS record management
  module "external_dns" {
    source = "git::git@github.com:implement1/terraform-aws-eks.git//modules/eks-k8s-external-dns"
    
    cluster_name = module.eks_cluster.eks_cluster_name
    
    # Route53 configuration
    route53_hosted_zone_ids = [var.route53_hosted_zone_id]
    
    # IAM role configuration
    iam_role_for_service_accounts_eks_oidc_provider_arn = module.eks_cluster.eks_oidc_provider_arn
  }

  3. Container Insights

  # CloudWatch Container Insights
  module "container_logs" {
    source = "git::git@github.com:implement1/terraform-aws-eks.git//modules/eks-container-logs"
    
    cluster_name = module.eks_cluster.eks_cluster_name
    
    # Log retention configuration
    cloudwatch_log_group_retention_in_days = 30

    # Performance monitoring
    enable_container_insights = true
  }

  Production Deployment Strategies

  1. Multi-Environment Pipeline

  # Development environment
  module "eks_dev" {
    source = "./modules/eks-cluster"
    
    cluster_name = "development"
    kubernetes_version = "1.28"
    
    # Cost-optimized configuration
    node_group_configurations = {
      dev = {
        desired_size = 1
        min_size     = 1
        max_size     = 3
        instance_types = ["t3.small"]
        capacity_type = "SPOT"
      }
    }
  }

  # Production environment
  module "eks_prod" {
    source = "./modules/eks-cluster"
    
    cluster_name = "production"
    kubernetes_version = "1.28"
    
    # High-availability configuration
    node_group_configurations = {
      prod_az1 = {
        desired_size = 2
        min_size     = 2
        max_size     = 10
        instance_types = ["t3.large"]
        capacity_type = "ON_DEMAND"
        subnet_ids = [local.usable_subnet_ids[0]]
      }
      prod_az2 = {
        desired_size = 2
        min_size     = 2
        max_size     = 10
        instance_types = ["t3.large"]
        capacity_type = "ON_DEMAND"
        subnet_ids = [local.usable_subnet_ids[1]]
      }
    }
  }

  2. GitOps Integration

  # ArgoCD configuration for continuous deployment
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: production-app
    namespace: argocd
  spec:
    project: default
    source:
      repoURL: https://github.com/company/k8s-manifests
      targetRevision: production
      path: applications/production
    destination:
      server: https://kubernetes.default.svc
      namespace: production
    syncPolicy:
      automated:
        prune: true
        selfHeal: true
      syncOptions:
      - CreateNamespace=true

  Troubleshooting and Known Issues

  1. ENI Management

  The README highlights a known issue with ENI cleanup:

  # Cleanup script for ENI issues during destroy
  #!/bin/bash
  echo "Cleaning up ENIs for cluster: $CLUSTER_NAME"

  # Find and delete ENIs associated with the cluster
  aws ec2 describe-network-interfaces \
    --filters "Name=description,Values=*$CLUSTER_NAME*" \
    --query 'NetworkInterfaces[*].NetworkInterfaceId' \
    --output text | \
  while read eni_id; do
    if [ ! -z "$eni_id" ]; then
      echo "Deleting ENI: $eni_id"
      aws ec2 delete-network-interface --network-interface-id $eni_id
    fi
  done

  2. Cluster Upgrade Strategy

  # Controlled cluster upgrade process
  resource "aws_eks_cluster" "main" {
    # ... other configuration

    version = var.kubernetes_version
    
    # Upgrade configuration
    upgrade_policy {
      support_type = "EXTENDED"  # Extended support for older versions
    }

    # Lifecycle management
    lifecycle {
      ignore_changes = [version]  # Manual version upgrades
    }
  }

  Conclusion

  This EKS implementation demonstrates enterprise-grade Kubernetes deployment patterns that address real-world containerized application challenges. The modular Terraform
  architecture provides:

  - Production Readiness: Multi-AZ deployment, managed node groups, and comprehensive monitoring
  - Security Excellence: Private networking, IAM integration, and security best practices
  - Operational Efficiency: Automated scaling, blue-green deployments, and managed updates
  - Cost Optimization: Spot instances, right-sizing, and resource optimization

  For Cloud Engineers: The implementation showcases advanced AWS service integration, Infrastructure as Code best practices, and production-ready security configurations.

  For DevOps Professionals: Automated scaling, comprehensive monitoring, and deployment automation capabilities reduce operational overhead while ensuring reliability.

  For Platform Teams: The modular design enables consistent deployments across environments while maintaining flexibility for specific requirements.

  By following these patterns and best practices, organizations can confidently deploy Kubernetes infrastructure that meets enterprise requirements for security, scalability,
  and reliability while leveraging AWS's managed services to reduce operational complexity.

  The combination of EKS's managed control plane, flexible worker node options, and comprehensive ecosystem integration makes it an ideal platform for running production
  containerized workloads at scale.

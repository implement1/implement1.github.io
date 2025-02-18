
# Automating EKS Clusters Using Terraform

This README provides an overview of automating the deployment and management of Amazon Elastic Kubernetes Service (EKS) clusters using Terraform. It covers the necessary steps to set up your environment, define your infrastructure, and ensure best practices for security and resource management.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Provider Configuration](#provider-configuration)
- [Creating a VPC](#creating-a-vpc)
- [Configuring the EKS Control Plane](#configuring-the-eks-control-plane)
- [Setting Up EKS Worker Pools](#setting-up-eks-worker-pools)
- [Security Considerations](#security-considerations)
- [IAM Role Mapping](#iam-role-mapping)
- [Conclusion](#conclusion)

## Prerequisites

Before you begin, ensure you have the following:

- [Terraform](https://www.terraform.io/downloads.html) installed.
- An AWS account with permissions to create EKS clusters and associated resources.
- AWS CLI configured with your credentials.

## Provider Configuration

Define a provider block to set up authentication credentials and secure connection details to connect Terraform to the EKS cluster.

```hcl
provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)

  exec {
    api_version = "client.authentication.k8s.io/v1alpha1"
    command     = "aws"
    args        = ["eks", "get-token", "--cluster-name", eks_cluster.eks_cluster_name]
  }
}
```
## Creating a VPC
Define a module designed to create a VPC and its related networking infrastructure in AWS, configured specifically for use with EKS. This VPC must be tagged to denote that it is shared with Kubernetes.
```hcl
module "vpc_app" {
  source = "aws-ia/vpc/aws"
  name   = "multi-az-vpc"
  cidr_block = "10.0.0.0/16"
  vpc_egress_only_internet_gateway = true
  az_count = 3
  custom_tags = module.vpc_tags.vpc_eks_tags
}
```
## Configuring the EKS Control Plane
Configure the control plane of the EKS cluster with specified parameters for networking, Kubernetes version, and additional add-ons.
```hcl
module "eks_cluster" {
  source = "terraform-aws-modules/eks/aws/eks-cluster-control-plane"
  cluster_name                  = var.eks_cluster_name
  vpc_id                        = module.vpc_app.vpc_id
  control_plane_subnet_ids      = local.usable_subnet_ids
  public_access_cidrs           = var.endpoint_public_access_cidrs
  kubernetes_version             = var.kubernetes_version
  eks_addons                     = var.eks_addons
  upgrade_script                 = false
}
```
## Setting Up EKS Worker Pools
We will create two distinct node pools:

workers: Dedicated to running application pods.
core_workers: Designed for running core services.
Why Use Multiple Node Pools?
Using multiple node pools enhances security and resource management by separating application workloads from core services. This allows for stricter security measures on core worker nodes.

## Locking Down Core Worker Nodes
The core_workers node pool will be restricted to authorized entities. Here are some strategies to achieve this:

Implement Security Groups: Limit inbound traffic to the core worker nodes.
Use IAM Roles: Assign IAM roles with fine-grained permissions.
Enable Private Access: Use private subnets for core worker nodes.
```hcl
module "eks_workers" {
  source = "terraform-aws-modules/eks/aws/eks-cluster-workers"
  name_prefix                = "applications-"
  cluster_name               = eks_cluster_name
  cluster_security_group      = true
  
  autoscaling_group_configurations = {
    asg = {
      min_size          = 1
      max_size          = 4
      asg_instance_type = "t3.medium"
      subnet_ids        = local.usable_subnet_ids
      
      tags = [
        {
          key                 = "type"
          value               = "application"
          propagate_at_launch = true
        }
      ]
    }
  }
  
  autoscaler_discovery_tags            = true
  asg_default_instance_user_data_base64 = base64encode(local.app_workers_user_data)
  keypair_name                         = var.keypair_name
  associate_public_ip_address          = true
}
```
## Security Considerations
For testing purposes, you may allow SSH from anywhere to the worker nodes. THIS SHOULD NOT BE DONE IN PROD.

```hcl
resource "aws_security_group_rule" "allow_ssh_from_anywhere" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = eks_worker_security_group_id
}
```
Similarly, allow access to node ports on the worker nodes for testing purposes only. THIS SHOULD NOT BE DONE IN PROD. INSTEAD, USE LOAD BALANCERS.
```hcl
resource "aws_security_group_rule" "allow_node_port_from_anywhere" {
  type              = "ingress"
  from_port         = 30000
  to_port           = 32767
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = eks_worker_security_group_id
}
```
## IAM Role Mapping
Create a mapping of IAM roles to Kubernetes RBAC groups in the EKS cluster to control access to the Kubernetes API and manage permissions for users and services effectively.

```hcl
module "eks_k8s_role_mapping" {
  source = "terraform-aws-modules/eks/aws/role-mapping"
  worker_role_arns = [eks_worker_iam_role_arn]
  
  role_rbac_group_mappings = {
    (local.real_arn) = ["system:masters"]
  }
  
  config_map = {
    "eks-cluster" = eks_cluster_name
  }
}
```
Conclusion
By following these guidelines, you can automate the deployment and management of EKS clusters effectively while adhering to best practices in security and resource management.

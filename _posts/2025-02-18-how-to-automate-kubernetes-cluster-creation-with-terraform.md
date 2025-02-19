
# Automating EKS Clusters Using Terraform

This post walks through the process of automating the deployment of Amazon Elastic Kubernetes Service (EKS) cluster using Terraform. It covers the steps to set up the environment, define the infrastructure, and ensure best practices.

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
- **Terraform** 
- **Packer**
- **(Optional) kubectl**
- An AWS account and IAM user with permissions to create EKS clusters and associated resources.
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
  source                          = "aws-ia/vpc/aws"

  name                            = "multi-az-vpc"
  cidr_block                      = "10.0.0.0/16"
  vpc_egress_only_internet_gateway = true
  az_count                        = 3
  custom_tags                     = module.vpc_tags.vpc_eks_tags
}
```
## Configuring the EKS Control Plane
Configure the control plane of the EKS cluster with specified parameters for networking, Kubernetes version, and additional add-ons such as core-dns, vpc-cni and kube-proxy.

```hcl
module "eks_cluster" {
  source                         = "terraform-aws-modules/eks/aws/eks-cluster-control-plane"

  cluster_name                   = var.eks_cluster_name

  vpc_id                         = module.vpc_app.vpc_id
  control_plane_subnet_ids       = local.usable_subnet_ids
  public_access_cidrs            = var.endpoint_public_access_cidrs
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

```hcl
module "eks_workers" {
  source                          = "terraform-aws-modules/eks/aws/eks-cluster-workers"
  name_prefix                    = "applications-"
  cluster_name                   = eks_cluster_name
  cluster_security_group          = true
  
  autoscaling_group_configurations = {
    asg = {
      min_size           = 1
      max_size           = 4
      asg_instance_type  = "t3.medium"
      subnet_ids         = local.usable_subnet_ids
      
      tags = [
        {
          key                 = "type"
          value               = "application"
          propagate_at_launch = true
        }
      ]
    }
  }
}
```
## Locking Down Core Worker Nodes
The core_workers node pool will be restricted to authorized entities. Here are some strategies to achieve this:
-  **Implement Security Groups**: Limit inbound traffic to the core worker nodes.
-  **Use IAM Roles**: Assign IAM roles with fine-grained permissions.
-  **Enable Private Access**: Use private subnets for core worker nodes.

## Security Considerations
For testing purposes, allow SSH from anywhere to the worker nodes. THIS SHOULD NOT BE DONE IN LIVE ENVIRONMENT.

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
Similarly, allow access to node ports on the worker nodes for testing purposes. THIS SHOULD NOT BE DONE IN LIVE ENVIRONMENT. INSTEAD, WE USE LOAD BALANCERS IN SUCH ENVIRONMENTS.

```hcl
resource "aws_security_group_rule" "allow_node_port_from_anywhere" {
  type                = "ingress"
  from_port           = 30000
  to_port             = 32767
  protocol            = "tcp"
  cidr_blocks         = ["0.0.0.0/0"]
  security_group_id   = eks_worker_security_group_id
}
```
## IAM Role Mapping
We create a mapping of IAM roles to Kubernetes RBAC groups in the EKS cluster to control access to the Kubernetes API and manage permissions for users and services.

```hcl
module "eks_k8s_role_mapping" {
  source                     = "terraform-aws-modules/eks/aws/role-mapping"
  worker_role_arns          = [eks_worker_iam_role_arn]
  
  role_rbac_group_mappings = {
    (local.real_arn) = ["system:masters"]
  }

  config_map = {
    "eks-cluster" = eks_cluster_name
  }
}
```
The following output provides a snapshot of the pods running in the kube system namespace indicating that the cluster is functionning normally.
```hcl
 kubectl get pods -n kube-system
 NAME                                  READY    STATUS      RESTARTS    AGE
 aws-node-h4k7v                   2/2       Running    0                18m
 aws-node-wtb2b                   2/2       Running    0                18m
 coredns-54d6f577c6-72nxn         1/1       Running    0                22m
 coredns-54d6f577c6-bhdhj         1/1       Running    0                22m
 kube-proxy-8\82s                 1/1       Running    0                18m
 kube-proxy-9hwqh                 1/1       Running    0                18m
```

## Conclusion
This post outlines the steps involved in automating the deployment and management of EKS clusters while adhering to best practices in security and resource management. In the next post, we will be covering the deployment of core services including - **FluentD**, **ALB Ingress Controller** and **External-DNS**.

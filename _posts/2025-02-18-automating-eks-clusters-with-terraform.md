“format this text in markdown format as a blog to be posted on github page”
Automating eks clusters using terraform
Define a provider block to set up authentication credentials and secure connection details to connect Terraform to the EKS cluster

 provider "kubernetes" {
  host                   = data.aws_eks_cluster.cluster.endpoint
  cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority.0.data)
  exec {
    api_version = "client.authentication.k8s.io/v1alpha1"
    command     = "aws"
    args = (
       ["eks", "get-token", "--cluster-name", eks_cluster.eks_cluster_name]
    )
  }
}

Define a module designed to create a VPC and its related networking infrastructure in AWS, configured specifically for use with Elastic Kubernetes Service (EKS) with custom tagging for resource identification and CIDR block for the VPC.
We will provision a new VPC because EKS requires a VPC that is tagged to 
denote that it is shared with Kubernetes. Specifically, both the VPC and 
the subnets that EKS resides in need to be tagged with
kubernetes.io/cluster/EKS_CLUSTER_NAME=shared

module "vpc_app" {
  source = “aws”-ia/vpc/aws”

  name = “multi-az-vpc”
  cidr_block = “10.0.0.0/16”
  vpc_egress_only_internet_gateway = true
  az_count = 3
  custom_tags = module.vpc_tags.vpc_eks_tags
}

Then we configure the control plane of EKS cluster with specified parameters for networking, Kubernetes version, and additional add-ons. We also manage the behavior of the upgrade process to ensure smooth deployments

module "eks_cluster" {
  source = "terraform-aws-modules/eks/aws/eks-cluster-control-plane"

  cluster_name                 	= var.eks_cluster_name
  vpc_id                       		= module.vpc_app.vpc_id
  control_plane_subnet_ids = local.usable_subnet_ids
  public_access_cidrs 		= var.endpoint_public_access_cidrs
  kubernetes_version           = var.kubernetes_version
  eks_addons 			= var.eks_addons

  upgrade_script = false
}


# Setting Up EKS Worker Pools

We will create two distinct node pools:

- **workers**: This node pool is dedicated to running application pods.
- **core_workers**: This node pool is designed for running core services.

## Why Use Multiple Node Pools?

The primary reason for using multiple node pools is to enhance security and resource management. By separating application workloads from core services, we can apply stricter security measures to the core worker nodes. This ensures that only trusted entities can access them, which is crucial since these nodes will be running sensitive pods.

### Locking Down Core Worker Nodes

The **core_workers** node pool will be locked down to restrict access. This means that only athorized entities will have the ability to perform actions such as SSH access. Here are some strategies to achieve this:

- **Implement Security Groups**: Create security groups that limit inbound traffic to the core worker nodes.
- **Use IAM Roles**: Assign IAM roles with fine-grained permissions to control access to the EKS cluster.
- **Enable Private Access**: Consider using private subnets for core worker nodes to prevent external access.

module "eks_workers" {
  # When using these modules in your own templates, you will need to use a 
  # Git URL with a ref attribute that pins you to a specific version of the 
  # modules.
  source = "terraform-aws-modules/eks/aws/eks-cluster-workers"

  name_prefix                = "applications-“
  cluster_name               = eks_cluster_name
  cluster_security_group = true

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
  asg_default_instance_user_data_base64        = base64encode(local.app_workers_user_data)
  keypair_name                = var.keypair_name
  associate_public_ip_address = true
}

Then we Allow SSH from anywhere to the worker nodes for test purposes only.
THIS SHOULD NOT BE DONE IN PROD.

resource "aws_security_group_rule" "allow_ssh_from_anywhere" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = eks_worker_security_group_id
}

Then we Allow access to node ports on the worker nodes for test purposes only.
# THIS SHOULD NOT BE DONE IN PROD. INSTEAD USE LOAD BALANCERS.

resource "aws_security_group_rule" "allow_node_port_from_anywhere" {
  type              = "ingress"
  from_port         = 30000
  to_port           = 32767
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = eks_worker_security_group_id
}

Then we create a mapping of IAM roles to Kubernetes RBAC groups in the EKS cluster to control access to the Kubernetes API and manage permissions for users and services effectively. 

module "eks_k8s_role_mapping" {
  source = "terraform-aws-modules/eks/aws/role-mapping"

worker_role_arns = [ eks_worker_iam_role_arn ]

  role_rbac_group_mappings = {
    (local.real_arn) = ["system:masters"]
  }

  config_map = {
    "eks-cluster" = eks_cluster_name
  }
}

## Deploying core services on Amazon Elastic Kubernetes Services EKS Cluster

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Conclusion](#Conclusion)

## Introduction

This module configures:

- A CloudWatch log group for the EKS cluster.
- Fluent Bit to collect and send container logs to CloudWatch.
- CloudWatch Agent for monitoring.
- AWS Application Load Balancer Ingress Controller for managing external access to services.
- Kubernetes Cluster Autoscaler for dynamic scaling based on workload demands.
- External DNS

## Prerequisites

- Terraform installed on your local machine.
- An AWS account with permissions to create EKS clusters and associated resources.
- An existing EKS cluster.

## Configuration
### CloudWatch Log Group
This module creates CloudWatch log group for the EKS cluster and configure Fluent Bit to collect and send container logs to this log group. It also includes settings for service accounts, log stream configuration, and pod tolerations to manage where and how logging pods can be scheduled within the Kubernetes cluster.
```hcl
resource "aws_cloudwatch_log_group" "eks" {
  name = "${var.cluster_name}-logs"
}
```
### Fluent Bit Configuration
This module unifies the log streams in the Kubernetes cluster to be shipped to an aggregation service like Cloudwatch or Kinesis on AWS. It installs fluent-bit that monitors the log files, parses custom log formats and ships them to a log aggregation service. 
```hcl
module "fluent_bit" {
  source = "git::https://github.com/DNXLabs/terraform-aws-eks-fluentbit.git"
  
  service_accounts        = var.service_accounts_config
  name_prefix             = var.eks_cluster_name
  cloudwatch_configuration = {
    region            = var.aws_region
    log_group_name    = aws_cloudwatch_log_group.eks.name
    log_stream_prefix = null
  }
  
  tolerations = [
    {
      key      = "tenancy"
      operator = "Equal"
      value    = "main"
      effect   = "NoSchedule"
    },
  ]
}
```
### CloudWatch Agent Deployment
This module deploys the CloudWatch Agent into the EKS cluster. The agent collects system level metrics from EC2 instances and ship them to Cloudwatch.It also sets a taint (tenancy = main) to ensure that only pods with this toleration can be scheduled on core nodes.
```hcl
module "cloudwatch_agent" {
  source = "terraform-aws-modules/cloudwatch/aws"
  
  cluster_name    = var.cluster_name
  service_accounts = var.service_accounts_config

   tolerations = [
    {
      key      = "tenancy"
      operator = "Equal"
      value    = "main"
      effect   = "NoSchedule"
    },
  ]
}
```

### ALB Ingress Controller

This module deploys the ALB Ingress Controller to integrate Kubernetes Service endpoints with Application Load balancer. Ingress resources configure and implement Layer 7 load balancers.
It also sets a taint (tenancy = main) to ensure that only pods with this toleration can be scheduled on core nodes.
```hcl
module "ingress" {
  source = "terraform-aws-modules/alb/aws"
  
  cluster_name     = var.cluster_name
  vpc_id           = var.vpc_id
  aws_region       = var.aws_region
  service_accounts = var.service_accounts_config
  
  tolerations = [
    {
      key      = "tenancy"
      operator = "Equal"
      value    = "main"
      effect   = "NoSchedule"
    },
  ]
  
  affinity = [
    {
      key      = "ec2.amazonaws.com/type"
      operator = "In"
      values   = ["main"]
    },
  ]
}
```
### Kubernetes Cluster Autoscaler
This module configures the Kubernetes Cluster Autoscaler to manage the scaling of nodes in the EKS cluster based on workload demands. In this example, if a node is underutilized for 10 minutes, it can be considered for scaling down. Likewise, after adding a new node, the autoscaler will wait 10 minutes before considering scaling down any nodes. With tolerations for the autoscaler pods, they are allowed to be scheduled on nodes labeled with tenancy = main. The priorities map ensures that resource-intensive applications are scheduled on larger instances when available.
```hcl
module "autoscaler" {
  source = "lablabs/cluster-autoscaler/aws"
  
  aws_region                             = var.aws_region
  cluster_name                           = var.cluster_name
  service_accounts                       = var.service_accounts_config
  version                                = var.autoscaler_version
  
  priorities = {
    20 = [".*t3\\.small.*"]
    30 = [".*t3\\.medium.*"]
  }
  
  scaling_strategy = var.scaling_strategy
  
  container_extra_args = {
    scale-down-unneeded-time   = "10m"
    scale-down-delay-after-add  = "10m"
  }
  
  pod_tolerations = [
    {
      key      = "tenancy"
      operator = "Equal"
      value    = "main"
      effect   = "NoSchedule"
    },
  ]
 }
```

### External DNS

This module  links a known domain name to an Ingress endpoint managed by Kubernetes by configuring Route 53 Hosted Zones to point DNS records to Ingress endpoints.
```hcl
module "external_dns" {
  source = "lablabs/eks-external-dns/aws"

  aws_region                           = var.aws_region
  cluster_name                         = var.cluster_name
  txt_owner_id                         = var.eks_cluster_name
  service_accounts                     = var.service_accounts_config

  route53_record_update_policy = "sync"

  zone_filters     = var.route53_zone_id_filters
  zone_tag_filters    = var.route53_zone_tag_filters
  zone_domain_filters = var.route53_zone_domain_filters

  trigger_loop_on_event = true
  poll_interval         = “10m"
  batch_change_size     = 2000

  tolerations = [
    {
      "key"      = “tenancy”
      "operator" = "Equal"
      "value"    = “main”
      "effect"   = "NoSchedule"
    },
  ]
}
```
The following output provides a snapshot of the pods running in the kube system namespace indicating that the cluster is functionning normally.

## Conclusion
In this post, we deployed core services that are essential for operating a production grade EKS cluster. In the next post, we will deploy a sample application leveraging all the components previously deployed. This application will be exposed externally through the AWS application load balancer ALB and mapped to a custom domain name using external DNS.
  

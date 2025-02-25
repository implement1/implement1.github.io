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
  
  scaling_strategy = "priority"
  
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

This module  links a known domain name to an Ingress endpoint managed by Kubernetes by configuring Route 53 Hosted Zones to point DNS records to Ingress endpoints. This automates the process of mapping domain names to ingress endpoint, and avoids waiting for the resource to become available. 
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

```hcl
aws-alb-ingress-controller-aws-load-balancer-controller-6bv85sn   1/1     Running   0          19m
aws-cloudwatch-agent-aws-cloudwatch-metrics-25dl2                 1/1     Running   0          19m
aws-cloudwatch-agent-aws-cloudwatch-metrics-mr657                 1/1     Running   0          19m
aws-for-fluent-bit-9m5ds                                          1/1     Running   0          19m
aws-for-fluent-bit-szhsz                                          1/1     Running   0          19m
aws-node-6znhx                                                    2/2     Running   0          38m
aws-node-ltd2q                                                    2/2     Running   0          39m
cluster-autoscaler-aws-cluster-autoscaler-6cc76774c4-lc568        1/1     Running   0          19m
coredns-789f8477df-88dqz                                          1/1     Running   0          46m
coredns-789f8477df-f4s4l                                          1/1     Running   0          46m
external-dns-665dfdfd4-xhkdg                                      1/1     Running   0          19m
kube-proxy-cg7b5                                                  1/1     Running   0          38m
kube-proxy-p2cs8                                                  1/1     Running   0          39m
```
#### Fluent-bit pod Logs
```hcl
containerinsights","labels":{"AutoScalingGroupName":"","ClusterName":"test-eks","ContainerName":"coredns","FullPodName":"aws-for-fluent-bit-szhsz","Namespace":"kube-system","NodeName":"ip-10-0-3-119.ec2.internal","PodName":"aws-cloudwatch-agent-aws-cloudwatch-metrics","Service":"kube-dns","Sources":"[\"apiserver\"]","Timestamp":"1740376723904","Type":"ClusterService","Version":"0","container_status":"Running","device":"/dev/nvme0n1","fstype":"vfs","interface":"eni39ad90ef0ad","kubernetes":"{\"namespace_name\":\"kube-system\",\"service_name\":\"kube-dns\"}","pod_status":"Running"}}
```
#### External DNS pod Logs
```hcl
time="2025-02-24T05:48:44Z" level=info msg="Instantiating new Kubernetes client"
time="2025-02-24T05:48:44Z" level=info msg="Using inCluster-config based on serviceaccount-token"
time="2025-02-24T05:48:44Z" level=info msg="Created Kubernetes client https://172.20.0.1:443"
time="2025-02-24T05:48:45Z" level=info msg="Applying provider record filter for domains: [ cloudresolve.net. .cloudresolve.net.]"
time="2025-02-24T05:48:45Z" level=info msg="All records are already up to date"
time="2025-02-24T05:48:50Z" level=info msg="Applying provider record filter for domains: [ cloudresolve.net. .cloudresolve.net.]"
```
#### Cloudwatch agent pod Logs
```hcl
2025-02-24T05:58:48Z I! {"caller":"awsemfexporter@v0.84.0/emf_exporter.go:109","msg":"Start processing resource metrics","kind":"exporter","data_type":"metrics","name":"awsemf/
Cloudwatch agent containerinsights","labels":{"AutoScalingGroupName":"","ClusterName":"test-eks","ContainerName":"coredns","FullPodName":"aws-for-fluent-bit-szhsz","Namespace":"kube-system","NodeName":"ip-10-0-3-119.ec2.internal","PodName":"aws-cloudwatch-agent-aws-cloudwatch-metrics","Service":"kube-dns","Sources":"[\"apiserver\"]","Timestamp":"1740376723904","Type":"ClusterService","Version":"0","container_status":"Running","device":"/dev/nvme0n1","fstype":"vfs","interface":"eni39ad90ef0ad","kubernetes":"{\"namespace_name\":\"kube-system\",\"service_name\":\"kube-dns\"}","pod_status":"Running"}}
```
#### Cluster auto scaler pod Logs
```hcl
I0224 06:10:39.728519       1 auto_scaling_groups.go:351] Regenerating instance to ASG map for ASGs: []
I0224 06:10:39.728549       1 auto_scaling.go:199] 0 launch configurations already in cache
I0224 06:10:39.728558       1 aws_manager.go:268] Refreshed ASG list, next refresh after 2025-02-24 06:11:39.728554067 +0000 UTC m=+1387.584293828
I0224 06:10:39.728981       1 filter_out_schedulable.go:65] Filtering out schedulables
I0224 06:10:39.729052       1 filter_out_schedulable.go:132] Filtered out 0 pods using hints
I0224 06:10:39.729081       1 filter_out_schedulable.go:170] 0 pods were kept as unschedulable based on caching
I0224 06:10:39.729132       1 filter_out_schedulable.go:171] 0 pods marked as unschedulable can be scheduled.
I0224 06:10:39.729180       1 filter_out_schedulable.go:82] No schedulable pods
I0224 06:10:39.729250       1 static_autoscaler.go:420] No unschedulable pods
I0224 06:10:39.729303       1 static_autoscaler.go:467] Calculating unneeded nodes
I0224 06:10:39.729365       1 pre_filtering_processor.go:57] Skipping ip-10-0-3-141.ec2.internal - no node group config
I0224 06:10:39.729411       1 pre_filtering_processor.go:57] Skipping ip-10-0-3-119.ec2.internal - no node group config
I0224 06:10:39.730201       1 static_autoscaler.go:521] Scale down status: unneededOnly=false lastScaleUpTime=2025-02-24 04:48:35.392619789 +0000 UTC m=-3596.751640416 lastScaleDownDeleteTime=2025-02-24 04:48:35.392619789 +0000 UTC m=-3596.751640416 lastScaleDownFailTime=2025-02-24 04:48:35.392619789 +0000 UTC m=-3596.751640416 scaleDownForbidden=false isDeleteInProgress=false scaleDownInCooldown=false
```

## Conclusion
This post discussed the deployment of services that are essential for operating a production grade EKS cluster. With all the services in place, in the next post, we will deploy an application on this cluster leveraging all the components previously installed. This application will be exposed externally through the AWS application load balancer ALB and mapped to a custom domain name using external DNS.
  

# Deploying Nginx Web Server on Kubernetes using Helm

This guide outlines the steps to deploy a Dockerized Apache web server application using Helm and Terraform in a Kubernetes environment.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Setup Terraform Connection](#setup-terraform-connection)
- [Configure Helm Connection](#configure-helm-connection)
- [Deploy Nginx Server using Helm Chart](#deploy-apache-web-server-using-helm-chart)
- [Manage Hosted Zones for Ingress Domain](#manage-hosted-zones-for-ingress-domain)
- [Route 53 Hosted Zone Configuration](#route-53-hosted-zone-configuration)
- [Conclusion](#conclusion)

## Prerequisites

- **Terraform**: Terraform is needed to automate deployments.
- **Helm**: Helm must be installed (version ~> 3.0).
- **AWS EKS Cluster**: An already existing EKS cluster all the necessary components.
- **AWS CLI**: AWS CLI configured and IAM user with appropriate permissions to interact with the EKS Cluster.

## Setting up Terraform Connection

To establish a connection to the Helm provider, include the following configuration in your Terraform files:

```hcl
terraform {
  required_version = ">= 1.11.0"

  required_providers {
    helm = {
      source  = "hashicorp/helm"
      version = "~> 3.0"
    }
  }
}
```
## Configuring the Helm Connection to EKS Cluster
Configure the Helm provider to connect to your Kubernetes cluster:
```hcl
provider "helm" {
  kubernetes {
    host                   = data.aws_eks_cluster.cluster.endpoint
    cluster_ca_certificate = base64decode(data.aws_eks_cluster.cluster.certificate_authority[0].data)
    token                  = data.aws_eks_cluster_auth.kubernetes_token.token
  }
}
```
## Deploying Nginx Web Server using Helm Chart
Use the following resource block to deploy the Apache web server:
```hcl
resource "helm_release" "apache" {
  name       = var.application_name
  repository = "https://artifacthub.io"
  chart      = "apache"
  version    = "11.3.4"
  namespace  = "kube-system"

  values = [
    templatefile(
      "${path.module}/values.yaml",
      {
        ports    = "[{\"HTTP\": 80}, {\"HTTPS\": 443}]"
        app_name = var.application_name
        hosts    = jsonencode([local.domain_name])
      },
    ),
  ]
}
```
### values.yaml Configuration
The values.yaml file specifies:

- **Image repository** The container image repository that should be used.
- **Image tag** The tag of the image (e.g., 'latest') that should be used.
- **Image pull repository** The image pull policy to determine when the image will be pulled.
- **number of replicas** The number of replica pods that should be deployed and maintained at any given point in time.
- **Liveness probe** Liveness probes to indicate when the container needs to be restarted to recover.
- **Readiness probe** Readiness probes to indicate when the container is unable to serve traffic.
- **Ingress** Ingress annotations to configure the ALB Ingress controller:
  - Enables an ALB Ingress resource that routes traffic to specified hosts and paths.
  - Sets up an internet-facing ALB with SSL termination using a specified certificate.
  - Configures DNS settings with a TTL for external DNS.
  - Defines the ports on which the ALB listens and the backend service port to which traffic will be directed.

```hcl
image:
  repository: nginx
  tag: 1.12.1
  digest: ""
  pullPolicy: IfNotPresent

replicaCount: 1

livenessProbe:
  httpGet:
    path: /
    port: http

readinessProbe:
  httpGet:
    path: /
    port: http

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: alb
    external-dns.alpha.kubernetes.io/ttl: '60'
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '${ports}'
    alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:us-east-1:XXXXXXXXXXXX:certificate/11g4d0-85h9-4171-8c65-80c041f52b4f"
  path: /
  pathType: Prefix
  hosts: ${hosts}
```
## Defining Ingress Domain variable settings
These variables are used in the Terraform configuration to set up DNS records related to the Apache web server deployment. The variables construct a fully qualified domain name (FQDN) by concatenating the string "apache." with the hostname of the Route 53 hosted zone.
```hcl
locals {
  domain_name = "nginx.${local.dns_name}"
  dns_name    = replace(
    element(
      concat(data.aws_route53_zone.ingress.*.name, [""]),
      0,
    ),
    "/\\.$/",
    "",
  )
}
```
## Route 53 Hosted Zone data source
Configure the Route 53 hosted zone:
```hcl
data "aws_route53_zone" "ingress" {
  name = "cloudresolve.net"
  tags = {
    public-access = true
  }
}
```
![Nginx web server on EKS](https://drive.google.com/uc?export=view&id=1AH2EgeZQR9WLqUCYOmoL9I50uv9lAR_K)


## Conclusion
We have successfully deployed an Apache web server helm chart onto Elastic Kubernetes Services EKS Cluster. This application is exposed through the custom domain name apache.cloudresolve.net with external-dns automatically updating the route53 dns record. This application is being exposed through AWS ALB which  terminates SSL connections using the specified certificate and routes incoming requests to the appropriate backend service.

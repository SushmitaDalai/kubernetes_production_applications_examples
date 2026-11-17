# Terraform handoff

This directory corresponds to your existing `infrastructure-live/iac/eks` root. The reference consumes its contract. Keep backend configuration and Terraform variables under your existing environment directories alongside the example Helm values.

| Terraform or platform output | App consumer |
|---|---|
| Account, region, cluster name and private API connectivity | Release runner and EKS access entry |
| VPC, private subnets across three AZs and ALB subnet discovery | AWS Load Balancer Controller and ingress |
| EC2 node groups and inference labels/taints | Scheduling constraints |
| OIDC provider and inference IAM role ARN | IRSA service account |
| Regional ACM ARN and private DNS zone | Internal ALB and DNS record |
| CoreDNS, VPC CNI with policy enabled, EBS CSI, Load Balancer Controller | Services, NetworkPolicy, PVCs and Ingress |
| Monitoring stack and ServiceMonitor CRD | Metrics discovery |
| Regional ECR repositories and model buckets | Immutable image and model promotion |

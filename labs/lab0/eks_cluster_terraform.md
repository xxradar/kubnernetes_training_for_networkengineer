# Lab 0 — EKS Cluster with Terraform (default AWS CNI)

This lab creates a managed [Amazon EKS](https://aws.amazon.com/eks/) cluster with [Terraform](https://developer.hashicorp.com/terraform) using the community `terraform-aws-modules` for VPC and EKS. It provisions the same size cluster as the eksctl lab, using the **default AWS VPC CNI** for pod networking (no Calico).

> **Note:** This is the infrastructure-as-code alternative to `eks_cluster.md`. Terraform manages the full lifecycle — VPC, subnets, IAM, the EKS control plane, a managed node group, and the core add-ons (`vpc-cni`, `kube-proxy`, `coredns`) — and can tear it all down again with a single `destroy`.

> **Cost / time:** An EKS control plane plus 3× `t2.xlarge` nodes and a NAT gateway run real AWS charges, and `terraform apply` takes ~15 minutes. Always `terraform destroy` when finished.

**What you build**

| Component | Value |
|-----------|-------|
| Cloud / region | AWS EKS, `eu-central-1` |
| Cluster | `training-cluster-tf` (Kubernetes 1.36) |
| Node group | 3× `t2.xlarge` (min 3, max 5), EKS-managed |
| Networking (CNI) | Default AWS VPC CNI (`vpc-cni` add-on) |
| VPC | `192.168.0.0/16`, 3 public + 3 private subnets, single NAT gateway |
| Modules | `terraform-aws-modules/vpc/aws ~> 5.0`, `terraform-aws-modules/eks/aws ~> 20.0` |

---

## 1. Prerequisites

Install [Terraform](https://developer.hashicorp.com/terraform/install) and the AWS CLI, then confirm you are authenticated.

```bash
terraform version
aws sts get-caller-identity
```

---

## 2. Write the Terraform configuration

Create a working directory and a `main.tf`:

```bash
mkdir -p eks-terraform && cd eks-terraform
cat > main.tf <<'EOF'
terraform {
  required_version = ">= 1.3"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

variable "region" {
  default = "eu-central-1"
}

variable "cluster_name" {
  default = "training-cluster-tf"
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "training-vpc"
  cidr = "192.168.0.0/16"

  azs             = ["${var.region}a", "${var.region}b", "${var.region}c"]
  private_subnets = ["192.168.128.0/19", "192.168.160.0/19", "192.168.192.0/19"]
  public_subnets  = ["192.168.32.0/19", "192.168.64.0/19", "192.168.96.0/19"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags  = { "kubernetes.io/role/elb" = 1 }
  private_subnet_tags = { "kubernetes.io/role/internal-elb" = 1 }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name                   = var.cluster_name
  cluster_version                = "1.36"
  cluster_endpoint_public_access = true

  # Grant the IAM identity running Terraform admin access to the cluster,
  # so kubectl works after apply (module default is false).
  enable_cluster_creator_admin_permissions = true

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # Default AWS networking: VPC CNI, kube-proxy, CoreDNS
  cluster_addons = {
    coredns    = {}
    kube-proxy = {}
    vpc-cni    = {}
  }

  eks_managed_node_groups = {
    worker-group = {
      instance_types = ["t2.xlarge"]
      min_size       = 3
      max_size       = 5
      desired_size   = 3
    }
  }
}

output "cluster_name" {
  value = module.eks.cluster_name
}

output "region" {
  value = var.region
}
EOF
```

---

## 3. Initialize and review

Download the providers and modules, then check the plan (read-only — creates nothing):

```bash
terraform init
terraform validate
terraform plan
```

> The plan should report roughly 59 resources to add — the VPC and subnets, IAM roles, the EKS control plane, the `worker-group` managed node group, and the `vpc-cni` / `kube-proxy` / `coredns` add-ons.

---

## 4. Apply

Create the cluster (this takes ~15 minutes):

```bash
terraform apply
```

Type `yes` to confirm (or use `terraform apply -auto-approve`).

---

## 5. Configure kubectl and verify

Point `kubectl` at the new cluster and confirm the nodes are `Ready`:

```bash
aws eks update-kubeconfig --name training-cluster-tf --region eu-central-1
kubectl get no -o wide
```

Check the default networking add-ons are running (pods get IPs from the VPC via `aws-node`):

```bash
kubectl get po -n kube-system
```

You should see `aws-node` (VPC CNI), `kube-proxy`, and `coredns` pods.

---

## 6. Destroy the cluster

Tear everything down when finished to stop AWS charges:

```bash
terraform destroy
```

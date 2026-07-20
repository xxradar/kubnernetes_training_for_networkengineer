# LAB01 - GKE Cluster with Terraform (default GKE networking)

This lab creates a regional [GKE](https://cloud.google.com/kubernetes-engine) cluster with [Terraform](https://developer.hashicorp.com/terraform), using GKE's default VPC-native networking. It provisions the same size cluster as the EKS and AKS labs.

> **Note:** The cluster is regional (spans 3 zones in `europe-west1`) with one node per zone, giving 3 nodes total. The default node pool is removed and a separately managed `worker` pool is created, which is the recommended pattern.

> **Cost / time:** A regional GKE cluster with 3× `e2-standard-4` nodes runs real GCP charges, and `terraform apply` takes ~10 minutes. Always `terraform destroy` when finished.

**What you build**

| Component | Value |
|-----------|-------|
| Cloud / region | GCP GKE, `europe-west1` (regional) |
| Cluster | `training-cluster-gke` |
| Node pool | 3× `e2-standard-4` (1 node/zone × 3 zones) |
| Networking (CNI) | Default GKE, VPC-native (alias IPs) |
| Module / provider | `hashicorp/google ~> 6.0` |

---

## 1. Prerequisites

Install [Terraform](https://developer.hashicorp.com/terraform/install) and the [gcloud CLI](https://cloud.google.com/sdk/docs/install), then authenticate and select your project.

```bash
gcloud auth application-default login
gcloud config set project <YOUR_PROJECT_ID>
terraform version
```

Enable the required APIs (once per project):

```bash
gcloud services enable container.googleapis.com compute.googleapis.com
```

---

## 2. Write the Terraform configuration

Create a working directory and a `main.tf`:

```bash
mkdir -p gke-terraform && cd gke-terraform
cat > main.tf <<'EOF'
terraform {
  required_version = ">= 1.3"
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
}

variable "project_id" {
  description = "GCP project ID"
  type        = string
}

variable "region" {
  default = "europe-west1"
}

variable "cluster_name" {
  default = "training-cluster-gke"
}

provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_container_cluster" "gke" {
  name     = var.cluster_name
  location = var.region

  # Manage our own node pool instead of the default one
  remove_default_node_pool = true
  initial_node_count       = 1

  # Default GKE networking (VPC-native / alias IPs)
  networking_mode = "VPC_NATIVE"
  ip_allocation_policy {}

  deletion_protection = false
}

resource "google_container_node_pool" "worker" {
  name     = "worker"
  cluster  = google_container_cluster.gke.id
  location = var.region

  # Regional cluster spans 3 zones; 1 node/zone = 3 nodes total
  node_count = 1

  node_config {
    machine_type = "e2-standard-4"
    oauth_scopes = ["https://www.googleapis.com/auth/cloud-platform"]
  }
}

output "cluster_name" {
  value = google_container_cluster.gke.name
}

output "region" {
  value = var.region
}
EOF
```

---

## 3. Initialize and review

Pass your project ID via a variable. The plan is read-only (creates nothing):

```bash
export TF_VAR_project_id=<YOUR_PROJECT_ID>
terraform init
terraform validate
terraform plan
```

---

## 4. Apply

Create the cluster (this takes ~10 minutes):

```bash
terraform apply
```

Type `yes` to confirm (or use `terraform apply -auto-approve`).

---

## 5. Configure kubectl and verify

Fetch credentials and confirm the 3 nodes are `Ready`:

```bash
gcloud container clusters get-credentials training-cluster-gke --region europe-west1
kubectl get no -o wide
```

---

## 6. Destroy the cluster

Tear everything down when finished to stop GCP charges:

```bash
terraform destroy
```

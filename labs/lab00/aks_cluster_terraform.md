# LAB00 - AKS Cluster with Terraform (default Azure CNI)

This lab creates a managed [AKS](https://azure.microsoft.com/products/kubernetes-service) cluster with [Terraform](https://developer.hashicorp.com/terraform), using the default Azure CNI networking. It provisions the same size cluster as the EKS and GKE labs.

> **Note:** Terraform manages the full lifecycle - a resource group, the AKS control plane, and a 3-node system node pool. AKS also auto-creates a second, node-infrastructure resource group named `MC_<rg>_<cluster>_<location>` that holds the VMSS, disks, and networking.

> **Cost / time:** An AKS cluster with 3× `Standard_D4s_v5` nodes runs real Azure charges, and `terraform apply` takes ~5 minutes. Always `terraform destroy` when finished.

> **vCPU quota:** 3× `Standard_D4s_v5` needs 12 vCPUs of the `standardDSv5Family` in your region. New/Pay-As-You-Go subscriptions often start with a low regional quota (e.g. 10) or 0 for a specific family. Check with `az vm list-usage --location westeurope -o table` and either request a quota increase or set a smaller `vm_size` (e.g. `Standard_B2ms`) via `-var vm_size=...`.

**What you build**

| Component | Value |
|-----------|-------|
| Cloud / region | Azure AKS, `westeurope` |
| Resource group | `training-aks-rg` |
| Cluster | `training-cluster-aks` |
| Node pool | 3× `Standard_D4s_v5` |
| Networking (CNI) | Default Azure CNI |
| Provider | `hashicorp/azurerm ~> 4.0` |

---

## 1. Prerequisites

Install [Terraform](https://developer.hashicorp.com/terraform/install) and the [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli), then sign in and select your subscription.

```bash
az login
az account show
export ARM_SUBSCRIPTION_ID=$(az account show --query id -o tsv)
terraform version
```

---

## 2. Write the Terraform configuration

Create a working directory and a `main.tf`:

```bash
mkdir -p aks-terraform && cd aks-terraform
cat > main.tf <<'EOF'
terraform {
  required_version = ">= 1.3"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

variable "location" {
  default = "westeurope"
}

variable "cluster_name" {
  default = "training-cluster-aks"
}

variable "vm_size" {
  default = "Standard_D4s_v5"
}

resource "azurerm_resource_group" "rg" {
  name     = "training-aks-rg"
  location = var.location
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = var.cluster_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "trainingaks"

  default_node_pool {
    name       = "worker"
    node_count = 3
    vm_size    = var.vm_size
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin = "azure"
  }
}

output "cluster_name" {
  value = azurerm_kubernetes_cluster.aks.name
}

output "resource_group" {
  value = azurerm_resource_group.rg.name
}
EOF
```

---

## 3. Initialize and review

The plan is read-only (creates nothing):

```bash
terraform init
terraform validate
terraform plan
```

---

## 4. Apply

Create the cluster (this takes ~5 minutes):

```bash
terraform apply
```

Type `yes` to confirm (or use `terraform apply -auto-approve`). To use a smaller VM if you are quota-limited:

```bash
terraform apply -var vm_size=Standard_B2ms
```

---

## 5. Configure kubectl and verify

Fetch credentials and confirm the 3 nodes are `Ready`:

```bash
az aks get-credentials --name training-cluster-aks --resource-group training-aks-rg
kubectl get no -o wide
```

Check the default networking pods (Azure CNI):

```bash
kubectl get po -n kube-system
```

You should see `azure-ip-masq-agent`, `kube-proxy`, and `coredns` pods.

---

## 6. Destroy the cluster

Tear everything down when finished to stop Azure charges:

```bash
terraform destroy
```

# LAB01 - Setting up a Kubernetes cluster

Before you can explore anything, you need a cluster to work on. This lab gives you several ways to get one, from a throwaway local cluster to a managed cloud cluster. For the rest of the course a single **kind** cluster on an Ubuntu host is enough, and it is the option we run through in detail. The cloud options (EKS, AKS, GKE) are here so you can see how the same concepts look on managed platforms with their own CNIs.

Pick one path and stick with it:

* **kind** on an Ubuntu VM: fast, cheap, and what the later labs assume. Start with the [common setup](kind_setup.md), then add [Cilium](kind_cilium.md) or [Calico](kind_calico.md).
* A managed cloud cluster if you want to compare networking on EKS, AKS, or GKE.

## Get the lab material onto your host

If you are using the kind path, first SSH into your Ubuntu host (see [common setup](kind_setup.md) for launching it), then install git and clone this repo so you have all the manifests locally:
```
sudo apt-get update && sudo apt-get install -y git
git clone https://github.com/xxradar/kubernetes_training_for_networkengineer.git
cd kubernetes_training_for_networkengineer
```
From here on, each lab lives under `labs/labXX/`.

## Cluster options

* [Rancher Desktop](https://rancherdesktop.io/)
* Kind cluster - [common setup](kind_setup.md), then [Cilium](kind_cilium.md) or [Calico](kind_calico.md)
* [EKS cluster - Calico (eksctl)](eks_cluster.md)
* [EKS cluster - Terraform (default AWS CNI)](eks_cluster_terraform.md)
* [AKS cluster - Terraform (default Azure CNI)](aks_cluster_terraform.md)
* [GKE cluster - Terraform (default GKE networking)](gke_cluster_terraform.md)
* [Kubeadm, containerd & K8S - Calico & Cilium](https://github.com/xxradar/k8s-calico-oss-install-containerd)

# Lab 0 — EKS Cluster with Calico Network Policy

This lab creates a managed [Amazon EKS](https://aws.amazon.com/eks/) cluster with [eksctl](https://eksctl.io/) using a cluster config file, then adds [Calico](https://docs.tigera.io/calico/latest/about/) **for network policy** on top of the AWS VPC CNI.

> **Note:** This is the **policy-only** setup — the AWS VPC CNI keeps handling pod networking (IPAM from your VPC), and Calico is layered on purely to enforce Kubernetes and Calico network policy. You do **not** replace the cluster's data plane. Do not also enable the VPC CNI's built-in network policy — it conflicts with Calico.

> **Cost / time:** An EKS control plane plus 3× `t2.xlarge` nodes runs real AWS charges, and `eksctl create cluster` takes ~15 minutes. Remember to delete the cluster (final step) when you are done.

**What you build**

| Component | Value |
|-----------|-------|
| Cloud / region | AWS EKS, `eu-central-1` |
| Cluster | `training-cluster` (eksctl config file), Kubernetes 1.36 |
| Node group | 3× `t2.xlarge` (min 3, max 5) |
| Networking (CNI) | AWS VPC CNI (pod IPs from the VPC) |
| Network policy | Calico (Tigera operator, `v3.32.1`, policy-only) |
| Control-plane logging | `audit`, `api`, `authenticator` → CloudWatch |

---

## 1. Prerequisites — AWS credentials

Configure the AWS CLI and confirm you are authenticated in the right account.

```bash
aws configure
aws sts get-caller-identity
```

---

## 2. Install eksctl

> The eksctl project moved from `weaveworks` to `eksctl-io`. Use the current URL below. This downloads the `amd64` Linux build; on ARM instances replace `amd64` with `arm64`.

```bash
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

## 3. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --client
```

---

## 4. Create the EKS cluster

Create a cluster config file called `cluster.yaml`:

```bash
cat >cluster.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: training-cluster
  region: eu-central-1
  version: "1.36"
nodeGroups:
  - name: worker-group
    instanceType: t2.xlarge
    desiredCapacity: 3
    minSize: 3
    maxSize: 5
cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator"]
EOF
```

> **Audit logging:** The `cloudWatch.clusterLogging` block enables EKS control-plane logging — `audit` (every request to the API server), plus `api` and `authenticator`. Logs go to the CloudWatch log group `/aws/eks/training-cluster/cluster`. Use `["all"]` to enable every type. Control-plane logs are **off by default** on EKS and incur CloudWatch ingestion/storage charges.

> **Tip:** Validate the config without creating anything using `eksctl create cluster -f cluster.yaml --dry-run` — it renders the fully-expanded config (AMI family, AZs, Kubernetes version) so you can catch schema errors first.

Create the cluster (this takes ~15 minutes):

```bash
eksctl create cluster -f cluster.yaml
```

Verify all nodes reach `Ready`:

```bash
kubectl get no -o wide
```

---

## 5. Install Calico for network policy

The AWS VPC CNI manifests that older guides referenced have been retired. The current method uses the **Tigera operator** in policy-only mode, following the [official Calico EKS install docs](https://docs.tigera.io/calico/latest/getting-started/kubernetes/managed-public-cloud/eks).

First, let the AWS VPC CNI annotate pods with their IPs so they propagate quickly to Calico (this grants the `aws-node` daemon set the `patch` permission on pods):

```bash
cat <<EOF > append.yaml
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - patch
EOF
kubectl apply -f <(cat <(kubectl get clusterrole aws-node -o yaml) append.yaml)
kubectl set env -n kube-system daemonset/aws-node ANNOTATE_POD_IP=true
```

Install the Tigera operator and CRDs:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/v1_crd_projectcalico_org.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/tigera-operator.yaml
```

Configure the Calico installation (policy-only: `cni.type: AmazonVPC`):

```bash
kubectl create -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  kubernetesProvider: EKS
  cni:
    type: AmazonVPC
  calicoNetwork:
    bgp: Disabled
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
```

> **Optional observability:** Calico 3.32 also ships the Goldmane flow aggregator and the Whisker UI. Add them by applying `Goldmane` and `Whisker` resources (`kind: Goldmane` / `kind: Whisker`, `metadata.name: default`) — see the official docs. They are not required for policy enforcement.

---

## 6. Verify Calico

Watch the operator roll out the Calico components into the `calico-system` namespace:

```bash
watch kubectl get tigerastatus
```

When `apiserver` and `calico` report `Available: True`, check the node agent daemon set:

```bash
kubectl get daemonset calico-node --namespace calico-system
```

You can now apply Kubernetes `NetworkPolicy` or Calico `GlobalNetworkPolicy` objects, enforced by Calico.

---

## 7. Delete the EKS cluster

Always tear the cluster down when finished to stop AWS charges:

```bash
eksctl delete cluster --name training-cluster --region eu-central-1
```

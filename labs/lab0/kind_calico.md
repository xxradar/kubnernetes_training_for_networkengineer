# Lab 0 — Kind Cluster with Calico CNI

This lab builds a 3-node Kubernetes cluster in [kind](https://kind.sigs.k8s.io/) (Kubernetes-in-Docker) on an AWS EC2 instance, then installs [Calico](https://docs.tigera.io/calico/latest/about/) as the CNI.

> **Note:** The default CNI is disabled in the kind config (`disableDefaultCNI: true`) so that Calico can take over pod networking. The pod CIDR (`192.168.0.0/16`) matches Calico's default IP pool.

**What you build**

| Component | Value |
|-----------|-------|
| Cloud / region | AWS EC2, `eu-west-3` |
| Instance | `t2.xlarge` (4 vCPU / 16 GB) |
| Cluster | 1 control-plane + 2 workers (kind) |
| Pod CIDR | `192.168.0.0/16` |
| Service CIDR | `10.11.0.0/16` |
| CNI | Calico v3.26.1 |

---

## 1. Prerequisites — Launch an EC2 instance

Create an SSH key pair and launch the training instance.

```bash
export AWS_PAGER=""

aws ec2 create-key-pair \
  --key-name training \
  --region eu-west-3 \
  --query 'KeyMaterial' \
  --output text > training.pem

chmod 400 training.pem

aws ec2 run-instances \
  --image-id ami-0a2387cb2c63a860e \
  --region eu-west-3 \
  --instance-type t2.xlarge \
  --key-name training \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=k8straining}]'
```

Grab the public IP once the instance is running, then SSH in with `ssh -i training.pem ubuntu@<PUBLIC_IP>`.

```bash
aws ec2 describe-instances \
  --region eu-west-3 \
  --filters \
    "Name=tag:Name,Values=k8straining" \
    "Name=instance-state-name,Values=running,pending" \
  --query 'Reservations[].Instances[].PublicIpAddress' \
  --output text
```

> **Tip:** The `--filters` tag value must match the `Name` tag you set above (`k8straining`), otherwise the query returns nothing.

---

## 2. Update the Ubuntu server

```bash
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common \
    net-tools
```

---

## 3. Install Docker

```bash
curl https://get.docker.com | bash
sudo usermod -aG docker $USER
newgrp docker
```

> `newgrp docker` activates the new group in the current shell so you can run `docker` without `sudo`.

---

## 4. Install Kubernetes tooling (kubectl)

```bash
VERSION=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
sudo curl -L "https://storage.googleapis.com/kubernetes-release/release/$VERSION/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl
sudo chmod +x /usr/local/bin/kubectl
kubectl version --client
```

---

## 5. Install kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.32.0/kind-linux-amd64
sudo mv ./kind /usr/local/bin/kind
sudo chmod +x /usr/local/bin/kind
kind get clusters
```

---

## 6. Create the cluster

Write the cluster config. Note `disableDefaultCNI: true` — pods stay in `Pending` until Calico is installed, which is expected.

```bash
cat <<EOF >kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
  serviceSubnet: "10.11.0.0/16"
EOF
```

```bash
kind create cluster --config=kind-config.yaml
kubectl cluster-info --context kind-kind
kubectl get no
```

Check that all pods are `Running` or `Pending`:

```bash
kubectl get po -A
```

> If you experience crashes, it is because of a bug in `kind`. Contact your instructor.

---

## 7. Install the Calico CNI

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

> This is the **manifest** install (not the Tigera operator), so `calico-node` and `calico-kube-controllers` are deployed into the **`kube-system`** namespace — there is no `calico-system` namespace with this method. It may take a minute or two to pull images and become ready.

---

## 8. Verify the cluster

Watch the Calico pods roll out (press `Ctrl+C` to exit the watch):

```bash
watch kubectl get po -n kube-system -l k8s-app=calico-node
```

When all `calico-node` pods are `Running`, the nodes should transition to `Ready`:

```bash
kubectl get no -o wide
```

Pods will get addresses from Calico's default IP pool inside the `192.168.0.0/16` pod subnet, for example:

```bash
kubectl get po -n kube-system -o wide
```

> **Next step:** Calico's `calicoctl` CLI (used in later labs) lets you inspect IP pools, BGP peers, and network policy. See `labs/lab8/calicoctl.md`.

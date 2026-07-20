# Lab 0 — kind cluster (common setup)

This lab provisions an Ubuntu EC2 instance, installs the tooling, and creates a 3-node [kind](https://kind.sigs.k8s.io/) cluster with the default CNI disabled and API-server audit logging enabled. Once the cluster is up, continue with a CNI install:

- [Cilium](kind_cilium.md)
- [Calico](kind_calico.md)

**What you build**

| Component | Value |
|-----------|-------|
| Cloud / region | AWS EC2, `eu-west-3` |
| Instance | `t2.xlarge`, Ubuntu 22.04 LTS (cgroup v2) |
| Cluster | 1 control-plane + 2 workers (kind) |
| Service CIDR | `10.11.0.0/16` |
| Pod CIDR | Cilium `10.10.0.0/16` · Calico `192.168.0.0/16` |
| Audit logging | kube-apiserver audit → `/var/log/kubernetes/audit/audit.log` |

---

## 1. Prerequisites — Launch an EC2 instance

Create an SSH key pair and launch the instance. The image ID is resolved from the Canonical SSM public parameter, so you always get the current Ubuntu 22.04 LTS AMI.

```bash
export AWS_PAGER=""

aws ec2 create-key-pair \
  --key-name training \
  --region eu-west-3 \
  --query 'KeyMaterial' \
  --output text > training.pem

chmod 400 training.pem

aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id \
  --region eu-west-3 \
  --instance-type t2.xlarge \
  --key-name training \
  --associate-public-ip-address \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=30,VolumeType=gp3,DeleteOnTermination=true}' \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=k8straining}]'
```

> The instance's security group must allow inbound TCP `22` from your IP so you can SSH in.

Get the public IP, then connect with `ssh -i training.pem ubuntu@<PUBLIC_IP>`.

```bash
aws ec2 describe-instances \
  --region eu-west-3 \
  --filters \
    "Name=tag:Name,Values=k8straining" \
    "Name=instance-state-name,Values=running,pending" \
  --query 'Reservations[].Instances[].PublicIpAddress' \
  --output text
```

Confirm the host uses cgroup v2 (required by current Kubernetes):

```bash
stat -fc %T /sys/fs/cgroup   # expect: cgroup2fs
```

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

---

## 4. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl
kubectl version --client
```

---

## 5. Install kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.32.0/kind-linux-amd64
sudo install -m 0755 ./kind /usr/local/bin/kind && rm ./kind
kind --version
```

---

## 6. Create the cluster

Set the pod CIDR for the CNI you plan to install:

```bash
export POD_CIDR=10.10.0.0/16       # Cilium
# export POD_CIDR=192.168.0.0/16   # Calico
```

Write the API-server audit policy (logs metadata for every request):

```bash
cat > audit-policy.yaml <<'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
EOF
```

Write the cluster config. The default CNI is disabled, and the control-plane node mounts the audit policy and enables audit logging:

```bash
cat > kind-config.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        audit-policy-file: /etc/kubernetes/audit/policy.yaml
        audit-log-path: /var/log/kubernetes/audit/audit.log
      extraVolumes:
      - name: audit-policy
        hostPath: /etc/kubernetes/audit/policy.yaml
        mountPath: /etc/kubernetes/audit/policy.yaml
        readOnly: true
        pathType: File
      - name: audit-logs
        hostPath: /var/log/kubernetes/audit/
        mountPath: /var/log/kubernetes/audit/
        readOnly: false
        pathType: DirectoryOrCreate
  extraMounts:
  - hostPath: ./audit-policy.yaml
    containerPath: /etc/kubernetes/audit/policy.yaml
    readOnly: true
- role: worker
- role: worker
networking:
  disableDefaultCNI: true
  podSubnet: "$POD_CIDR"
  serviceSubnet: "10.11.0.0/16"
EOF
```

```bash
kind create cluster --config=kind-config.yaml
kubectl cluster-info --context kind-kind
kubectl get no
```

The nodes stay `NotReady` and CoreDNS stays `Pending` until you install a CNI — this is expected.

```bash
kubectl get po -A
```

---

## 7. Verify audit logging

```bash
docker exec kind-control-plane tail -2 /var/log/kubernetes/audit/audit.log
```

You should see `audit.k8s.io/v1` JSON events.

---

## 8. Install a CNI

Continue with one of:

- [Cilium](kind_cilium.md) — pod CIDR `10.10.0.0/16`
- [Calico](kind_calico.md) — pod CIDR `192.168.0.0/16`

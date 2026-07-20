# Lab 0 — Kind Cluster with Cilium CNI

This lab builds a 3-node Kubernetes cluster in [kind](https://kind.sigs.k8s.io/) (Kubernetes-in-Docker) on an AWS EC2 instance, then installs [Cilium](https://cilium.io/) as the CNI. You end up with WireGuard pod encryption, kube-proxy replacement, an ingress controller, and Hubble observability.

> **Note:** The default CNI is disabled in the kind config (`disableDefaultCNI: true`) so that Cilium can take over pod networking.

**What you build**

| Component | Value |
|-----------|-------|
| Cloud / region | AWS EC2, `eu-west-3` |
| Instance | `t2.xlarge` (4 vCPU / 16 GB) |
| Cluster | 1 control-plane + 2 workers (kind) |
| Pod CIDR | `10.10.0.0/16` |
| Service CIDR | `10.11.0.0/16` |
| CNI | Cilium 1.19.6 (WireGuard, kube-proxy replacement, ingress, Hubble) |

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

Write the cluster config. Note `disableDefaultCNI: true` — pods stay in `Pending` until Cilium is installed, which is expected.

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
  podSubnet: "10.10.0.0/16"
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

## 7. Install Helm

```bash
sudo snap install helm --classic
```

---

## 8. Install the Cilium and Hubble CLIs

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

```bash
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz{,.sha256sum}
```

> The `cilium` CLI is a management helper; `hubble` is the observability CLI. Both are separate from the Cilium agent installed via Helm below.

---

## 9. Install the Cilium CNI

```bash
helm repo add cilium https://helm.cilium.io/
```

Install with WireGuard transparent encryption, kube-proxy replacement, and the ingress controller enabled:

```bash
helm install cilium cilium/cilium --version 1.19.6 \
    --namespace kube-system \
    --set ipam.mode=cluster-pool \
    --set ipam.operator.clusterPoolIPv4PodCIDRList={10.10.0.0/16} \
    --set ipam.operator.clusterPoolIPv4MaskSize=24 \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set encryption.enabled=true \
    --set encryption.type=wireguard \
    --set kubeProxyReplacement=true \
    --set ingressController.enabled=true \
    --set ingressController.loadbalancerMode=shared \
    --set ingressController.service.type="NodePort" \
    --set loadBalancer.l7.backend=envoy
```

| Flag | Purpose |
|------|---------|
| `ipam.operator.clusterPoolIPv4PodCIDRList={10.10.0.0/16}` | Pod IP pool — set to match the kind `podSubnet` so pods land in `10.10.0.0/16` (Cilium otherwise uses its own `10.0.0.0/8` default) |
| `ipam.operator.clusterPoolIPv4MaskSize=24` | Per-node slice of the pool (`/24` = 254 pods/node) |
| `encryption.type=wireguard` | Transparent pod-to-pod encryption |
| `kubeProxyReplacement=true` | Cilium handles service routing in eBPF, replacing kube-proxy |
| `ingressController.enabled=true` | Built-in Cilium ingress |
| `loadBalancer.l7.backend=envoy` | Envoy-based L7 load balancing |

> **Why these IPAM flags?** In its default `cluster-pool` mode, Cilium allocates pod IPs from its own pool and ignores the kind `podSubnet`. These flags point that pool at `10.10.0.0/16` so the CIDR in your kind config is actually used.

Wait for Cilium to come up, then (optionally) run the connectivity test:

```bash
cilium status --wait
cilium connectivity test   # optional, takes several minutes
```

```bash
kubectl get no
```

---

## 10. Enable Cilium observability (Hubble)

```bash
helm upgrade cilium cilium/cilium --version 1.19.6 \
   --namespace kube-system \
   --reuse-values \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true
```

Port-forward the Hubble relay in the background:

```bash
cilium hubble port-forward&
```

```bash
hubble status
```

Example output:

```text
Healthcheck (via localhost:4245): Ok
Current/Max Flows: 4096/4096 (100.00%)
Flows/s: 12.34
Connected Nodes: 3/3
```

> The cluster has 3 nodes (`kind-control-plane`, `kind-worker`, `kind-worker2`). Right after starting the port-forward you may briefly see `Connected Nodes: 0/3` until the relay connects to every agent.

Observe live flows:

```bash
hubble observe
```

Example output:

```text
Jan 29 14:47:36.330: 10.10.0.103:33058 <> 10.10.1.88:4240 to-overlay FORWARDED (TCP Flags: ACK)
Jan 29 14:47:36.330: 10.10.0.103:43218 <> 10.10.2.243:4240 to-overlay FORWARDED (TCP Flags: ACK)
Jan 29 14:47:41.454: 10.10.0.180:4240 <> 10.10.1.221:58542 to-overlay FORWARDED (TCP Flags: ACK)
Jan 29 14:47:41.455: 10.10.1.221:58542 -> 10.10.0.180:4240 to-endpoint FORWARDED (TCP Flags: ACK)
Jan 29 14:47:44.267: 10.10.0.180:4240 <> 10.10.2.12:56052 to-overlay FORWARDED (TCP Flags: ACK)
Jan 29 14:47:44.267: 10.10.2.12:56052 -> 10.10.0.180:4240 to-endpoint FORWARDED (TCP Flags: ACK)
Jan 29 14:47:46.062: 10.10.0.180:4240 <> 10.10.2.226:43864 to-overlay FORWARDED (TCP Flags: ACK)
Jan 29 14:47:46.063: 10.10.2.226:43864 -> 10.10.0.180:4240 to-endpoint FORWARDED (TCP Flags: ACK)
Jan 29 14:47:49.134: 10.10.0.103:37572 <- 10.10.0.180:4240 to-stack FORWARDED (TCP Flags: ACK)
```

---

## 11. Access the Hubble UI

```bash
kubectl port-forward -n kube-system svc/hubble-ui --address 0.0.0.0 --address :: 12000:80
```

Then open `http://<INSTANCE_PUBLIC_IP>:12000` in your browser.

> **Security note:** `--address 0.0.0.0` exposes the UI on all interfaces. Make sure your EC2 security group only allows your own IP on port `12000`.

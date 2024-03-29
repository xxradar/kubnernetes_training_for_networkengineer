### Prerequisites
```
export AWS_PAGER=""

aws ec2 run-instances \
  --image-id ami-0a21d1c76ac56fee7 \
  --region eu-west-3 \
  --instance-type t2.xlarge \
  --key-name <SSH_KEYNAME> \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyInstance}]' \
  --block-device-mappings 'DeviceName=/dev/sda1,Ebs={VolumeSize=40,Encrypted=true,VolumeType=gp2,DeleteOnTermination=true}'
```
### Update Ubuntu server 
```
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common \
    net-tools
```
### Install docker
```
curl https://get.docker.com | bash
sudo usermod -aG docker $USER
newgrp docker
```
### Install K8S tooling
```
sudo curl -L "https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl
sudo chmod +x /usr/local/bin/kubectl
kubectl version --client
```
### Create a K8S cluster with kind
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
sudo mv ./kind /usr/local/bin/kind
sudo chmod +x /usr/local/bin/kind
kind get clusters
```
 
```
cat  <<EOF >kind-config.yaml 
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
```
kind create cluster --config=kind-config.yaml
kubectl cluster-info --context kind-kind
kubectl get no
   .... 
```
Check if all pods are `running` or `pending`
```
kubectl get po -A
```
If you experience crashes, it is because of the bug in `kind`. Contact your instructor.
### Install HELM
```
sudo snap install helm --classic
```
### Install Cilium and Hubble CLI
```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```
```
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz{,.sha256sum}
```
### Install Cilium CNI
```
helm repo add cilium https://helm.cilium.io/
```
```
helm install cilium cilium/cilium --version 1.14.1 \
    --namespace kube-system \
    --set authentication.mutual.spire.enabled=true \
    --set authentication.mutual.spire.install.enabled=true \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set encryption.enabled=true \
    --set encryption.type=wireguard \
    --set kube-proxy-replacement=strict \
    --set ingressController.enabled=partial \
    --set ingressController.loadbalancerMode=shared \
    --set ingressController.service.type="NodePort" \
    --set loadBalancer.l7.backend=envoy

```
```
cilium status --wait
cilium connectivity test #optional
```

### Install Cilium observablity
```
helm upgrade cilium cilium/cilium --version 1.14.1 \
   --namespace kube-system \
   --reuse-values \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true
```

```
cilium hubble port-forward&
```
```
$ hubble status
Healthcheck (via localhost:4245): Ok
Current/Max Flows: 0/0
Flows/s: N/A
Connected Nodes: 0/4
Unavailable Nodes: 4
  - kind-control-plane
  - kind-worker
  - kind-worker2
  - kind-worker3hubble observe
```
```
$ hubble observe
Jan 29 14:47:36.330: 10.10.0.103:33058 <> 10.10.1.88:4240 to-overlay FORWARDED (TCP Flags: ACK)
Jan 29 14:47:36.330: 10.10.0.103:43218 <> 10.10.2.243:4240 to-overlay FORWARDED (TCP Flags: ACK)
Jan 29 14:47:41.454: 10.10.0.180:4240 <> 10.10.1.221:58542 to-overlay FORWARDED (TCP Flags: ACK)
Jan 29 14:47:41.455: 10.10.1.221:58542 -> 10.10.0.180:4240 to-endpoint FORWARDED (TCP Flags: ACK)
Jan 29 14:47:44.267: 10.10.0.180:4240 <> 10.10.3.12:56052 to-overlay FORWARDED (TCP Flags: ACK)
Jan 29 14:47:44.267: 10.10.3.12:56052 -> 10.10.0.180:4240 to-endpoint FORWARDED (TCP Flags: ACK)
Jan 29 14:47:46.062: 10.10.0.180:4240 <> 10.10.2.226:43864 to-overlay FORWARDED (TCP Flags: ACK)
Jan 29 14:47:46.063: 10.10.2.226:43864 -> 10.10.0.180:4240 to-endpoint FORWARDED (TCP Flags: ACK)
Jan 29 14:47:49.134: 10.10.0.103:37572 <- 10.10.0.180:4240 to-stack FORWARDED (TCP Flags: ACK)
```
### Access Hubble UI WebUI
```
kubectl port-forward -n kube-system svc/hubble-ui --address 0.0.0.0 --address :: 12000:80
```


### Update Ubuntu server (ex. EC2 instance - t2.xlarge - pulic)
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
# (logout and login again)
```
### Fix bug in kind
```
sudo sysctl net/netfilter/nf_conntrack_max=262144
```
### Install K8S tooling
```
sudo curl -L "https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl
sudo chmod +x /usr/local/bin/kubectl
kubectl version --short --client
```
### Create a K8S cluster with kind
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.11.1/kind-linux-amd64
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
curl https://get.helm.sh/helm-v3.8.0-linux-amd64.tar.gz -o helm-v3.8.0-linux-amd64.tar.gz
tar -zxvf helm-v3.8.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

### Install Cilium CNI
```
helm repo add cilium https://helm.cilium.io/
```
```
docker pull quay.io/cilium/cilium:v1.11.1
kind load docker-image quay.io/cilium/cilium:v1.11.1
```
```
helm install cilium cilium/cilium --version 1.11.1 \
   --namespace kube-system \
   --set kubeProxyReplacement=partial \
   --set hostServices.enabled=false \
   --set externalIPs.enabled=true \
   --set nodePort.enabled=true \
   --set hostPort.enabled=true \
   --set bpf.masquerade=false \
   --set image.pullPolicy=IfNotPresent \
   --set ipam.mode=kubernetes
```
### Install Cilium CLI
```
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
```
```
cilium status --wait
cilium connectivity test #optional
```

### Install Cilium observablity
```
helm upgrade cilium cilium/cilium --version 1.11.1 \
   --namespace kube-system \
   --reuse-values \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true
```
```
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz{,.sha256sum}
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

```
kubectl port-forward -n kube-system svc/hubble-ui --address 0.0.0.0 --address :: 12000:80
```

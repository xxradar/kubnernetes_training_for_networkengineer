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
### Install Calico CNI
```
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```
Adjust the pod cidr in custom-resources.yaml to 10.10.0.0/16
```
wget https://docs.projectcalico.org/manifests/custom-resources.yaml
```
```
vi  custom-resources.yaml 
...
```
```
kubectl apply -f custom-resources.yaml
```
### Verify cluster 
```
watch kubectl get po -n calico-system
```
When all pods are running ... the nodes should be ready.
```
kubectl get no -o wide
```
  

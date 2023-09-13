### Prerequisites
For eu-west-3
```
export AWS_PAGER=""

aws ec2 run-instances \
  --image-id ami-0a21d1c76ac56fee7 \
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
  podSubnet: 192.168.0.0/16
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

### Install Calico CNI
```
kubectl apply -f  https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

### Verify cluster 
```
watch kubectl get po -n calico-system
```
When all pods are running ... the nodes should be ready.
```
kubectl get no -o wide
```
  

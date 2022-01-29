## Installing EKS via cluster config file

### Installing eksctl
```
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
$ sudo mv /tmp/eksctl /usr/local/bin
$ eksctl version
```

### Installing kubectl 
```
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod 700 ./kubectl
$ sudo mv ./kubectl /usr/local/bin
$ kubectl version
```

### Installing EKS cluster 

Create a file called `cluster.yaml`
```
cat >cluster.yaml <<EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: training-cluster
  region: eu-central-1
nodeGroups:
  - name: worker-group
    instanceType: t2.xlarge
    desiredCapacity: 3
    minSize: 3
    maxSize: 5
EOF
```
Create the cluster
```
eksctl create cluster -f cluster.yaml
```
This may take a while (15min). Verify the cluster to see everything is `Ready`
```
kubectl get no -o wide
```
### Installing support for Calico Network Security policies
```
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/calico-operator.yaml
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/calico-crs.yaml
```
Check if everything is ready ...
```
kubectl get daemonset calico-node --namespace calico-system
```
### Deleting the EKS cluster
```
eksctl delete cluster --name training-cluster --region eu-central-1
```

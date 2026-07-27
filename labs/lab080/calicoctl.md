### Install calicoctl
The verion must match the version of calico currently installed
```
curl -o calicoctl -O -L  "https://github.com/projectcalico/calicoctl/releases/download/v3.20.0/calicoctl"
chmod +x calicoctl
mv ./calicoctl /usr/local/bin/calicoctl
```

### Configure calicoctl
```
sudo mkdir /etc/calico

sudo vi etc/calico/calicoctl.cfg 

apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "kubernetes"
  kubeconfig: "/home/ubuntu/.kube/config"
```

## Lab 8 - Calico global network policy example
_Note: this will not work on EKS. For EKS you must install the correct version of calicoctl ([install](calicoctl.md)). To apply the policies use calicoctl instead of kubectl_
```
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: quarantine
spec:
  order: 100
  selector: quarantine == "true"
  namespaceSelector: ''
  serviceAccountSelector: ''
  ingress:
    - action: Log
      source: {}
      destination: {}
    - action: Deny
      source: {}
      destination: {}
  egress:
    - action: Log
      source: {}
      destination: {}
    - action: Deny
      source: {}
      destination: {}
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
EOF
```
```
kubectl get netpol -n prod-nginx
...
kubectl describe netpol allow-http -n prod-nginx
...
kubectl get globalnetworkpolicy
...
kubectl describe globalnetworkpolicy quarantine
```
```
kubectl run -it --rm --image xxradar/hackon -n prod-nginx debug
curl my-nginx-clusterip
...
kubectl run -it --rm --image xxradar/hackon -n prod-nginx debug -l quarantine=true
curl my-nginx-clusterip
...
```
A small variation ...
```
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: quarantine-ns
spec:
  order: 100
  selector: ''
  namespaceSelector: quarantine == "true"
  serviceAccountSelector: ''
  ingress:
    - action: Log
      source: {}
      destination: {}
    - action: Deny
      source: {}
      destination: {}
  egress:
    - action: Log
      source: {}
      destination: {}
    - action: Deny
      source: {}
      destination: {}
  doNotTrack: false
  applyOnForward: false
  preDNAT: false
  types:
    - Ingress
    - Egress
EOF
```
```
kubectl get globalnetworkpolicy
```
```
kubectl run -it --rm  --image xxradar/hackon debug -n myhackns -l mode=debug
curl my-nginx-clusterip.prod-nginx 
```
```
kubectl label ns myhackns quarantine=true
```
```
kubectl run -it --rm  --image xxradar/hackon debug -n myhackns -l mode=debug
curl my-nginx-clusterip.prod-nginx 
```

## LAB 8-bis Cilium cluster wide network policy example
```
kubectl apply -f -<<EOF
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "quarantine"
spec:
  endpointSelector:
    matchLabels:
      quarantine: "true"
  egressDeny:
  - toEntities:
    - "world"
EOF
```
```
kubectl run -it --rm --image xxradar/hackon debug
...
kubectl run -it --rm --image xxradar/hackon debug -l quarantine=true
```

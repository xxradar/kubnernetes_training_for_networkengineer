## Lab 8 - Calico global network policy example

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
```
```
kubectl run -it --rm --image xxradar/hackon debug
...
kubectl run -it --rm --image xxradar/hackon debug -l quarantine=true
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
kubectl label ns myhackns quarantine=true
```
```
kubectl run -it --rm -n myhackns --image xxradar/hackon debug
```

## LAB 8-bis
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

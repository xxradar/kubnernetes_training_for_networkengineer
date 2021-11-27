## Lab 8 

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
kubectl run -it --rm --image xxradar/hackon debug
...
kubectl run -it --rm --image xxradar/hackon debug -l quarantine=true
```

## LAB 7 - Network Security Policies
Check connectivity ...

```
kubectl get po -n prod-nginx -o wide  --show-labels
...
kubectl get svc -n prod-nginx -o wide --show-labels
...
```
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon debug
curl curl my-nginx-clusterip
curl <pod_ip>
```
Apply a default-deny all policy
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
   - Ingress
   - Egress
EOF
```
Check connectivity
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon debug
curl curl my-nginx-clusterip
curl <pod_ip>
```
Enable access on port 80
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels: {}
    ports:
    - protocol: TCP
      port: 80
EOF
```
Check connectivity
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon debug
curl curl my-nginx-clusterip
curl <pod_ip>
```
Connectivity form a different namespace ...

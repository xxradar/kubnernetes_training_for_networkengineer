## LAB 7 - Network Security Policies
Check connectivity ...

```
kubectl get po -n prod-nginx 
...
kubectl get svc -n prod-nginx
...
```
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon debug
curl curl my-nginx-clusterip
curl <pod_ip>
```

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

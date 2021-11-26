## LAB 7 - Network Security Policies

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

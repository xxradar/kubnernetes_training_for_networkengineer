## LAB 4
Create a service of type NodePort 
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-nodeport
  namespace: prod-nginx
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
EOF
```
```
kubectl get svc -n prod-nginx -o wide
```
```
kubectl get ep my-nginx-clusterip -n prod-nginx -o yaml
```

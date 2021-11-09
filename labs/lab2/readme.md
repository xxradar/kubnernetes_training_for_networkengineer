## Lab2

```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```
```
kubectl get deploy -o wide
```
### Execercise
Try to add a pod 
```
 kubectl run  --image nginx -l app=nginx -l pod-template-hash=7848d4b86f testnginx
 ```
 

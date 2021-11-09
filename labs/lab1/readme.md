### Lab 1
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    name: nginx
    environment: dev
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
EOF
```

```
kubectl get po
```
```
kubectl get po -o wide
```
```
kubectl get po -o wide --show-labels
```
### Execrcise
- Create additional pods 
- Delete one of these pods. Does it come back?

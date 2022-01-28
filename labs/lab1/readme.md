### Lab 1
Create a namespace
```
kubectl create ns prod-nginx
```
Deploy a a single pod in the newly create namespace
```
kubectl apply -f pod.yaml
```
or 
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: prod-nginx
  labels:
    name: nginx
    environment: prod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
EOF
```
Analyse the output of kubectl with the different flags
```
kubectl get po                #show pods in default namespace
kubectl get po -n prod-nginx  #show pods in namespace prod-nginx
kubectl get po -A             #show all pods in all namespaces
```
```
kubectl get po -n prod-nginx -o wide
```
```
kubectl get po -n prod-nginx -o wide --show-labels
```
### Execrcise
- Create additional pods 
- Delete one of these pods. Does it come back?

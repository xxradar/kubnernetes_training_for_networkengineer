## LAB 3

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-clusterip
  namespace: prod-nginx
spec:
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

## Exercise
Create a test pod
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon hackpod -- bash
```
```
ifconfig
...
cat /etc/resolv
...
```
```
curl my-nginx-clusterip
...
curl my-nginx-clusterip.default
```
Create a test pod in another namespace
```
kubectl create ns myhackns 
```
```
kubectl run -it --rm -n myhackns --image xxradar/hackon hackpod -- bash
```
```
curl my-nginx-clusterip
...
curl my-nginx-clusterip.default
```

## Additional exercise
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: dev-nginx
EOF
```
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev-nginx
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
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-clusterip
  namespace: dev-nginx
spec:
  ports:
  - port: 8765
    targetPort: 80
    protocol: TCP
  selector:
    app: nginx
EOF
```
```
kubectl run -it --rm -n myhackns --image xxradar/hackon hackpod -- bash
curl my-nginx-clusterip                 # Does this work?
curl my-nginx-clusterip.prod-nginx      # Does this work?
curl my-nginx-clusterip.dev-nginx       # Does this work?
curl my-nginx-clusterip.dev-nginx:8765  # Does this work?
```

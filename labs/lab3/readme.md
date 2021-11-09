## LAB 3

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-clusterip
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
EOF
```
```
kubectl get svc -o wide
```
```
kubectl get ep my-nginx-clusterip -o yaml
```

## Exercise
Create a test pod
```
kubectl run -it --image xxradar/hackon hackpod -- bash
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
kubectl run -it -n myhackns --image xxradar/hackon hackpod -- bash
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


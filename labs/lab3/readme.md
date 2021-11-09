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
```


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
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE    SELECTOR
my-nginx-clusterip   ClusterIP   10.99.180.103   <none>        80/TCP         13m    app=nginx
my-nginx-nodeport    NodePort    10.107.221.36   <none>        80:31062/TCP   105s   app=nginx
```
Try to reach the NodePort
```
curl http:///127.0.0.1:31062
...
curl http://<node_ip>:31062
```
Compare the endpoints of the two services
```
kubectl get ep my-nginx-clusterip -n prod-nginx -o yaml
...
kubectl get ep my-nginx-nodeport -n prod-nginx -o yaml
...
```


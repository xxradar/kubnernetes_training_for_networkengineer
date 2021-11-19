## LAB 5
Create a service of type LoadBalancer 
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-lb
  namespace: prod-nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
EOF
```
```
kubectl get svc -n prod-nginx -o wide
NAME                 TYPE          CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE    SELECTOR
my-nginx-clusterip   ClusterIP     10.99.180.103   <none>        80/TCP         13m    app=nginx
my-nginx-lb          LoadBalancer  10.107.221.36   <Pending>     80:31062/TCP   105s   app=nginx
```
Try to reach the NodePort
```
curl http:///127.0.0.1:31062
```


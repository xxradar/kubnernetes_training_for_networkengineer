## LAB 5
This lab assumes your K8S can spin up a LoadBalancer. When you're using a managed K8S cluster, this should work as explained. Self-managed clusters do not have an integrated external load balancer configured by default. If you feel up to it, you can use the instructions [](https://kind.sigs.k8s.io/docs/user/loadbalancer/) to configure MetalLB with Kind. You need to carefullty check the Docker IPAM range and update the MetalLB configmap.<br>

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
Try to reach the NodePort and the LB external ip.
```
curl http:///127.0.0.1:31062
...
curl http://$NODE:31062
...
curl http:///<external-ip>:80
...
```


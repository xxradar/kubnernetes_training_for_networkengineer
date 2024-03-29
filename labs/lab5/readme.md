## LAB 5
This lab assumes your K8S can spin up a LoadBalancer. When you're using a managed K8S cluster, this should work as explained. Self-managed clusters do not have an integrated external load balancer configured by default. If you feel up to it, you can use the instructions to configure [MetalLB](https://kind.sigs.k8s.io/docs/user/loadbalancer/) with Kind. You need to carefullty check the Docker IPAM range and update the MetalLB configmap.<br>

### Setting up MetalLB
Create the metallb namespace
```
kubectl apply -fhttps://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
```
Wait for metallb pods to have a status of Running
```
kubectl get pods -n metallb-system --watch
```
Check the Docker IPAM range in use
```
docker network inspect -f '{{.IPAM.Config}}' kind 
```
Create the MetalLB configmap and match the addresses with the docker IPAM range network.
```
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.18.255.200-172.18.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
EOF
```
You can access the LoadBalancer IP from your lab host. <br>
If you want to make it accissible to the outside world on a KIND cluster, you must add some IPtables rules.
```
sudo iptables -t nat -A PREROUTING -p tcp -i eth0 --dport 80 -j DNAT --to-destination 172.18.255.200:80
sudo iptables -A FORWARD -p tcp -d 172.18.255.200 --dport 80 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```

### Create a service of type LoadBalancer 
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


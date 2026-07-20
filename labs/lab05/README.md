# LAB15 - Services - LoadBalancer

A **LoadBalancer** Service asks the underlying platform to provision an **external IP** that routes into the service. It is a superset of the previous types: it still gets a cluster-internal VIP (ClusterIP) and a node port (NodePort), and adds a real external address on top.

For network engineers: on a managed cloud (EKS, AKS, GKE) this provisions a cloud load balancer and the `EXTERNAL-IP` fills in automatically. On a self-managed or KIND cluster there is no external load balancer by default, so the `EXTERNAL-IP` stays `<Pending>` until you add something like [MetalLB](https://kind.sigs.k8s.io/docs/user/loadbalancer/) to hand out addresses.

> Continues from LAB14: the `prod-nginx` namespace and the nginx deployment already exist.

## Setting up MetalLB (KIND only)
On a managed cloud cluster you can skip this section. On KIND you need MetalLB so a LoadBalancer service gets an external IP. Check the Docker IPAM range carefully and match the MetalLB pool to it.

Install MetalLB:
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
```
Wait for the MetalLB pods to reach `Running`:
```
kubectl get pods -n metallb-system --watch
```
Check the Docker IPAM range in use:
```
docker network inspect -f '{{.IPAM.Config}}' kind
```
Create the MetalLB pool, matching the addresses to the Docker `kind` network range:
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
You can reach the LoadBalancer IP from your lab host. To make it reachable from the outside world on a KIND cluster, add NAT rules on the host:
```
sudo iptables -t nat -A PREROUTING -p tcp -i eth0 --dport 80 -j DNAT --to-destination 172.18.255.200:80
sudo iptables -A FORWARD -p tcp -d 172.18.255.200 --dport 80 -m state --state NEW,ESTABLISHED,RELATED -j ACCEPT
```

## Create a service of type LoadBalancer
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
NAME                 TYPE          CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE    SELECTOR
my-nginx-clusterip   ClusterIP     10.99.180.103   <none>           80/TCP         13m    app=nginx
my-nginx-lb          LoadBalancer  10.107.221.36   172.18.255.200   80:31062/TCP   105s   app=nginx
```
Without MetalLB (or a cloud provider) the `EXTERNAL-IP` stays `<Pending>`. Note that the LoadBalancer service also has a node port (here `31062`).

## Reach the service every way
A LoadBalancer service is reachable at all three levels. Grab a node IP first:
```
export NODE=$(kubectl get no kind-worker2 -o=jsonpath="{.status.addresses[0].address}")
```
```
curl http://127.0.0.1:31062       # via the node port on localhost
...
curl http://$NODE:31062           # via the node port on a node IP
...
curl http://<external-ip>:80      # via the external LoadBalancer IP
...
```

### Explore it yourself
* On KIND, what does `EXTERNAL-IP` show before you install MetalLB, and after?
* The LoadBalancer, NodePort, and ClusterIP for this app all forward to the same pods. Confirm with `kubectl get ep my-nginx-lb -n prod-nginx -o yaml`.
* On a managed cloud, where does the external IP come from, and what actually sits in the traffic path compared to the KIND/MetalLB case?

> Takeaway for network engineers: the three service types stack. ClusterIP gives an internal VIP, NodePort adds a port on every node, and LoadBalancer adds an external address on top. Which you use is about where you need to reach the app from, not about how it selects pods (that is always the label selector).

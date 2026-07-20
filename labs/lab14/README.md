# LAB14 - Services - NodePort

A **NodePort** Service exposes your app on a static port on **every node** in the cluster. Traffic to `<any-node-IP>:<nodePort>` is picked up by the dataplane (kube-proxy, or Cilium eBPF) and load-balanced to the pods behind the service. A NodePort is a superset of a ClusterIP: it still gets a cluster-internal VIP, and adds the node-level port on top.

For network engineers: the port (default range `30000-32767`) is open on **all** nodes, even the ones not running a matching pod, because the node forwards the traffic internally. That makes it a simple way to reach an app from outside the cluster without a cloud load balancer, at the cost of exposing raw node IPs and high ports.

> Continues from LAB13: the nginx deployment and the `my-nginx-clusterip` service already exist.

## Create a service of type NodePort
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
Look at the two services side by side. Note the `PORT(S)` column: the NodePort shows `80:<nodePort>/TCP`.
```
kubectl get svc -n prod-nginx -o wide
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE    SELECTOR
my-nginx-clusterip   ClusterIP   10.99.180.103   <none>        80/TCP         13m    app=nginx
my-nginx-nodeport    NodePort    10.107.221.36   <none>        80:31062/TCP   105s   app=nginx
```
Note your own allocated node port (here `31062`) and use it below.

## Reach the NodePort
Grab a node IP:
```
export NODE=$(kubectl get no kind-worker2 -o=jsonpath="{.status.addresses[0].address}")
```
> On EKS/AKS/GKE, use the external public IP of a node and open the node port in the security group / firewall.
> On a KIND cluster the nodes are containers on the `kind` docker network, so run a client on that network: `docker run -it --network kind xxradar/hackon`, then issue the curl commands from there.
```
curl http://127.0.0.1:31062
...
curl http://$NODE:31062
```
Compare the endpoints of the two services (they select the same pods, so the Endpoints list is identical):
```
kubectl get ep my-nginx-clusterip -n prod-nginx -o yaml
...
kubectl get ep my-nginx-nodeport -n prod-nginx -o yaml
...
```

### Explore it yourself
* Does the node port answer on a node that is **not** running an nginx pod? Try a different node IP. What does that tell you about how the traffic gets forwarded?
* Which of the two services is reachable from **outside** the cluster, and which only from inside?
* What port range did the node port come from, and how would you pin a specific one?

> Takeaway for network engineers: a NodePort opens the same high port on every node and forwards inward to the pods. It is the building block that a LoadBalancer service (LAB15) sits on top of.

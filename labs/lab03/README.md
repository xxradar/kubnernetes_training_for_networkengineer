# LAB03 - Services - ClusterIP

A **Service** of type **ClusterIP** gives you a stable virtual IP (a VIP) and a DNS name in front of a changing set of pods. It load-balances to every pod whose labels match the service `selector`, and the VIP is reachable only from **inside** the cluster.

For network engineers: the ClusterIP is not bound to any interface, it is a virtual address that the dataplane (kube-proxy, or Cilium eBPF in our setup) DNATs to a real pod IP. The **Endpoints** object is the live list of pod IPs behind the service, updated automatically as pods come and go. The DNS name is `<service>.<namespace>` (fully qualified: `<service>.<namespace>.svc.cluster.local`).

> Continues from LAB02: the `prod-nginx` namespace and the nginx deployment already exist.

## Create a ClusterIP service
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
## Inspect the service
See the service and its ClusterIP:
```
kubectl get svc -n prod-nginx -o wide
```
Look at the details, including the selector and the port mapping:
```
kubectl describe svc -n prod-nginx my-nginx-clusterip
```
List the Endpoints, the live set of pod IPs behind the service:
```
kubectl get ep my-nginx-clusterip -n prod-nginx -o yaml
```
Compare the Endpoints list with the pod IPs from LAB02. They should be the same set, and the service picks pods purely by the `selector` labels.

## Quick exercise: scaling and the service
Scale the deployment and watch the service track it. The ClusterIP (the VIP) never changes, but the Endpoints list grows and shrinks with the pods.
```
kubectl scale -n prod-nginx --replicas=5 deploy/nginx-deployment
kubectl get ep my-nginx-clusterip -n prod-nginx -o wide
...
kubectl scale -n prod-nginx --replicas=2 deploy/nginx-deployment
kubectl get ep my-nginx-clusterip -n prod-nginx -o wide
...
```
* How many IPs are in the Endpoints after each scale? Does the ClusterIP address itself change?
* Scale down to `--replicas=0`. What does the Endpoints list look like now, and what happens if you `curl` the service?

## Exercise: reach the service by DNS
Create a test pod (this image ships with networking tools):
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon hackpod -- bash
```
From inside the pod, look at its own network config:
```
ifconfig
```
and its DNS config (note the `search` domains and the CoreDNS server):
```
cat /etc/resolv.conf
```
Now reach the app three different ways and reason about each. Straight to a pod IP:
```
curl <a_prod-nginx_pod_ip>
```
By short DNS name, in the same namespace:
```
curl my-nginx-clusterip
```
By name.namespace:
```
curl my-nginx-clusterip.prod-nginx
```
Which of these would keep working after the pods are recreated with new IPs?

### From another namespace
Now repeat from a pod in **another** namespace
```
kubectl create ns myhackns
```
```
kubectl run -it --rm -n myhackns --image xxradar/hackon hackpod -- bash
```
From inside the pod
```
curl <a_prod-nginx_pod_ip>          # Does this work?
curl my-nginx-clusterip             # Does this work? Why or why not?
curl my-nginx-clusterip.prod-nginx  # Does this work?
```
The short name only resolves in the pod's own namespace, so from `myhackns` you need `name.namespace`.

## Additional exercise: service port vs targetPort
Create a second app and service in a `dev-nginx` namespace, using a different service `port` (8765) that maps to the container's `targetPort` (80).
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
        env: dev
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
You now have two services called `my-nginx-clusterip`: one in `prod-nginx` on port `80`, and one in `dev-nginx` on port `8765`. Start a test pod in a third namespace, `myhackns`:
```
kubectl run -it --rm -n myhackns --image xxradar/hackon hackpod -- bash
```
Work through the checks below from inside that pod and reason about each result. Two things decide the outcome: which namespace a name resolves in, and which port the service listens on.

### By IP
An IP is reachable cluster-wide, regardless of namespace. Straight to a prod-nginx pod IP:
```
curl <a_prod-nginx_pod_ip>          # Does this work?
```
Straight to a service ClusterIP:
```
curl <a_svc_ip>                     # Does this work?
```

### By short name (namespace matters)
A short name resolves in the pod's own namespace (`myhackns`), where no such service exists:
```
curl my-nginx-clusterip             # Does this work?
curl my-nginx-clusterip:8765        # Does this work?
```

### By name.namespace (namespace and port matter)
Now target the services explicitly. Remember the prod service listens on `80` and the dev service on `8765`:
```
curl my-nginx-clusterip.prod-nginx        # Does this work?
curl my-nginx-clusterip.dev-nginx         # Does this work?
curl my-nginx-clusterip.dev-nginx:8765    # Does this work?
```

> Takeaway: a ClusterIP is a stable in-cluster VIP that load-balances by label selector to the current Endpoints, and you reach it by DNS name, not by IP. Remember the two gotchas: the short name only resolves inside its own namespace (use `name.namespace` across namespaces), and the service `port` can differ from the container `targetPort`.

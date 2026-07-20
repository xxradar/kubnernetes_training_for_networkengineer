# LAB03 - Services - ClusterIP

A **Service** of type **ClusterIP** gives you a stable virtual IP (a VIP) and a DNS name in front of a changing set of pods. It load-balances to every pod whose labels match the service `selector`, and the VIP is reachable only from **inside** the cluster.

For network engineers: the ClusterIP is not bound to any interface, it is a virtual address that the dataplane (kube-proxy, or Cilium eBPF in our setup) DNATs to a real pod IP. The **Endpoints** object is the live list of pod IPs behind the service, updated automatically as pods come and go. The DNS name is `<service>.<namespace>` (fully qualified: `<service>.<namespace>.svc.cluster.local`).

Create a service of type ClusterIP
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
Inspect the service, its details, and its endpoints
```
kubectl get svc -n prod-nginx -o wide
kubectl describe svc -n prod-nginx my-nginx-clusterip
kubectl get ep my-nginx-clusterip -n prod-nginx -o yaml
```
Compare the Endpoints list with the pod IPs from LAB02. They should be the same set, and the service picks pods purely by the `selector` labels.

## Exercise
Create a test pod (this image ships with networking tools)
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon hackpod -- bash
```
From inside the pod, look at its own network and DNS config
```
ifconfig
...
cat /etc/resolv.conf
...
```
Now reach the app three different ways and reason about each
```
curl <a_prod-nginx_pod_ip>          # straight to a pod IP
curl my-nginx-clusterip             # short DNS name, same namespace
curl my-nginx-clusterip.prod-nginx  # name.namespace
```
Which of these would keep working after the pods are recreated with new IPs?

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

## Additional exercise
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
From a pod in `myhackns`, work out why each of these does or does not work (mind the namespace and the port)
```
kubectl run -it --rm -n myhackns --image xxradar/hackon hackpod -- bash
curl <a_prod-nginx_pod_ip>              # Does this work?
curl <a_svc_ip>                         # Does this work?
curl my-nginx-clusterip                 # Does this work?
curl my-nginx-clusterip:8765            # Does this work?
curl my-nginx-clusterip.prod-nginx      # Does this work?
curl my-nginx-clusterip.dev-nginx       # Does this work?
curl my-nginx-clusterip.dev-nginx:8765  # Does this work?
```

> Takeaway for network engineers: a ClusterIP is a stable in-cluster VIP that load-balances by label selector to the current Endpoints, and you reach it by DNS name, not by IP. Remember the two gotchas: the short name only resolves inside its own namespace (use `name.namespace` across namespaces), and the service `port` can differ from the container `targetPort`.

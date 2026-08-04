# LAB037 - Traffic locality (internalTrafficPolicy, topology-aware routing)

> **Optional, standalone lab.** It uses its own namespace `topo-demo` and does not depend on the other labs. It needs a cluster with at least **two worker nodes**.

By default a ClusterIP load-balances east-west (in-cluster) traffic across **all** ready endpoints, wherever they run. Two knobs change that to keep traffic node-local or zone-local, which lowers latency and, on a cloud, avoids cross-zone egress cost:

- `internalTrafficPolicy: Local` - only send to endpoints on the **same node** as the client.
- `spec.trafficDistribution` - prefer endpoints close to the client: `PreferSameZone` (same zone) or `PreferSameNode` (same node). On older clusters this value was called `PreferClose` (now the deprecated alias of `PreferSameZone`).

## Setup
Create the namespace and an app that reveals which pod answered (`traefik/whoami` returns its own hostname), spread across the nodes:
```
kubectl create ns topo-demo
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: topo-demo
  labels:
    app: whoami
spec:
  replicas: 4
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: whoami
      containers:
      - name: whoami
        image: traefik/whoami
        ports:
        - containerPort: 80
EOF
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: topo-demo
spec:
  selector:
    app: whoami
  ports:
  - port: 80
    targetPort: 80
EOF
```
See where the pods landed (note the `NODE` column, they should be split across your workers):
```
kubectl get po -n topo-demo -o wide -l app=whoami
```
Start a client pinned to **one** worker node, so you control where traffic originates. Replace `<worker>` with one of your worker node names:
```
kubectl run client -n topo-demo --image xxradar/hackon \
  --overrides='{"spec":{"nodeName":"<worker>"}}' --command -- sleep 3600
kubectl wait --for=condition=Ready pod/client -n topo-demo --timeout=60s
```

## A. internalTrafficPolicy: Local
Curl the service several times with the **default** policy. You hit pods on **all** nodes:
```
for i in $(seq 8); do kubectl exec -n topo-demo client -- curl -s whoami | grep Hostname; done | sort | uniq -c
```
Switch the service to node-local:
```
kubectl patch svc whoami -n topo-demo -p '{"spec":{"internalTrafficPolicy":"Local"}}'
```
Curl again. Now you only ever hit the whoami pods running on the client's node:
```
for i in $(seq 8); do kubectl exec -n topo-demo client -- curl -s whoami | grep Hostname; done | sort | uniq -c
```
The trade-off: if the client's node has **no** whoami pod, the request fails. There is no fallback.

Reset:
```
kubectl patch svc whoami -n topo-demo -p '{"spec":{"internalTrafficPolicy":"Cluster"}}'
```

## B. trafficDistribution (topology / zones)
kind nodes have no real zones, so label two workers to simulate them. Replace the names with your workers:
```
kubectl label no <worker-1> topology.kubernetes.io/zone=zone-a --overwrite
kubectl label no <worker-2> topology.kubernetes.io/zone=zone-b --overwrite
```
Turn on same-zone routing (make sure the `client` pod is on a node in `zone-a`):
```
kubectl patch svc whoami -n topo-demo -p '{"spec":{"trafficDistribution":"PreferSameZone"}}'
```
Confirm the EndpointSlice now carries per-zone hints:
```
kubectl get endpointslices -n topo-demo -l kubernetes.io/service-name=whoami -o yaml | grep -A2 hints
```
Curl again. Traffic now prefers endpoints in the client's zone (`zone-a`) while they are ready, and only spills to `zone-b` if `zone-a` has none:
```
for i in $(seq 8); do kubectl exec -n topo-demo client -- curl -s whoami | grep Hostname; done | sort | uniq -c
```
`PreferSameNode` behaves like `internalTrafficPolicy: Local` (client's node only), through the same field:
```
kubectl patch svc whoami -n topo-demo -p '{"spec":{"trafficDistribution":"PreferSameNode"}}'
for i in $(seq 8); do kubectl exec -n topo-demo client -- curl -s whoami | grep Hostname; done | sort | uniq -c
```

## Explore it yourself
* With `internalTrafficPolicy: Local`, scale whoami to 1 replica. If it lands on a node other than the client's, what happens to your curls?
* How is `internalTrafficPolicy: Local` / `PreferSameNode` (node) different from `PreferSameZone` (zone)? When would you pick each?
* These tune **east-west** traffic. How does that differ from `externalTrafficPolicy` in LAB040, which tunes **north-south** ingress?

## Cleanup
```
kubectl delete ns topo-demo
kubectl label no <worker-1> topology.kubernetes.io/zone-
kubectl label no <worker-2> topology.kubernetes.io/zone-
```

> Takeaway: by default a ClusterIP spreads across every ready endpoint. `internalTrafficPolicy: Local` (or `trafficDistribution: PreferSameNode`) keeps traffic on the client's node; `trafficDistribution: PreferSameZone` keeps it in the client's zone. Both cut latency and cross-zone cost, at the price of less even spreading and a weaker fallback when the local set is empty.

# LAB037 - Traffic locality (internalTrafficPolicy and topology-aware routing)

> **Optional, standalone lab.** It uses its own namespace `topo-demo` and does not depend on the other labs. It needs a cluster with at least **two worker nodes**.

By default a ClusterIP load-balances east-west (in-cluster) traffic across **all** ready endpoints, wherever they run. You can bias that toward locality (lower latency, and on a cloud lower cross-zone egress cost) in two ways, but they behave very differently depending on the **dataplane**:

- `internalTrafficPolicy: Local` - only send to endpoints on the **same node** as the client. Honored by **every** dataplane (kube-proxy and Cilium).
- `spec.trafficDistribution` (`PreferSameZone` / `PreferSameNode`) - topology-aware routing. Honored by **kube-proxy**, but **not enforced by Cilium's kube-proxy replacement** in current versions (our course clusters). Treat this part as a lesson in "the Service API expresses intent, but the dataplane has to implement it."

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
kubectl wait --for=condition=Ready pod/client -n topo-demo --timeout=90s
```

## A. internalTrafficPolicy: Local (works on any dataplane)
Curl the service several times with the **default** policy. You hit pods on **all** nodes:
```
for i in $(seq 12); do kubectl exec -n topo-demo client -- curl -s whoami | grep Hostname; done | sort | uniq -c
```
Switch the service to node-local:
```
kubectl patch svc whoami -n topo-demo -p '{"spec":{"internalTrafficPolicy":"Local"}}'
```
Curl again. Now you only ever hit the whoami pods running on the client's node:
```
for i in $(seq 12); do kubectl exec -n topo-demo client -- curl -s whoami | grep Hostname; done | sort | uniq -c
```
The trade-off: if the client's node has **no** whoami pod, the request fails. There is no fallback. Reset when done:
```
kubectl patch svc whoami -n topo-demo -p '{"spec":{"internalTrafficPolicy":"Cluster"}}'
```

## B. trafficDistribution: topology-aware routing (dataplane-dependent)
kind nodes have no real zones, so label two workers to simulate them. Replace the names with your workers, and make sure the `client` pod is on the node you label `zone-a`:
```
kubectl label no <worker-1> topology.kubernetes.io/zone=zone-a --overwrite
kubectl label no <worker-2> topology.kubernetes.io/zone=zone-b --overwrite
```
Ask for same-zone routing:
```
kubectl patch svc whoami -n topo-demo -p '{"spec":{"trafficDistribution":"PreferSameZone"}}'
```
The **control plane** does its part: the EndpointSlice controller adds per-zone hints. Confirm:
```
kubectl get endpointslices -n topo-demo -l kubernetes.io/service-name=whoami \
  -o jsonpath='{range .items[*].endpoints[*]}{.targetRef.name}{"  zone="}{.zone}{"  hint="}{.hints.forZones[*].name}{"\n"}{end}'
```
(If `zone=` is empty, the pods were created before you labeled the nodes; run `kubectl rollout restart deploy/whoami -n topo-demo` and re-check.)

Now curl from the zone-a client:
```
for i in $(seq 12); do kubectl exec -n topo-demo client -- curl -s whoami | grep Hostname; done | sort | uniq -c
```
What you observe depends on the dataplane:

- On a **kube-proxy** cluster (iptables / IPVS), traffic stays on the zone-a pods, the hints are enforced.
- On a **Cilium kube-proxy replacement** cluster (our course setup), traffic **still spreads across both zones**. The hints are present, but Cilium does not enforce `trafficDistribution` by default. Enabling `--set loadBalancer.serviceTopology=true` on the Cilium install helps only partially in current versions, and `PreferSameNode` is not honored at all.

That difference is the real takeaway: the same Service YAML gives you zone-local routing under kube-proxy and does nothing under Cilium's dataplane. For reliable node-locality on Cilium, use `internalTrafficPolicy: Local` from Part A instead.

## Explore it yourself
* With `internalTrafficPolicy: Local`, scale whoami to 1 replica. If it lands on a node other than the client's, what happens to your curls?
* On your cluster, does `trafficDistribution: PreferSameZone` actually restrict traffic, or just set hints? What does that tell you about who enforces it?
* These tune **east-west** traffic. How does that differ from `externalTrafficPolicy` in LAB040, which tunes **north-south** ingress?

## Cleanup
```
kubectl delete ns topo-demo
kubectl label no <worker-1> topology.kubernetes.io/zone-
kubectl label no <worker-2> topology.kubernetes.io/zone-
```

> Takeaway: `internalTrafficPolicy: Local` keeps traffic on the client's node and is honored by every dataplane. `trafficDistribution` (topology-aware routing) expresses zone/node preference, but whether it is actually enforced depends on the dataplane: kube-proxy honors it, Cilium's kube-proxy replacement currently does not. The Service API states intent; the dataplane decides.

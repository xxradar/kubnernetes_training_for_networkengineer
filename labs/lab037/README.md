# LAB037 - Traffic locality (spreading, affinity, internalTrafficPolicy, topology-aware routing)

> **Optional, standalone lab.** It uses its own namespace `topo-demo` and does not depend on the other labs. It needs a cluster with at least **two worker nodes**.

By default a ClusterIP load-balances east-west (in-cluster) traffic across **all** ready endpoints, wherever they run. This lab is about controlling **where the pods run** and **which endpoints receive traffic**, to keep traffic node-local or zone-local (lower latency, and on a cloud, lower cross-zone egress cost).

A recurring theme: some of these knobs are plain Kubernetes API and work everywhere; the **topology-aware** ones depend on the **dataplane** (kube-proxy vs Cilium), and they behave differently.

## Setup
Create the namespace and an app that reveals which pod answered (`traefik/whoami` returns its own hostname):
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
See where the pods landed:
```
kubectl get po -n topo-demo -o wide -l app=whoami
```

### What is that `topologySpreadConstraints` block?
Without it the scheduler is free to pile all replicas onto one node. `topologySpreadConstraints` tells the scheduler to **spread pods evenly across a topology domain**:

- `topologyKey: kubernetes.io/hostname` - the domain is "the node". So spread across nodes. Use `topology.kubernetes.io/zone` to spread across zones instead.
- `maxSkew: 1` - the pod counts between any two domains may differ by at most 1 (so on 2 nodes you get 2+2, not 3+1).
- `whenUnsatisfiable` - `ScheduleAnyway` (soft, best effort) or `DoNotSchedule` (hard, leave the pod Pending rather than break the skew).
- `labelSelector` - which pods count toward the skew (here, other `whoami` pods).

Why it matters for this lab: an **even spread is a prerequisite for topology-aware routing**. The topology-aware-hints algorithm only keeps traffic in-zone when each zone has a share of endpoints **proportional to its capacity**; if one zone has far fewer endpoints, it deliberately sends cross-zone traffic rather than overload the small zone. So "spread the pods" and "route locally" go together.

## Pinning pods with node affinity
> **Scheduler, not dataplane.** Everything about pod **placement** here, `topologySpreadConstraints`, `nodeSelector`, node affinity, pod affinity/anti-affinity, and DaemonSets, is decided by the kube-scheduler and behaves the **same on any CNI** (Cilium, Calico, kube-proxy). Only the **traffic-routing** parts further down (`internalTrafficPolicy` and topology-aware routing) have dataplane-specific behavior.

`topologySpreadConstraints` spreads pods out. Sometimes you instead want to **pin** pods to specific nodes (a zone, GPU nodes, an ingress tier). Two ways:

`nodeSelector` is the simple exact-match form:
```
spec:
  template:
    spec:
      nodeSelector:
        topology.kubernetes.io/zone: zone-a
```

`nodeAffinity` is the expressive form, with hard and soft rules:
```
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          # hard: only schedule where this matches
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["zone-a", "zone-b"]
          # soft: prefer these, but schedule elsewhere if needed
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
```
`IgnoredDuringExecution` means the rule is only checked at **scheduling** time; a pod is not evicted if the node's labels change later.

### Pod affinity and anti-affinity
Node affinity places pods relative to **node labels**. **Pod** affinity/anti-affinity places pods relative to **other pods**, using a `topologyKey` to define the domain (`kubernetes.io/hostname` = same node, `topology.kubernetes.io/zone` = same zone).

- **podAffinity**: schedule this pod **into the same domain** as pods matching a selector. Use it to co-locate components that interact a lot, for example placing an inline inspection or proxy pod on the **same node** as the workloads it protects, so that traffic never leaves the node before it is inspected.
- **podAntiAffinity**: schedule this pod **away from** pods matching a selector. Use it for high availability, for example spreading the replicas of a security gateway across different nodes or zones so a single node failure cannot take the whole tier down.

Example: keep at most one `waf` replica per node (anti-affinity for HA) and prefer to land on the same node as the `app` it fronts (affinity for locality):
```
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: waf
            topologyKey: kubernetes.io/hostname
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: app
              topologyKey: kubernetes.io/hostname
```
As with node affinity, `required...` is hard (leave the pod Pending if it cannot be satisfied) and `preferred...` is soft. Note that required anti-affinity across many pods gets expensive for the scheduler at scale, so `preferred` is often the pragmatic choice. This is also how you would pin a per-workload security sidecar next to the app it guards, or guarantee two replicas of an appliance never share a node.

> **DaemonSet vs anti-affinity.** Both can produce "one pod per node", but they answer different questions. A **DaemonSet** runs one copy on **every** matching node and follows the node set: add a node and a pod appears there, remove a node and its pod goes away. A **Deployment with `replicas: N` + required anti-affinity** runs a **fixed count** spread across distinct nodes: adding a node does nothing on its own, and setting `replicas` above the node count leaves the extras `Pending`. Use a DaemonSet for "every node needs this agent" (CNI, log or metrics collector, per-node security agent, see LAB020); use anti-affinity for "N replicas of my app, just not stacked on one node".

Start a client pinned to **one** worker node (using `nodeName`, the bluntest pin of all), so you control where traffic originates. Replace `<worker>`:
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
Curl again. Now you only ever hit the whoami pods on the client's node:
```
for i in $(seq 12); do kubectl exec -n topo-demo client -- curl -s whoami | grep Hostname; done | sort | uniq -c
```
The trade-off: if the client's node has **no** whoami pod, the request fails, there is no fallback. Reset when done:
```
kubectl patch svc whoami -n topo-demo -p '{"spec":{"internalTrafficPolicy":"Cluster"}}'
```

## B. Zone-local routing (topology-aware) - and why the dataplane matters
This part keeps traffic in the client's **zone**. First give yourself a balanced 2+2 spread across two simulated zones (kind has no real zones). Label the nodes, then re-roll whoami so it spreads by zone:
```
kubectl label no <worker-1> topology.kubernetes.io/zone=zone-a --overwrite
kubectl label no <worker-2> topology.kubernetes.io/zone=zone-b --overwrite
```
Make sure your `client` pod is on the `zone-a` node.

There are two ways to ask for zone-local routing, and on Cilium they are **not** equivalent:

### The modern field: `spec.trafficDistribution`
```
kubectl patch svc whoami -n topo-demo -p '{"spec":{"trafficDistribution":"PreferSameZone"}}'
```
The control plane sets per-zone hints on the EndpointSlice:
```
kubectl get endpointslices -n topo-demo -l kubernetes.io/service-name=whoami \
  -o jsonpath='{range .items[*].endpoints[*]}{.targetRef.name}{"  zone="}{.zone}{"  hint="}{.hints.forZones[*].name}{"\n"}{end}'
```
- On a **kube-proxy** cluster this is enough, traffic stays in-zone.
- On a **Cilium kube-proxy replacement** cluster (our course setup, tested on 1.19.6) it is **ignored**, traffic still spreads across both zones. Cilium does not act on the `trafficDistribution` field in this version.

### Cilium's way: the topology-mode annotation + serviceTopology
Cilium's topology-aware routing is driven by the older annotation plus a Helm flag. Enable the flag on the Cilium install:
```
helm upgrade cilium cilium/cilium --version 1.19.6 -n kube-system --reuse-values \
  --set loadBalancer.serviceTopology=true
kubectl -n kube-system rollout restart ds/cilium
```
Then annotate the service (drop the `trafficDistribution` field first so they do not fight):
```
kubectl patch svc whoami -n topo-demo --type=json -p='[{"op":"remove","path":"/spec/trafficDistribution"}]'
kubectl annotate svc whoami -n topo-demo service.kubernetes.io/topology-mode=Auto --overwrite
```
Now a `zone-a` client only hits the two `zone-a` pods:
```
for i in $(seq 16); do kubectl exec -n topo-demo client -- curl -s whoami | grep Hostname; done | sort | uniq -c
```
If you still see zone-b pods, check that the spread is balanced (2+2) and that the `client` is really on the `zone-a` node; with an unbalanced spread the safety heuristic falls back to cross-zone on purpose.

## Explore it yourself
* Recreate whoami with `nodeSelector: topology.kubernetes.io/zone: zone-a`. All pods now live in zone-a. What happens to a `zone-b` client under zone-local routing, and under `internalTrafficPolicy: Local`?
* Delete one zone-a pod so the spread becomes 1+2. Does zone-local routing still keep traffic in-zone, or does the safety heuristic kick in?
* These tune **east-west** traffic. How does that differ from `externalTrafficPolicy` in LAB040, which tunes **north-south** ingress?

## Cleanup
```
kubectl delete ns topo-demo
kubectl label no <worker-1> topology.kubernetes.io/zone-
kubectl label no <worker-2> topology.kubernetes.io/zone-
```
If you enabled it, you can also turn Cilium service topology back off with another `helm upgrade ... --set loadBalancer.serviceTopology=false` and a rollout restart.

> Takeaway: scheduling controls (`topologySpreadConstraints`, `nodeSelector`, `nodeAffinity`) decide **where pods run**; that placement is a prerequisite for locality. `internalTrafficPolicy: Local` keeps traffic on the client's node and is honored by every dataplane. Zone-local routing is dataplane-specific: kube-proxy honors `spec.trafficDistribution`, while Cilium honors the `service.kubernetes.io/topology-mode: Auto` annotation with `loadBalancer.serviceTopology=true`. Same intent, different switch, know which dataplane you are on.

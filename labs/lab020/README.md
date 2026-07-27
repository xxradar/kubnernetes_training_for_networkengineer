# LAB020 - Deployments and ReplicaSets

A **Deployment** declares the desired state of your app and manages a **ReplicaSet**, which keeps a fixed number of identical **pod** replicas running. If a pod dies or a node fails, the ReplicaSet recreates it. This is the self-healing reconciliation loop (desired state versus actual state), which a network engineer can picture as an auto-recovering pool of identical backends.

Two things matter for networking. Every replacement pod gets a **new IP**, and pods are tied to their ReplicaSet by **labels** (the `selector`). That constant churn is exactly why you never target a pod IP directly, and why Services (LAB030) exist.

> LAB020 to LAB050 run in sequence in the `prod-nginx` namespace, created in LAB010. Each lab builds on the previous one.

## Create a deployment
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: prod-nginx
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
        env: prod
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```
## Inspect the deployment
See the deployment itself:
```
kubectl get deploy -n prod-nginx -o wide
```
See the pods it created:
```
kubectl get po -n prod-nginx -o wide
```
Look at the deployment in detail, its update strategy, replica counts, and the ReplicaSet it points at:
```
kubectl describe deploy -n prod-nginx nginx-deployment
```
List the ReplicaSet the deployment created:
```
kubectl get rs -n prod-nginx
```
Inspect that ReplicaSet (use the name from the previous command):
```
kubectl describe rs -n prod-nginx <your_rs>
```
Look at the pod names: `nginx-deployment-<replicaset-hash>-<random>`. The `pod-template-hash` label is what ties each pod to its ReplicaSet, and the `describe deploy` output points at the `NewReplicaSet` that owns them.

## Labels and selectors
**Labels** are key/value tags you attach to objects (here `app: nginx` and `env: prod`). On their own they do nothing. A **selector** is a query that matches objects by their labels, and this is the glue that loosely couples things in Kubernetes without anyone hard-coding a pod IP or name.

* The Deployment's ReplicaSet owns exactly the pods that match its `spec.selector.matchLabels` (here `app: nginx`).
* A Service (LAB030) picks its backend pods the same way, with a label selector.
* The `pod-template-hash` label is added automatically by the Deployment, so pods from different rollout revisions can be told apart.

Filter pods by label from the CLI. Show every label on each pod:
```
kubectl get po -n prod-nginx --show-labels
```
Only the pods matching one label:
```
kubectl get po -n prod-nginx -l app=nginx
```
The AND of two labels:
```
kubectl get po -n prod-nginx -l env=prod,app=nginx
```
Show label values as their own columns:
```
kubectl get po -n prod-nginx -L app -L env
```
This is exactly the matching the ReplicaSet does to decide which pods it owns, and what a Service does to decide where to send traffic.

## Scaling
Scaling just changes the **desired** replica count, and the ReplicaSet reconciles by adding or removing pods until actual matches desired. Scale imperatively:
```
kubectl scale -n prod-nginx --replicas=5 deploy/nginx-deployment
```
Then look at the pods:
```
kubectl get po -n prod-nginx -o wide
```
Or declaratively by changing `spec.replicas` in the manifest and re-applying it.

For network engineers: every pod added by scaling up is a new pod with a **new IP**, possibly on a different node, and every pod removed by scaling down takes its IP with it. A Service in front (LAB030) tracks this automatically through its Endpoints, so the backend pool grows and shrinks while the address clients use stays the same.

Watch it happen live: run this in one terminal, then scale up or down in another.
```
kubectl get po -n prod-nginx -o wide -w
```

## Host network vs pod network (a first DaemonSet)
Normally a pod gets its **own IP** from the pod CIDR and its own network namespace, which is what you have seen so far. A pod with `hostNetwork: true` instead **shares the node's network namespace**: it has no separate pod IP, it uses the **node's IP**, and any `containerPort` binds directly on the node's interface.

For network engineers: a host-network pod bypasses the CNI and the overlay entirely, so there is no pod IP and no NAT, the process just listens on the node. This is how node-level agents (the CNI agent, kube-proxy, log and metrics collectors) run, and they are almost always deployed as a **DaemonSet**, which runs exactly one pod per node.

Deploy nginx as a host-network DaemonSet:
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-hostnet
  namespace: prod-nginx
  labels:
    app: nginx-hostnet
spec:
  selector:
    matchLabels:
      app: nginx-hostnet
  template:
    metadata:
      labels:
        app: nginx-hostnet
    spec:
      hostNetwork: true
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```
Now compare the addressing with the Deployment's pods:
```
kubectl get po -n prod-nginx -o wide -l app=nginx-hostnet
```
Two things stand out. There is **one pod per worker node** (that is the DaemonSet), and each pod's IP is the **node's IP**, not a `10.10.x` pod-CIDR address. The nginx process is now listening on port 80 of the node itself.

> One gotcha: because it binds the node's port 80 directly, only one host-network pod can use that port per node. If something else already holds port 80 on the node, the pod fails to start. That direct binding, with no pod IP and no NAT, is exactly the point of hostNetwork.

### Exercise
Work through these and reason about the "why".

* Show the pod labels: `kubectl get po -n prod-nginx -o wide --show-labels`
* Manually add a pod that matches the ReplicaSet selector, reusing the current `pod-template-hash`:
  `kubectl run -n prod-nginx --image nginx testnginx -l app=nginx,env=prod,pod-template-hash=<your_hash>`
  Re-check the ReplicaSet. What happened to your extra pod, and why?
* Delete one pod from the deployment. Does it come back? Same name? Same IP?
  `kubectl delete po -n prod-nginx <a_pod_name>`
* Scale the deployment and watch where the new pods land (which nodes):
  `kubectl scale -n prod-nginx --replicas=6 deploy/nginx-deployment`
  `kubectl get po -n prod-nginx -o wide`

> Takeaway: the ReplicaSet continuously reconciles to the desired replica count, so pods and their IPs come and go. You manage the set through labels, not through individual pods. That moving target is what a Service sits in front of, next in LAB030.

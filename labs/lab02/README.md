# LAB02 - Deployments and ReplicaSets

A **Deployment** declares the desired state of your app and manages a **ReplicaSet**, which keeps a fixed number of identical **pod** replicas running. If a pod dies or a node fails, the ReplicaSet recreates it. This is the self-healing reconciliation loop (desired state versus actual state), which a network engineer can picture as an auto-recovering pool of identical backends.

Two things matter for networking. Every replacement pod gets a **new IP**, and pods are tied to their ReplicaSet by **labels** (the `selector`). That constant churn is exactly why you never target a pod IP directly, and why Services (LAB03) exist.

> LAB02 to LAB05 run in sequence in the `prod-nginx` namespace, created in LAB01. Each lab builds on the previous one.

Create a deployment
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
Inspect the deployment, its pods, and the ReplicaSet it created
```
kubectl get deploy -n prod-nginx -o wide
kubectl get po -n prod-nginx -o wide
kubectl describe deploy -n prod-nginx nginx-deployment
kubectl get rs -n prod-nginx
kubectl describe rs -n prod-nginx <your_rs>
```
Look at the pod names: `nginx-deployment-<replicaset-hash>-<random>`. The `pod-template-hash` label is what ties each pod to its ReplicaSet, and the `describe deploy` output points at the `NewReplicaSet` that owns them.

## Labels and selectors
**Labels** are key/value tags you attach to objects (here `app: nginx` and `env: prod`). On their own they do nothing. A **selector** is a query that matches objects by their labels, and this is the glue that loosely couples things in Kubernetes without anyone hard-coding a pod IP or name.

* The Deployment's ReplicaSet owns exactly the pods that match its `spec.selector.matchLabels` (here `app: nginx`).
* A Service (LAB03) picks its backend pods the same way, with a label selector.
* The `pod-template-hash` label is added automatically by the Deployment, so pods from different rollout revisions can be told apart.

Look at, and filter by, labels from the CLI:
```
kubectl get po -n prod-nginx --show-labels               # show every label on each pod
kubectl get po -n prod-nginx -l app=nginx                # only pods matching this label
kubectl get po -n prod-nginx -l env=prod,app=nginx       # AND of both labels
kubectl get po -n prod-nginx -L app -L env               # show these label values as columns
```
This is exactly the matching the ReplicaSet does to decide which pods it owns, and what a Service does to decide where to send traffic.

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

> Takeaway for network engineers: the ReplicaSet continuously reconciles to the desired replica count, so pods and their IPs come and go. You manage the set through labels, not through individual pods. That moving target is what a Service sits in front of, next in LAB03.

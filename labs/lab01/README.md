# LAB04 - Learning about pods

A **pod** is the smallest deployable unit in Kubernetes, one or more containers that share a single network namespace. In network terms: every pod gets its **own IP address** (from the pod CIDR you configured in LAB01), and containers inside a pod reach each other over `localhost`. Pod-to-pod traffic is routed flat, with **no NAT** between pods.

A **namespace** is a logical boundary for grouping and isolating resources, think of it like a tenant or VRF for your objects (it scopes names and, later, network policy).

First, list the namespaces that already exist on the cluster:
```
kubectl get ns
```
You will see the built-in ones such as `default`, `kube-system`, and `kube-public`.

Create a namespace
```
kubectl create ns prod-nginx
```
Deploy a a single pod in the newly create namespace
```
kubectl apply -f pod.yaml
```
or 
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: prod-nginx
  labels:
    name: nginx
    environment: prod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
EOF
```
Analyse the output of kubectl with the different flags
```
kubectl get po                #show pods in default namespace
kubectl get po -n prod-nginx  #show pods in namespace prod-nginx
kubectl get po -A             #show all pods in all namespaces
```
```
kubectl get po -n prod-nginx -o wide    #show additional information like POD IP address and NODE information
NAME                                READY   STATUS    RESTARTS   AGE    IP              NODE           NOMINATED NODE   READINESS GATES
nginx-pod                           1/1     Running   0          136m   10.10.162.130   kind-worker    <none>           <none>
```
```
kubectl get po -n prod-nginx -o wide --show-labels   #show pod label information
NAME                                READY   STATUS    RESTARTS   AGE    IP              NODE           NOMINATED NODE   READINESS GATES   LABELS
nginx-pod                           1/1     Running   0          138m   10.10.162.130   kind-worker    <none>           <none>            environment=prod,name=nginx
```
### Exercise
Work through these yourself, the interesting part is figuring out the "why".

* Create a new namespace `lab01-exercise` (namespace names can't contain underscores, so no `lab01_exercise`)
* Create an `nginx` pod in the new namespace
* Find the IP address of the new pod
* Start an interactive throwaway pod in the same namespace:
  `kubectl run tmp -it -n lab01-exercise --rm --image ubuntu -- bash`
* From inside it, try to `curl` the nginx pod's IP
* If it fails, work out why and fix it (what does a bare `ubuntu` image not ship with?)
* Exit the pod (`exit`)
* Start the ubuntu pod again. What do you see, and what does that tell you about a pod's filesystem?
* Now run the throwaway pod in a **different** namespace (for example `default`) and `curl` the same nginx pod IP over in `lab01-exercise`. Does it still work? What does that tell you about whether a namespace is a network boundary by default?

> Takeaway for network engineers: pod IPs are ephemeral and a pod's filesystem resets on restart, so **Services** (next labs) give you a stable virtual IP in front of changing pods. And a namespace isolates names, not traffic, cross-namespace pod-to-pod reachability is open by default until you add a **NetworkPolicy** (LAB20 and LAB21).

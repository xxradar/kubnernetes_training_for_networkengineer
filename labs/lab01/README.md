# LAB01 - Learning about pods

A **pod** is the smallest deployable unit in Kubernetes - one or more containers that share a single network namespace. In network terms: every pod gets its **own IP address** (from the pod CIDR you configured in LAB00), and containers inside a pod reach each other over `localhost`. Pod-to-pod traffic is routed flat, with **no NAT** between pods.

A **namespace** is a logical boundary for grouping and isolating resources - think of it like a tenant or VRF for your objects (it scopes names and, later, network policy).

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
- Create additional pods
- Delete one of these pods. Does it come back?

### Explore it yourself
Start poking at the network behavior - the answers are more interesting than the commands:

- Which subnet do the pod IPs come from, and where was that decided? (Hint: LAB00.)
- From one pod, can you reach another pod's IP directly? Try:
  `kubectl exec -n prod-nginx nginx-pod -- curl -s <other-pod-ip>`
- Delete the pod and recreate it - does it get the **same IP** back? What does that imply for anything that hard-codes a pod IP?
- Look inside the pod's network stack: `kubectl exec -n prod-nginx nginx-pod -- ip addr`. How many interfaces? Where does the other end live?
- Are two pods on **different nodes** reachable without NAT? Check the `NODE` column with `-o wide` and test.
- How does a pod's IP relate to its **node's** IP (`kubectl get nodes -o wide`)? Same subnet or different?

> Takeaway for network engineers: pods are cattle, not pets - their IPs are ephemeral. That's exactly why **Services** (next labs) exist: a stable virtual IP in front of a changing set of pod IPs.

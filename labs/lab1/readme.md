### Lab 1
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
### Execrcise
- Create additional pods 
- Delete one of these pods. Does it come back?

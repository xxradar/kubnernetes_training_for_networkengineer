## Lab2
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
```
kubectl get deploy -n prod-nginx -o wide
```
```
kubectl get po -n prod-nginx -o wide
```
```
kubectl describe deploy -n prod-nginx nginx-deployment
```
```
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Tue, 09 Nov 2021 09:44:01 +0100
Labels:                 app=nginx-deployment
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-7848d4b86f (3/3 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  27m   deployment-controller  Scaled up replica set nginx-deployment-7848d4b86f to 3
```
```
kubectl get rs -n prod-nginx 
```
```
kubectl describe  rs -n prod-nginx <your_rs>
```
 
 
### Exercise
Check the POD labels
```
kubectl get po -n prod-nginx -o wide --show-labels
```
Try to add a pod, what happens ? (verify and change the `pod-template-hash`)
```
kubectl run  -n prod-nginx --image nginx testnginx -l app=nginx,env=prod,pod-template-hash=7848d4b86f 
 ```
 Check the replication set again ...
 ```
 kubectl describe rs -n prod-nginx <your_rs>
 ...
 ```
 Try to delete a pod from the deployment. What happens?
 ```
 kubectl delete po -n prod-nginx nginx-deployment-755b69f8f9-h9dvg
 ```
 Try to scale the deployment, what happens?
 ```
 kubectl scale -n prod-nginx --replicas=6 deploy/nginx-deployment
 ```
 ```
 kubectl describe rs -n prod-nginx <your_rs>
 ```
 
### Exercise
 - delete some pods from the deployment
 - check if the pods come back? What about IP addresses ?

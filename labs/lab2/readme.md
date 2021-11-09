## Lab2

```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```
```
kubectl get deploy -o wide
```
```
kubectl describe deploy nginx-deployment
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
### Exercise
Check the POD lables
```
kubectl get po -o wide --show-labels
```
Try to add a pod 
```
kubectl run  --image nginx -l app=nginx,pod-template-hash=7848d4b86f testnginx
 ```
 Check the replication set
 ```
 kubectl get rs
 ```
 ```
 kubectl describe rs nginx-deployment-7848d4b86f
 ...
 ```
 ```
 kubectl scale --replicas=6 deploy/nginx-deployment
 ```
 

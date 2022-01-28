##accessing kubernetes services
In a lab environment, you won't always have access via a load-balancer ...<br>
Also please keep in mind the network policies that are in place, they still apply.

```
kubectl port-forward svc/my-nginx-clusterip -n dev-nginx 8888:8765 &
```

```
curl http://127.0.0.1:8888
```
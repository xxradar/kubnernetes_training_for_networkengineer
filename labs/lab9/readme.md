## Accessing kubernetes services
In a lab environment, you won't always have access via a load-balancer ...<br>
Also please keep in mind the network policies that are in place, they still apply.

### `kubect port-forward` is an SSH style port forwarding mechanism baSED ON HTTP/2
```
kubectl port-forward svc/my-nginx-clusterip -n dev-nginx 8888:8765 &
```
```
curl http://127.0.0.1:8888
```
### `kubect port-forward` can listen for incoming connection on --address 0.0.0.0 (allows external access)
```
netstat -anpt tcp | grep 8888
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 127.0.0.1:8888          0.0.0.0:*               LISTEN      429505/kubectl
tcp6       0      0 ::1:8888                :::*                    LISTEN      429505/kubectl
```

```
kubectl port-forward svc/my-nginx-clusterip -n dev-nginx --address 0.0.0.0 8890:8765 &
```
```
curl http://<your-jumpbox>:8890
```
```
netstat -anpt tcp | grep 8890
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:8890            0.0.0.0:*               LISTEN      439248/kubectl
```
### An alternative way to access kubernetes is via `kubectl proxy`
Although possible, DON'T expose on --address 0.0.0.0. This allows full UN-AUTHENTICATED acesss to the kubernete API !!!
```
kubectl proxy &
```
```
curl http://localhost:8001/api/v1/namespaces/dev-nginx/services/http:my-nginx-clusterip:8765/proxy/
```
### Playing with the kubernetes API
```
curl http://localhost:8001/api/v1/
```

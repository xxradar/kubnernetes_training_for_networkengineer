# LAB035 - Accessing kubernetes services

In a lab, or on a locked-down cluster, you will not always have a LoadBalancer or Ingress to reach a service. `kubectl` gives you a couple of ways to reach a ClusterIP service directly from your workstation, tunnelled through the API server. Keep in mind that any network policies in place still apply.

This lab uses the `my-nginx-clusterip` service in `prod-nginx` (service port `80`) from the earlier labs.

## kubectl port-forward (local only)
`kubectl port-forward` is an SSH-style port forward over HTTP/2, tunnelled through the API server. Forward a local port to the service:
```
kubectl port-forward svc/my-nginx-clusterip -n prod-nginx 8888:80 &
```
Reach it on localhost:
```
curl http://127.0.0.1:8888
```
Check what is listening. By default it only binds `127.0.0.1`, so it is reachable from this host only:
```
netstat -anpt tcp | grep 8888
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 127.0.0.1:8888          0.0.0.0:*               LISTEN      429505/kubectl
tcp6       0      0 ::1:8888                :::*                    LISTEN      429505/kubectl
```

## kubectl port-forward on all interfaces (external access)
Add `--address 0.0.0.0` to also listen on the host's external interfaces, so you can reach it through your jumpbox:
```
kubectl port-forward svc/my-nginx-clusterip -n prod-nginx --address 0.0.0.0 8890:80 &
```
```
curl http://<your-jumpbox>:8890
```
Now it binds all interfaces:
```
netstat -anpt tcp | grep 8890
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:8890            0.0.0.0:*               LISTEN      439248/kubectl
```

## kubectl proxy
`kubectl proxy` opens an authenticated local proxy to the Kubernetes API, and you can reach a service through the API's service-proxy subresource.

> Do NOT expose this on `--address 0.0.0.0`. It allows full UN-AUTHENTICATED access to the Kubernetes API.
```
kubectl proxy &
```
Reach the service through the API proxy (note the `namespaces/<ns>/services/http:<name>:<port>/proxy/` path):
```
curl http://localhost:8001/api/v1/namespaces/prod-nginx/services/http:my-nginx-clusterip:80/proxy/
```

## Playing with the kubernetes API
The same proxy lets you browse the API directly:
```
curl http://localhost:8001/api/v1/
```

## Cleanup (important)
The commands above run in the **background** (`&`) and keep listening after the lab. This matters for security: if anyone used `--address 0.0.0.0` (on the port-forward, or worse on `kubectl proxy`), the service, or an **unauthenticated path to the Kubernetes API**, is exposed on every interface of the host, reachable from outside the jumpbox. Always shut them down.

Kill the port-forwards and the proxy:
```
pkill -f "kubectl port-forward"
pkill -f "kubectl proxy"
```
Verify nothing is still listening on those ports (should print nothing):
```
ss -lntp | grep -E ':8888|:8890|:8001' || echo "clean"
```
`jobs` also shows any still-running background jobs in this shell; `kill %1 %2 ...` clears them by job number.

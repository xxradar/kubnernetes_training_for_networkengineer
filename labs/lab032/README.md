# LAB032 - Health probes (readiness, liveness, startup)

> Continues from LAB030: the `prod-nginx` namespace, the `nginx-deployment`, and the `my-nginx-clusterip` service already exist.

Kubernetes uses three probes to decide the state of a container. For a network engineer the key one is the **readiness** probe, because it controls whether a pod is in the Service Endpoints, and therefore whether it receives traffic.

## The three probes
- **readiness**: is the pod ready to serve? While it fails, the pod is removed from the Service Endpoints, so it gets **no traffic**. The container is **not** restarted. This is the traffic control.
- **liveness**: is the container still healthy? When it fails, the kubelet **restarts** the container.
- **startup**: has a slow app finished starting? Until it passes, liveness and readiness are held off, so a slow starter is not killed before it is up.

## Add probes to the nginx deployment
Re-apply the `nginx-deployment` from the earlier labs, this time with probes added. The container writes a `/healthz` file (served over HTTP) and a `/tmp/alive` file before starting nginx. The readiness and startup probes check `/healthz` over HTTP, the liveness probe checks `/tmp/alive`. Toggling those two files lets you drive each probe by hand.
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
        command: ["/bin/sh","-c","echo ok > /usr/share/nginx/html/healthz; touch /tmp/alive; exec nginx -g 'daemon off;'"]
        ports:
        - containerPort: 80
        startupProbe:
          httpGet:
            path: /healthz
            port: 80
          failureThreshold: 30
          periodSeconds: 2
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
          periodSeconds: 3
        livenessProbe:
          exec:
            command: ["cat", "/tmp/alive"]
          periodSeconds: 5
EOF
```
This rolls the deployment. All pods should become `READY 1/1`, and their IPs should appear in the Endpoints of the existing `my-nginx-clusterip` service:
```
kubectl get po -n prod-nginx -l app=nginx -o wide
```
```
kubectl get endpointslices -n prod-nginx -l kubernetes.io/service-name=my-nginx-clusterip
```

## Readiness gates the Endpoints
Break the readiness check on **one** pod by deleting its `/healthz` file, so `GET /healthz` starts returning 404:
```
kubectl exec -n prod-nginx <one-nginx-pod> -- rm /usr/share/nginx/html/healthz
```
Watch that pod go `READY 0/1` within a few seconds, while its `RESTARTS` stays at `0`:
```
kubectl get po -n prod-nginx -l app=nginx -w
```
Now look at the Endpoints again. The NotReady pod's IP is **gone**, so the Service no longer sends it traffic:
```
kubectl get endpointslices -n prod-nginx -l kubernetes.io/service-name=my-nginx-clusterip
```
Put the file back and the pod returns to the Endpoints:
```
kubectl exec -n prod-nginx <one-nginx-pod> -- sh -c 'echo ok > /usr/share/nginx/html/healthz'
```

## Liveness restarts the container
Break the liveness check by removing `/tmp/alive`:
```
kubectl exec -n prod-nginx <one-nginx-pod> -- rm /tmp/alive
```
After a few failed checks the kubelet restarts the container. Watch `RESTARTS` go up:
```
kubectl get po -n prod-nginx -l app=nginx -w
```
On restart the container command recreates both files, so the pod recovers on its own. Note the difference from readiness: liveness **restarts**, readiness only **removes from Endpoints**.

## Startup probe
The startup probe runs first and holds off the readiness and liveness probes until it passes (here `failureThreshold: 30` x `periodSeconds: 2` = up to 60s of grace). It exists so a slow-starting app is not killed by the liveness probe before it has finished coming up. nginx starts instantly, so you will not see it flap here, but the pattern matters for real workloads (JVMs, databases, apps that warm caches).

## Explore it yourself
* Break readiness on one pod, then `curl my-nginx-clusterip` a few times from a test pod. Do you ever hit the broken pod?
* What is the difference in `RESTARTS` between the readiness break and the liveness break?
* Why would a liveness probe on a slow-starting app, without a startup probe, cause a restart loop?

> Takeaway: readiness controls **traffic** (in or out of the Endpoints), liveness controls **restarts**, and startup gives a slow app **grace** before the other two apply. Readiness is the one that plugs straight into the Service and Endpoints model from LAB030.

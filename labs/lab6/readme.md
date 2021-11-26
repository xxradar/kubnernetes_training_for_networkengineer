## LAB6 - Ingress <br>
Checkout this repo for a detailed Ingress Lab example using NGINX <br>
https://github.com/xxradar/ingress_kubernetes_workshop

If you're using Rancher Desktop, ingress is implemented via Traefik.<br>
Verify the service node ports.
```
kubectl get svc -n kube-system   traefik
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
traefik   LoadBalancer   10.43.45.176   192.168.5.15   80:31056/TCP,443:31101/TCP   21h
xxradar@Philippes-MacBook-Pro-2 dev-fortinet %
```
Let's store the node ports into an env variable for easy use.
```
export SECUREWEB=$(kubectl get svc -n kube-system  traefik -o=jsonpath="{.spec.ports[?(@.port==443)].nodePort}")
export WEB=$(kubectl get svc -n kube-system   traefik -o=jsonpath="{.spec.ports[?(@.port==80)].nodePort}")
echo $WEB
echo $SECUREWEB 
```
Let's create an ingress resource for HTTP

```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app1-ingress
spec:
  rules:
  - host: app1.dockersec.me
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-nginx-clusterip
            port:
              number: 80
EOF
```
Verify access
```
curl -kv http://localhost:$WEB
...
curl -kv  -H "Host: app1.dockersec.me" http://localhost:$WEB
```
In order to enable TLS, we need to create a certificate.
```
openssl req -x509 -newkey rsa:2048 -keyout tls.key -out tls.crt -days 365 -nodes -subj "/CN=tlsapp1.dockersec.me"
kubectl create secret tls tlscertsapp1 -n app1 --cert=./tls.crt --key=./tls.key
kubectl describe secret -n app1 tlscertsapp1
```
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
  annotations:
spec:
  tls:
  - hosts:
      - tlsapp1.dockersec.me
    secretName: tlscertsapp1
  rules:
  - host: tlsapp1.dockersec.me
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-nginx-clusterip
            port:
              number: 80
EOF
```
```
kubectl get ingress -n prod-nginx
```
Update your hostfile !!!

```
Add to /etc/hosts
127.0.0.1 tlsapp1.dockersec.me
```
Check carefully the TLS handshake.
```
curl -kv  -H "Host: tlsapp1.dockersec.me" https://tlsapp1.dockersec.me:$SECUREWEB
curl -kv  -H "Host: tlsapp1.dockersec.me" https://localhost:$SECUREWEB
```

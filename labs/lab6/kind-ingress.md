## LAB 6 - Ingress <br>
### KIND - Nginx ingress
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.45.0/deploy/static/provider/baremetal/deploy.yaml
```
```
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.11.22.255   <none>        80:32060/TCP,443:30340/TCP   8m53s
ingress-nginx-controller-admission   ClusterIP   10.11.22.36    <none>        443/TCP                      8m53s
```
```
export SECUREWEB=$(kubectl get svc ingress-nginx-controller -n ingress-nginx  -n ingress-nginx -o=jsonpath="{.spec.ports[?(@.port==443)].nodePort}")
export WEB=$(kubectl get svc ingress-nginx-controller -n ingress-nginx  -n ingress-nginx -o=jsonpath="{.spec.ports[?(@.port==80)].nodePort}")
export WEB=$(kubectl get svc ingress-nginx-controller -n ingress-nginx  -n ingress-nginx -o=jsonpath="{.spec.ports[?(@.port==80)].nodePort}")
export NODE=$(kubectl get no kind-worker2 -o=jsonpath="{.status.addresses[0].address}")
echo $WEB
echo $SECUREWEB 
echo $NODE
```
```
curl -kv http://$NODE:$WEB  #change the portnumber according kubectl svc -n ingress-nginx
curl -kv https://$NODE:$SECUREWEB  #change the portnumber according kubectl svc -n ingress-nginx
```
Let's create an ingress resource for HTTP

```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: app1-ingress
spec:
  rules:
  - host: app1.dockersec.me
    http:
      paths:
      - backend:
          serviceName: my-nginx-clusterip
          servicePort: 80
EOF
```
Verify access
```
curl -kv http://$NODE:$WEB
...
curl -kv  -H "Host: app1.dockersec.me" http://$NODE:$WEB
```
In order to enable TLS, we need to create a certificate.
```
openssl req -x509 -newkey rsa:2048 -keyout tls.key -out tls.crt -days 365 -nodes -subj "/CN=tlsapp1.dockersec.me"
kubectl create secret tls tlscertsapp1 -n prod-nginx --cert=./tls.crt --key=./tls.key
kubectl describe secret -n prod-nginx tlscertsapp1
```
Create an ingress resource for HTTPS
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: tls-example-ingress
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
        backend:
          serviceName: my-nginx-clusterip
          servicePort: 80
EOF
```
```
kubectl get ingress -n prod-nginx
```
Update your hostfile !!!

```
Add to /etc/hosts
<NODE_ADDRESS> tlsapp1.dockersec.me
```
Check carefully the TLS handshake.
```
curl -kv  -H "Host: tlsapp1.dockersec.me" https://tlsapp1.dockersec.me:$SECUREWEB
curl -kv  -H "Host: tlsapp1.dockersec.me" https://$NODE:$SECUREWEB
```

## LAB 6 - Ingress <br>
### EKS - Nginx ingress
```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/aws/nlb-with-tls-termination/deploy.yaml
```
```
kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP                                                                        PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.85.87     a62334251f0114b9aaf34d86202c2882-40e81f7ab83cdc16.elb.eu-central-1.amazonaws.com   80:31384/TCP,443:32434/TCP   9m23s
ingress-nginx-controller-admission   ClusterIP      10.100.148.204   <none>                                                                             443/TCP                      9m23s
```
```
export NODE=a62334251f0114b9aaf34d86202c2882-40e81f7ab83cdc16.elb.eu-central-1.amazonaws.com
```
```
curl -kv http://$NODE:80  #change the portnumber according kubectl svc -n ingress-nginx
curl -kv https://$NODE:443  #change the portnumber according kubectl svc -n ingress-nginx
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
apiVersion: networking.k8s.io/v1
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

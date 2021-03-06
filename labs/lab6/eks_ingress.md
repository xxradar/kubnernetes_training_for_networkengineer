## LAB 6 - Ingress <br>
### Install NGIX Ingress controller
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
### Create an ingress resource for HTTP

```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app1-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: "nginx"
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
curl -kv http://$NODE:80
...
curl -kv  -H "Host: app1.dockersec.me" http://$NODE:80
```
### Create an ingress resource for HTTPS

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
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: "nginx"
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
<LB_ADDRESS> tlsapp1.dockersec.me
```
Check carefully the TLS handshake.
```
curl -kv  -H "Host: tlsapp1.dockersec.me" https://tlsapp1.dockersec.me
curl -kv  -H "Host: tlsapp1.dockersec.me" https://$NODE
```

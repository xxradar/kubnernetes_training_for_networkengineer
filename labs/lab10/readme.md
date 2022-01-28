## Troubleshooting

```
kubectl get po -n prod-nginx
...

kubectl exec -it -n prod-nginx nginx-deployment-77b8976db4-2n6gp -- bash
root@nginx-deployment-77b8976db4-2n6gp:/# # ls -la
total 88
drwxr-xr-x   1 root root 4096 Jan 28 20:13 .
drwxr-xr-x   1 root root 4096 Jan 28 20:13 ..
drwxr-xr-x   2 root root 4096 Jan 25 00:00 bin
drwxr-xr-x   2 root root 4096 Dec 11 17:25 boot
drwxr-xr-x   5 root root  360 Jan 28 20:13 dev
drwxr-xr-x   1 root root 4096 Jan 26 08:58 docker-entrypoint.d
-rwxrwxr-x   1 root root 1202 Jan 26 08:58 docker-entrypoint.sh
drwxr-xr-x   1 root root 4096 Jan 28 20:13 etc
drwxr-xr-x   2 root root 4096 Dec 11 17:25 home
drwxr-xr-x   1 root root 4096 Jan 25 00:00 lib
...
```

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
```
kubectl logs -n prod-nginx nginx-deployment-77b8976db4-2n6gp
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2022/01/28 20:13:23 [notice] 1#1: using the "epoll" event method
2022/01/28 20:13:23 [notice] 1#1: nginx/1.21.6
2022/01/28 20:13:23 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6)
2022/01/28 20:13:23 [notice] 1#1: OS: Linux 5.11.0-1022-aws
2022/01/28 20:13:23 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/01/28 20:13:23 [notice] 1#1: start worker processes
2022/01/28 20:13:23 [notice] 1#1: start worker process 31
2022/01/28 20:13:23 [notice] 1#1: start worker process 32
2022/01/28 20:13:23 [notice] 1#1: start worker process 33
2022/01/28 20:13:23 [notice] 1#1: start worker process 34
10.10.162.135 - - [28/Jan/2022:20:29:34 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.68.0" "172.18.0.5"
10.10.110.139 - - [28/Jan/2022:20:30:37 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.68.0" "-"
10.10.110.143 - - [28/Jan/2022:20:34:10 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.68.0" "-"
10.10.110.157 - - [28/Jan/2022:20:51:33 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.68.0" "-"
...
```

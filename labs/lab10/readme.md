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
kubectl run -it --rm debug  --restart=Never --image=xxradar/hackon --overrides='{"kind":"Pod", "apiVersion":"v1", "spec": {"hostNetwork":true}}'
If you don't see a command prompt, try pressing enter.
root@kind-worker2:/#
...
```
```
root@kind-worker2:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: calibb5a86c99dc@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
4: vxlan.calico: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 66:03:c5:fc:0f:e5 brd ff:ff:ff:ff:ff:ff
    inet 10.10.110.128/32 scope global vxlan.calico
       valid_lft forever preferred_lft forever
5: cali968e7ff47c5@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
6: calif4e18bca3de@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 3
10: calie634c06af99@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 4
83: eth0@if84: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:05 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.18.0.5/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fc00:f853:ccd:e793::5/64 scope global nodad
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe12:5/64 scope link
       valid_lft forever preferred_lft forever
...
```
```
root@kind-worker2:/# tcpdump -i any -n
21:42:26.770551 IP 172.18.0.3.37542 > 172.18.0.5.10250: Flags [.], ack 541207, win 501, options [nop,nop,TS val 1037616719 ecr 3538320227], length 0
21:42:26.770619 IP 127.0.0.1.57908 > 127.0.0.1.42161: Flags [.], ack 526104, win 10741, options [nop,nop,TS val 1256196285 ecr 1256196285], length 0
21:42:26.770634 IP 172.18.0.5.10250 > 172.18.0.3.37542: Flags [P.], seq 541207:541233, ack 0, win 501, options [nop,nop,TS val 3538320227 ecr 1037616719], length 26
21:42:26.770665 IP 172.18.0.3.37542 > 172.18.0.5.10250: Flags [.], ack 541233, win 501, options [nop,nop,TS val 1037616719 ecr 3538320227], length 0
21:42:26.770729 IP 127.0.0.1.42161 > 127.0.0.1.57908: Flags [P.], seq 526104:527530, ack 1, win 512, options [nop,nop,TS val 1256196285 ecr 1256196285], length 1426
21:42:26.770784 IP 127.0.0.1.57908 > 127.0.0.1.42161: Flags [.], ack 527530, win 10741, options [nop,nop,TS val 1256196285 ecr 1256196285], length 0
21:42:26.770882 IP 127.0.0.1.42161 > 127.0.0.1.57908: Flags [P.], seq 527530:527534, ack 1, win 512, options [nop,nop,TS val 1256196285 ecr 1256196285], length 4
21:42:26.770902 IP 127.0.0.1.42161 > 127.0.0.1.57908: Flags [P.], seq 527534:527538, ack 1, win 512, options [nop,nop,TS val 1256196285 ecr 1256196285], length 4
21:42:26.770912 IP 127.0.0.1.42161 > 127.0.0.1.57908: Flags [P.], seq 527538:528962, ack 1, win 512, options [nop,nop,TS val 1256196285 ecr 1256196285], length 1424
21:42:26.770934 IP 127.0.0.1.57908 > 127.0.0.1.42161: Flags [.], ack 527538, win 10741, options [nop,nop,TS val 1256196285 ecr 1256196285], length 0
21:42:26.770985 IP 172.18.0.5.10250 > 172.18.0.3.37542: Flags [P.], seq 542681:542711, ack 0, win 501, options [nop,nop,TS val 3538320227 ecr 1037616719], length 30
21:42:26.771172 IP 172.18.0.3.37542 > 172.18.0.5.10250: Flags [.], ack 544483, win 501, options [nop,nop,TS val 1037616720 ecr 3538320228], length 0

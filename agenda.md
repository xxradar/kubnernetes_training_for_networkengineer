# Kubernetes for Network & Security Engineers

## Session 1 - Foundations
- Intro to k8s + control plane (api-server, etcd, scheduler, controller-manager, kubelet/CRI)
- Setting up a cluster (kind)
- Networking and CNI (Calico / Cilium)
- CoreDNS / service discovery
- Running a first pod
- Pods, ReplicaSets, Deployments
+ labs 000, 010, 020

## Session 2 - Workloads & Security Context

- DaemonSets / StatefulSets
- Namespaces
- Health probes (liveness / readiness)
- securityContext hardening (runAsNonRoot, readOnlyRootFS, drop caps, seccomp)
- Resource requests / limits (as a security control)
- Replicas and scaling
- Intro storage
+ labs

## Session 3 - Services & Networking
- Services (ClusterIP, NodePort, LoadBalancer) + Endpoints / EndpointSlices
- kube-proxy dataplane (iptables / IPVS / eBPF, Cilium kube-proxy replacement)
- Ingress
- Gateway API
- Network policies (default-deny)
+ labs 030, 040, 050

## Session 4 - Operations
- Secrets and ConfigMaps
- RBAC + ServiceAccounts
- Introduction to Helm
- Observability & metrics (metrics-server, Prometheus model, logs, Hubble)
+ labs

## Session 5 - Security Posture
- Pod Security Standards
- Admission control (Kyverno / Gatekeeper)
- Image security (scanning: Trivy / signing: cosign, verified at admission)
- Audit logs
- Events
+ labs

# LAB00 - Cilium CNI

Installs [Cilium](https://cilium.io/) as the CNI on the kind cluster from [kind_setup.md](kind_setup.md), with WireGuard pod encryption, kube-proxy replacement, and Hubble observability.

---

## 1. Install Helm

```bash
sudo snap install helm --classic
```

---

## 2. Install the Cilium and Hubble CLIs

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

```bash
export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz{,.sha256sum}
```

---

## 3. Install Cilium

```bash
helm repo add cilium https://helm.cilium.io/
```

```bash
helm install cilium cilium/cilium --version 1.19.6 \
    --namespace kube-system \
    --set ipam.mode=cluster-pool \
    --set ipam.operator.clusterPoolIPv4PodCIDRList={10.10.0.0/16} \
    --set ipam.operator.clusterPoolIPv4MaskSize=24 \
    --set kubeProxyReplacement=true \
    --set encryption.enabled=true \
    --set encryption.type=wireguard \
    --set hubble.relay.enabled=true \
    --set hubble.ui.enabled=true \
    --set ingressController.enabled=true \
    --set ingressController.loadbalancerMode=shared \
    --set ingressController.service.type="NodePort" \
    --set loadBalancer.l7.backend=envoy
```

| Flag | Purpose |
|------|---------|
| `ipam.operator.clusterPoolIPv4PodCIDRList={10.10.0.0/16}` | Pod IP pool, matching the kind `podSubnet` |
| `ipam.operator.clusterPoolIPv4MaskSize=24` | `/24` slice per node (254 pods/node) |
| `kubeProxyReplacement=true` | Cilium handles service routing in eBPF, replacing kube-proxy |
| `encryption.type=wireguard` | Transparent pod-to-pod encryption |
| `ingressController.enabled=true` | Built-in Cilium ingress |
| `loadBalancer.l7.backend=envoy` | Envoy-based L7 load balancing |

Wait for Cilium to be ready:

```bash
cilium status --wait
cilium connectivity test   # optional, takes several minutes
```

```bash
kubectl get no
```

Confirm kube-proxy replacement and WireGuard are active:

```bash
kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium status | grep -iE "KubeProxyReplacement|Encryption"
```

---

## 4. Hubble observability

```bash
cilium hubble port-forward&
hubble status
hubble observe
```

Access the Hubble UI:

```bash
kubectl port-forward -n kube-system svc/hubble-ui --address 0.0.0.0 --address :: 12000:80
```

Then open `http://<INSTANCE_PUBLIC_IP>:12000` (allow TCP `12000` from your IP in the security group).

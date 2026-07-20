# Lab 0 — Calico CNI

Installs [Calico](https://docs.tigera.io/calico/latest/about/) as the CNI on the kind cluster from [kind_setup.md](kind_setup.md).

---

## 1. Install Calico

Download the Calico manifest and set the default IP pool to match the cluster's pod CIDR (`10.10.0.0/16`):

```bash
curl -sLO https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/calico.yaml
sed -i \
  -e 's|# - name: CALICO_IPV4POOL_CIDR|- name: CALICO_IPV4POOL_CIDR|' \
  -e 's|#   value: "192.168.0.0/16"|  value: "10.10.0.0/16"|' \
  calico.yaml
```

Apply it. `calico-node` and `calico-kube-controllers` are deployed into the `kube-system` namespace:

```bash
kubectl apply -f calico.yaml
```

---

## 2. Verify

Watch the Calico pods roll out (`Ctrl+C` to exit):

```bash
watch kubectl get po -n kube-system -l k8s-app=calico-node
```

When all `calico-node` pods are `Running`, the nodes become `Ready`:

```bash
kubectl get no -o wide
```

Confirm the IP pool matches the cluster pod CIDR and pods get `10.10.x` addresses:

```bash
kubectl get ippools -o jsonpath='{.items[*].spec.cidr}'; echo
kubectl get po -n kube-system -o wide
```

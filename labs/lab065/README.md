# LAB065 - Gateway API (Envoy Gateway)

The **Gateway API** is the modern successor to Ingress (LAB063). Same job, expose HTTP(S) services at L7, but a cleaner, **role-oriented** model built from typed objects instead of controller-specific annotations:

- **GatewayClass** - which controller/implementation handles gateways (the "vendor"). Cluster-scoped, set by the platform.
- **Gateway** - the listeners and the external address (the VIP / entry point). Owned by the **platform/infra team**.
- **HTTPRoute** (and `TLSRoute`, `GRPCRoute`, …) - the actual routing rules. Owned by the **app teams**, in their own namespaces, attached to a Gateway.

That separation is the whole point: infra owns the entry point, app teams own their routes, and features like traffic-splitting or header matching are **first-class fields**, not opaque annotations.

We use **Envoy Gateway** as the implementation. The Gateway is exposed through a LoadBalancer Service, so we reuse the **MetalLB** pool from LAB050 to give it an external IP.

> Standalone lab. Runs on the kind + Cilium cluster from LAB000. **Requires the MetalLB LoadBalancer from LAB050** so the Gateway gets an external IP. Uses namespaces `envoy-gateway-system` (the Gateway) and `demo-app` (the route + app).

## Prerequisite: a LoadBalancer (MetalLB from LAB050)
Envoy Gateway exposes the Gateway through a Service of type `LoadBalancer`, so the cluster needs something to hand it an external IP. On KIND that is the MetalLB pool from LAB050. Confirm it is present:
```
kubectl get ipaddresspool -n metallb-system
```
If you skipped LAB050, install MetalLB and its pool first (see LAB050). On a managed cloud (EKS/AKS/GKE) the cloud load balancer fills this role automatically, nothing to install.

## 1. Install the Gateway API CRDs
The API types are not built in; install the **standard channel** (GatewayClass, Gateway, HTTPRoute, …):
```
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
kubectl get crds | grep gateway.networking.k8s.io
```

## 2. Install Envoy Gateway
Envoy Gateway is the controller that watches Gateway/HTTPRoute objects and programs actual Envoy proxies. Install it with Helm (skip CRDs, we just installed the Gateway API ones):
```
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.3 -n envoy-gateway-system --create-namespace --skip-crds
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

## 3. Create the GatewayClass
The GatewayClass points at the Envoy Gateway controller (the "pick your vendor" step):
```
kubectl apply -f gatewayclass.yaml
kubectl get gatewayclass
```
```
NAME   CONTROLLER                                      ACCEPTED   AGE
eg     gateway.envoyproxy.io/gatewayclass-controller   True       5s
```
`ACCEPTED: True` means the controller recognised it.

## 4. Create the Gateway
This defines an HTTP listener on port 80 and, via `allowedRoutes`, lets routes from any namespace attach. Envoy Gateway creates a LoadBalancer Service for it, which **MetalLB** assigns an IP from the LAB050 pool:
```
kubectl apply -f gateway.yaml
kubectl wait --timeout=5m -n envoy-gateway-system gateway/eg-gateway --for=condition=Programmed
kubectl get gateway -n envoy-gateway-system
```
The `ADDRESS` column shows the external IP (e.g. `172.18.255.200`). If it stays empty, MetalLB isn't handing out an IP, recheck the prerequisite above.

## 5. Deploy the app and attach a route
A simple echo app plus its Service, and an **HTTPRoute** that attaches to the Gateway and sends `webapp.local.dev` to the Service:
```
kubectl apply -f webapp.yaml
kubectl apply -f httproute.yaml
kubectl get httproute -n demo-app
```

## 6. Test it
Grab the Gateway's address and curl it with the matching Host header:
```
export GW=$(kubectl get gateway eg-gateway -n envoy-gateway-system -o jsonpath='{.status.addresses[0].value}')
curl -H "Host: webapp.local.dev" http://$GW/
```
You should get `Hello from Gateway API! Pod: webapp-...`. Repeat and watch the pod name change as it load-balances.
> On KIND the LB IP lives on the `kind` docker network. If your shell can't reach it directly, run the client on that network: `docker run --rm --network kind curlimages/curl -H "Host: webapp.local.dev" http://$GW/`.

## How this differs from Ingress (LAB063)
- **Roles are separated.** The `Gateway` (infra) and the `HTTPRoute` (app team, in `demo-app`) are distinct objects with distinct RBAC. In Ingress it was one blob.
- **Routing is typed, not annotated.** Header matches, traffic splitting, method matches, cross-namespace refs are **fields** in the spec. In Ingress each of those needed a controller-specific annotation.
- **Portable.** Swap Envoy Gateway for another Gateway API implementation and the `Gateway`/`HTTPRoute` objects stay the same, only the `GatewayClass` changes.

## Explore it yourself
* **Traffic split / canary:** add a second Deployment (`version: v2`) and a second `backendRefs` entry in the HTTPRoute with `weight: 10` vs `weight: 90`. No annotations, just weights.
* **Header match:** add a rule that matches header `X-Canary: true` and routes it to v2. Compare that to what LAB063's Ingress would have needed.
* **Second route:** add another HTTPRoute (different hostname) attached to the **same** Gateway, that is the shared-entry-point / role-split model in action.
* **TLS:** add an HTTPS listener (port 443, `mode: Terminate`) with a TLS Secret and re-test over https.

## Cleanup
```
kubectl delete -f httproute.yaml -f webapp.yaml -f gateway.yaml -f gatewayclass.yaml
helm uninstall eg -n envoy-gateway-system
```
(You can leave the Gateway API CRDs and MetalLB in place for later labs.)

> Takeaway: the Gateway API replaces Ingress with a typed, role-oriented model, `GatewayClass` (vendor), `Gateway` (infra's listeners + VIP), `HTTPRoute` (app team's rules). Here Envoy Gateway is the implementation and MetalLB (from LAB050) provides the external IP. Advanced routing (canary, header match, cross-namespace) is built into the API instead of hiding in controller-specific annotations.

# TODO

## After the training
- [ ] Migrate the course clusters from Cilium 1.19.x to **Cilium 1.20** to stay current.
  - 1.20 adds kube-proxy-replacement support for the `trafficDistribution` field (`PreferSameZone` / `PreferSameNode`), which 1.19.x ignored (it only honored the `service.kubernetes.io/topology-mode: Auto` annotation). Still gated behind `loadBalancer.serviceTopology=true`.
  - Re-test LAB037 (traffic locality) thoroughly on 1.20: does `trafficDistribution: PreferSameZone` now keep traffic in-zone on Cilium? Check the no-local-endpoint fallback behavior (see cilium/cilium issues #41022 and #40883).
  - If it works, update LAB037 to prefer the `trafficDistribution` field over the legacy annotation, and add a "Cilium 1.19 vs 1.20" version note.
  - Cross-cluster locality is separate: verify ClusterMesh `service.cilium.io/affinity: local` on global services (`service.cilium.io/global: "true"`).

- [ ] Build a small **custom app container** (Node.js) with a real app and proper health endpoints (`/livez`, `/readyz`, maybe `/startupz`), plus a runtime toggle to flip readiness/liveness on demand (e.g. `POST /toggle/ready`), so we have a clean code -> image -> registry flow.
  - Use this image as "our app" across the labs instead of `nginx`.
  - Rework LAB032 (probes) on top of it: drop the `/healthz` file and `/tmp/alive` file tricks and drive the probes through the app's real endpoints and toggle. Makes the probe demos cleaner and sequential.
  - Build it at the **end of the training**, so we fold in all the gotchas we hit along the way.

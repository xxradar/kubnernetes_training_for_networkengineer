# TODO

## After the training
- [ ] Migrate the course clusters from Cilium 1.19.x to **Cilium 1.20** to stay current.
  - 1.20 adds kube-proxy-replacement support for the `trafficDistribution` field (`PreferSameZone` / `PreferSameNode`), which 1.19.x ignored (it only honored the `service.kubernetes.io/topology-mode: Auto` annotation). Still gated behind `loadBalancer.serviceTopology=true`.
  - Re-test LAB037 (traffic locality) thoroughly on 1.20: does `trafficDistribution: PreferSameZone` now keep traffic in-zone on Cilium? Check the no-local-endpoint fallback behavior (see cilium/cilium issues #41022 and #40883).
  - If it works, update LAB037 to prefer the `trafficDistribution` field over the legacy annotation, and add a "Cilium 1.19 vs 1.20" version note.
  - Cross-cluster locality is separate: verify ClusterMesh `service.cilium.io/affinity: local` on global services (`service.cilium.io/global: "true"`).

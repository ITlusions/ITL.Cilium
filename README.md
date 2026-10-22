# ITL.Cilium

[Cilium](https://github.com/cilium/cilium) CNI for the ITL Talos cluster ([`ITlusions/ITL.K8s`](https://github.com/ITlusions/ITL.K8s)) - a thin wrapper chart around the upstream chart, same pattern as [`ITL.Longhorn`](https://github.com/ITlusions/ITL.Longhorn), [`ITL.Traefik`](https://github.com/ITlusions/ITL.Traefik), and [`ITL.K8s.MetricsServer`](https://github.com/ITlusions/ITL.K8s.MetricsServer). Replaces kube-proxy and iptables entirely with eBPF for networking, load balancing, and network policy enforcement. Ships [Hubble](https://github.com/cilium/hubble) for eBPF-based network flow observability (enabled by default in `values.yaml`).

## Deploy

```bash
cd itl.cilium
helm dependency update
helm upgrade --install cilium . \
  -n kube-system \
  --wait --timeout 5m
```

Installing cluster-wide on a live cluster restarts every Cilium agent pod (brief network interruption on every node, not scoped to one app) - confirm before applying to a running cluster, not just on first bootstrap.

## Why the Talos-specific overrides

- **`cgroup.autoMount.enabled: false`** - Talos mounts cgroup2 itself; Cilium must not try to mount it again.
- **`securityContext.capabilities`** - Talos blocks `CAP_SYS_MODULE` and `CAP_SYS_BOOT` node-wide (no pod may load kernel modules or reboot the host), but they're in Cilium's default capability set. Without this override, the `cilium-agent` and its `clean-cilium-state` init container crash-loop with `unable to apply caps: can't apply capabilities: operation not permitted`. See [Talos process capabilities docs](https://docs.siderolabs.com/talos/v1.14/learn-more/process-capabilities).
- **`k8sServiceHost`** - the cluster's control-plane VIP (`10.99.100.10`), since `kubeProxyReplacement: false` means Cilium still needs to know where the API server is directly.

## Hubble

Enabled by default: `hubble-relay` (cluster-wide flow aggregation) and `hubble-ui` (service-dependency graph) both deploy alongside the agents.

```bash
kubectl get pods -n kube-system -l 'k8s-app in (hubble-relay,hubble-ui)'
kubectl port-forward -n kube-system svc/hubble-ui 12000:80   # then open http://localhost:12000
```

## Verify

```bash
kubectl get pods -n kube-system -l k8s-app=cilium
cilium status --wait   # requires the cilium CLI: https://github.com/cilium/cilium-cli
```

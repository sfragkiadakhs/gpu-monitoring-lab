# Architecture

Snapshot of 26.10.2026 (IPs from `k9s` and `docker network inspect kind`; Pod IPs change on restart).

```
Laptop: Windows 11 → WSL Ubuntu → Docker network "kind" (172.22.0.0/16)

┌─ kind node: gpu-lab-control-plane  172.22.0.2 ───────────────┐
│                                                              │
│  ┌──────────────────────┐          ┌──────────────────────┐  │
│  │ Pod prometheus       │  scrape  │ Service node-exporter│  │
│  │ 10.244.0.3           ├─────────►│ 10.96.88.156:9100    │  │
│  │ config: ConfigMap    │          └──────────┬───────────┘  │
│  │ → /etc/prometheus    │                     ▼              │
│  └──────────┬───────────┘          ┌──────────────────────┐  │
│             │                      │ Pod node-exporter    │  │
│             │                      │ 10.244.0.6 DaemonSet │  │
│             │                      └──────────────────────┘  │
│             │                                                │
│             │  Service prometheus 10.96.139.120:9090 → UI    │
│             │  Service kube-dns   10.96.0.10:53     → DNS    │
└─────────────┼────────────────────────────────────────────────┘
              │ scrape dcgm-exporter:9400
              │ (no Service: CoreDNS forwards → Docker DNS on "kind")
              ▼
┌─ Compose container: monitoring-dcgm-exporter-1 ─┐
│ 172.22.0.3:9400   has the GPU                   │
└─────────────────────────────────────────────────┘
```

Three networks: **10.244.x.x** = Pods (IP changes on restart) · **10.96.x.x** = Services (stable IP + DNS name) · **172.22.x.x** = Docker's `kind` network.

| Scrape job | Target | Found via |
|---|---|---|
| `node` | `node-exporter:9100` | Kubernetes Service (CoreDNS answers) |
| `gpu` | `dcgm-exporter:9400` | Docker network alias on `kind` (CoreDNS forwards) |

DCGM stays in Compose because kind can't pass the GPU into the node container. In production I'd run it in the cluster as a DaemonSet via the NVIDIA GPU Operator.

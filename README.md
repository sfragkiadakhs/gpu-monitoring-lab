# gpu-monitoring-lab

On GPU servers you need to see utilization, temperature and power to catch idle, overloaded or overheating GPUs. This is a GPU monitoring stack I built and debugged step by step on my own laptop: GPU in Docker, metrics with NVIDIA DCGM and Prometheus, dashboards in Grafana provisioned from git, and a first move to Kubernetes.

**Tech:** Linux · Docker / Docker Compose · NVIDIA Container Toolkit · DCGM · Prometheus · Grafana · Kubernetes (kind) · WSL2

![GPU utilization and temperature in Grafana during a load test](monitoring/screenshots/grafana-gpu-load.png)

## What I built
- **GPU in containers:** Docker + NVIDIA Container Toolkit on WSL2, verified with `nvidia-smi` inside a CUDA container. [→ docs](docs/gpu-in-containers.md)
- **Monitoring stack** (Docker Compose): DCGM exporter (GPU) and node_exporter (CPU, RAM, disk) → Prometheus → Grafana. [→ docs](docs/monitoring.md)
- **Dashboard as code:** the Grafana datasource and dashboard are files in git, loaded at startup. Recreating the container brings everything back with no manual clicks. [→ docs](docs/monitoring.md#dashboard-as-code)
- **Kubernetes (in progress):** a local kind cluster, Prometheus as a Deployment manifest, self-healing tested, a Service as a fixed address in front of the Pod. [→ docs](docs/kubernetes.md)

```
RTX 4060 ──► DCGM exporter :9400 ─┐
                                  ├──► Prometheus :9090 ──► Grafana :3000
CPU/RAM  ──► node_exporter :9100 ─┘    scrapes every 15 s,   dashboards from git
                                       stores time series
```

## Run it
Needs: an NVIDIA GPU + driver, Docker, and the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html). Tested on Ubuntu in WSL2.

```bash
cd monitoring
docker compose up -d
```
Grafana: http://localhost:3000 (default login `admin` / `admin`, change it on first login) → folder **GPU**. Prometheus: http://localhost:9090.

## Troubleshooting highlights
Real problems I hit, written up as incident reports (symptom → diagnosis → root cause → fix → lesson) in [docs/troubleshooting.md](docs/troubleshooting.md):
- **GPU not visible in Docker:** two Docker engines were installed, and the sandboxed snap one couldn't reach the driver.
- **No DNS in WSL:** ping by IP worked, names didn't, so the problem was DNS (a WSL setting). The second time, the cause was an active VPN.
- **Impossible metrics:** 590 W idle on a 45 W GPU. Check physical limits before trusting a dashboard.
- **Provisioned dashboard showed "datasource not found":** an import placeholder that only the UI fills in, fixed with a fixed datasource uid.
- **kind cluster gone:** a Docker daemon restart stopped the node container, and its restart policy doesn't cover that. `sudo kubectl` was a red herring (root has no kubeconfig).

## Repo layout
```
monitoring/   Docker Compose stack, Prometheus config, Grafana provisioning + dashboard
k8s/          Kubernetes manifests
hardware/     raw outputs (lscpu, nvidia-smi, GPU-in-Docker test)
docs/         details per topic + troubleshooting
```

## Roadmap
- [x] GPU in Docker (NVIDIA Container Toolkit)
- [x] Monitoring with Prometheus + node_exporter + DCGM exporter → Grafana
- [x] Grafana dashboard as code (provisioned from git)
- [x] kind cluster, Prometheus Deployment as a manifest, self-healing test
- [x] Pinned image versions (compose + k8s), checked against the running containers
- [x] Service for Prometheus (fixed address, tested by replacing the Pod)
- [ ] Run the full monitoring stack on kind (Prometheus config as ConfigMap, Grafana)

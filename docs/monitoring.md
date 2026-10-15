# Monitoring stack

All files are in [monitoring/](../monitoring/), started with `docker compose up -d`.

| Service | Port | Job |
|---|---|---|
| `dcgm-exporter` | 9400 | GPU metrics via NVIDIA DCGM (`gpus: all` → NVIDIA Container Toolkit) |
| `node-exporter` | 9100 | CPU, RAM, disk, network of the machine |
| `prometheus` | 9090 | Scrapes both exporters every 15 s and stores the time series ([config](../monitoring/prometheus.yml)) |
| `grafana` | 3000 | Dashboards, provisioned from git (see below) |

- Prometheus **pulls** (scrapes) from the exporters. Targets are service names (`dcgm-exporter:9400`), resolved by Docker's internal DNS.
- WSL note: node-exporter's usual `/:/host:ro,rslave` mount fails because `/` is not a shared mount in WSL, so it uses `/:/host:ro`.

## Load test
`nbody` CUDA sample (1M bodies, ~1 min):

![GPU utilization and temperature in Grafana](../monitoring/screenshots/grafana-gpu-load.png)

Utilization 0 → 100 %, temperature 39 → 63 °C, power ~45 W (= the GPU's power limit). Idle readings were not trustworthy: see [troubleshooting #4](troubleshooting.md#4-impossible-gpu-metrics).

## Dashboard as code
**Problem:** a dashboard imported in the UI lives only inside the Grafana container and is lost when the container is recreated.
**Solution:** the dashboard is a JSON file in git that Grafana loads at startup (provisioning). That gives the three R's: **rebuild** it anywhere, every change has a **revision** in git, and changes can be **reviewed** like code.

```
git repo (monitoring/grafana/)        Grafana container
datasource.yml  ───────────────────►  /etc/grafana/provisioning/datasources/   where the data comes from
dashboards.yml  ───────────────────►  /etc/grafana/provisioning/dashboards/    where to find dashboards
dashboards/gpu.json  ──────────────►  /var/lib/grafana/dashboards/             the dashboard itself
```

- [dashboards.yml](../monitoring/grafana/dashboards.yml) is a provider: "load every JSON in `/var/lib/grafana/dashboards` into the folder GPU". New dashboards only need a new JSON file.
- All mounts are read-only (`:ro`), so the container cannot change the files in git.
- Dashboard: NVIDIA DCGM Exporter Dashboard (Grafana ID 12239). Making it work from a file needed one fix: [troubleshooting #5](troubleshooting.md#5-provisioned-dashboard-datasource-not-found).
- Test: `docker compose up -d --force-recreate grafana` → the dashboard comes back with data, no clicks.

# Troubleshooting

Real problems I hit while building this lab, written as short incident reports.

- [1. GPU not visible inside Docker](#1-gpu-not-visible-inside-docker)
- [2. No DNS inside WSL](#2-no-dns-inside-wsl)
- [3. DNS broken again: the VPN](#3-dns-broken-again-the-vpn)
- [4. Impossible GPU metrics](#4-impossible-gpu-metrics)
- [5. Provisioned dashboard: "datasource not found"](#5-provisioned-dashboard-datasource-not-found)

---

## 1. GPU not visible inside Docker

**Symptom**
```
docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi
→ nvidia-container-cli: load library failed: libnvidia-ml.so.1: cannot open shared object file
```

**Diagnosis**
- Checked which Docker installation was actually running: `docker info`, `snap list`, `dpkg -l`.
- Two engines were installed: `docker.io` (apt) and `docker` (snap). The snap engine was serving the Docker socket.

**Root cause:** the snap runs in a sandbox, so its NVIDIA hook could not reach the driver library in `/usr/lib/wsl/lib`.

**Fix**
- Before removing anything, checked what would be lost: all images, containers and volumes lived in the snap's storage (`/var/snap/docker/...`). Images are rebuildable and no volume held data worth keeping.
- Removed the snap engine (`snap remove --purge docker`) and kept apt `docker.io`.
- The removal also deleted `/run/docker.sock`: the daemon was running, but the client could not connect. Fixed with `systemctl restart docker.socket docker`.
- Installed the NVIDIA Container Toolkit and registered it: `nvidia-ctk runtime configure --runtime=docker`.

**Result:** `nvidia-smi` runs inside the CUDA container ([output](../hardware/docker-gpu-test.txt)).

**Lesson:** confirm which installation is really running before debugging deeper.

---

## 2. No DNS inside WSL

**Symptom:** `docker pull` failed because image names did not resolve.

**Diagnosis**
- `ping 8.8.8.8` worked, so the network was fine and only DNS was broken.
- `/etc/resolv.conf` pointed to a DNS proxy on the Windows host (`172.17.176.1`), which did not answer.

**Root cause:** `dnsTunneling=false` in the Windows file `%USERPROFILE%\.wslconfig`.

**Fix:** `dnsTunneling=true` + `wsl --shutdown`. DNS queries now go through Windows' own network stack.

**Lesson:** test by IP first, then by name. That separates "network down" from "DNS down".

---

## 3. DNS broken again: the VPN

**Symptom:** same as #2: ping by IP works, `getent hosts` fails.

**Diagnosis**
- `.wslconfig` was still correct.
- Windows itself could resolve names, so the problem was between WSL and Windows.

**Root cause:** an active VPN. WSL's DNS tunnel (`nameserver 10.255.255.254`) could not get through it.

**Fix:** VPN off → names resolve again.

**Lesson:** when Windows resolves but WSL doesn't, check for a VPN first.

---

## 4. Impossible GPU metrics

**Symptom:** the Grafana dashboard showed **590 W** and **1 W** idle power on a GPU with a 45 W limit.

**Diagnosis**
- Values under load were realistic (~45 W = the power limit), so only the idle readings were wrong.
- After the load test, temperature stayed flat at 63 °C for 7 minutes and utilization had gaps.

**Root cause (likely):** the GPU's deep power-saving state in WSL. While it sleeps, the exporter reports invalid or repeated values.

**Fix:** none needed for the lab. I treat idle readings as unreliable.

**Lesson:** sanity-check metrics against physical limits. On a real server, compare with `nvidia-smi` and the BMC's power reading before trusting a dashboard.

---

## 5. Provisioned dashboard: "datasource not found"

**Symptom:** after loading the dashboard from a file (provisioning), the panels could not find their datasource.

**Diagnosis:** dashboards exported from grafana.com contain the placeholder `${DS_PROMETHEUS}` (10× in this one). The UI import asks which datasource to use and fills it in. Provisioning loads the file as it is.

**Fix**
- Gave the datasource a fixed `uid: prometheus` in [datasource.yml](../monitoring/grafana/datasource.yml).
- Replaced the placeholder: `sed -i 's/\${DS_PROMETHEUS}/prometheus/g' gpu.json`.

**Test:** `docker compose up -d --force-recreate grafana` → logs show `finished to provision dashboards`, and the dashboard is back with data.

**Lesson:** a file that works through the UI is not automatically ready for automation. Check for import-time placeholders.

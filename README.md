# gpu-monitoring-lab

Hands-on lab to learn how GPU systems are built and run: describing hardware architectures and how the components connect, GPU drivers, monitoring (Prometheus, Grafana, DCGM) and cloud-native deployment.

Setup: my laptop (i7-13620H, 16 GB RAM, RTX 4060 Laptop GPU), Ubuntu on WSL2, Docker.

## Session 1: hardware inventory

Raw outputs are in [hardware/](hardware/).

### CPU (`lscpu`, `lstopo`)
- 1 socket, 1 NUMA node. Each core has its own L1 (instructions + data) and L2; all cores share the 24 MB L3.
- Speed order: L1 (fastest, smallest) → L2 → L3 (largest, shared) → RAM.
- The real chip is hybrid: 6 P-cores (2 threads each) + 4 E-cores (1 thread each) = 10 cores / 16 threads.
- **WSL difference:** the hypervisor shows 8 identical cores × 2 threads, and WSL only gets ~8 GB of the 16 GB RAM (default WSL config). Inside a VM, tools show what the hypervisor allows.

### GPU (`lspci`, `nvidia-smi`)
- NVIDIA GeForce RTX 4060 Laptop GPU (Ada Lovelace), 8 GB VRAM, 45 W power limit, PCIe Gen4 x8.
- Driver 610.78, supports CUDA up to 13.3.
- No ECC, no MIG, no NVLink: these are datacenter GPU features.
- **WSL difference:** `lspci` shows only "Microsoft Basic Render Driver" and Virtio devices, with PCI addresses that change on every reboot. WSL does not see the GPU as a PCIe device; it uses it through the Windows driver (GPU paravirtualization, libraries in `/usr/lib/wsl/lib`).

### Topology (`nvidia-smi topo -m`)
- Shows how GPUs connect to each other (NVLink, PCIe switch, host bridge, across sockets) and which CPU cores and NUMA node each GPU belongs to.
- On a multi-GPU server it is used to place jobs on close GPUs and on the CPU cores of the same NUMA node, and to spot broken links (e.g. `SYS` where `NV18` is expected).
- On my laptop it shows only `GPU0 X`: one GPU, and WSL hides the CPU/NUMA affinity.

### Finding: GPU in Docker failed
```
docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi
→ nvidia-container-cli: load library failed: libnvidia-ml.so.1: cannot open shared object file
```
- Two Docker engines were installed: `docker.io` (apt) and `docker` (snap). The snap one was serving the Docker socket.
- The snap runs in a sandbox, so its NVIDIA hook could not reach the driver library in `/usr/lib/wsl/lib`.
- Lesson: check which installation is actually running (`docker info`, `snap list`, `dpkg -l`) before debugging further.
- Fix: next session (remove the snap engine, install the NVIDIA Container Toolkit).

## Roadmap
- [x] Session 1: hardware inventory
- [ ] Drivers: NVIDIA Container Toolkit, GPU in Docker, how drivers work on Linux servers
- [ ] Monitoring: Prometheus + node_exporter + GPU exporter → Grafana
- [ ] Cloud-native: run the monitoring stack on kind/k3s

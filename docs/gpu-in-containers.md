# GPU in containers

## Hardware and the WSL view
Laptop: i7-13620H, 16 GB RAM, NVIDIA RTX 4060 Laptop GPU, Ubuntu on WSL2. Raw outputs are in [hardware/](../hardware/).

- **CPU:** the real chip is hybrid (6 P-cores with 2 threads each + 4 E-cores) = 10 cores / 16 threads. Inside WSL the hypervisor shows 8 identical cores × 2 threads, and WSL gets ~8 GB of the 16 GB RAM by default. Inside a VM, tools show what the hypervisor allows.
- **GPU:** RTX 4060 Laptop (Ada Lovelace), 8 GB VRAM, 45 W power limit. Driver 610.78, supports CUDA up to 13.3. No ECC, MIG or NVLink (datacenter features).
- **WSL does not see the GPU as a PCIe device.** `lspci` shows only virtual devices; the GPU is used through the Windows driver (GPU paravirtualization, libraries in `/usr/lib/wsl/lib`). So no Linux NVIDIA driver is installed inside WSL.

## How a container gets the GPU
```
Windows NVIDIA driver ──► WSL (/usr/lib/wsl/lib) ──► NVIDIA Container Toolkit ──► container (own CUDA libraries)
```
- The **NVIDIA Container Toolkit** is not a driver. When a container starts with `--gpus all`, it mounts the host's GPU devices and driver libraries into the container.
- Registered with Docker via `nvidia-ctk runtime configure --runtime=docker` (writes `/etc/docker/daemon.json`).
- Each container brings its own CUDA version; the host only needs the driver.

## The CUDA rule
The driver's CUDA version must be **≥** the container's CUDA version. Here: driver supports 13.3, container uses 12.6 → works.
If a container needs a newer CUDA than the driver supports: use an older image or update the driver.

Test output: [docker-gpu-test.txt](../hardware/docker-gpu-test.txt). The setup problems on the way are in [troubleshooting #1](troubleshooting.md#1-gpu-not-visible-inside-docker).

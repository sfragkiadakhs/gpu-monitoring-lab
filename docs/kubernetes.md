# Kubernetes (in progress)

**Goal:** move the monitoring stack from Docker Compose to Kubernetes. Done so far: a local cluster and Prometheus as a Deployment.

## Cluster: kind
- `kind create cluster --name gpu-lab` → one node, `gpu-lab-control-plane`.
- kind = **K**ubernetes **in** **D**ocker: each node is a container (visible in `docker ps`). kubectl talks to the API server on the node's port 6443.
- Why kind: built for creating and deleting test clusters on a laptop. k3s would be the choice for real small servers; kubectl and the manifests are the same.

```
Laptop → WSL → Docker → [kind node container] → Kubernetes → Pod (Prometheus)
```

## Prometheus as a manifest
- Generated with `kubectl create deployment … --dry-run=client -o yaml`, then cleaned up: [k8s/prometheus-deployment.yaml](../k8s/prometheus-deployment.yaml).
- The Deployment holds the desired state (`replicas: 1`). Its Pod template puts the label `app: prometheus` on every Pod; the selector counts Pods by that label.
- `kubectl apply -f` makes the cluster match the file and is **idempotent**: same file → `unchanged`; `replicas: 1 → 2` → `configured` and a second Pod. Same idea as dashboard as code: the desired state lives in git.

## Self-healing test
Deleted the running Pod → a new Pod with a new name was running 5 s later. The controller saw "1 wanted, 0 running" and created a replacement.

## Limit: GPU in kind
The DCGM exporter needs the GPU, but a kind node is a container inside Docker, so passing the GPU through is extra work. Plan: Prometheus and Grafana move to Kubernetes, the DCGM exporter stays in Docker for now.

# Kubernetes (in progress)

**Goal:** move the monitoring stack from Docker Compose to Kubernetes. Done so far: a local cluster, Prometheus as a Deployment, and a Service in front of it.

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

## Service: a fixed address
**Problem:** a Pod's IP changes whenever the Pod is replaced, and even when it restarts (after a Docker restart: same Pod, `10.244.0.6 → 10.244.0.4`). Anything that talks to Prometheus by Pod IP would break.

- Generated with `kubectl expose deployment prometheus --port=9090 --dry-run=client -o yaml`, then cleaned up: [k8s/prometheus-service.yaml](../k8s/prometheus-service.yaml).
- The Service gets a fixed cluster IP and the DNS name `prometheus`. Its selector `app: prometheus` finds the Pods by label, the same way the Deployment does, and its endpoint list follows them.
- `port` = where the Service listens, `targetPort` = the port inside the Pod (both 9090).

**Test:** deleted the Pod and compared before/after.

| | Before | After |
|---|---|---|
| Pod IP | 10.244.0.4 | 10.244.0.6 |
| Service cluster IP | 10.96.139.120 | 10.96.139.120 |
| Endpoints (`kubectl get endpointslices`) | 10.244.0.4:9090 | 10.244.0.6:9090 |

Two controllers, two jobs: the Deployment replaces the Pod, the Service updates where traffic goes. Grafana will connect to `prometheus:9090` and never see the change.

## Limit: GPU in kind
The DCGM exporter needs the GPU, but a kind node is a container inside Docker, so passing the GPU through is extra work. Plan: Prometheus and Grafana move to Kubernetes, the DCGM exporter stays in Docker for now.

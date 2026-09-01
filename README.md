# Kubernetes Learning Repository

A hands-on collection of Kubernetes manifests and Helm charts covering core concepts — from basic workloads to autoscaling, storage, networking, and security. The cluster is spun up locally using [kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker).

---

## Cluster Setup

The cluster is defined in `cluster-config.yml` using kind with one control-plane and three worker nodes (Kubernetes v1.31.2). One worker node exposes host ports for NodePort and HTTPS access.

```bash
kind create cluster --config cluster-config.yml
```

---

## Repository Structure

```
k8s/
├── cluster-config.yml          # kind cluster definition
├── namespace.yml               # nginx-ns namespace
├── pod.yml                     # standalone nginx pod
├── deployment.yml              # nginx Deployment (3 replicas)
├── replicaset.yml              # nginx ReplicaSet (5 replicas)
├── service.yml                 # ClusterIP service for nginx
├── daemonset.yml               # nginx DaemonSet
├── job.yml                     # batch job with busybox
├── cronjob.yml                 # scheduled backup CronJob
├── persistentVolume.yml        # hostPath PersistentVolume (1.5Gi)
├── persistentVolumeClaim.yml   # PVC requesting 1Gi
├── persistentVolumeDeployment.yml  # nginx Deployment with PVC mounted
│
├── HPA/                        # Horizontal Pod Autoscaler
├── VPA/                        # Vertical Pod Autoscaler
├── RBAC/                       # Role-Based Access Control
├── Dashboard/                  # Kubernetes Dashboard admin setup
├── ingress/                    # Ingress routing (nginx + NoteApp)
├── probe/                      # Liveness & readiness probes
├── init-container/             # Init container example
├── sidecar-container/          # Sidecar container pattern
├── postgreSQL/                 # StatefulSet with ConfigMap & Secrets
├── Helm/                       # Helm charts (apache, notespace)
└── monitoring/                 # (placeholder for monitoring stack)
```

---

## Topics Covered

### Core Workloads

| File | Description |
|---|---|
| `pod.yml` | Single nginx pod in `nginx-ns` |
| `deployment.yml` | nginx Deployment with 3 replicas |
| `replicaset.yml` | nginx ReplicaSet with 5 replicas |
| `daemonset.yml` | nginx DaemonSet — runs one pod per node |
| `job.yml` | One-off batch job (2 completions, 3 parallel) |
| `cronjob.yml` | CronJob that runs a backup script every minute |

### Networking & Services

| Directory / File | Description |
|---|---|
| `service.yml` | ClusterIP service exposing nginx on port 82 |
| `ingress/` | Ingress routing `/nginx` to nginx and `/` to NoteApp |

The ingress uses the NGINX Ingress Controller with `rewrite-target: /`. Two backends are configured in `nginx-ns`:
- `nginx-service` → nginx Deployment
- `note-app-service` → NoteApp Deployment (`jayvaghela0304/notespace:v1`)

### Storage

| File | Description |
|---|---|
| `persistentVolume.yml` | `hostPath` PV with 1.5 Gi, `Retain` reclaim policy |
| `persistentVolumeClaim.yml` | PVC requesting 1 Gi from `local-storage` class |
| `persistentVolumeDeployment.yml` | nginx Deployment mounting the PVC at `/var/www/html` |

### Autoscaling

#### HPA (`HPA/`)
Horizontal Pod Autoscaler for an Apache (`httpd:2.4`) Deployment in `apache-ns`. Scales between 3 and 10 replicas based on CPU utilization (target: 5%).

```bash
kubectl apply -f HPA/namespace.yml
kubectl apply -f HPA/deployment.yml
kubectl apply -f HPA/service.yml
kubectl apply -f HPA/hpa.yml
```

#### VPA (`VPA/`)
Vertical Pod Autoscaler targeting the same Apache Deployment with `updateMode: Auto`.

```bash
kubectl apply -f VPA/vpa.yml
```

### RBAC (`RBAC/`)

Scoped to `apache-ns`:
- `service-account.yml` — ServiceAccount `apache-user`
- `role.yml` — Role `apache-manager` with full access to Deployments, Pods, and Services
- `role-binding.yml` — Binds the role to `apache-user`

```bash
kubectl apply -f RBAC/service-account.yml
kubectl apply -f RBAC/role.yml
kubectl apply -f RBAC/role-binding.yml
```

### Kubernetes Dashboard (`Dashboard/`)

Creates an `admin-user` ServiceAccount in `kubernetes-dashboard` and binds it to the `cluster-admin` ClusterRole for full dashboard access.

```bash
kubectl apply -f Dashboard/dashboard-admin-user.yml
kubectl -n kubernetes-dashboard create token admin-user
```

### Probes (`probe/`)

nginx Deployment with both `livenessProbe` and `readinessProbe` configured as HTTP GET checks against `/` on port 80.

### Init Container (`init-container/`)

A Pod that runs a busybox init container (simulates a 20-second initialization delay) before the main container starts.

### Sidecar Container (`sidecar-container/`)

A Pod with two containers sharing an `emptyDir` volume:
- **main-con** — writes log lines to `/var/log/app.log` every 5 seconds
- **sidecar-con** — tails the same log file in real time

### PostgreSQL StatefulSet (`postgreSQL/`)

A 4-replica PostgreSQL StatefulSet in `postgres-ns` with per-pod persistent storage (1 Gi each via `volumeClaimTemplates`).

Configuration is split across three patterns for comparison:

| Sub-folder | Approach |
|---|---|
| `statefulset.yml` | Env vars inline |
| `ConfigMap/` | DB name via ConfigMap |
| `secret/` | Password via base64-encoded Secret |

```bash
kubectl apply -f postgreSQL/namespace.yml
kubectl apply -f postgreSQL/ConfigMap/ConfigMap.yml
kubectl apply -f postgreSQL/secret/secret.yml
kubectl apply -f postgreSQL/service.yml
kubectl apply -f postgreSQL/ConfigMap/StatefulSetUsingConfigMap.yml
```

### Helm Charts (`Helm/`)

Two Helm charts are included:

#### `apache-helm`
Deploys the Apache (`httpd:2.4`) web server. Key defaults:
- 3 replicas
- ClusterIP service on port 83
- CPU/memory limits set (100m / 128Mi)
- Autoscaling disabled by default

```bash
helm install apache ./Helm/apache-helm
```

A packaged release is also available: `Helm/apache-helm-0.1.0.tgz`

#### `notespace`
Deploys the custom NoteApp (`jayvaghela0304/notespace:v1`). Key defaults:
- 4 replicas
- ClusterIP service on port 5000
- Autoscaling disabled by default

```bash
helm install notespace ./Helm/notespace
```

---

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/) (for Helm charts)
- [metrics-server](https://github.com/kubernetes-sigs/metrics-server) (required for HPA)

---

## Quick Start

```bash
# 1. Create the cluster
kind create cluster --config cluster-config.yml

# 2. Apply a namespace and basic workloads
kubectl apply -f namespace.yml
kubectl apply -f deployment.yml
kubectl apply -f service.yml

# 3. Verify
kubectl get all -n nginx-ns
```

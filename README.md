# Project 3: Cloud Mesh

Containerize and deploy Project 2 (AI Procurement Agent) with Docker and Kubernetes manifests.

This project focuses on packaging a Python CLI into a container image and running it in a local Kubernetes cluster.

## What this demonstrates
- Docker image build and run.
- Kubernetes manifests as code (YAML).
- Secret injection for API keys (no secrets committed).
- Running a run-to-completion pipeline as a Kubernetes `Job`.

## Quick glossary (minimal)
- **Container image**: an immutable bundle of your app + dependencies.
- **Container**: a running instance of an image.
- **Kubernetes cluster**: a system that schedules containers across nodes.
- **Node**: a machine (or VM/container) that runs workloads.
- **Pod**: the smallest deployable unit; one or more containers that share networking/storage.
- **Job**: runs a Pod until it finishes successfully (great for CLIs/pipelines).
- **Manifest**: YAML that declares the desired state (Job/Secret/etc).
- **Namespace**: logical isolation within a cluster.
- **Secret**: stores sensitive config (API keys); referenced by Pods as env vars or files.

## Prerequisites (local)
- Docker installed and working.
- `kubectl` installed (Kubernetes CLI).
- `kind` installed (runs Kubernetes in Docker).

## Step-by-step: run the pipeline on Kubernetes (local)

### 1) Build the container image (Project 2)
From the Project 2 repo:
```bash
cd /path/to/project-2-ai-procurement-agent
docker build -t ai-procurement-agent:local .
```

### 2) Create a local Kubernetes cluster (kind)
```bash
kind create cluster --name cloud-mesh
kubectl cluster-info
```

### 3) Load the local image into the cluster
`kind` clusters can’t pull your local Docker images unless you load them:
```bash
kind load docker-image ai-procurement-agent:local --name cloud-mesh
```

### 4) Create the namespace
```bash
kubectl apply -f k8s/namespace.yaml
```

### 5) Create the Secret (API key)
Do **not** commit real keys. Create the secret locally:
```bash
kubectl -n cloud-mesh create secret generic google-api-key \
  --from-literal=GOOGLE_API_KEY='REPLACE_ME'
```

### 6) Create the persistent volume claim (PVC)
By default, Pods and `emptyDir` volumes are ephemeral. A `PersistentVolumeClaim` gives you storage that survives across Jobs.

```bash
kubectl apply -f k8s/pvc-chroma.yaml
kubectl -n cloud-mesh get pvc
```

### 7) Run the index Job (writes to the PVC)
```bash
kubectl apply -f k8s/job-procurement-index.yaml
kubectl -n cloud-mesh wait --for=condition=complete job/procurement-index --timeout=300s
kubectl -n cloud-mesh logs job/procurement-index
```

### 8) Run the query Job (reads from the same PVC)
```bash
kubectl apply -f k8s/job-procurement-query.yaml
kubectl -n cloud-mesh wait --for=condition=complete job/procurement-query --timeout=300s
kubectl -n cloud-mesh logs job/procurement-query
```

### 9) Cleanup
```bash
kubectl -n cloud-mesh delete job procurement-index procurement-query
kubectl -n cloud-mesh delete pvc procurement-chroma
kind delete cluster --name cloud-mesh
```

## Files
- `k8s/namespace.yaml`: namespace used for the demo.
- `k8s/secret.example.yaml`: safe example secret manifest (placeholder value).
- `k8s/pvc-chroma.yaml`: persistent storage for the Chroma index.
- `k8s/job-procurement-index.yaml`: generates data + indexes documents into Chroma (PVC-backed).
- `k8s/job-procurement-query.yaml`: queries the existing Chroma index (PVC-backed).

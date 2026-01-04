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

### 6) Run the pipeline as a Job
```bash
kubectl apply -f k8s/job-procurement-pipeline.yaml
kubectl -n cloud-mesh get jobs,pods -w
```

### 7) Read logs (the “output” of the Job)
```bash
kubectl -n cloud-mesh logs job/procurement-pipeline
```

### 8) Cleanup
```bash
kubectl -n cloud-mesh delete job procurement-pipeline
kind delete cluster --name cloud-mesh
```

## Files
- `k8s/namespace.yaml`: namespace used for the demo.
- `k8s/secret.example.yaml`: safe example secret manifest (placeholder value).
- `k8s/job-procurement-pipeline.yaml`: runs Project 2 end-to-end in a Kubernetes Job.

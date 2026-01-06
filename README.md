# K8s Procurement Pipeline Lab

Containerize and run the AI Procurement RAG pipeline on Kubernetes using Jobs, a PVC-backed index, and a scheduled CronJob. This lab is focused on operational readiness and reproducible batch execution.

## Contents
- [What I Built & Why](#what-i-built--why)
- [Decisions & Tradeoffs](#decisions--tradeoffs)
- [Architecture & Flow](#architecture--flow)
- [Components & Versions](#components--versions)
- [Runbook (Build -> Deploy -> Run)](#runbook-build---deploy---run)
- [Optional Scheduling (CronJob)](#optional-scheduling-cronjob)
- [Results & Evidence](#results--evidence)
- [Kubernetes Manifests](#kubernetes-manifests)

## What I Built & Why
- **Docker image**: containerized the procurement RAG CLI so it can run in a cluster.
- **Kubernetes Jobs**: run-to-completion batch pipeline for indexing and querying.
- **PVC-backed persistence**: Chroma index survives across Jobs to avoid re-indexing.
- **Secret injection**: API keys stay out of git and are injected at runtime.

## Decisions & Tradeoffs
- **Jobs instead of Deployments** because the pipeline is batch; tradeoff is no always-on API.
- **kind** for local Kubernetes; tradeoff is single-node, local-only testing.
- **PVC for Chroma** to persist embeddings; tradeoff is extra storage management.

## Architecture & Flow
```mermaid
graph TD
  A[CronJob schedule (optional)] --> B[Job: procurement-index]
  B --> C[Pod: ai-procurement-agent:local]
  C --> D[PVC: procurement-chroma]
  C --> E[Secret: google-api-key]
  F[Job: procurement-query] --> C
```

## Components & Versions
- Docker + local image `ai-procurement-agent:local`
- Kubernetes (kind) + kubectl
- PVC + Secret + Job primitives; CronJob optional

## Runbook (Build -> Deploy -> Run)
1) Build the container image (from the RAG lab):
   ```bash
   cd /path/to/ai-procurement-rag-lab
   docker build -t ai-procurement-agent:local .
   ```
2) Create a local Kubernetes cluster:
   ```bash
   kind create cluster --name cloud-mesh
   kubectl cluster-info
   ```
3) Load the local image into the cluster:
   ```bash
   kind load docker-image ai-procurement-agent:local --name cloud-mesh
   ```
4) Create namespace, Secret, and PVC:
   ```bash
   kubectl apply -f k8s/namespace.yaml
   kubectl -n cloud-mesh create secret generic google-api-key \
     --from-literal=GOOGLE_API_KEY='REPLACE_ME'
   kubectl apply -f k8s/pvc-chroma.yaml
   ```
5) Run the index Job:
   ```bash
   kubectl apply -f k8s/job-procurement-index.yaml
   kubectl -n cloud-mesh wait --for=condition=complete job/procurement-index --timeout=300s
   kubectl -n cloud-mesh logs job/procurement-index
   ```
6) Run the query Job:
   ```bash
   kubectl apply -f k8s/job-procurement-query.yaml
   kubectl -n cloud-mesh wait --for=condition=complete job/procurement-query --timeout=300s
   kubectl -n cloud-mesh logs job/procurement-query
   ```
7) Cleanup:
   ```bash
   kubectl -n cloud-mesh delete job procurement-index procurement-query
   kubectl -n cloud-mesh delete pvc procurement-chroma
   kind delete cluster --name cloud-mesh
   ```

## Optional Scheduling (CronJob)
If you want to show scheduling, use the CronJob manifest to re-index on a schedule.

```bash
kubectl apply -f k8s/cronjob-procurement-index.yaml
kubectl -n cloud-mesh create job --from=cronjob/procurement-index-cron procurement-index-manual-001
kubectl -n cloud-mesh wait --for=condition=complete job/procurement-index-manual-001 --timeout=300s
kubectl -n cloud-mesh logs job/procurement-index-manual-001
```

Cleanup if you created the CronJob:
```bash
kubectl -n cloud-mesh delete job procurement-index-manual-001
kubectl -n cloud-mesh delete cronjob procurement-index-cron
```

## Results & Evidence
- Index Job completes end-to-end (generate -> detect -> index) and persists the Chroma index.
- Query Job reads from the PVC without re-indexing.
- Optional CronJob manifest is included for scheduled re-indexing.
- The screenshot below shows both Jobs completing successfully in the `cloud-mesh` namespace.

![Kubernetes Jobs](docs/screenshots/k8s_jobs.svg)

## Kubernetes Manifests
- `k8s/namespace.yaml`: namespace for the demo.
- `k8s/secret.example.yaml`: safe example secret manifest.
- `k8s/pvc-chroma.yaml`: persistent storage for the Chroma index.
- `k8s/job-procurement-index.yaml`: generates data + indexes documents (PVC-backed).
- `k8s/job-procurement-query.yaml`: queries the existing index (PVC-backed).
- `k8s/cronjob-procurement-index.yaml`: optional scheduled re-indexing.

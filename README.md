# Open WebUI Deployment

Production-grade deployment configuration for [Open WebUI](https://github.com/open-webui/open-webui)
as the user-facing chat and agent interface for the **furseal** AI workspace on Kind Kubernetes (Mac Studio).

- Exposes `http://open-webui.localhost` via ingress.
- Uses upstream image `ghcr.io/open-webui/open-webui:main`.
- Persistent state is mounted from a local hostPath volume (Kind node) at `/app/backend/data`, backed by `./data/` on the host.

> This repository contains **deployment manifests and runtime data only**. It is *not* the Open WebUI source code; build/test/lint of application code does not apply here.

## Architecture & Data Flow

```
User Browser ──http──► Ingress (open-webui.localhost)
                          │
                          ▼
 Open Web UI Pod ◄────────┤   Service :8080
  container port=8080     │
  image: ghcr.io/open-webui/open-webui:main
                          │
                          ├─ envFrom secretRef(open-webui-secret)
                          │     WEBUI_SECRET_KEY=...e4756d...
                          └─ volumeMount /app/backend/data  (hostPath)
                          │
                          ▼
Bifrost Gateway http://bifrost:8080/v1 -─────────────────┐
      OpenAI-compatible → Ollama (local models)          │
                                  + Cloud LLMs           │
                                          │              │
                  Tracing / Langfuse  (http://langfuse:3000) ◄── prompts, traces, evals
                                          │
                 Observability Stack      ▼
                                      active Kubernetes context
```

Open Web UI issues streaming chat/completion requests to the Bifrost gateway at `http://bifrost:8080/v1` and forwards LLM traces to Langfuse for async prompt logging, call tracing, and evaluation.

## Repository Structure

```
open-webui/               # deploy config + runtime data (NOT upstream source)
├── .gitignore            # excludes data/, *.log, *-secret.yaml, .DS_Store
├── AGENTS.md             # maintainer notes for OpenCode sessions
├── README.md             # this file
├── k8s/
│   ├── open-webui-deployment.yaml   # Deployment + Service (port 8080) + Ingress (nginx)
│   └── open-webui-secret.yaml       # Secret: WEBUI_SECRET_KEY (git-ignored pattern)
└── data/                 # git-ignored hostPath runtime volume → /app/backend/data
    ├── webui.db*         # SQLite WAL DB (profiles, chat history)
    ├── vector_db/        # embedding index files
    ├── cache/            # HuggingFace model caches
    ├── uploads/          # user-uploaded documents
    └── audit.log         # request log
```

## Prerequisites

- Kind Kubernetes cluster running on Mac Studio with nginx ingress controller installed.
- Manifests are namespace-less; apply to your active context (`kubectl config view`). A hostPath `DirectoryOrCreate` auto-provisions `/mnt/workspaces/open-webui/data` on first start.
- Bifrost gateway reachable at `http://bifrost:8080/v1` (OpenAI-compatible) within the same namespace/network exposing local Ollama and cloud LLMs.
- Postgres available for Open Web UI data persistence.
- Langfuse reachable at `http://langfuse:3000` with API credentials prepared for admin configuration.

## Deployment Guide

Apply manifests in order using prefix-sorted filenames so dependencies resolve correctly (Secrets → Service/Deployment → Ingress):

```sh
# 1. Secret first (envFrom source)
kubectl apply -f k8s/open-webui-secret.yaml          # WEBUI_SECRET_KEY

# 2. Deployment + Service
kubectl apply -f k8s/open-webui-deployment.yaml      # port=8080 → ClusterIP svc:open-webui:8080

# 3. Ingress routes traffic to the service (nginx)
```

The deployment mounts hostPath `/mnt/workspaces/open-webui/data` into the container at `/app/backend/data`; `type: DirectoryOrCreate` ensures it is auto-created on first start. No local build needed — it runs the published upstream image directly.

## Initial Setup & Admin Configuration

1. On first launch, open `http://open-webui.localhost`. When no admin user exists yet, visit `/auth/admin/setup` to create the initial Admin account and password.
2. In **Admin Settings → Configure → Network Connection**, set the API endpoint/endpoint configuration to point at your OpenAI-compatible gateway:
   - Endpoint: `http://bifrost:8080/v1`
   - Confirm `/v1/models`, `/v1/chat/completions`, and `/v1/embeddings` resolve through Bifrost.
3. In **Admin Settings → Configure → Observability / Langfuse**, enable tracing with:
   ```env
   # paste keys (names per the Secret manifest)
   LANGFUSE_PUBLIC_KEY=…
   LANGFUSE_SECRET_KEY=…
   LANGFUSE_HOST=http://langfuse:3000
   ```

## Maintenance & Logs

```sh
# live pod logs
kubectl logs -f deploy/open-webui

# rolling restart after config changes
kubectl rollout restart deployment open-webui

# confirm healthy status (ClusterIP svc + ingress expected)
kubectl get pods,svc,ingress,pvc

# backup the runtime state directory on the host node:
rsync -av data/ /mnt/backups/open-webui-data-$(date +%F)/
```

> `audit.log`, `*.db*`, and `data/` are git-ignored by design; do not re-commit regenerated state. Rotate `WEBUI_SECRET_KEY` via edit of `k8s/open-webui-secret.yaml` (not in plaintext history).
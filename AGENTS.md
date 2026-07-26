# AGENTS.md

This is NOT the open-webui source repo. It contains only Kubernetes
deployment config and local runtime data for a containerized instance.

## What this directory holds

- `k8s/` — Helm-style manifests deploying `ghcr.io/open-webui/open-webui:main`
  into namespace `furseal`. Image is published upstream; no local build needed.
- `data/` (gitignored) — runtime volume mounted at `/app/backend/data`:
  `webui.db*` SQLite files, `vector_db/`, `cache/` (HF models),
  `uploads/`, and `audit.log`.
- `.gitignore` excludes `data/`, `*.log`, k8s secrets, macOS metadata.

## Editing these manifests

The ingress hosts are `open-webui.localhost` (local) and `ai.furseal.net`
(public). Secrets live in `k8s/open-webui-secret.yaml` — keep values out of
git and rotate rather than edit plaintext.

No local build/test/lint commands apply here: this is declarative k8s config.
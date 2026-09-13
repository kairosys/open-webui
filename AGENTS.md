# AGENTS.md — Open WebUI deployment config

This repo is **declarative Kubernetes manifests + local runtime data** for a
containerized `open-webui` (the upstream project's app code, NOT here).
No `package.json`, `pyproject.toml`, or other source/build tooling exists; do
not run application-level build/test/lint/fix commands. Treat these as k8s manifests only.

## Repo contents (what an agent may edit)
- `k8s/open-webui-deployment.yaml` — Ingress (`open-webui.localhost`) + Service (`ClusterIP:8080`) + Deployment, all namespace-less. **Authoritative.**
- `k8s/open-webui-external-ingress.yaml` — gitignored; the `ai.furseal.net` Ingress, split out of the main manifest.
- `k8s/open-webui-secret.yaml` — gitignored; the local copy holds the live `WEBUI_SECRET_KEY` (the only key — verified in-cluster). Never commit it.
- `data/` — gitignored; **the same directory as the live hostPath volume** (same inode on the node): running ~500MB SQLite, `vector_db/`, `cache/`, `uploads/`. Not a source-tree copy.
- `.gitignore` excludes `data/`, `*.log`, `k8s/*-secret.yaml`, `k8s/*-external-ingress.yaml`.

## What an agent should NOT do here
- No build/test/lint targets for Open Web UI's Python/Go code (not present); don't try to make the upstream toolchain work against this repo.
- Don't hand-edit `data/` like source, and don't modify `webui.db` while the pod runs — back it up (rsync below) and `kubectl scale deploy open-webui --replicas=0` first.

## Deployment facts (verified against manifest + live cluster)
- Image `ghcr.io/open-webui/open-webui:v0.11.0`, `imagePullPolicy: Always`. Don't re-tag to `:main` without intent.
- Kind cluster `mac-studio` (kubectl context `kind-mac-studio`), kind running on **podman** — the node container is `mac-studio-control-plane`; use `podman exec`, `docker` is not on PATH.
- hostPath: node `/mnt/workspaces/open-webui/data` → container `/app/backend/data` (`DirectoryOrCreate`); on the Mac `~/Workspaces` is bind-mounted, so in-repo `data/` == the live volume.
- Manifests are namespace-less; they deploy into the current kubectl context (default: `default`).
- The secret carries only `WEBUI_SECRET_KEY`; pod env = `TZ` + that. **Langfuse keys are NOT injected via env** — the README's Configuration table claiming otherwise is stale; manifest/cluster is canonical.
- The cluster also hosts the rest of the furseal stack (bifrost, langfuse, pgadmin, redis, rustfs, scriberr, searxng); only `open-webui*` resources belong to this repo — don't touch the others.

## Apply order (matters, or pods start without env vars / fail on the Secret ref)
```sh
kubectl apply -f k8s/open-webui-secret.yaml              # 1. Deployment's envFrom refs it
kubectl apply -f k8s/open-webui-deployment.yaml         # 2. Ingress + Service + Deployment
kubectl apply -f k8s/open-webui-external-ingress.yaml   # 3. optional: ai.furseal.net (gitignored; absent in fresh clones)
```

## Initial admin bootstrap (one-time per fresh empty DB only)
1. Visit `/auth/admin/setup` on first boot — no users exist yet.
2. Admin → Configure Networking: API endpoint = `http://bifrost:8080/v1`; verify `/v1/models`, `/v1/chat/completions`, `/v1/embeddings`.
3. Optional Langfuse: configure in the Admin UI (Obs/observability; host `http://langfuse:3000`) — keys are not supplied via the secret/env.

## Runtime ops cheat sheet
```sh
kubectl logs -f deploy/open-webui           # live pod logs
kubectl rollout restart deployment open-webui  # after config/secret changes
kubectl get pods,svc,ingress                # current context, namespace-less
# stop writers, then back up / restore the runtime volume:
kubectl scale deploy open-webui --replicas=0
rsync -av /mnt/workspaces/open-webui/data/ "/mnt/backups/open-webui-data-$(date +%F)/"
```

## Verification (after any manifest edit)
`kubectl apply --dry-run=client -o yaml -f k8s/` then
`kubectl diff -f k8s/open-webui-deployment.yaml`; ensure `envFrom.secretRef.name: open-webui-secret` resolves before rollout.

## Gotchas
- Rotating `WEBUI_SECRET_KEY` invalidates all sessions; re-apply secret + `rollout restart`.
- Keep `ai.furseal.net` out of `open-webui-deployment.yaml` — it belongs in the gitignored external ingress, or it leaks into commits (the repo can't ship without a real cert).
- `open-webui.localhost` resolution is host-side DNS, not k8s'.
- README drifts from the manifests (Configuration table, Ingress steps); when they disagree, trust `k8s/*.yaml` and the live cluster.

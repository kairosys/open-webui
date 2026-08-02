# AGENTS.md — Open WebUI deployment config

This repo is **declarative Kubernetes manifests + local runtime data** for a
containerized `open-webui` (the upstream project's app code, NOT here).  
No `package.json`, `pyproject.toml`, or other source/build tooling exists; do
not run application-level build/test/lint/fix commands. Treat these as k8s manifests only.

## Repo contents (what an agent may edit)
- `k8s/open-webui-deployment.yaml` — Service (`ClusterIP:8080`) + Deployment + Ingress, all namespace-less. **Authoritative.**
- `k8s/open-webui-secret.yaml` — gitignored; defines `open-webui-secret` with `WEBUI_SECRET_KEY`. Generate yours; do not reuse the committed placeholder or commit real keys.
- `.gitignore` excludes `data/`, `*.log`, and `k8s/*-secret.yaml`.

## What an agent should NOT edit / expect to run here
- No build/test/lint targets for Open Web UI's Python/Go code (not present).
- Do not try to make the upstream open-webui toolchain work against this repo.
- Don't `kubectl apply`-then-edit runtime DB inside; `data/` is a hostPath volume, see below.

## Deployment fact-check (verify here, not in README prose)
Authoritative source of truth = `k8s/*.yaml`. Read these diverge:
1. **Image tag.** Manifest pins the image to `ghcr.io/open-webui/open-webui:v0.11.0` with `imagePullPolicy: Always`. (README's table says `:main`; the manifest is what k8s actually runs.) Do not change tags without intent.
2. **hostPath on Kind node.** Deployment mounts host `/mnt/workspaces/open-webui/data` → container `/app/backend/data`, `type: DirectoryOrCreate`. The in-repo `./data/` dir (gitignored) holds the live SQLite/cache/vector/upload state for that mount point, so back up and edit it at that on-host path — not by treating repo files as a "source tree."
3. **Namespace.** Manifests carry no `namespace:` — they deploy into the
   current kubectl context's namespace (default: `default`). See `kubectl config view`.

## Apply order (matters, or pods start without env vars / fail to resolve Secret)
```sh
# 1. secret first — Deployment's envFrom refs it explicitly
kubectl apply -f k8s/open-webui-secret.yaml

# 2. service + deployment + ingress are all in this file; applied together
kubectl apply -f k8s/open-webui-deployment.yaml    # ClusterIP svc → Ingress(open-web-ui.localhost, ai.furseal.net) -> svc:8080
```
Note: the README's "step 3 = Ingress" line is misleading — ingress is defined in `open-webui-deployment.yaml`, not a third file.

## Initial admin bootstrap (one-time per fresh empty DB only)
1. Visit `/auth/admin/setup` on first boot → no users exist yet.
2. Admin → Configure Networking: API endpoint = `http://bifrost:8080/v1`; verify `/v1/models`, `/v1/chat/completions`, `/v1/embeddings`.
3. Admin → Observability/Langfuse: set `LANGFUSE_PUBLIC_KEY`, `LANGFURSE_SECRET_KEY`, `LANGFUSE_HOST=http://langfuse:3000` (values live in the secret manifest; paste there, never plaintext).

## Runtime ops cheat sheet
```sh
kubectl logs -f deploy/open-webui           # live pod logs
kubectl rollout restart deployment open-webui  # after config/secret changes
kubectl get pods,svc,ingress,pvc              # into current context namespace
# backup runtime volume on the Kind node:
rsync -av /mnt/workspaces/open-webui/data/ "/mnt/backups/open-webui-data-$(date +%F)/"
```

## Verification step (after any manifest edit)
`kubectl apply --dry-run=client -o yaml -f k8s/` then
`kubectl diff -f k8s/open-webui-deployment.yaml`; ensure `envFrom.secretRef.name: open-webui-secret` resolves to a present Secret before rollout.

## Gotchas an agent is likely to hit otherwise
- Editing the image tag to remove `.v`-pin in favor of `:main` will drift from what this repo ships.
- Rotating `WEBUI_SECRET_KEY` invalidates sessions; re-apply secret + `rollout restart`.
- Ingress exposes `open-webui.localhost`; local DNS/`.localhost` resolution is your own host setup, not k8s'.
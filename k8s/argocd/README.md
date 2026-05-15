# Argo CD on k3s

GitOps controller for the homelab cluster. Watches this repo and reconciles cluster state to match.

URL: <http://homelab:8090>

## Layout

| File | Purpose |
|---|---|
| `00-namespace.yaml` | `argocd` namespace |
| `01-server-loadbalancer.yaml` | LoadBalancer Service exposing `argocd-server` on `:8090` |
| `02-server-cmd-params.yaml` | ConfigMap that puts `argocd-server` in `--insecure` mode (plain HTTP, since Tailscale handles encryption) |
| `apps/root.yaml` | The "App of Apps" — Argo's own root Application that watches `k8s/` and onboards every other app |
| `apps/*.yaml` | One Application per managed app (vault, headlamp, minio, jellyfin, argocd itself) |

## First-time install (the only manual step)

```bash
# 1. Namespace
kubectl apply -f 00-namespace.yaml

# 2. Official Argo install (~50 resources, pulls latest stable)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Our overrides: insecure mode + LoadBalancer
kubectl apply -f 02-server-cmd-params.yaml
kubectl apply -f 01-server-loadbalancer.yaml

# 4. Restart argocd-server so it picks up --insecure
kubectl -n argocd rollout restart deploy/argocd-server

# 5. Wait for everything
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd rollout status statefulset/argocd-application-controller
```

## Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

Username: `admin`. Log in at <http://homelab:8090>.

After first login, change the password from the UI (User Info → Update Password) and **delete the initial secret**:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

## Bootstrap the App of Apps

After Argo is up:

```bash
kubectl apply -f apps/root.yaml
```

This single Application tells Argo to watch `homelab/k8s/argocd/apps/` — every other Application YAML in that folder gets picked up automatically. Adding a new app from now on = drop a new YAML in `apps/`, commit, push.

## Day-to-day ops

| Task | How |
|---|---|
| Add a new app | Create folder under `k8s/<app>/` with manifests, add `apps/<app>.yaml` Application, commit, push |
| Update an existing app | Edit the manifests, commit, push — Argo syncs within ~3 min |
| Force a sync now | UI → app → "Sync", or `kubectl -n argocd patch application <name> --type merge -p '{"operation":{"sync":{}}}'` |
| Stop self-heal on an app | Edit `apps/<app>.yaml`, set `syncPolicy.automated.selfHeal: false` |

## Components (reference)

| Pod | Job |
|---|---|
| `argocd-application-controller` | The brain — watches Git, diffs cluster, applies changes |
| `argocd-repo-server` | Clones Git, renders manifests |
| `argocd-server` | Web UI + API |
| `argocd-redis` | Cache for the API |
| `argocd-dex-server` | SSO (unused; we use built-in admin) |

## Notes

- **Insecure mode is intentional**: Tailscale encrypts the wire; no value in double-TLS.
- **Auto-prune is OFF by default** in our App of Apps — accidental `git rm` won't delete cluster resources. Turn on per-app where safe.
- **The CRD-before-instance rule**: never apply an `Application` resource before Argo is fully running, or kubectl will reject it with `no matches for kind`.

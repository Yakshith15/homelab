# Vault on k3s

Manifests for deploying the [vault](https://github.com/Yakshith15/vault) app to the homelab k3s cluster.

## Layout

| File | Purpose |
|---|---|
| `00-namespace.yaml` | `vault` namespace |
| `01-secret.template.yaml` | Template — **do not commit the real key**. Create the real secret directly (see below). |
| `02-configmap.yaml` | Backend env vars (DB path, content dir, model, throttle, CORS) |
| `03-pvc.yaml` | 2 Gi PVC at `/data` (local-path storage class) |
| `04-backend.yaml` | Backend Deployment + ClusterIP Service on port 8000 |
| `05-frontend.yaml` | Frontend Deployment + ClusterIP Service on port 3000 |
| `06-ingress.yaml` | Traefik strip-prefix Middleware + two Ingresses (`/api/*` → backend, `/*` → frontend) |

## First-time deploy

Run from WSL (where kubectl is configured against k3s).

```bash
# 1. Namespace
kubectl apply -f 00-namespace.yaml

# 2. Real secret (do NOT use the template). Replace <your-key> with the value from backend/.env.
kubectl -n vault create secret generic vault-secrets \
  --from-literal=GEMINI_API_KEY='<your-key>'

# 3. Everything else
kubectl apply -f 02-configmap.yaml
kubectl apply -f 03-pvc.yaml
kubectl apply -f 04-backend.yaml
kubectl apply -f 05-frontend.yaml
kubectl apply -f 06-ingress.yaml

# Or in one shot (after creating the secret manually):
kubectl apply -f .  # skips the .template.yaml because of the suffix? No — it doesn't.
                    # Use explicit files OR delete the template before apply -f .
```

## Verify

```bash
kubectl -n vault get pods -w        # wait until both Running and Ready
kubectl -n vault get svc,ingress
kubectl -n vault logs deploy/vault-backend
kubectl -n vault logs deploy/vault-frontend
```

Then from your Mac (on Tailscale):
- Frontend: <http://100.76.108.54>
- Backend health: <http://100.76.108.54/api/health>

## Rolling a new image

GHA pushes `:latest` and `:sha-<commit>` on every push to vault `main`. Since the deployments use `imagePullPolicy: Always` and `image: ...:latest`, a restart picks up the new image:

```bash
kubectl -n vault rollout restart deploy/vault-backend
kubectl -n vault rollout restart deploy/vault-frontend
kubectl -n vault rollout status deploy/vault-backend
```

(When Argo CD lands, this becomes automatic — Argo will see the new commit in this repo and reconcile.)

## Migrating existing data into the PVC

When you already have a `vault.db` and `content/` from running vault locally (Mac or anywhere else), you can copy them into the cluster's PVC. The PVC starts empty on first deploy — the local file and the in-cluster file are completely separate volumes.

**On the source machine (e.g., Mac):**
```bash
scp ~/Desktop/claude/projects/vault/backend/data/vault.db wsl:/tmp/vault.db
scp -r ~/Desktop/claude/projects/vault/backend/data/content wsl:/tmp/content
```

If SQLite was running on the source recently, flush the WAL first to avoid losing in-flight writes:
```bash
sqlite3 ~/Desktop/claude/projects/vault/backend/data/vault.db "PRAGMA wal_checkpoint(TRUNCATE);"
```

**On WSL:**
```bash
# Stop the backend so SQLite isn't open while we overwrite it
kubectl -n vault scale deploy/vault-backend --replicas=0

# Find the PVC's host path
PVC_PATH=$(sudo find /var/lib/rancher/k3s/storage -maxdepth 1 -type d -name "*vault-data*")
echo "PVC at: $PVC_PATH"

# Copy in
sudo cp /tmp/vault.db "$PVC_PATH/vault.db"
sudo mkdir -p "$PVC_PATH/content"
sudo cp -r /tmp/content/* "$PVC_PATH/content/" 2>/dev/null || true

# Backend container runs as UID 10001 — fix ownership so it can read/write
sudo chown -R 10001:10001 "$PVC_PATH"

# Bring the backend back up
kubectl -n vault scale deploy/vault-backend --replicas=1
kubectl -n vault rollout status deploy/vault-backend
```

Refresh the frontend at <http://100.76.108.54> to confirm your data shows up.

## Updating env vars

Edit `02-configmap.yaml`, `kubectl apply -f 02-configmap.yaml`, then restart the backend (envFrom doesn't hot-reload).

## Updating the API key

```bash
kubectl -n vault delete secret vault-secrets
kubectl -n vault create secret generic vault-secrets \
  --from-literal=GEMINI_API_KEY='<new-key>'
kubectl -n vault rollout restart deploy/vault-backend
```

## Tear down

```bash
kubectl delete namespace vault
```

PVC data is deleted with the namespace. Back up `/var/lib/rancher/k3s/storage/<pvc-id>` on the WSL host first if you care about it.

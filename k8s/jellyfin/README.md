# Jellyfin on k3s

Self-hosted media streaming server. Migrated from Docker Compose to k3s.

URL: <http://homelab:8096>

## Storage layout

| Volume | Type | Path on host | Why |
|---|---|---|---|
| `config` | PVC `jellyfin-config` (5 Gi, `local-path`) | WSL ext4 vhd | Holds SQLite DB, users, library state. Small + performance-sensitive. SQLite on NTFS-via-DrvFs is risky → keep on native ext4. |
| `cache` | hostPath | `D:\jellyfin-cache\` (HDD) | Transcoded chunks + image thumbnails. Can grow to 10+ GB. Sequential I/O, safe on NTFS. Keeps the C: SSD free. |
| `media` | hostPath (read-only) | `E:\courses\masterclass\` (HDD) | Source video library. Same as Docker setup — zero copy. |

## Layout

| File | Purpose |
|---|---|
| `00-namespace.yaml` | `jellyfin` namespace |
| `01-pvc.yaml` | Config PVC (cache is hostPath, see deployment) |
| `02-deployment.yaml` | Deployment + LoadBalancer Service on `:8096` |

## First-time deploy

```bash
# 1. Create the cache directory on D: (the hostPath needs to exist or DirectoryOrCreate will make it; this just sets perms)
sudo mkdir -p /mnt/d/jellyfin-cache
# Jellyfin runs as root in the container, so no chown needed.

# 2. Apply manifests
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-pvc.yaml
kubectl apply -f 02-deployment.yaml

# 3. Watch it come up
kubectl -n jellyfin rollout status deploy/jellyfin
```

First image pull is ~700 MB (Jellyfin is a fat container) — give it 1–3 minutes.

## Migrating from Docker (one-time, this is what we did)

Existing `~/jellyfin/config` from the Docker setup holds users, library scans, and watch history. Copy it into the new PVC so Jellyfin comes up exactly as before.

```bash
# 1. Stop the Docker Jellyfin container
cd ~/homelab
docker compose --profile media stop jellyfin
docker compose --profile media rm -f jellyfin

# 2. Apply k3s manifests (creates PVC, starts a fresh Jellyfin)
cd k8s/jellyfin
kubectl apply -f .
kubectl -n jellyfin rollout status deploy/jellyfin

# 3. Stop the new pod so we can populate the config PVC underneath it
kubectl -n jellyfin scale deploy/jellyfin --replicas=0

# 4. Find the config PVC's host path
PVC_PATH=$(sudo find /var/lib/rancher/k3s/storage -maxdepth 1 -type d -name "*jellyfin-config*")
echo "Config PVC at: $PVC_PATH"

# 5. Wipe the auto-created empty config and copy the old one in
sudo rm -rf "$PVC_PATH"/*
sudo cp -r ~/jellyfin/config/. "$PVC_PATH/"

# 6. Bring Jellyfin back up — it sees its old DB and resumes
kubectl -n jellyfin scale deploy/jellyfin --replicas=1
kubectl -n jellyfin rollout status deploy/jellyfin
```

Open <http://homelab:8096> — should drop you straight to the login screen with your existing user, library, and watch history intact.

(Note: cache from the old Docker setup is **not** copied — it's expendable. Jellyfin re-generates thumbnails/transcodes on demand.)

## Verify

```bash
kubectl -n jellyfin get pods
kubectl -n jellyfin get svc
kubectl -n jellyfin logs deploy/jellyfin
```

Service should show `EXTERNAL-IP` = your tailscale IP and ports `8096:xxxxx/TCP`.

Then in browser: <http://homelab:8096> → log in with your existing Jellyfin credentials → library/watch history should be exactly as before.

## Day-to-day ops

| Task | Command |
|---|---|
| Stop Jellyfin (free RAM/CPU) | `kubectl -n jellyfin scale deploy/jellyfin --replicas=0` |
| Start Jellyfin | `kubectl -n jellyfin scale deploy/jellyfin --replicas=1` |
| Restart (after image update) | `kubectl -n jellyfin rollout restart deploy/jellyfin` |
| Tail logs | `kubectl -n jellyfin logs -f deploy/jellyfin` |
| Bump to latest image | `kubectl -n jellyfin rollout restart deploy/jellyfin` (image is `:latest` with `IfNotPresent` — first pull a fresh image manually if needed) |

Or do all of the above through Headlamp at <http://homelab/headlamp/>.

## Adding more media folders

Edit `02-deployment.yaml`, add another `volumes:` and `volumeMounts:` block:

```yaml
volumes:
  - name: movies
    hostPath:
      path: /mnt/d/movies
      type: Directory

volumeMounts:
  - name: movies
    mountPath: /movies
    readOnly: true
```

Then `kubectl apply -f 02-deployment.yaml` and add the new library in Jellyfin's UI: **Dashboard → Libraries → Add Media Library** → folder `/movies`.

## Clients

Same as before:
- **Mac browser**: <http://homelab:8096>
- **iOS**: Swiftfin (free) or Infuse (paid). Server URL: `http://homelab:8096`
- **Android**: Findroid or official Jellyfin app
- **Mac native**: Jellyfin Media Player

## Notes

- **No HW transcoding**: WSL2 doesn't expose the GPU to containers cleanly. Software transcoding only — fine for course videos at typical bitrates.
- **No HTTPS**: Tailscale already encrypts traffic. Some Jellyfin clients may complain about HTTP for "remote" access — that's a UI warning, traffic is still secure on tailnet.
- **Resource limits**: 2 CPU / 4 Gi max. Bump if you hit transcoding ceilings.
- **Image pinned to `:latest`**: rolling restart picks up new versions. Pin to a specific tag for stability.
- **TZ set to `Asia/Kolkata`**: change in `02-deployment.yaml` env block if you move time zones.

## Tear down

```bash
kubectl delete namespace jellyfin
```

The hostPath cache at `D:\jellyfin-cache\` is **NOT** deleted (it's outside the cluster). Remove manually if desired:
```bash
sudo rm -rf /mnt/d/jellyfin-cache
```

Media at `E:\courses\masterclass\` is read-only and untouched.

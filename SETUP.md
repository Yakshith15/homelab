# Personal Homelab Setup

A complete reference for the homelab: a laptop running bare-metal Ubuntu with a single-node k3s cluster, reachable from the Mac and phone over Tailscale.

> **History.** Until 2026-10-08 this ran inside WSL2 on Windows. The Windows side was reset and the laptop now dual-boots Windows + Ubuntu, with the homelab on the Ubuntu side. §10 is the rebuild runbook that was followed; §18 keeps the short version of the WSL era and what the move cost.

---

## 1. Goal

Use the laptop as an always-on personal server running a k3s cluster. Access everything from:
- Mac (primary dev machine)
- Phone
- Anywhere on the internet (without exposing ports publicly)

Constraints we cared about:
- Shared Wi-Fi → can't trust the LAN (and devices on it can't always reach each other)
- No public IP / port forwarding
- Free / personal-use tier

---

## 2. Architecture

```
  ┌─────────────┐         Tailscale tailnet         ┌──────────────────────────┐
  │     Mac     │ ◄──────── (encrypted) ──────────► │  Laptop — Ubuntu 26.04   │
  │  (client)   │                                   │  (`homelab`)             │
  └─────────────┘                                   │   - Tailscale            │
         ▲                                          │   - SSH server           │
         │                                          │   - k3s                  │
  ┌─────────────┐                                   │     - Traefik            │
  │   Phone     │ ◄──────── (encrypted) ──────────► │     - argocd             │
  │  (client)   │                                   │       + image-updater    │
  └─────────────┘                                   │     - vault  - headlamp  │
                                                    │     - minio  - jellyfin  │
                                                    │     - blog   - mylife    │
                                                    │     - neo                │
                                                    │   HDD: /mnt/d  /mnt/e    │
                                                    └──────────────────────────┘
```

Two Tailscale nodes matter: the Mac and `homelab`. The phone is a third client. The Ubuntu node runs everything — the others just connect to it.

Why this design:
- **Tailscale** = private mesh VPN, no public exposure, works through NAT.
- **Bare-metal Ubuntu** gives k3s all 8 GB of RAM and a normal Linux network stack (no WSL DNS/idle-shutdown workarounds).
- **Dual boot, not a wipe**: Windows is still installed. Only one OS runs at a time — booting Windows takes the homelab offline until the next reboot into Ubuntu.

---

## 3. Devices & IPs

| Device | Role | Tailscale IP | Login |
|---|---|---|---|
| Mac | Dev client | (check `tailscale status`) | macOS user |
| Laptop (Ubuntu) | Server (k3s, SSH) | **100.109.54.25** | `yakshith` (uid 1000) |

OS hostname: `yakshith-ubuntu` (this is also the k3s node name). Wi-Fi LAN IP: `192.168.1.49` (DHCP, may change; nothing depends on it).

Tailscale machine name (MagicDNS): **`homelab`**. Resolves on any tailnet device:
- Short: `http://homelab/` (root currently 404 — reserved for a future dashboard at `/`)
- FQDN: `http://homelab.tailbed621.ts.net/`

App URLs:
- Vault: `http://homelab/vault/`
- Headlamp: `http://homelab/headlamp/`
- mylife: `http://homelab/mylife/`
- Blog: `http://homelab/blog/`
- Argo CD: `http://homelab:8090/`
- Jellyfin: `http://homelab:8096/`
- MinIO API: `http://homelab:9000/`, Console: `http://homelab:9001/`
- neo: no ingress (ClusterIP only; talks out via Telegram)

SSH alias on Mac: `ssh wsl` → connects to the Ubuntu node. (The alias name is a leftover from the WSL era; it points at `homelab`.)

---

## 4. Tailscale setup

### On Mac
1. Installed via Mac App Store, signed in with the main Google account.
2. Approved "Add VPN Configuration" prompt.
3. CLI symlink (because the App Store version doesn't add `tailscale` to PATH):
   ```bash
   sudo ln -s /Applications/Tailscale.app/Contents/MacOS/Tailscale /usr/local/bin/tailscale
   ```

### On Ubuntu
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --hostname=homelab
```
Open the URL it prints, authenticate with the same Google account. `tailscaled` is enabled in systemd by the installer and reconnects on boot.

**The name `homelab` must be free first.** If an old `homelab` machine is still listed in the admin console, the new node silently becomes `homelab-1` and every URL in this repo breaks. Remove dead machines before running `tailscale up`.

### Admin console housekeeping
- https://login.tailscale.com/admin/machines
- **MagicDNS enabled** (DNS tab) — lets you use machine names instead of IPs
- **Disable key expiry** on `homelab` (`…` menu → Disable key expiry) so it isn't logged out every ~180 days

### Verify
```bash
tailscale status            # lists all peers
tailscale ip -4             # this node's IP
```

---

## 5. Ubuntu base setup

Ubuntu 26.04 LTS (desktop install), root filesystem on the NVMe SSD (`nvme0n1p7`, ext4). Windows lives on the same SSD (`nvme0n1p3`).

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y openssh-server ntfs-3g curl git
sudo systemctl enable --now ssh
```

Docker is **not** installed — k3s brings its own containerd and nothing here needs raw Docker.

Power settings (never sleep, ignore the lid) are in §12.

---

## 6. SSH setup

### On Ubuntu (server side)
`openssh-server` installed and enabled in §5.

### On Mac (client side)

`~/.ssh/config`:
```
Host wsl
  HostName homelab
  User yakshith
```

Install the Mac's public key (asks for the Ubuntu password once):
```bash
ssh-copy-id yakshith@homelab
```

Now from anywhere on the tailnet:
```bash
ssh wsl
```

If the node is ever reinstalled, clear the old host key first: `ssh-keygen -R homelab`.

---

## 7. Storage: the HDD and its mounts

The laptop has a second disk (HDD, `sda`) with three NTFS partitions left over from Windows. Two of them hold app data and are mounted at the paths WSL used to expose, so **no manifest had to change** in the move to Ubuntu.

| Partition | Size | Was (Windows) | Mounted at | Holds |
|---|---|---|---|---|
| `sda1` | 296 G | `D:` | `/mnt/d` | `minio-data`, `mylife-data`, `photos`, `jellyfin-cache` |
| `sda2` | 313 G | `E:` | `/mnt/e` | `courses/masterclass` (Jellyfin media), other personal folders |
| `sda3` | 322 G | third volume | not mounted | personal files, not used by any app |

`/etc/fstab`:
```
UUID=A292C28692C25E83  /mnt/d  ntfs-3g  defaults,nofail,uid=1000,gid=1000,umask=022  0  0
UUID=CA18C70E18C6F909  /mnt/e  ntfs-3g  defaults,nofail,uid=1000,gid=1000,umask=022  0  0
```

- `uid=1000,gid=1000` — NTFS has no Linux ownership, so the mount assigns it. MinIO and mylife run as uid 1000 and need to write.
- `nofail` — a missing or unreadable HDD must not drop the boot into emergency mode.
- All three partitions are labelled "New Volume"; identify them by UUID or by contents (`lsblk -f`), never by label.

**k3s waits for the mounts.** `nofail` means boot continues without the drives, and MinIO's `hostPath` is `DirectoryOrCreate` — if k3s started first it would create an empty `/mnt/d/minio-data` on the root disk and serve that. A systemd drop-in prevents it:

```bash
sudo mkdir -p /etc/systemd/system/k3s.service.d
printf "[Unit]\nRequiresMountsFor=/mnt/d /mnt/e\n" | sudo tee /etc/systemd/system/k3s.service.d/mounts.conf
sudo systemctl daemon-reload
```

In the desktop Files app the drives don't appear in the sidebar: press Ctrl+L, type `/mnt/d`, then Ctrl+D to bookmark.

PVC-backed data (`vault-data`, `jellyfin-config`, `neo-data`) lives on the root SSD under `/var/lib/rancher/k3s/storage/`.

---

## 8. DNS

Ubuntu uses `systemd-resolved`. Out of the box the only general resolver was the Wi-Fi router (`192.168.1.1`), and it dropped a lookup during the first image pulls (see §15). Cloudflare and Google are now the global resolvers on any network:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
printf "[Resolve]\nDNS=1.1.1.1 8.8.8.8\nDomains=~.\n" | sudo tee /etc/systemd/resolved.conf.d/dns.conf
sudo systemctl restart systemd-resolved
kubectl -n kube-system rollout restart deploy/coredns     # CoreDNS copies the node's resolvers at pod start
```

- `Domains=~.` routes every normal lookup to the global servers.
- Tailscale still answers `*.ts.net` names through its own interface (split DNS) — MagicDNS works on the node, unlike under WSL.
- Verify: `resolvectl status | head -8` shows `1.1.1.1 8.8.8.8` under **Global**.
- Undo: delete `dns.conf`, restart `systemd-resolved`.

---

## 9. Secrets applied out-of-band

None of these are in git. A fresh cluster has none of them, and each app that needs one stays broken until it exists.

| Secret | Namespace | Used by | What it holds |
|---|---|---|---|
| `repo-github-yakshith` (label `argocd.argoproj.io/secret-type=repo-creds`) | `argocd` | Argo CD, Image Updater (`write-back-method: git`) | Fine-grained GitHub PAT — repos `homelab`, `neo`, `blog`, `mylife`, Contents: Read and write |
| `argocd-image-updater-secret` | `argocd` | Image Updater write-back for `vault` and `neo` | Same fine-grained PAT, keys `username` / `password` |
| `ghcr-pull-secret` | `neo`, `blog`, `mylife` | Image pulls + Image Updater digest polls | **Classic** PAT with `read:packages` |
| `vault-secrets` | `vault` | vault backend | `GEMINI_API_KEY` |
| `minio-secrets` | `minio` | MinIO | `MINIO_ROOT_USER`, `MINIO_ROOT_PASSWORD` |
| `neo-secrets` | `neo` | neo | `anthropic-api-key`, `telegram-bot-token` (required); `github-token`, `gemini-api-key`, `calendar-ical-url` (optional) |

Two different GitHub tokens are needed because **GHCR does not accept fine-grained tokens** — only classic ones.

Creation commands are in §10 step 6. Enter values with `read -s` so they never land in shell history or scrollback:
```bash
read -s -p "token: " T; echo
```

The MinIO root password is also saved on the Mac in the `mc` alias (`mc alias list homelab`) — that is how it was recovered for the rebuild.

---

## 10. Rebuild runbook (new machine or reinstall)

What was done on 2026-10-08, in order. Roughly 1–2 hours.

1. **Base OS** — §5.
2. **Mount the HDD** at `/mnt/d` and `/mnt/e` — §7. Check: `findmnt /mnt/d`, `findmnt /mnt/e`, and `touch /mnt/d/.t && rm /mnt/d/.t`.
3. **Tailscale** — remove the dead `homelab` node in the admin console, then §4. Disable key expiry. Update the Mac's SSH config and key (§6).
4. **Power + DNS** — §12 and §8.
5. **k3s** — §11 (install, kubeconfig, mounts drop-in, `tls-san`). Reboot once and confirm mounts, Tailscale and k3s all return unaided.
6. **Argo CD + secrets** (full detail in [`ARGO.md`](ARGO.md) §3):
   ```bash
   mkdir -p ~/projects && git clone https://github.com/Yakshith15/homelab.git ~/projects/homelab
   cd ~/projects/homelab
   kubectl apply -f k8s/argocd/00-namespace.yaml
   kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   kubectl apply -f k8s/argocd/02-server-cmd-params.yaml -f k8s/argocd/01-server-loadbalancer.yaml
   kubectl -n argocd rollout restart deploy/argocd-server
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml

   # $GITHUB_PAT = fine-grained token, $GHCR_PAT = classic read:packages token (see §9)
   kubectl -n argocd create secret generic repo-github-yakshith \
     --from-literal=type=git --from-literal=url=https://github.com/Yakshith15 \
     --from-literal=username=git --from-literal=password="$GITHUB_PAT"
   kubectl -n argocd label secret repo-github-yakshith argocd.argoproj.io/secret-type=repo-creds
   kubectl -n argocd patch secret argocd-image-updater-secret --type merge \
     -p "{\"stringData\":{\"username\":\"git\",\"password\":\"$GITHUB_PAT\"}}"
   kubectl -n argocd rollout restart deploy/argocd-image-updater-controller

   for ns in vault minio neo blog mylife; do kubectl create ns $ns; done
   for ns in neo blog mylife; do
     kubectl -n $ns create secret docker-registry ghcr-pull-secret \
       --docker-server=ghcr.io --docker-username=Yakshith15 --docker-password="$GHCR_PAT"
   done
   kubectl -n vault create secret generic vault-secrets --from-literal=GEMINI_API_KEY="$V"
   kubectl -n minio create secret generic minio-secrets \
     --from-literal=MINIO_ROOT_USER=admin --from-literal=MINIO_ROOT_PASSWORD="$M"
   # neo-secrets: see neo/k8s/02-secret.template.yaml
   ```
7. **Bootstrap** — `kubectl apply -f k8s/argocd/apps/root.yaml`, wait ~3 min, then:
   ```bash
   kubectl -n argocd get app                                  # 9 apps, all Synced
   kubectl get pods -A | grep -v -E 'Running|Completed'       # should be empty
   ```
8. **Per-app first run**
   - Argo CD: read the initial admin password, change it in the UI, delete `argocd-initial-admin-secret`.
   - Headlamp: `kubectl -n headlamp create token headlamp --duration=8760h`.
   - Jellyfin: first-run wizard; add the library as **Home Videos and Photos** on `/media` (§11.4).
   - Vault: restore `vault.db` + `content/` into the PVC (`k8s/vault/README.md`), then re-import anything newer.

Pre-creating the namespaces and secrets (step 6) before the bootstrap (step 7) is what lets every app come up green on the first sync.

---

## 11. k3s (lightweight Kubernetes)

k3s is a single-binary Kubernetes distribution that runs the entire control plane + worker on one node.

### Architecture (current)
- **One node:** `yakshith-ubuntu` is both control plane and worker.
- **Bundled components:** containerd (runtime), Traefik (ingress), CoreDNS, ServiceLB, local-path-provisioner (default StorageClass), metrics-server.
- **Datastore:** SQLite (not etcd) — fine for single-node.
- **Version at rebuild:** v1.36.5+k3s1.

### Install
```bash
curl -sfL https://get.k3s.io | sh -
```
Creates and enables `k3s.service`; `kubectl` is symlinked at `/usr/local/bin/kubectl`.

### kubectl without sudo, `k` alias, tab completion
```bash
mkdir -p ~/.kube && sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config && chmod 600 ~/.kube/config

cat >> ~/.bashrc <<'EOF'
export KUBECONFIG=$HOME/.kube/config
alias k=kubectl
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
EOF
source ~/.bashrc
```
The k3s-bundled `kubectl` reads the root-owned `/etc/rancher/k3s/k3s.yaml` unless `KUBECONFIG` says otherwise.

### Server config
`/etc/rancher/k3s/config.yaml`:
```yaml
tls-san:
  - 100.109.54.25
  - homelab
  - homelab.tailbed621.ts.net
```
Adds the Tailscale address and names to the API server certificate, so agents (and a `kubectl` on the Mac) can connect over the tailnet. Apply with `sudo systemctl restart k3s` — pods keep running through a k3s restart.

Plus the mounts drop-in from §7 (`/etc/systemd/system/k3s.service.d/mounts.conf`).

### Day-to-day commands
```bash
kubectl get nodes                       # cluster nodes
kubectl get pods -A                     # all pods, all namespaces
kubectl get svc -A                      # all services
kubectl describe pod <pod> -n <ns>      # pod details + events (debug)
kubectl logs -f <pod> -n <ns>           # follow logs
kubectl top nodes                       # node CPU/RAM
kubectl top pods -A                     # pod CPU/RAM
```

### Service operations
```bash
sudo systemctl status k3s               # daemon state
sudo systemctl restart k3s              # restart the server process (pods keep running)
sudo systemctl stop k3s                 # stop the server process
```

### Adding a node (tested 2026-10-08)
A second node joins with the server URL and the token from `/var/lib/rancher/k3s/server/node-token`. A throwaway agent in Docker on the Mac worked:
```bash
docker run -d --name k3s-mac-agent --hostname mac-agent --privileged --tmpfs /run --tmpfs /var/run \
  -e K3S_URL=https://100.109.54.25:6443 -e K3S_TOKEN="$K3S_TOKEN" \
  rancher/k3s:v1.36.5-k3s1 agent --node-taint experiment=true:NoSchedule
```
- **Always taint a new node** (or pin the apps). MinIO, mylife and Jellyfin use `hostPath`s that exist only on `yakshith-ubuntu`; the PVCs are node-local too.
- Joining over Tailscale needs the `tls-san` entries above — the Mac could not reach the Wi-Fi IP at all.
- The node went `Ready` and ran a pod, but **cross-node pod traffic does not work** this way: Docker on a Mac hides the container behind NAT. Real multi-node networking across networks needs k3s's Tailscale integration (`--vpn-auth`) on the server and every agent — not done.
- Mac is arm64, the homelab amd64; our own GHCR images are built for amd64.
- Remove: `kubectl delete node mac-agent`, `kubectl -n kube-system delete secret mac-agent.node-password.k3s` (otherwise a re-join under the same name is refused), `docker rm -f k3s-mac-agent`.

### Adding the second Windows laptop as a node (WSL2 Ubuntu) — plan, NOT yet tested

The join itself is the same as above. The work is in the networking: by default WSL2 sits behind NAT exactly like Docker on the Mac, so the node would join but its pods would be unreachable. Giving both nodes a Tailscale address and running the cluster network over it fixes that — and it means changing the **server** too. Do it when there's time to check every app afterwards.

**A. On the Windows laptop — prepare WSL2**

1. Admin PowerShell: `wsl --install -d Ubuntu`, reboot, create the user.
2. Inside Ubuntu, enable systemd and stop WSL rewriting DNS:
   ```bash
   printf "[boot]\nsystemd=true\n\n[network]\ngenerateResolvConf = false\n" | sudo tee /etc/wsl.conf
   ```
   Then `wsl --shutdown` in PowerShell and reopen Ubuntu.
3. Tailscale inside WSL (it gets its own `100.x` address, separate from Windows):
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up --hostname=wsl-node --accept-dns=false
   sudo rm -f /etc/resolv.conf && printf "nameserver 1.1.1.1\nnameserver 8.8.8.8\n" | sudo tee /etc/resolv.conf
   tailscale ip -4          # note this: <WSL_TS_IP>
   ```
   `--accept-dns=false` plus the pinned `resolv.conf` avoids the 2026-09-30 outage (§18). Disable key expiry for `wsl-node` in the admin console.
4. Keep WSL alive: Task Scheduler job at logon running `wsl.exe -d Ubuntu -u root sleep infinity`; set Windows to never sleep on AC and "do nothing" on lid close. Without these the node drops out whenever WSL idles or Windows sleeps.
5. Optional `C:\Users\<you>\.wslconfig` to size it: `[wsl2]` / `memory=…` / `processors=…`.

**B. On the homelab — move the cluster network onto Tailscale**

Add to `/etc/rancher/k3s/config.yaml` (keep the existing `tls-san` block):
```yaml
node-ip: 100.109.54.25
flannel-iface: tailscale0
```
Make k3s start after Tailscale, then restart:
```bash
printf "[Unit]\nAfter=tailscaled.service\nWants=tailscaled.service\n" | sudo tee /etc/systemd/system/k3s.service.d/tailscale.conf
sudo systemctl daemon-reload && sudo systemctl restart k3s
kubectl get nodes -o wide          # INTERNAL-IP should now be 100.109.54.25
```
Then re-check every URL in §3 and reboot once to confirm it still comes up unaided. Roll back by deleting the two lines and the drop-in, and restarting k3s.

**C. Join the WSL node**

Token from the homelab: `sudo cat /var/lib/rancher/k3s/server/node-token`. In WSL:
```bash
read -s -p "token: " K3S_TOKEN; echo
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.36.5+k3s1 \
  K3S_URL=https://100.109.54.25:6443 K3S_TOKEN="$K3S_TOKEN" sh -s - agent \
  --node-name wsl-node --node-ip <WSL_TS_IP> --flannel-iface tailscale0 \
  --node-taint experiment=true:NoSchedule
```
- Match `INSTALL_K3S_VERSION` to the server (`kubectl get nodes`).
- `--node-name` matters: WSL's hostname is the Windows machine name.
- The taint keeps existing apps on `yakshith-ubuntu` (see the bullets above). Remove it later only for workloads that carry their own node selection.

**D. Verify pods can talk across nodes** (the part that failed with Docker on the Mac)
```bash
kubectl run net-test --image=nginx --restart=Never --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"wsl-node"},"tolerations":[{"key":"experiment","operator":"Exists","effect":"NoSchedule"}]}}'
kubectl get pod net-test -o wide                    # note its 10.42.x.x IP
curl -s -o /dev/null -w '%{http_code}\n' http://<pod-ip>     # from the homelab: 200 = cross-node traffic works
kubectl exec net-test -- getent hosts kubernetes.default      # in-cluster DNS from the new node
kubectl delete pod net-test
```

**Unknowns to watch for** (this has not been run): whether k3s is content with the server's node IP changing on an existing cluster; MTU over Tailscale (flannel should pick it up from `tailscale0`); and whether ServiceLB ports stay reachable exactly as before. If the simple route misbehaves, the alternative is k3s's own Tailscale integration (`--vpn-auth`, §19 link).

**Same network instead of Tailscale?** If both laptops sit on one LAN that lets devices reach each other, section B can be skipped: set `networkingMode=mirrored` under `[wsl2]` in `.wslconfig` (Windows 11) so WSL takes the laptop's LAN address, allow UDP 8472 and TCP 10250 through the Windows firewall, and join with the homelab's LAN IP. The current Wi-Fi did not let the Mac reach the homelab, so don't count on it here.

**Remove the node:** `kubectl drain wsl-node --ignore-daemonsets`, `kubectl delete node wsl-node`, `kubectl -n kube-system delete secret wsl-node.node-password.k3s`, and in WSL `/usr/local/bin/k3s-agent-uninstall.sh`.

### Why k3s vs alternatives
- **vs minikube/KIND:** those are dev-focused, designed to be torn down. k3s runs in systemd and persists.
- **vs full kubeadm Kubernetes:** k3s is one binary; same Kubernetes API, much less ops burden.

---

## 11.1. Vault (deployed app)

Personal knowledge-vault app — FastAPI backend + Next.js frontend + SQLite + Gemini analysis. Lives in [a separate repo](https://github.com/Yakshith15/vault); deployed to this cluster from manifests in `k8s/vault/`.

URL: <http://homelab/vault/>

Manifests + ops docs (deploy, scale, rolling, data migration, env updates, teardown) live in [`k8s/vault/README.md`](k8s/vault/README.md).

Key facts:
- Namespace: `vault`
- Image source: GHCR (`ghcr.io/yakshith15/vault-backend:latest`, `ghcr.io/yakshith15/vault-frontend:latest`) — built by GHA on every push to vault repo `main`.
- Frontend served at `/vault` via Next.js `basePath: '/vault'`. Browser requests `/vault/api/...` → Traefik middleware `strip-vault-api` rewrites to `/...` → backend.
- Probes hit `/vault` (matches basePath); hitting `/` would 404.
- **Managed as a Kustomize source** (`k8s/vault/kustomization.yaml`) — required by Argo CD Image Updater. The `images:` block is where Image Updater rewrites the digest.
- New images picked up automatically by Image Updater (~2 min poll), written back to git, then synced by Argo.
- Data: `vault-data` PVC (2 Gi, local-path) — SQLite DB + content directory. The backend runs as uid 10001.
- **2026-10-08:** the PVC was lost with the WSL disk. Restored from the copy on the Mac (`vault/backend/data`, 428 items, May 2026) and re-imported the rest; the import endpoint skips URLs it already has.
- Stop/start: `kubectl -n vault scale deploy --all --replicas=0/1` or click in Headlamp.

---

## 11.2. Headlamp (k8s dashboard)

In-cluster web UI for managing the cluster — scale, logs, exec, edit manifests.

URL: <http://homelab/headlamp/>

Manifests + ops docs live in [`k8s/headlamp/README.md`](k8s/headlamp/README.md).

Key facts:
- Namespace: `headlamp`
- ServiceAccount bound to `cluster-admin` (single-user homelab; scope down if more users are added).
- Login: Bearer token. Mint with `kubectl -n headlamp create token headlamp --duration=8760h` (1-year token). Tokens from the old cluster are invalid.
- `-base-url=/headlamp` arg on the deployment lets it serve cleanly behind the `/headlamp/` ingress path.

---

## 11.3. MinIO (S3-compatible object storage)

Self-hosted S3 replacement — the homelab's general blob store.

| Endpoint | URL |
|---|---|
| S3 API (SDKs, `aws cli`, `mc`, rclone) | <http://homelab:9000> |
| Web console | <http://homelab:9001> |

Manifests + ops docs live in [`k8s/minio/README.md`](k8s/minio/README.md).

Key facts:
- Namespace: `minio`
- **Storage lives on the HDD**: `hostPath: /mnt/d/minio-data` (NTFS via ntfs-3g, see §7). The data survived the move from Windows untouched.
- **Image: `cgr.dev/chainguard/minio:latest`.** `minio/minio` was removed from Docker Hub (404 as of 2026-10) and a fresh node cannot pull it. Chainguard publishes a build of the same server; it started cleanly on the existing data (RELEASE.2026-09-22).
- Service type **LoadBalancer** (not Ingress) — path-prefixed Ingress would break presigned URLs and S3 SDK host expectations.
- Single-node, single-drive mode — no erasure coding/replication.
- Credentials in `minio-secrets`. Reusing the old root password kept everything readable.
- Pod runs as uid 1000, matching the mount's `uid=1000`.
- **Don't browse `/mnt/d/minio-data` directly**: MinIO stores objects in its internal "XL" format, not as plain files. Use the console, `mc`, or an S3 client.

---

## 11.4. Jellyfin (media streaming)

Self-hosted media server.

URL: <http://homelab:8096>

Manifests + ops docs live in [`k8s/jellyfin/README.md`](k8s/jellyfin/README.md).

Key facts:
- Namespace: `jellyfin`
- **Hybrid storage**:
  - `config` → PVC on the root SSD (5 Gi, `local-path`) — SQLite DB, users, library state. Kept on ext4, off NTFS.
  - `cache` → hostPath `/mnt/d/jellyfin-cache` (HDD) — transcodes/thumbnails.
  - `media` → hostPath `/mnt/e/courses/masterclass` read-only.
- Service type **LoadBalancer** on `:8096`.
- Image `jellyfin/jellyfin:latest`, runs as root inside the container.
- No HW transcoding configured. (It was impossible under WSL2; on bare metal it could be set up, but hasn't been.)
- **2026-10-08:** config PVC lost with the WSL disk, so users and watch history started over. Clients show a "Server Mismatch" prompt once — the server identity changed at the same address; "Connect Anyway" is correct.
- **Library type matters.** As **Movies**, Jellyfin merged the "- I" / "- II" lecture pairs into single entries (9 shown for 15 files). Use **Home Videos and Photos**, which lists every file as-is.
- Stop/start: `kubectl -n jellyfin scale deploy/jellyfin --replicas=0/1` or click in Headlamp.
- Clients: browser at <http://homelab:8096>, Swiftfin/Infuse on iOS, Findroid on Android.

---

## 11.4.1. mylife (life-in-weeks memory journal)

Personal "life in weeks" journal — one box per week, memories with photos and video. Zero-dependency Node server + one HTML page. Code, Dockerfile, GHA build and k8s manifests live in [its own repo](https://github.com/Yakshith15/mylife) (`k8s/` is a Kustomize source), same split as neo and blog.

URL: <http://homelab/mylife/>

Key facts:
- Namespace: `mylife`. Argo Application `k8s/argocd/apps/mylife.yaml`, in the `ImageUpdater` CR, digest strategy, plain `git` write-back (like blog).
- **Stateful, `prune: false`. Storage on the HDD, like MinIO:**
  - `hostPath: /mnt/d/mylife-data` → `/data`, read-write — `me.json`, per-profile `db.json`, uploads. The only copy of the memories. Must pre-exist (the Deployment uses `type: Directory`, so a missing folder fails loudly instead of being created root-owned).
  - `hostPath: /mnt/d/photos` → `/library`, **read-only** — the photo/video library. Fill it from the Mac with `rsync -avh ~/Pictures/for-homelab/ wsl:/mnt/d/photos/`.
- Served under `/mylife` with no Traefik strip-prefix: the server runs with `BASE_PATH=/mylife`. Probes hit `/healthz`.
- **No auth** — reachable by anything on the tailnet, nothing more. Never give it a public route.
- GHCR package is private → `ghcr-pull-secret` in `mylife` ns + Role letting Image Updater read it.
- `Recreate` strategy, single replica. Runs as uid 1000 with a read-only root filesystem.
- Survived the 2026-10-08 move intact — its data was on the HDD.

---

## 11.4.2. blog and neo

Both live in their own private repos (`Yakshith15/blog`, `Yakshith15/neo`) with a Kustomize `k8s/` folder, an Argo Application here under `k8s/argocd/apps/`, and a private GHCR image (so each needs `ghcr-pull-secret`).

- **blog** — <http://homelab/blog/>. Stateless, `prune: true`.
- **neo** — no ingress. Stateful (`neo-data` PVC), needs `neo-secrets` (§9). As of 2026-10-08 the secret has not been recreated, so the pod sits in `CreateContainerConfigError`; the PVC also started empty.

---

## 11.5. Argo CD (GitOps controller)

GitOps for the cluster. Watches this repo and reconciles every Application to match git within ~3 minutes.

URL: <http://homelab:8090/> (LoadBalancer, plain HTTP — Tailscale handles encryption)

Full reference: [`ARGO.md`](ARGO.md). Short ops notes: [`k8s/argocd/README.md`](k8s/argocd/README.md).

Key facts:
- Namespace: `argocd`. Installed from the upstream `stable` manifest with `--server-side` (v3.5.4 at rebuild).
- Our overrides: LoadBalancer Service `argocd-server-lb` on `:8090`, and `server.insecure: "true"`.
- **App-of-Apps**: `k8s/argocd/apps/root.yaml` watches `k8s/argocd/apps/`; it is the only Application applied by hand.
- All apps `selfHeal: true`; `prune: false` except `headlamp` and `blog`.
- Private repos are read through one credential template, `repo-github-yakshith` (§9). A repo created later must be added to that token's repository list.
- Login: `admin`. Initial password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`.
- Stop/start: `kubectl -n argocd scale deploy --all --replicas=0/1 && kubectl -n argocd scale statefulset argocd-application-controller --replicas=0/1`.

---

## 11.5.1. Argo CD Image Updater

Polls GHCR for new image digests and writes them back to git, so Argo's normal sync rolls the pods. Covers `vault`, `neo`, `blog`, `mylife`.

- The Deployment is **`argocd-image-updater-controller`** (it was `argocd-image-updater` in older releases). Logs: `kubectl -n argocd logs -f deploy/argocd-image-updater-controller`.
- Install path is `/config/install.yaml`, NOT `/manifests/install.yaml`.
- Write-back credentials: `vault` and `neo` use `argocd-image-updater-secret`; `blog` and `mylife` use plain `git`, which reuses Argo's repo credentials.
- Confirmed working on the new cluster the same day: it pushed `build: automatic update of vault` within minutes of the bootstrap.

Everything else (annotations, strategies, PAT rotation, gotchas) is in [`ARGO.md`](ARGO.md) §10.

---

## 12. Power settings (always-on server mode)

Goal: the laptop keeps running with the lid closed and never suspends.

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
printf "[Login]\nHandleLidSwitch=ignore\nHandleLidSwitchExternalPower=ignore\nHandleLidSwitchDocked=ignore\n" \
  | sudo tee /etc/systemd/logind.conf.d/lid.conf
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

- The lid setting applies from the next reboot. Don't `restart systemd-logind` on a desktop install — it can end the graphical session.
- Verify: `systemctl is-enabled sleep.target suspend.target` → `masked` twice.
- Consequence: under Ubuntu the laptop **never** sleeps, on battery too. Don't bag it while running; `sudo poweroff` when it should be off.
- Undo: delete `lid.conf`, `sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target`, reboot.
- Windows keeps its own power settings; none of this affects it.

---

## 13. Auto-start on boot

Nothing to configure beyond what the installers did:

1. GRUB boots Ubuntu by default.
2. `/etc/fstab` mounts `/mnt/d` and `/mnt/e`.
3. `tailscaled` reconnects as `homelab`.
4. `k3s` starts once the mounts are up (§7 drop-in) and brings every pod back.

Verified with a reboot on 2026-10-08: mounts, Tailscale and all pods returned without touching the machine. Allow a minute or two.

---

## 14. Common commands cheatsheet

### From Mac
```bash
ssh wsl                               # connect to the Ubuntu node
tailscale status                      # peers
mc ls homelab                         # MinIO buckets
```

### On Ubuntu
```bash
# Tailscale
tailscale status
tailscale ip -4
sudo systemctl status tailscaled

# Drives
findmnt /mnt/d                        # one path per call — two args mean "source target"
findmnt /mnt/e
lsblk -f

# DNS
resolvectl status | head -8
getent hosts ghcr.io

# Cluster
k get nodes
k get pods -A | grep -v -E 'Running|Completed'
k -n argocd get app
```

---

## 15. Things we learned (gotchas)

### One flaky resolver stalls image pulls
During the rebuild the dex pod sat in `ErrImagePull` with `dial tcp: lookup ghcr.io: Try again` while `quay.io` had resolved a minute earlier. The only general resolver was the Wi-Fi router. Fix: global resolvers in systemd-resolved (§8), then recreate CoreDNS and delete the stuck pod.

### CoreDNS keeps the old resolver after a DNS change
CoreDNS forwards to the resolvers it copied when its pod started. After any change to the node's DNS:
```bash
kubectl -n kube-system rollout restart deploy/coredns && kubectl -n kube-system rollout status deploy/coredns
kubectl -n argocd annotate app <name> argocd.argoproj.io/refresh=hard --overwrite
```
Symptom when forgotten: Argo apps `Unknown` with `lookup github.com on 10.43.0.10:53: server misbehaving`.

### `minio/minio` is gone from Docker Hub
`pull access denied, repository does not exist` on a node with no cached image. The old cluster only kept working from its cache. Now on `cgr.dev/chainguard/minio` (§11.3). General lesson: `:latest` + `IfNotPresent` hides a vanished upstream until the day the node is rebuilt.

### GHCR needs a classic token
Fine-grained PATs cannot pull from `ghcr.io`. Repo access (Argo, Image Updater write-back) and image pulls are two separate tokens (§9).

### New private repo → add it to Argo's GitHub token
Argo reads every `https://github.com/Yakshith15/*` repo through the `repo-github-yakshith` credential template. Its fine-grained PAT is scoped to a list of repos, so a repo created later is invisible to Argo.

Symptom seen (mylife, 2026-09-30): app stuck `Unknown` with `authorization failed: Write access to repository not granted.` Fix: open the token on GitHub → Repository access → add the repo. The token value doesn't change; hard-refresh the app.

List what Argo has (names + URLs only, no tokens):
```bash
kubectl -n argocd get secrets -l 'argocd.argoproj.io/secret-type in (repository,repo-creds)' \
  -o go-template='{{range .items}}{{.metadata.name}}  {{index .metadata.labels "argocd.argoproj.io/secret-type"}}  {{.data.url | base64decode}}{{"\n"}}{{end}}'
```

### Pull secret for a new private GHCR image
Each namespace with a private `ghcr.io/yakshith15/*` image needs its own `ghcr-pull-secret`. Copy a working one rather than minting a token:
```bash
kubectl -n blog get secret ghcr-pull-secret -o yaml \
  | grep -vE '^\s+(namespace|resourceVersion|uid|creationTimestamp):' \
  | kubectl -n <new-ns> apply -f -
```

### NTFS mounts read-only after Windows
If Windows hibernated or shut down with Fast Startup on, the NTFS volumes are left "dirty" and `ntfs-3g` refuses to mount them read-write — MinIO and mylife then fail to write. Fix from Windows: turn Fast Startup off (Control Panel → Power Options → Choose what the power buttons do) and do a full shutdown.

### Booting Windows = homelab off
Only one OS runs at a time. Also: a Windows update can overwrite GRUB (laptop boots straight to Windows — repair with `boot-repair` from an Ubuntu USB), and Windows may show the wrong clock after Linux (`sudo timedatectl set-local-rtc 1 --adjust-system-clock` on the Linux side).

### Pasting several lines into the terminal
The last line of a multi-line paste doesn't run until Enter is pressed, and lines pasted while `sudo` is asking for a password can be swallowed. Both happened during the rebuild (`systemctl mask` and the `.bashrc` block silently didn't run). Paste one command at a time and check each one's output.

### Tailscale name collisions
A new node registering while an old `homelab` still exists becomes `homelab-1`. Remove the old machine first (§4).

### Tailscale "idle" vs "offline"
- **idle** = peer reachable, no active traffic → fine
- **offline** = peer unreachable → check the peer's Tailscale state
- **active** = traffic flowing right now

### Tailscale CLI not in PATH on Mac (App Store install)
Symlink to `/usr/local/bin/tailscale` — see §4.

---

## 16. Resume / restart procedure

After the laptop has been off:

1. Power on; let GRUB boot Ubuntu. No login needed — services start at boot.
2. On Mac, ensure Tailscale is connected (menu bar icon).
3. `ssh wsl` from Mac → should drop straight in.

If `ssh wsl` fails:
- Did it boot Windows instead? Check the screen.
- `tailscale status` on Mac — is `homelab` listed and not offline?
- On the laptop: `sudo systemctl status tailscaled`; restart it if needed.

Then check the cluster came back:
```bash
findmnt /mnt/d && findmnt /mnt/e                          # both mounted, rw
kubectl get pods -A | grep -v -E 'Running|Completed'      # should be (nearly) empty
kubectl -n argocd get app                                 # all Synced
```
- k3s not starting → a drive didn't mount (the drop-in blocks k3s on purpose). `sudo mount -a`, read the error, see §15 "NTFS mounts read-only".
- Everything in `ImagePullBackOff` → DNS, §8 / §15.
- Apps `Unknown` → CoreDNS, §15.

---

## 17. Next steps (TODO)

- [ ] **Backups — top priority.** The 2026-10-08 move lost `vault-data`, `jellyfin-config` and `neo-data` because nothing was copied off the machine. Now at risk on the root SSD: the same three PVCs. On the HDD: `/mnt/d/mylife-data`, `/mnt/d/photos`, `/mnt/d/minio-data`. Plan: scheduled rsync/rclone to another disk or a cloud bucket.
- [ ] **neo** — recreate `neo-secrets` (§9).
- [ ] **Vaultwarden** (password manager) — next service. Needs HTTPS first (browsers refuse the web vault's crypto over plain HTTP) and a backup before real passwords go in.
- [ ] **HTTPS via Tailscale** (`tailscale serve` / certs for `homelab.tailbed621.ts.net`) — now a prerequisite for Vaultwarden.
- [ ] **Multi-node over Tailscale** — k3s `--vpn-auth` integration, with a VM rather than Docker on the Mac (§11).
- [ ] **Vault CORS** — `VAULT_CORS_ORIGINS` in `k8s/vault/02-configmap.yaml` still names the old WSL IP. Harmless while the frontend calls the API same-origin; clean up when next touching vault.
- [ ] **More services**: Homepage (dashboard at `/`), Gitea, Uptime Kuma, *arr stack.
- [ ] **Jellyfin HW transcoding** — possible on bare metal now.

Done: Tailscale mesh, k3s, Vault, Headlamp, MinIO, Jellyfin, Argo CD + App-of-Apps, Image Updater, blog, neo, mylife, friendly hostname, move from WSL2 to bare-metal Ubuntu (2026-10-08).

---

## 18. History: the WSL2 era (May–Oct 2026)

The first version ran Ubuntu inside WSL2 on Windows, with Tailscale inside the WSL instance (IP `100.76.108.54`) and the Windows drives visible at `/mnt/d` and `/mnt/e` through DrvFs. Things that setup needed and this one doesn't:

- `/etc/wsl.conf` with `systemd=true` and `generateResolvConf = false`, a hand-pinned `/etc/resolv.conf`, and `tailscale set --accept-dns=false` — WSL and Tailscale both kept rewriting the resolver (outage on 2026-09-30).
- A Task Scheduler job running `wsl.exe ... sleep infinity`, because WSL shuts down when idle.
- A 5 GB RAM cap via `.wslconfig`.
- Windows power settings for lid/sleep; `tailscaled` needed a restart after every Windows sleep.

Before k3s, services ran under Docker Compose (Portainer, Jellyfin); both were retired in May 2026 — Portainer replaced by Headlamp, Jellyfin moved into the cluster.

**What the move cost.** Everything in git and everything on the HDD came across. Lost: the WSL virtual disk and with it the three local-path PVCs (vault's database, Jellyfin's users/history, neo's data), plus every out-of-band secret. Vault was partly recovered from a May copy on the Mac. The fix for next time is the backup item in §17, not a better runbook.

The full WSL-era document is in git history (`git log -- SETUP.md`).

---

## 19. Useful references

- Tailscale admin: https://login.tailscale.com/admin/machines
- Tailscale docs: https://tailscale.com/kb
- k3s docs: https://docs.k3s.io/
- k3s + Tailscale (multi-node): https://docs.k3s.io/networking/distributed-multicloud
- Headlamp docs: https://headlamp.dev/docs/latest/
- Argo CD docs: https://argo-cd.readthedocs.io/en/stable/
- Argo CD App-of-Apps pattern: https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/
- MinIO docs: https://min.io/docs/minio/linux/index.html
- MinIO Client (`mc`): https://min.io/docs/minio/linux/reference/minio-mc.html
- Traefik (k3s bundled) docs: https://doc.traefik.io/traefik/
- Jellyfin docs: https://jellyfin.org/docs/
- Swiftfin (iOS client): https://github.com/jellyfin/Swiftfin

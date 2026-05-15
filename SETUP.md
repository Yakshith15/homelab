# Personal Homelab Setup

A complete reference for the homelab built across Mac + Windows + WSL2 Ubuntu, connected via Tailscale, hosting Docker workloads.

---

## 1. Goal

Use the Windows laptop as an always-on personal server running Docker containers (and eventually k3s/Kubernetes, terraform, media services). Access everything from:
- Mac (primary dev machine)
- Phone
- Anywhere on the internet (without exposing ports publicly)

Constraints we cared about:
- Shared hostel Wi-Fi → can't trust the LAN
- No public IP / port forwarding
- Free / personal-use tier

---

## 2. Architecture

```
  ┌─────────────┐         Tailscale tailnet         ┌─────────────────────┐
  │     Mac     │ ◄──────── (encrypted) ──────────► │  Windows laptop     │
  │  (client)   │                                   │  ┌───────────────┐  │
  └─────────────┘                                   │  │ WSL2 Ubuntu   │  │
         ▲                                          │  │ (`homelab`)   │  │
         │                                          │  │  - Tailscale  │  │
  ┌─────────────┐                                   │  │  - SSH server │  │
  │   Phone     │ ◄──────── (encrypted) ──────────► │  │  - Docker +   │  │
  │  (client)   │                                   │  │    Jellyfin   │  │
  └─────────────┘                                   │  │  - k3s        │  │
                                                    │  │    - Traefik  │  │
                                                    │  │    - vault    │  │
                                                    │  │    - headlamp │  │
                                                    │  └───────────────┘  │
                                                    └─────────────────────┘
```

Three Tailscale nodes total: Mac, Windows host, Ubuntu (inside WSL2). The Ubuntu node is the one running services — everything else just connects to it.

Why this design:
- **Tailscale** = private mesh VPN, no public exposure, works through NAT.
- **WSL2 Ubuntu** instead of native Windows Docker because k3s/Linux-native tools work properly there, and Docker performance is better.
- **Tailscale inside WSL** (not on the Windows host alone) so the Ubuntu node gets its own `100.x.x.x` IP — avoids WSL networking gymnastics.

---

## 3. Devices & IPs

| Device | Role | Tailscale IP | Login |
|---|---|---|---|
| Mac | Dev client | (check `tailscale status`) | macOS user |
| Windows host | Hypervisor + Tailscale relay | 100.82.126.74 | Windows user |
| Ubuntu (WSL) | Server (Docker, k3s, SSH) | **100.76.108.54** | `yakshith` |

Hostname inside WSL: `DESKTOP-6BCH3H9` (same as Windows host).

Tailscale machine name (MagicDNS): **`homelab`** (renamed in Tailscale admin from the original `desktop-6bch3h9-1`). Resolves on any tailnet device:
- Short: `http://homelab/`
- FQDN: `http://homelab.tailbed621.ts.net/`

SSH alias on Mac: `ssh wsl` → connects to Ubuntu.

---

## 4. Tailscale setup (done)

### On Windows
1. Installed from https://tailscale.com/download/windows
2. Signed in with main Google account
3. Tray icon shows "Connected"

### On Mac
1. Installed via Mac App Store
2. Signed in with **same** Google account
3. Approved "Add VPN Configuration" prompt (required, safe — Tailscale uses WireGuard end-to-end encrypted)
4. CLI symlink (because App Store version doesn't add `tailscale` to PATH):
   ```bash
   sudo ln -s /Applications/Tailscale.app/Contents/MacOS/Tailscale /usr/local/bin/tailscale
   ```

### On Ubuntu (WSL)
1. Install (after DNS fix below):
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   ```
2. Bring up:
   ```bash
   sudo tailscale up
   ```
   Paste the URL it prints into a browser, authenticate, return.
3. Auto-start on WSL boot (requires systemd, see WSL section):
   ```bash
   sudo systemctl enable tailscaled
   ```

### Admin console housekeeping
- https://login.tailscale.com/admin/machines
- **Enable MagicDNS** (DNS tab) — lets you use machine names instead of IPs
- **Disable key expiry** on each machine (`…` menu → Disable key expiry) so they don't get auto-logged-out every ~180 days

### Verify
```bash
tailscale status            # lists all peers
ping 100.76.108.54          # confirm tunnel works
```

---

## 5. WSL2 + Ubuntu setup

### Install
On Windows in **Admin PowerShell**:
```powershell
wsl --install -d Ubuntu
```
Reboot when prompted. Ubuntu terminal auto-opens after reboot → set UNIX username (`yakshith`) + password.

### Enable systemd + persistent DNS
Inside Ubuntu, write `/etc/wsl.conf`:
```bash
echo -e "[boot]\nsystemd=true\n\n[network]\ngenerateResolvConf = false" | sudo tee /etc/wsl.conf
```

Set DNS (WSL's default sometimes fails — happened twice during setup):
```bash
sudo rm /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

Apply changes: close Ubuntu terminal, then in Windows PowerShell:
```powershell
wsl --shutdown
```
Reopen Ubuntu. Verify:
```bash
systemctl --version          # systemd is alive
cat /etc/resolv.conf         # still shows 1.1.1.1
ping -c 2 google.com         # DNS works
```

### Why we chose WSL2 over native Windows OpenSSH
Tried Windows OpenSSH first; sshd service refused to start (error 1053). Even running `sshd.exe -d` manually worked for one connection, but the Windows service wrapper was broken. Switched to WSL2 because:
- It's the proper environment for Docker / k3s / terraform anyway
- Standard Linux OpenSSH "just works"
- Skills transfer 1:1 to a future dedicated Linux server

Tailscale SSH was considered but it doesn't run as a server on Windows hosts.

---

## 6. SSH setup

### On Ubuntu (server side)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh    # should be "active (running)"
```

### On Mac (client side)

Generate key pair (if no existing key at `~/.ssh/id_ed25519`):
```bash
ssh-keygen -t ed25519        # 3x Enter for defaults / no passphrase
```

Install public key on Ubuntu:
```bash
ssh-copy-id yakshith@100.76.108.54
```

Create alias in `~/.ssh/config`:
```
Host wsl
  HostName 100.76.108.54
  User yakshith
```

Now from anywhere:
```bash
ssh wsl
```

---

## 7. Docker setup (in Ubuntu)

Install via the official script:
```bash
curl -fsSL https://get.docker.com | sh
```

Add your user to the docker group so `sudo` isn't needed:
```bash
sudo usermod -aG docker $USER
```
**Then log out and back in** (group membership only kicks in on fresh login).

Verify:
```bash
docker --version
docker run --rm hello-world
```

Note: we **uninstalled Docker Desktop on Windows** before this — Docker Desktop conflicts with k3s and adds unnecessary indirection. Native Docker inside Ubuntu is the right path for our use case.

---

## 8. Portainer (REMOVED)

Originally installed as a Docker UI. **Removed** once everything moved into k3s — Portainer's k8s support is shallower than purpose-built k8s dashboards, and we no longer have a meaningful Docker workload outside k3s (Jellyfin is the only one left, managed via compose).

Replaced by **Headlamp** (in-cluster k8s dashboard) — see §11.2.

Teardown was just `docker stop portainer && docker rm portainer && docker volume rm portainer_data && docker rmi portainer/portainer-ce`.

---

## 9. Docker Compose (homelab service orchestration)

All services are now declared in a single `docker-compose.yml` file at `~/homelab/docker-compose.yml` (on the Ubuntu node). Profiles let us bring up subsets of services on demand.

### File location
- **On Ubuntu (live, what Docker reads):** `/home/yakshith/homelab/docker-compose.yml`
- **In this repo (version-controlled copy):** `docker-compose.yml` next to this file.

Keep them in sync — when you edit on one side, copy to the other. (Future improvement: clone this GitHub repo on the Ubuntu node directly, so there's only one source.)

### Profiles
Each service is tagged with one or more profiles. Currently:
- `portainer` → profiles `admin`, `all`
- `jellyfin` → profiles `media`, `all`

### Day-to-day commands

From `~/homelab/` on Ubuntu:

```bash
# Bring up everything
docker compose --profile all up -d

# Bring up just the media stack
docker compose --profile media up -d

# Bring up just admin tools
docker compose --profile admin up -d

# Stop everything (containers persist, just stopped)
docker compose stop

# Stop + remove containers (volumes/data stay)
docker compose down

# Stop + remove a single service
docker compose stop jellyfin
docker compose rm -f jellyfin

# Tail logs
docker compose logs -f jellyfin

# Validate yaml without running
docker compose --profile all config

# Recreate after editing the yaml
docker compose --profile all up -d --force-recreate
```

### Adding a new service

1. Add a new block under `services:` in `docker-compose.yml`.
2. Pick its profile(s) — `media`, `admin`, or a new one like `dev`, `monitoring`, etc.
3. `docker compose --profile <profile> up -d` to start it.
4. Commit + push the updated yaml.

### Migration history (one-time, done)
The original Portainer and Jellyfin containers were created with raw `docker run` commands. We migrated to compose by:
1. `docker stop portainer jellyfin && docker rm portainer jellyfin` (data preserved in named volume + bind mounts).
2. Wrote `docker-compose.yml` declaring both services. Used `external: true` on `portainer_data` volume to reuse the existing one.
3. `docker compose --profile all up -d`.
4. Verified both services came back with all data and settings intact.

---

## 10. Jellyfin (media streaming server)

A Docker container that serves your video library to phone, Mac, anywhere on the tailnet. Free, open source, no account/tracking.

### Prep
Verify the Windows drive folder is accessible from WSL:
```bash
ls /mnt/e/courses/masterclass
```
(WSL exposes Windows drives at `/mnt/<drive-letter>/`.)

Create config + cache directories on Linux side (Jellyfin database lives here — must NOT be on the Windows mount):
```bash
mkdir -p ~/jellyfin/config ~/jellyfin/cache
```

### Run
```bash
docker run -d \
  --name jellyfin \
  --restart=unless-stopped \
  -p 8096:8096 \
  -v ~/jellyfin/config:/config \
  -v ~/jellyfin/cache:/cache \
  -v /mnt/e/courses/masterclass:/media:ro \
  jellyfin/jellyfin
```

Flag breakdown:
- `-p 8096:8096` — Jellyfin web UI
- `-v ~/jellyfin/config:/config` — Jellyfin's database/settings (persistent)
- `-v ~/jellyfin/cache:/cache` — image/transcode cache
- `-v /mnt/e/courses/masterclass:/media:ro` — your video folder, **read-only** (`:ro`) so Jellyfin can't modify files
- `--restart=unless-stopped` — comes back after reboots / docker restarts

### First-run setup
Open `http://100.76.108.54:8096` in browser. Wizard:
1. Language: English, Server name: anything.
2. Create admin user.
3. Add media library:
   - Content type: **Home Videos** for courses (skips Movie/TV metadata matching, which won't find anything for course files).
   - Folder: `/media` (this is what the container sees — actually maps to `/mnt/e/courses/masterclass` outside).
4. Metadata language + country → English / India.
5. Remote access: leave **Allow remote connections** checked.

### Clients
Server is the same — clients differ:
- **Mac**: browser at `http://100.76.108.54:8096` (or Jellyfin Media Player app).
- **iOS**: **Swiftfin** (free, recommended) or **Infuse** (paid, fancier) or **Jellyfin Mobile** (official, basic).
- **Android**: Findroid (free, native) or official Jellyfin app.

In any client: Add Server → URL `http://100.76.108.54:8096` → sign in.

### Adding more media later
Just add another `-v` mount. E.g. for movies on D drive:
```bash
docker stop jellyfin && docker rm jellyfin
# then re-run docker run with an additional: -v /mnt/d/Movies:/movies:ro
```
Then in Jellyfin web UI: Dashboard → Libraries → Add Media Library → folder `/movies`.

Or (cleaner) migrate to docker-compose so changes don't require recreating containers manually.

---

## 11. k3s (lightweight Kubernetes)

k3s is a single-binary Kubernetes distribution that ships in ~70 MB and runs the entire control plane + worker on one node. It's the standard choice for homelab/edge clusters. We installed it inside the WSL2 Ubuntu node.

### Architecture (current)
- **One node:** the WSL Ubuntu instance is both control plane AND worker (k3s untaints control plane to allow scheduling workloads on it).
- **Bundled components:** containerd (runtime), Traefik (ingress), CoreDNS, ServiceLB, local-path-provisioner (default StorageClass), metrics-server.
- **Datastore:** SQLite (not etcd) — fine for single-node, would switch to etcd if going multi-node HA.

### Install (one-time, done)
```bash
curl -sfL https://get.k3s.io | sh -
```
Auto-creates `k3s.service` in systemd, starts it, configures kubectl symlink at `/usr/local/bin/kubectl`. Auto-starts on WSL boot (`systemctl is-enabled k3s` → `enabled`).

### kubectl without sudo (one-time)
```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
```

The k3s-bundled `kubectl` defaults to reading `/etc/rancher/k3s/k3s.yaml` (root-owned) instead of `~/.kube/config`, even when no `$KUBECONFIG` env var is set. Override it explicitly:
```bash
export KUBECONFIG=$HOME/.kube/config
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
```

After this, plain `kubectl get nodes` works as your user.

### Day-to-day commands
```bash
kubectl get nodes                       # cluster nodes
kubectl get pods -A                     # all pods, all namespaces
kubectl get pods -n kube-system         # system pods only
kubectl get svc -A                      # all services
kubectl describe pod <pod> -n <ns>      # pod details + events (debug)
kubectl logs <pod> -n <ns>              # pod logs
kubectl logs -f <pod> -n <ns>           # follow logs
kubectl apply -f <manifest.yaml>        # create/update resources
kubectl delete -f <manifest.yaml>       # delete resources
kubectl top nodes                       # node CPU/RAM (needs metrics-server, included)
kubectl top pods -A                     # pod CPU/RAM
```

### Service operations
```bash
sudo systemctl status k3s               # daemon state
sudo systemctl restart k3s              # restart cluster
sudo systemctl stop k3s                 # stop cluster (containers stop with it)
```

### Resource footprint (idle)
- k3s daemon: ~700 MB–1.4 GB RAM (varies, settles after first few minutes)
- ~6 system pods running by default
- Plenty of headroom on a 5 GB WSL allocation for Argo CD + a handful of small apps

### Why k3s vs alternatives
- **vs minikube/KIND:** those are dev-focused, designed to be torn down. k3s is production-grade (used in real edge deployments), runs in systemd, persists.
- **vs full kubeadm Kubernetes:** k3s is one binary; kubeadm needs ~6 services. Same Kubernetes API, much less ops burden.

### TODO
- [x] **First app deploy** — done. Vault (FastAPI + Next.js + SQLite + Gemini) deployed to k3s. See §11.1.
- [x] **Ingress setup** — done. Traefik routes `/api/*` → backend, `/*` → frontend, `/headlamp/*` → headlamp.
- [x] **PersistentVolumeClaims** — done. `vault-data` PVC (2 Gi, local-path) holds vault's SQLite + content directory at `/var/lib/rancher/k3s/storage/`.
- [x] **k8s dashboard** — done. Headlamp at `http://homelab/headlamp/`. See §11.2.
- [ ] **Install Argo CD** for GitOps (push to GitHub → auto-deploy to cluster)
- [ ] **HTTPS via Tailscale Serve** — deferred (Tailscale already encrypts at network layer). Path documented in `k8s/vault/README.md` if/when needed.
- [ ] **Backup strategy** for the `vault-data` PVC (rsync host-path to NAS or S3)

---

## 11.1. Vault (deployed app)

Personal knowledge-vault app — FastAPI backend + Next.js frontend + SQLite + Claude/Gemini analysis. Lives at `vault/` in [a separate repo](https://github.com/Yakshith15/vault); deployed to this cluster from manifests in `k8s/vault/`.

URL: <http://homelab/>

Manifests + ops docs (deploy, scale, rolling, data migration, env updates, teardown) live in [`k8s/vault/README.md`](k8s/vault/README.md).

Key facts:
- Namespace: `vault`
- Image source: GHCR (`ghcr.io/yakshith15/vault-backend:latest`, `ghcr.io/yakshith15/vault-frontend:latest`) — built by GHA on every push to vault repo `main`.
- Frontend `NEXT_PUBLIC_API_URL=/api` (relative URL → same-origin → no CORS issues regardless of which hostname the user hits).
- New images picked up via `kubectl -n vault rollout restart deploy/vault-frontend` (or backend).
- Stop/start workflow: scale to 0 / 1 either via `kubectl -n vault scale deploy --all --replicas=0/1` or click in Headlamp.

---

## 11.2. Headlamp (k8s dashboard)

In-cluster web UI for managing the cluster — scale, logs, exec, edit manifests, etc. Replaced Portainer once everything moved into k3s.

URL: <http://homelab/headlamp/>

Manifests + ops docs (deploy, token generation, daily-use shortcuts, teardown) live in [`k8s/headlamp/README.md`](k8s/headlamp/README.md).

Key facts:
- Namespace: `headlamp`
- ServiceAccount bound to `cluster-admin` (single-user homelab; scope down if more users are added).
- Login: Bearer token. Mint with `kubectl -n headlamp create token headlamp --duration=8760h` (1-year token).
- `-base-url=/headlamp` arg on the deployment lets it serve cleanly behind the `/headlamp/` ingress path (no stripPrefix middleware needed).

---

## 12. Windows power settings (always-on server mode)

Goal: laptop keeps running when lid is closed and never sleeps (when plugged in), so the homelab is genuinely always-on.

### Lid close behavior
1. Win → search **Control Panel** → open.
2. **Hardware and Sound** → **Power Options**.
3. Left sidebar → **Choose what closing the lid does**.
4. Set **When I close the lid + Plugged in** → **Do nothing**.
5. Leave **On battery** as **Sleep** (battery acts as a safety net; if power dies, laptop sleeps gracefully instead of running until battery dead).
6. **Save changes**.

### Sleep timer
1. Settings → **System** → **Power & battery** → **Screen and sleep**.
2. **When plugged in, put my device to sleep after** → **Never**.
3. Leave the screen-off timer alone (just blanks the display, doesn't affect services).
4. Leave battery sleep timer alone (e.g. 10 min).

Result: laptop runs full speed with lid closed, plugged in. Closes the "Tailscale offline after sleep" issue (see gotchas).

---

## 13. Auto-start on Windows boot

Goal: when Windows boots, WSL spins up automatically → systemd starts → tailscaled + ssh + docker containers all come online — no manual intervention.

### Task Scheduler config
Created scheduled task: **Start WSL on logon**

- **Triggers**: At log on (of your user)
- **Action**:
  - Program: `wsl.exe`
  - Arguments: `-d Ubuntu -u root sleep infinity`
- **Conditions**: unchecked "Start only on AC power"
- **Settings**: checked "Run task as soon as possible after a scheduled start is missed"

Why `sleep infinity`: WSL shuts down when no processes are running. The sleep keeps it alive indefinitely.

### Verify
After Windows login, give it ~30 sec, then on Mac:
```bash
ssh wsl
```
Should connect with no Windows interaction needed.

If it doesn't:
```powershell
wsl --list --running          # is Ubuntu running?
```

---

## 14. Common commands cheatsheet

### From Mac

```bash
# Connect to Ubuntu
ssh wsl

# Tailscale status
tailscale status

# Test connectivity
ping 100.76.108.54
```

### On Ubuntu (inside SSH session)

```bash
# Tailscale
tailscale status                      # peers + connection state
tailscale ip -4                       # this node's IP
sudo systemctl status tailscaled

# SSH server
sudo systemctl status ssh

# Docker basics
docker ps                             # running containers
docker ps -a                          # all containers (incl. stopped)
docker images                         # images on disk
docker logs <name>                    # tail container logs
docker logs -f <name>                 # follow logs
docker stop <name>                    # stop container
docker rm <name>                      # remove stopped container
docker rm -f <name>                   # force-remove running container
docker exec -it <name> bash           # shell inside container

# Cleanup
docker container prune                # remove all stopped containers
docker image prune                    # remove dangling images
docker system prune -a                # nuke everything unused (careful)
```

### On Windows PowerShell

```powershell
wsl --list --running                  # which distros are up
wsl --shutdown                        # stop ALL WSL distros
wsl --list --verbose                  # all distros + their state
```

---

## 15. Things we learned (gotchas)

### Tailscale CLI not in PATH on Mac (App Store install)
Symlink to `/usr/local/bin/tailscale` — see Tailscale section.

### WSL DNS keeps breaking
WSL regenerates `/etc/resolv.conf` on every restart by default. The `[network] generateResolvConf = false` line in `/etc/wsl.conf` prevents this. Without it, DNS dies after every reboot.

### Systemd not enabled by default in WSL
Newer WSL2 supports systemd but it's off by default. Without it, `systemctl` commands fail with "System has not been booted with systemd as init system". Fix: `[boot] systemd=true` in `/etc/wsl.conf` + `wsl --shutdown`.

### WSL shuts down when idle
Even after `wsl --shutdown` for the systemd config, WSL also auto-shuts when no processes run inside it (~60s idle). Task Scheduler with `sleep infinity` keeps it alive.

### `ssh-copy-id` for passwordless SSH
After this, no more password prompts — uses your ed25519 key pair.

### Tailscale "idle" vs "offline"
- **idle** = peer reachable, no active traffic → fine
- **offline** = peer unreachable → check the peer's Tailscale state
- **active** = traffic flowing right now

### Docker Desktop vs native Docker
Docker Desktop on Windows uses WSL2 under the hood anyway; using native Docker inside Ubuntu cuts out the middleware and makes k3s work properly.

### Windows OpenSSH (skipped)
Tried installing OpenSSH Server on Windows. Service wouldn't start (error 1053 — host keys, permissions, none of the standard fixes worked). Pivoted to WSL2 instead. Don't go back down this path unless something forces it.

### Tailscale "offline" inside WSL after Windows sleep
**Symptom:** `tailscale status` from Ubuntu shows the Linux node as `offline` (sometimes even with `# Health check: Unable to connect to the Tailscale coordination server` warning). Daemon is `active (running)` per `systemctl status tailscaled`, but log shows repeated:
```
control: map response long-poll timed out!
control: lite map update error after 2m0.002s: Post "https://controlplane.tailscale.com/m..."
```

**Cause:** When Windows sleeps, WSL's network gets severed. `tailscaled` keeps trying to reuse a dead HTTPS connection to the control plane.

**Fix:**
```bash
sudo systemctl restart tailscaled
```
Daemon reconnects cleanly within a few seconds.

**Permanent fix:** done — disabled Windows sleep when plugged in (section 11). With Windows always awake, the issue doesn't trigger.

### Compose profiles affect EVERY subcommand
If you tag services with `profiles: [...]`, then `docker compose <subcommand>` without `--profile <name>` ignores those services entirely. Affects: `up`, `down`, `stop`, `ps`, `config`, `logs`, etc.

**Symptom:** `docker compose down` returns silently but `docker ps` still shows containers running.

**Fix:** always pass the profile flag (or use `--profile all` to act on everything):
```bash
docker compose --profile all down
docker compose --profile all ps
docker compose --profile all logs -f
```

Workaround for ad-hoc operations on a single service: name it explicitly — profile filtering is skipped when you name services. E.g. `docker compose up -d portainer`, `docker compose logs jellyfin`.

---

## 16. Resume / restart procedure

After both laptops have been off:

1. Boot Windows → login. Wait ~30 sec for Task Scheduler to spin up WSL + Tailscale.
2. On Mac, ensure Tailscale is connected (menu bar icon).
3. `ssh wsl` from Mac → should drop straight in, no password.

If `ssh wsl` hangs:
- Check `tailscale status` on Mac — is the Ubuntu node listed and idle/active (not offline)?
- If offline: on Windows, open Ubuntu terminal manually to wake it up.
- If still offline: in Ubuntu, `sudo systemctl status tailscaled`. Restart if needed: `sudo systemctl restart tailscaled && sudo tailscale up`.

---

## 16. Next steps (TODO)

In rough order of priority:

- [x] **Windows power settings** — done. Lid close = "Do nothing" (plugged in), sleep timer = Never (plugged in).
- [x] **Tailscale on mobile** — done. Phone is on the tailnet.
- [x] **First real service: Jellyfin** — done. Streaming course videos from E drive to Mac + iPhone (via Swiftfin).
- [x] **Docker Compose** — done. Both Portainer and Jellyfin now managed via `~/homelab/docker-compose.yml` with profiles (`admin`, `media`, `all`). YAML version-controlled in this repo.
- [x] **Bump WSL RAM allocation** — done. `C:\Users\<you>\.wslconfig` now contains:
  ```ini
  [wsl2]
  memory=5GB
  processors=4
  swap=4GB
  ```
  Host has 8 GB total → WSL gets 5 GB, Windows keeps ~3 GB. Bigger swap (4 GB) compensates for tight RAM. Apply with `wsl --shutdown` then reopen Ubuntu. Verify with `free -h` (should show ~5 GB total, 4 GB swap).
- [x] **k3s** — done. Single-node cluster (control plane + worker on same node), v1.35.4+k3s1. See §11.
- [x] **First k3s app** — done. Vault deployed (§11.1) end-to-end via manifests in `k8s/vault/`.
- [x] **k3s dashboard** — done. Headlamp at `http://homelab/headlamp/` (§11.2).
- [x] **Friendly hostname** — done. Tailscale machine renamed to `homelab` → reachable at `http://homelab/` from any tailnet device.
- [ ] **More services** — pick based on need:
  - Vaultwarden (password manager)
  - Homepage (dashboard)
  - Gitea (self-hosted git)
  - *arr stack (Sonarr/Radarr) if you want auto-organized media
- [ ] **Argo CD** — install in cluster for GitOps deploys (push to GitHub → auto-deploy to k3s). Worth it once we have 3+ services.
- [ ] **HTTPS via Tailscale Serve** — deferred; Tailscale already encrypts at network layer. Steps documented in `k8s/vault/README.md`.
- [ ] **Persist WSL DNS fix** — set `[network] generateResolvConf = false` AND `tailscale set --accept-dns=false` so manual nameservers in `/etc/resolv.conf` survive sleep/resume + WSL restarts. Currently fixed manually each time it bites.
- [ ] **Backup strategy** for the `vault-data` PVC (rsync host-path to NAS or S3) and any other PVCs added later.
- [ ] **(Eventual) dual-boot Linux** — see section 17 for the full migration plan.

---

## 17. Future: Dual-boot Windows + Linux migration plan

When WSL2 hits its limits (5 GB RAM ceiling, k3s + many services, Windows update flakiness), the plan is to **dual-boot** the laptop with Windows and Ubuntu side by side. Boot into Linux for the homelab, boot into Windows when needed for Windows-only stuff. GRUB handles the boot menu.

### Critical tradeoff: only one OS runs at a time
Dual boot is **not** "two OSes at once" — that would be virtualization. It's "two OSes installed, choose one at boot time." Implications:

- **Booted into Linux** → homelab is up, Windows is off.
- **Booted into Windows** → homelab is **down** (no containers running, Tailscale node offline).
- **Switching = reboot.** No live switching.

So: only worth dual-booting if you genuinely need Windows occasionally. If you don't, full wipe is cleaner. The decision rule:
- "I'll boot Windows < 5% of the time, only for one specific app" → dual-boot is fine
- "I might need Windows for X someday" → still go full wipe; reinstall Windows in a VM later if you actually need it
- "I'm not sure" → stay on WSL2 longer

### Why dual-boot vs full wipe
| | Dual-boot | Full wipe |
|---|---|---|
| Linux RAM | All 8 GB | All 8 GB |
| Windows still available | Yes | No (would need reinstall) |
| Disk space for Linux | Whatever you allocate (~half) | Entire disk |
| Always-on homelab | Only when booted into Linux | Always |
| Setup complexity | Higher (partitioning, GRUB) | Lower (just install) |
| Windows update breaking GRUB | Possible (annoying but fixable) | N/A |

### About the masterclass folder (good news)
With dual boot, `E:\courses\masterclass` **does not need backing up**. The E: partition is separate from the Windows C: partition. As long as you do **manual partitioning** during Ubuntu install (don't let it auto-resize everything), E: stays intact.

From Linux you can mount the NTFS E: partition and read the files directly:
```bash
sudo mkdir -p /mnt/e-drive
sudo mount -t ntfs-3g /dev/nvme0n1p? /mnt/e-drive   # ? = partition number, find with lsblk
```
Then in `docker-compose.yml`, change Jellyfin's mount from `/mnt/e/courses/masterclass:/media:ro` to `/mnt/e-drive/courses/masterclass:/media:ro`. Same files, no copy.

To make the mount permanent, add to `/etc/fstab`:
```
UUID=<get-via-blkid>  /mnt/e-drive  ntfs-3g  defaults,uid=1000,gid=1000  0  0
```

### Pre-install checklist (do ALL before booting the installer)

1. **Back up Jellyfin config from WSL** — settings, library, watch history. WSL's filesystem isn't accessible from bare-metal Linux:
   ```bash
   tar czf ~/jellyfin-backup.tar.gz ~/jellyfin/config
   scp ~/jellyfin-backup.tar.gz mac:~/Desktop/
   ```
2. **Back up Portainer Docker volume** — same reason:
   ```bash
   docker run --rm -v portainer_data:/data -v $HOME:/backup alpine tar czf /backup/portainer-backup.tar.gz -C /data .
   scp ~/portainer-backup.tar.gz mac:~/Desktop/
   ```
3. **Confirm this repo is pushed clean** — `cd ~/Desktop/claude/projects/homelab && git status`. The `docker-compose.yml` is the rebuild spec.
4. **Defragment Windows C: drive** — Win+R → `dfrgui` → optimize C:. Helps the Ubuntu installer shrink the partition without errors.
5. **Disable Windows Fast Startup** — Control Panel → Power Options → "Choose what the power buttons do" → uncheck "Turn on fast startup." Otherwise Windows leaves the disk in a half-hibernated state and Linux can't safely mount NTFS partitions.
6. **Disable BitLocker on C: AND E:** if enabled — Settings → Privacy & Security → Device encryption. Linux can't read encrypted Windows partitions. (Probably not enabled on your laptop, but check.)
7. **Note your current Windows partition layout** — Win+R → `diskmgmt.msc`, screenshot. Useful if anything goes wrong.
8. **Decide partition split** — recommend ~60–80 GB for Ubuntu root (`/`), the rest stays Windows. Leave E: alone entirely.
9. **Make a Ubuntu Server 24.04 LTS USB** — download ISO from ubuntu.com, flash with Rufus (on Windows) or balenaEtcher.
10. **In BIOS/UEFI: confirm Secure Boot is OFF or that you'll enable shim signing.** Ubuntu's installer handles signed Secure Boot fine, but disabling avoids surprises.
11. **Save Windows recovery key** (BitLocker recovery, account.microsoft.com) just in case.

### Install steps (dual-boot specific)

1. **Boot from USB** — F12 / F2 / Esc at startup (varies by laptop) to pick boot device.
2. **Choose "Try or Install Ubuntu Server"**.
3. **Network setup** — connect to Wi-Fi from the installer.
4. **Storage layout — CRITICAL STEP**:
   - Pick **"Custom storage layout"** (NOT "Use entire disk").
   - You'll see existing partitions: `nvme0n1p1` (EFI), `nvme0n1p2` (Microsoft reserved), `nvme0n1p3` (Windows C:), `nvme0n1p4` (recovery), and likely `nvme0n1p?` for E:.
   - **Shrink the Windows C: partition** (`p3`) to free space. Right-click → resize → leave ~half for Windows, free up ~80–100 GB.
   - In the freed space, create:
     - **`/` (root)** — ext4, ~60–80 GB
     - **swap** — 4 GB (matches your current WSL swap)
   - Mount the **existing EFI partition** (`p1`, ~100–500 MB) at `/boot/efi` — **do NOT format it**. GRUB goes here alongside the Windows boot manager.
   - Leave E: partition completely untouched.
5. **Username**: `yakshith`. **Hostname**: e.g. `homelab`.
6. **OpenSSH server: yes**. (Lets you ssh in immediately.)
7. Let installer finish (~20 min). Reboot, remove USB.
8. **First boot — GRUB menu appears** with two entries:
   - Ubuntu (default)
   - Windows Boot Manager
   Pick Ubuntu. Test Windows boots later.

### Post-install setup (in Ubuntu)

1. **Get on Wi-Fi** — `nmtui` for a TUI picker if not already connected.
2. **Find LAN IP**: `ip a`. SSH from Mac one-time:
   ```bash
   ssh yakshith@<lan-ip>
   ```
3. **Update**: `sudo apt update && sudo apt upgrade -y`.
4. **Install ntfs-3g for E: drive access:**
   ```bash
   sudo apt install -y ntfs-3g
   sudo mkdir -p /mnt/e-drive
   lsblk -f                       # find E: partition's UUID
   sudo blkid                     # alternative
   ```
   Add to `/etc/fstab`:
   ```
   UUID=<the-uuid>  /mnt/e-drive  ntfs-3g  defaults,uid=1000,gid=1000,umask=022  0  0
   ```
   Mount: `sudo mount -a`. Verify: `ls /mnt/e-drive/courses/masterclass`.
5. **Install Tailscale**:
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
   Auth in browser. Note new Tailscale IP.
6. **Update Mac `~/.ssh/config`** — point `Host wsl` (or rename to `Host homelab`) at the new Tailscale IP.
7. **Copy SSH key**: `ssh-copy-id yakshith@<new-tailscale-ip>` from Mac.
8. **In Tailscale admin** — disable the old WSL node and the Windows host node (won't come back when you boot Linux). Re-enable them only on the rare reboots into Windows.
9. **Install Docker**:
   ```bash
   curl -fsSL https://get.docker.com | sh
   sudo usermod -aG docker $USER
   ```
   Log out / back in.
10. **Clone repo**:
    ```bash
    git clone https://github.com/Yakshith15/homelab.git ~/homelab
    cd ~/homelab
    ```
11. **Update `docker-compose.yml`** Jellyfin volume:
    `/mnt/e/courses/masterclass:/media:ro` → `/mnt/e-drive/courses/masterclass:/media:ro`. Commit + push.
12. **Restore Jellyfin config**:
    ```bash
    mkdir -p ~/jellyfin/cache
    tar xzf ~/jellyfin-backup.tar.gz -C ~
    ```
13. **Restore Portainer volume**:
    ```bash
    docker volume create portainer_data
    docker run --rm -v portainer_data:/data -v $HOME:/backup alpine sh -c "cd /data && tar xzf /backup/portainer-backup.tar.gz"
    ```
14. **Bring services up**:
    ```bash
    cd ~/homelab
    docker compose --profile all up -d
    ```
15. **Re-install k3s** (same one-liner, see k3s section). YAML manifests in git apply cleanly.
16. **Lid-close behavior** — for a closed-lid server, edit `/etc/systemd/logind.conf`:
    ```
    HandleLidSwitch=ignore
    HandleLidSwitchExternalPower=ignore
    ```
    Then `sudo systemctl restart systemd-logind`.
17. **Default boot to Ubuntu** — already the GRUB default unless you changed it. To boot Windows occasionally, just pick it from GRUB menu at startup.
18. **Update SETUP.md** — replace WSL sections with bare-metal Ubuntu. Update Tailscale IP. Add this dual-boot context.

### Dual-boot gotchas to watch for

- **Windows updates can overwrite GRUB** — symptom: laptop boots straight into Windows, no menu. Fix: boot Ubuntu USB in "Try" mode, run `boot-repair` (or `sudo grub-install` + `sudo update-grub`).
- **Windows clock is wrong after booting back to Windows** — Linux uses UTC for hardware clock, Windows uses local time, they fight. Fix in Linux: `sudo timedatectl set-local-rtc 1 --adjust-system-clock`.
- **NTFS partition shows as read-only** — usually means Windows didn't shut down cleanly (Fast Startup left it in hibernation). Boot Windows, properly shut down (Shift+click Shut Down), then back to Linux.
- **Booting into Windows = homelab is OFF** — for ~30 min of Windows usage, you take the homelab down. Plan accordingly.

### What you DON'T need to redo
- Tailscale account
- Mac SSH config (just update IPs)
- Phone Tailscale + Swiftfin (just update server URL)
- The `docker-compose.yml` itself (only one path string changes)
- GitHub repo

### Estimated downtime
- Backup: ~30 min
- Install: ~45 min (incl. partitioning carefully)
- Tailscale + Docker + restore: ~30 min
- Total: **~2 hours** (no media copy needed since E: is preserved)

### When to actually pull the trigger
- WSL2 RAM ceiling becomes a real problem (you'll see OOM kills on pods, sluggish containers)
- You want monitoring (Prometheus + Grafana adds ~1 GB by itself)
- Windows update breaks WSL/Tailscale (it happens)
- You're comfortable enough with Linux to live in it primarily

Until then, WSL2 is fine.

---

## 18. Useful references

- Tailscale admin: https://login.tailscale.com/admin/machines
- Tailscale docs: https://tailscale.com/kb
- WSL docs: https://learn.microsoft.com/en-us/windows/wsl/
- Docker docs: https://docs.docker.com/
- k3s docs: https://docs.k3s.io/
- Headlamp docs: https://headlamp.dev/docs/latest/
- Traefik (k3s bundled) docs: https://doc.traefik.io/traefik/
- Jellyfin docs: https://jellyfin.org/docs/
- Swiftfin (iOS client): https://github.com/jellyfin/Swiftfin

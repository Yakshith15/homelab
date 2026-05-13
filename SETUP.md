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
         ▲                                          │  │  - Tailscale  │  │
         │                                          │  │  - SSH server │  │
  ┌─────────────┐                                   │  │  - Docker     │  │
  │   Phone     │ ◄──────── (encrypted) ──────────► │  │  - Portainer  │  │
  │  (client)   │                                   │  │  - Jellyfin   │  │
  └─────────────┘                                   │  └───────────────┘  │
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
| Ubuntu (WSL) | Server (Docker, SSH) | **100.76.108.54** | `yakshith` |

Hostname inside WSL: `DESKTOP-6BCH3H9` (same as Windows host).

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

## 8. Portainer (web UI for Docker)

One-time install:
```bash
docker volume create portainer_data
docker run -d -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access from Mac browser: **https://100.76.108.54:9443** (HTTPS, not HTTP — accept self-signed cert warning).

Set admin user/password on first visit. Click into the **local** environment to manage everything.

`--restart=always` means it auto-comes-back after reboots.

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

## 11. Windows power settings (always-on server mode)

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

## 12. Auto-start on Windows boot

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

## 13. Common commands cheatsheet

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

## 14. Things we learned (gotchas)

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

## 15. Resume / restart procedure

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
- [ ] **More services** — pick based on need:
  - Vaultwarden (password manager)
  - Homepage (dashboard)
  - Gitea (self-hosted git)
  - *arr stack (Sonarr/Radarr) if you want auto-organized media
- [ ] **k3s** — lightweight Kubernetes. `curl -sfL https://get.k3s.io | sh -` inside Ubuntu.
- [ ] **Reverse proxy** (Caddy or Traefik) — clean URLs (`jellyfin.local`) instead of `100.x.x.x:8096`.
- [ ] **Backup strategy** for persistent Docker volumes.
- [ ] **(Eventual) bare-metal Linux** — wipe Windows, install Ubuntu Server. Everything we built here transfers 1:1 (same compose files, same configs).

---

## 17. Future: Bare-metal Linux migration plan

When the Windows + WSL2 setup hits its limits (RAM ceiling, k3s + many services, wanting full Linux control), the long-term plan is to wipe Windows and install Ubuntu Server directly on the laptop. Everything we built transfers over — same compose files, same Tailscale IPs (mostly), same workflow.

### Why migrate eventually
- Full 8 GB RAM available to the OS (no Windows overhead, no WSL cap)
- No `wsl --shutdown` quirks, no Windows updates rebooting your server
- `/mnt/e/...` paths become real Linux paths (faster, no NTFS translation)
- Closer to "real" production environments — better for learning

### Pre-wipe checklist (CRITICAL — do all before formatting)

1. **Back up media** — `E:\courses\masterclass` (and any other course/video folders). After the wipe, the NTFS partition is gone forever.
   - Option A: external USB drive (fastest for ~50+ GB of video)
   - Option B: copy to Mac over Tailscale: `scp -r yakshith@100.76.108.54:/mnt/e/courses/masterclass ~/Desktop/masterclass-backup`
   - Option C: cloud (Google Drive / OneDrive) — slow on hostel Wi-Fi, last resort
2. **Back up Jellyfin config** — `~/jellyfin/config` inside WSL has all your library settings, watch history, users:
   ```bash
   tar czf ~/jellyfin-backup.tar.gz ~/jellyfin/config
   scp ~/jellyfin-backup.tar.gz mac:~/Desktop/
   ```
3. **Back up Portainer data** — the `portainer_data` Docker volume:
   ```bash
   docker run --rm -v portainer_data:/data -v $HOME:/backup alpine tar czf /backup/portainer-backup.tar.gz -C /data .
   scp ~/portainer-backup.tar.gz mac:~/Desktop/
   ```
4. **Confirm this repo is pushed** — `cd ~/Desktop/claude/projects/homelab && git status` should show clean. The `docker-compose.yml` and `SETUP.md` are the source of truth for rebuilding services.
5. **Note current Tailscale IP** of the WSL node (`100.76.108.54`). After migration the new bare-metal node gets a fresh IP — update the SSH alias on Mac and any bookmarks.
6. **Export any Docker volumes you care about** that aren't already covered (e.g. databases for future services).
7. **Save Windows product key** (run `wmic path softwarelicensingservice get OA3xOriginalProductKey`) in case you ever want to dual-boot or VM Windows back.
8. **Personal files** — Documents, Downloads, Desktop, browser bookmarks, anything in `C:\Users\<you>\`. The Windows "Reset this PC → Remove everything" still wipes the user folder.

### Steps after bootloading Ubuntu Server

Order matters — each step depends on the previous.

1. **Install Ubuntu Server 24.04 LTS** (or whatever's current LTS) from a USB. Pick:
   - OpenSSH server: **yes** (lets you ssh in immediately from Mac via local Wi-Fi)
   - Username: `yakshith` (same as WSL — keeps muscle memory)
   - Disk: use the whole disk (no dual-boot — the point is to free all resources for Linux)
2. **Boot into Ubuntu, log in locally once** to confirm it works.
3. **Get on Wi-Fi** — `nmtui` for a TUI Wi-Fi picker if not already connected.
4. **SSH from Mac over local Wi-Fi** to the laptop's LAN IP (one-time, before Tailscale):
   ```bash
   ssh yakshith@<lan-ip>   # find via `ip a` on the laptop
   ```
5. **Install Tailscale** (same as WSL section):
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
   Auth in browser. Note the new Tailscale IP (`tailscale ip -4`).
6. **Update Mac SSH config** — change `~/.ssh/config` `Host wsl` block to point at the new Tailscale IP. Optionally rename the alias from `wsl` to `homelab` (no more WSL involved).
7. **Copy SSH key over** (or re-run `ssh-copy-id`) for passwordless access.
8. **Disable old WSL node in Tailscale admin** (https://login.tailscale.com/admin/machines) — won't come back, just clutters the list. Also disable the Windows host node.
9. **Install Docker** (same one-liner):
   ```bash
   curl -fsSL https://get.docker.com | sh
   sudo usermod -aG docker $USER
   ```
   Log out and back in.
10. **Clone this repo:**
    ```bash
    git clone https://github.com/Yakshith15/homelab.git ~/homelab
    cd ~/homelab
    ```
    Now the compose file lives natively on Linux — no more "keep WSL copy and repo copy in sync."
11. **Restore media** — copy the masterclass backup to a sensible Linux path (e.g. `/home/yakshith/media/masterclass`). Update `docker-compose.yml` jellyfin volume mount from `/mnt/e/courses/masterclass:/media:ro` to `/home/yakshith/media/masterclass:/media:ro`. Commit + push.
12. **Restore Jellyfin config:**
    ```bash
    mkdir -p ~/jellyfin/cache
    tar xzf ~/jellyfin-backup.tar.gz -C ~  # restores ~/jellyfin/config
    ```
13. **Restore Portainer volume:**
    ```bash
    docker volume create portainer_data
    docker run --rm -v portainer_data:/data -v $HOME:/backup alpine sh -c "cd /data && tar xzf /backup/portainer-backup.tar.gz"
    ```
14. **Bring services up:**
    ```bash
    cd ~/homelab
    docker compose --profile all up -d
    ```
    Verify Portainer at `https://<new-tailscale-ip>:9443` and Jellyfin at `http://<new-tailscale-ip>:8096`.
15. **Re-install k3s** (same one-liner, see k3s section once it's added). All your YAML manifests in git apply cleanly to the new cluster — `kubectl apply -f` and you're back.
16. **Set up auto-start** — Ubuntu Server systemd already auto-starts Docker, Tailscale, k3s on boot. Nothing to configure (unlike WSL Task Scheduler hacks).
17. **Power management** — for a closed-lid server, edit `/etc/systemd/logind.conf`:
    ```
    HandleLidSwitch=ignore
    HandleLidSwitchExternalPower=ignore
    ```
    Then `sudo systemctl restart systemd-logind`. Equivalent to the Windows power settings step but cleaner.
18. **Update SETUP.md** to reflect the new architecture (no more WSL section, new Tailscale IP, bare-metal commands instead of WSL-specific ones).

### What you DON'T need to redo
- Tailscale account + machines (just add new node, remove old)
- Mac side setup (ssh key, alias — just update IPs)
- Phone Tailscale + Swiftfin (just update server URL in Swiftfin)
- The `docker-compose.yml` (works as-is, only paths change)
- GitHub repo connections

### Estimated downtime
~2–4 hours total (Ubuntu install ~30 min, Tailscale + Docker ~15 min, restoring data ~varies by media size, reconfig ~1 hr).

### When to actually pull the trigger
- You're consistently bumping into the 5 GB WSL RAM limit
- You want to add monitoring (Prometheus + Grafana) — they alone use 1+ GB
- A Windows update breaks WSL or Tailscale (it happens)
- You no longer need Windows for anything else on this laptop

Until then, the WSL setup is fine for learning.

---

## 18. Useful references

- Tailscale admin: https://login.tailscale.com/admin/machines
- Tailscale docs: https://tailscale.com/kb
- WSL docs: https://learn.microsoft.com/en-us/windows/wsl/
- Docker docs: https://docs.docker.com/
- Portainer docs: https://docs.portainer.io/
- Jellyfin docs: https://jellyfin.org/docs/
- Swiftfin (iOS client): https://github.com/jellyfin/Swiftfin

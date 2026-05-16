# Argo CD Reference (Homelab)

Standalone, comprehensive reference for everything Argo CD in this homelab. If everything else burned to the ground and only this file survived, this is enough to rebuild it.

Target audience: someone who knows Kubernetes basics. Practical, terse, copy-pasteable commands.

Repo: `Yakshith15/homelab` — Argo manifests live under `k8s/argocd/`.

---

## 1. What's running

| Component | Where | Why |
|---|---|---|
| Argo CD core (server, repo-server, application-controller, redis, dex) | `argocd` ns, upstream install | GitOps controller — watches this repo, reconciles cluster |
| `argocd-server-lb` Service | `argocd` ns, our manifest | Exposes the UI on `:8090` over the tailnet |
| `argocd-cmd-params-cm` ConfigMap | `argocd` ns, our manifest | Forces `argocd-server` into `--insecure` mode (plain HTTP) |
| Argo CD Image Updater | `argocd` ns, upstream install | Polls GHCR for new image digests, writes them back to git |
| `ImageUpdater` CR `vault` | `argocd` ns, our manifest | Tells the updater which Application to watch |

URL: <http://homelab:8090> (over Tailscale).

---

## 2. Architecture overview

### App-of-Apps pattern

```
                root.yaml (Application)
                       │
                       │ watches k8s/argocd/apps/
                       ▼
        ┌──────────────┴──────────────┐
        │                              │
   argocd.yaml   vault.yaml   headlamp.yaml   minio.yaml   jellyfin.yaml
        │              │             │              │             │
   watches        watches      watches        watches       watches
  k8s/argocd/    k8s/vault/  k8s/headlamp/  k8s/minio/    k8s/jellyfin/
  (excl apps/**)
```

- `root` is the only Application that needs to be `kubectl apply`d by hand — everything else gets onboarded via the loop above.
- Each child Application points at its own `k8s/<app>/` folder. To add a new app, drop a new YAML in `apps/`, commit, push. `root` picks it up on next sync (~3 min).
- `root` also has `automated.selfHeal: true`, so even if you delete a child Application by mistake, `root` recreates it from git.

### Reconciliation loop

For each Application, on a ~3-minute cycle:

1. `argocd-repo-server` clones the source repo and renders manifests (kustomize build, helm template, or raw YAML).
2. `argocd-application-controller` diffs the rendered output against live cluster state.
3. If `automated.selfHeal: true` and there's a drift, controller re-applies the git version.
4. If `automated.prune: true` and a resource exists in cluster but not in git, controller deletes it.

### Image Updater loop (orthogonal)

On a separate ~2-minute cycle:

1. Image Updater reads each `ImageUpdater` CR + the annotated Application.
2. For each image in the annotation `image-list`, polls the registry.
3. If a new digest is found and `update-strategy: digest` matches, commits the new digest into the git source (mutating `kustomization.yaml`'s `images:` block in our case).
4. Argo's normal reconcile loop then sees the git change and rolls the pods.

---

## 3. Installation (from scratch)

### Prereqs

- k3s up, kubectl works, `KUBECONFIG=$HOME/.kube/config`.
- This repo cloned at `~/homelab`.

### Step 1 — Argo CD core

```bash
# Namespace
kubectl apply -f ~/homelab/k8s/argocd/00-namespace.yaml

# Upstream install — server-side apply is REQUIRED.
# The ApplicationSet CRD definition exceeds the 256 KB annotation limit
# that client-side apply uses for last-applied-configuration tracking.
# Without --server-side, you get:
#   "metadata.annotations: Too long: must have at most 262144 bytes"
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 2 — Our overrides

```bash
kubectl apply -f ~/homelab/k8s/argocd/02-server-cmd-params.yaml
kubectl apply -f ~/homelab/k8s/argocd/01-server-loadbalancer.yaml

# Pick up --insecure
kubectl -n argocd rollout restart deploy/argocd-server

# Wait
kubectl -n argocd rollout status deploy/argocd-server
kubectl -n argocd rollout status statefulset/argocd-application-controller
```

### Step 3 — Bootstrap the App of Apps

```bash
kubectl apply -f ~/homelab/k8s/argocd/apps/root.yaml
```

`root` then picks up every other YAML in `apps/` automatically.

### Step 4 — Image Updater

```bash
# Install the controller. NOTE: /config/install.yaml, NOT /manifests/install.yaml.
# The old /manifests path 404s on current releases.
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml

# Wait
kubectl -n argocd rollout status deploy/argocd-image-updater
```

The `ImageUpdater` CR at `k8s/argocd/03-image-updater.yaml` will be applied by Argo once the `argocd` self-Application syncs.

### Step 5 — Wire the GitHub PAT for Image Updater

See §10 for full PAT setup. Without this the controller logs `403 Forbidden` on every git push attempt.

### Step 6 — Login + first sync

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

Browse <http://homelab:8090>, login as `admin`, change the password in User Info → Update Password, then:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

---

## 4. Why `--server-side` apply

The upstream Argo CD install bundle defines several large CRDs (`ApplicationSet`, `Application`). When applied client-side, kubectl writes the entire submitted manifest into the `kubectl.kubernetes.io/last-applied-configuration` annotation. That annotation has a hard 256 KB cap (etcd limit). The `ApplicationSet` CRD alone breaches it.

Server-side apply skips that annotation entirely, using a fields-ownership model in etcd instead.

Rule: **always use `--server-side` when applying the upstream Argo install bundle.** Our `argocd` self-Application includes `syncOptions: ServerSideApply=true` for the same reason — so Argo can re-apply the install bundle when upgrading.

---

## 5. Custom config explained

### LoadBalancer on `:8090`

File: `k8s/argocd/01-server-loadbalancer.yaml`.

k3s ships ServiceLB (klipper-lb), which gives every `LoadBalancer` Service an external IP equal to the node's IP. So `argocd-server-lb` ends up listening on `WSL_IP:8090`, which over the tailnet resolves to `homelab:8090`.

Why a separate Service instead of editing the bundled `argocd-server`: the upstream install owns that Service. Our own Service is additive and survives Argo upgrades cleanly.

### `server.insecure: "true"`

File: `k8s/argocd/02-server-cmd-params.yaml` (ConfigMap `argocd-cmd-params-cm`).

By default `argocd-server` terminates TLS internally with a self-signed cert. That breaks browser access without cert juggling. We disable it because:

- The wire is already encrypted by Tailscale's WireGuard tunnel between Mac and the WSL node.
- The traffic between Mac → WSL node never touches the LAN or the public internet unencrypted.
- Self-signed TLS on `:8090` adds zero security and a lot of friction (browser warnings, CLI `--insecure` flags everywhere).

Don't use this on a node that isn't behind a private mesh.

---

## 6. Bootstrap sequence (the chicken-and-egg)

Argo manages itself. The order matters on first install:

1. `kubectl apply 00-namespace.yaml` — manual.
2. `kubectl apply --server-side <upstream install>` — manual.
3. `kubectl apply 02-server-cmd-params.yaml` + `01-server-loadbalancer.yaml` — manual (one-shot; Argo will own these from now on).
4. `kubectl apply apps/root.yaml` — manual; this is the only Application you ever hand-apply.
5. `root` reconciles → discovers every other Application YAML in `apps/` → creates them.
6. `argocd` (self-) Application then claims ownership of `00-namespace.yaml`, `01-server-loadbalancer.yaml`, `02-server-cmd-params.yaml`, `03-image-updater.yaml` and re-applies them on every cycle.

After step 6, you can edit any of those files in git and the change will sync automatically. You never need to `kubectl apply` to Argo's namespace again, except to upgrade the upstream Argo install itself (and even that can be done via git by bumping the source URL — though we currently treat upstream upgrades as a manual `kubectl apply --server-side`).

---

## 7. Applications inventory

All Applications live in `k8s/argocd/apps/`. Each one targets `namespace: argocd` (the Application resource itself lives there) but its `spec.destination.namespace` points at the app's actual namespace.

| File | App name | Source path | Dest ns | `prune` | `selfHeal` | Notes |
|---|---|---|---|---|---|---|
| `root.yaml` | `root` | `k8s/argocd/apps` | `argocd` | `false` | `true` | The App-of-Apps root. Manages every other Application. |
| `argocd.yaml` | `argocd` | `k8s/argocd` (excl `apps/**`) | `argocd` | `false` | `true` | Argo manages itself. Has `ServerSideApply=true` for the install bundle. |
| `vault.yaml` | `vault` | `k8s/vault` (Kustomize) | `vault` | `false` | `true` | Kustomize source. Annotated for Image Updater. |
| `headlamp.yaml` | `headlamp` | `k8s/headlamp` | `headlamp` | **`true`** | `true` | Stateless, safe to prune. |
| `minio.yaml` | `minio` | `k8s/minio` (excl `*.template.yaml`) | `minio` | `false` | `true` | `ignoreDifferences` on `spec.replicas` for manual scale-to-0. |
| `jellyfin.yaml` | `jellyfin` | `k8s/jellyfin` | `jellyfin` | `false` | `true` | `ignoreDifferences` on `spec.replicas` for manual scale-to-0. |

### Why `prune: false` on most apps

Pruning means "delete cluster resources that no longer exist in git." It sounds safe but it isn't for stateful apps:

- A typo'd `git rm`, a botched rebase, or a renamed resource would delete PVCs / Secrets / ConfigMaps wholesale.
- Vault, MinIO, Jellyfin all have data on PVCs or hostPaths — accidentally pruning the PVC = data loss.
- With `prune: false`, Argo flags the app `OutOfSync` with `Requires Pruning: true` until you manually approve via UI ("Sync" → check "Prune") or `kubectl delete` the orphan yourself.

Only `headlamp` has `prune: true` — fully stateless, no risk.

### Why `ignoreDifferences` on `spec.replicas` for MinIO and Jellyfin

Sometimes we manually scale to 0 to save resources (e.g. Jellyfin when no one's streaming). Without `ignoreDifferences`, Argo would see "git says 1, cluster says 0" and immediately scale back up because `selfHeal: true`. The `ignoreDifferences` block tells Argo to skip the `replicas` field when diffing.

Vault doesn't have this — its replicas are managed entirely by git.

---

## 8. Sync policies — full reference

### `automated.selfHeal`

If `true`, Argo re-applies the git version any time live cluster state drifts. If `false`, Argo only syncs on a git change or manual trigger.

We use `true` everywhere — without it, ad-hoc `kubectl edit` survives indefinitely, defeating the point of GitOps.

### `automated.prune`

If `true`, Argo deletes cluster resources that disappear from git. Discussed above; `false` everywhere except headlamp.

### `syncOptions: CreateNamespace=true`

If the destination namespace doesn't exist, create it before applying resources. We rely on this for every app — none of the `k8s/<app>/00-namespace.yaml` files would apply otherwise on a fresh cluster.

### `syncOptions: ServerSideApply=true`

Only on the `argocd` self-Application. Required because the upstream Argo install bundle exceeds the client-side annotation size limit (see §4).

### `ignoreDifferences`

Tells Argo to skip specific fields when diffing live vs git. Used for `spec.replicas` on MinIO + Jellyfin to allow manual scaling.

Format:
```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas
```

### Per-resource sync options (annotations)

You can annotate individual resources to override Application-level policy. Examples:

```yaml
metadata:
  annotations:
    # Never prune this resource even if Application has prune: true
    argocd.argoproj.io/sync-options: Prune=false
    # Apply this before everything else (sync wave)
    argocd.argoproj.io/sync-wave: "-1"
```

See §9 for an incident where `Prune=false` per-resource saved us.

---

## 9. Common gotchas

### `directory.recurse: false` gets stripped

Argo normalizes Application specs at admission, dropping fields that match the default value. `directory.recurse` defaults to `false`, so writing it explicitly causes Argo to remove it on apply, then immediately flag the Application as `OutOfSync` because git says `false` and cluster says `<absent>`.

Fix: just omit it. If you need `recurse: true`, write that explicitly; if you want `false`, write nothing.

### Orphan resource pruning incident (`strip-api` → `strip-vault-api`)

We renamed a Traefik Middleware from `strip-api` to `strip-vault-api` in git. With `prune: false`, the old `strip-api` Middleware stayed in the cluster as an orphan, and Argo flagged the app `OutOfSync` with `Requires Pruning: true`.

The right fix in this case was to actually delete the orphan (it was no longer referenced by any Ingress):

```bash
kubectl -n vault delete middleware strip-api
```

For resources we *want* to keep alive across renames (rare — e.g. a Secret we manage out-of-band), the per-resource annotation works:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: Prune=false
```

This way, even if the Application has `prune: true`, this specific resource is exempted.

### ConfigMap changes don't trigger pod restarts

Mounting a ConfigMap as env vars or as a file does NOT auto-restart pods when the ConfigMap changes. Argo will happily apply the new ConfigMap, but pods keep the old values until they restart.

Fixes (any one):

1. Manual: `kubectl rollout restart deploy/<name>` after the ConfigMap change syncs.
2. Stakater Reloader (extra controller): watches ConfigMaps/Secrets and rolls dependent Deployments automatically.
3. Embed a hash of the ConfigMap into the Deployment's pod template annotations (kustomize `configMapGenerator` does this automatically).

Currently we manual-restart. Reloader is a future TODO.

### Argo CD upgrade requires `--server-side`

When bumping Argo to a new version, re-running `kubectl apply -f <new install.yaml>` without `--server-side` will fail on the `ApplicationSet` CRD with the 256 KB annotation error. Always:

```bash
kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

`--force-conflicts` is needed because our `argocd` self-Application also claims ownership of some of those fields (via `ServerSideApply=true`).

---

## 10. Argo CD Image Updater

### What it does

Polls a container registry (we use GHCR) on a ~2-minute cycle. When it sees a new image digest matching the configured constraint, it writes the new digest back to the git source so Argo's normal sync loop picks it up and rolls the pods.

Closes the manual `kubectl rollout restart` loop after every GHA build.

### Conceptual decision: `argocd` vs `git` write-back

Image Updater supports two ways to update an Application:

**Option A: `write-back-method: argocd`**

- Writes the new digest to the Application's `spec.source.helm.parameters` or `spec.source.kustomize.images` field via the Argo CD API.
- Doesn't touch git. The override lives in-cluster only.
- Doesn't work for plain-YAML sources (no override mechanism exists for raw directory sources).
- Lost if the Application is deleted and recreated from git (because git doesn't know about the override).

**Option B: `write-back-method: git`** (what we use)

- Commits the new digest to the source repo, mutating the actual `kustomization.yaml` (or Helm values) in git.
- Works for any source type that Image Updater supports (Helm, Kustomize).
- Survives Application recreate.
- Full git audit trail of every image bump.
- Requires a PAT (the controller needs to push commits).

We chose Option B because:

1. Vault was originally a plain-YAML source — Option A wasn't available. Even after converting to Kustomize, Option B is strictly more durable.
2. Git history of image bumps is valuable (can git-blame a regression to a specific image version).
3. Survives full cluster wipe-and-restore.

### Why Vault was converted from plain YAML to Kustomize

Image Updater **only supports Helm or Kustomize sources**. Plain YAML directories have no concept of parameterized image references, so there's nothing for the updater to mutate.

Conversion was minimal:

- Added `k8s/vault/kustomization.yaml` listing each existing YAML as a `resources:` entry.
- Added an `images:` block in `kustomization.yaml` with `name` + `newTag` for each image.
- Removed `spec.source.directory.exclude` from `apps/vault.yaml` — Argo treats the presence of `directory:` as "this is a plain-YAML source," which conflicts with auto-detected Kustomize.

After conversion, Argo auto-detects the `kustomization.yaml` and uses `kustomize build` to render.

### The `ImageUpdater` CR (v1.2.0+)

In Image Updater v1.2.0 the architecture moved from "pure annotations on Applications" to "an `ImageUpdater` CR that points at one or more Applications, plus annotations for per-image config."

Our CR (`k8s/argocd/03-image-updater.yaml`):

```yaml
apiVersion: argocd-image-updater.argoproj.io/v1alpha1
kind: ImageUpdater
metadata:
  name: vault
  namespace: argocd
spec:
  applicationRefs:
    - namePattern: vault
      useAnnotations: true
```

`useAnnotations: true` keeps the familiar annotation-driven config — per-image strategy and write-back lives on the Application, not the CR. To onboard a new app: add a second entry under `applicationRefs` (or create a separate `ImageUpdater` CR — either works).

### Application annotations (on `apps/vault.yaml`)

```yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: frontend=ghcr.io/yakshith15/vault-frontend:latest,backend=ghcr.io/yakshith15/vault-backend:latest
  argocd-image-updater.argoproj.io/frontend.update-strategy: digest
  argocd-image-updater.argoproj.io/backend.update-strategy: digest
  argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/argocd-image-updater-secret
  argocd-image-updater.argoproj.io/git-branch: main
```

Breakdown:

- `image-list`: `<alias>=<registry>/<image>:<constraint>` — comma-separated. Alias is arbitrary; used as the prefix for per-image annotations below.
- `<alias>.update-strategy: digest` — match by digest of the `:latest` tag. Alternatives: `semver`, `latest` (tag-name lexical), `name` (alphabetical).
- `write-back-method: git:secret:argocd/argocd-image-updater-secret` — push to git using credentials from the named secret. Syntax: `git:secret:<namespace>/<secret-name>`.
- `git-branch: main` — what to push to.

### Why `:latest` is in the image-list, not in `allow-tags`

The `digest` update strategy needs a **version constraint** — a tag name to monitor for digest changes. Without one, the updater logs:

```
no valid version found for image ...
```

The correct place to put the constraint is in the image-list entry itself (`:latest` after the image name). Putting it in a separate `allow-tags` annotation does NOT work for `digest` strategy — `allow-tags` is for `semver` or `latest` strategies that need to filter tag candidates.

### Secret format (changed in v1.2.0)

Old syntax (pre-v1.2.0):
```yaml
write-back-method: git:secret:argocd/argocd-image-updater-secret#password
```
where `#password` named the key inside the Secret.

New syntax (v1.2.0+):
```yaml
write-back-method: git:secret:argocd/argocd-image-updater-secret
```
The `#key` suffix was dropped. v1.2.0+ expects literal keys named `username` and `password` in the Secret. No way to override.

### GitHub PAT setup (one-time)

```bash
# 1. GitHub → Settings → Developer settings → Personal access tokens → Fine-grained
#    Scope: only Yakshith15/homelab
#    Permissions: Contents: Read and write
#    Copy the token (you only see it once).

# 2. Patch the secret (out-of-band — DO NOT commit the PAT to git):
export GITHUB_PAT='github_pat_xxx...'
kubectl -n argocd patch secret argocd-image-updater-secret \
  --type merge \
  -p "{\"stringData\":{\"username\":\"git\",\"password\":\"$GITHUB_PAT\"}}"

# (The `argocd-image-updater-secret` Secret is created by the upstream install bundle
#  as an empty placeholder — we patch in the credentials.)

# 3. Restart the controller:
kubectl -n argocd rollout restart deploy/argocd-image-updater

# 4. Watch logs for the next poll cycle:
kubectl -n argocd logs -f deploy/argocd-image-updater
```

The `username` value is irrelevant for PAT auth (GitHub uses the PAT as the password and ignores the username); we use `git` by convention.

### Rotating the PAT

```bash
# 1. Generate a new fine-grained PAT in GitHub with the same scope/permissions.
# 2. Patch the secret in-place:
export NEW_PAT='github_pat_yyy...'
kubectl -n argocd patch secret argocd-image-updater-secret \
  --type merge \
  -p "{\"stringData\":{\"username\":\"git\",\"password\":\"$NEW_PAT\"}}"

# 3. Restart the controller:
kubectl -n argocd rollout restart deploy/argocd-image-updater

# 4. Verify next poll cycle pushes successfully (no 403 in logs):
kubectl -n argocd logs -f deploy/argocd-image-updater | grep -i "push\|403\|forbidden"
```

No git changes; no Application annotation changes. Just patch + restart.

### Verifying end-to-end

```bash
# 1. Push a commit to the vault app repo (Yakshith15/vault) on main.
# 2. GHA builds and pushes new images to GHCR.
# 3. Wait ~2 min, then check Image Updater logs:
kubectl -n argocd logs -f deploy/argocd-image-updater | grep vault
#    Expect: "Setting new image to ghcr.io/yakshith15/vault-frontend@sha256:..."
#            "Successfully updated image ..."
#            "Committing 2 parameter update(s) for application vault"

# 4. Verify a new commit appeared on origin/main:
git -C ~/homelab fetch && git -C ~/homelab log --oneline origin/main -5
#    Look for a commit authored by the Image Updater bot mutating
#    k8s/vault/kustomization.yaml's images: block.

# 5. Within ~3 min, Argo syncs the kustomization change and rolls vault pods:
kubectl -n vault get pods -w
```

### Defaults worth knowing

- **Poll interval**: 2 minutes (`--interval` flag). Fine for our use case.
- **Registry credentials**: not needed for public GHCR images. If we ever go private, add a `Secret` with `.dockerconfigjson` and reference it via `pull-secret` annotation.
- **Git commit author**: defaults to `argocd-image-updater <noreply@argoproj.io>`. Configurable via the controller's args, but we leave it as-is.

---

## 11. Operations

### Get the admin password (initial)

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

After first login, set a real password in the UI (User Info → Update Password), then delete the initial secret:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

### Reset the admin password (if forgotten)

```bash
# Delete the bcrypt hash in argocd-secret → Argo regenerates initial-admin-secret on restart.
kubectl -n argocd patch secret argocd-secret \
  -p '{"data":{"admin.password":null,"admin.passwordMtime":null}}'
kubectl -n argocd rollout restart deploy/argocd-server
# Then read the initial-admin-secret again.
```

### Access the UI

<http://homelab:8090> over Tailscale. Username `admin`, password whatever you set.

### Manually trigger a sync

UI: navigate to the app → click `SYNC` button.

CLI:
```bash
kubectl -n argocd patch app <name> --type merge -p '{"operation":{"sync":{}}}'
# Examples:
kubectl -n argocd patch app vault    --type merge -p '{"operation":{"sync":{}}}'
kubectl -n argocd patch app jellyfin --type merge -p '{"operation":{"sync":{}}}'
```

This kicks an immediate reconcile, skipping the ~3-min wait.

### Refresh from git (force re-clone)

If you suspect Argo has a stale copy of the repo:

```bash
kubectl -n argocd patch app <name> --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

This forces `repo-server` to re-clone before the next sync.

### Stop / start everything

```bash
# Stop
kubectl -n argocd scale deploy --all --replicas=0
kubectl -n argocd scale statefulset argocd-application-controller --replicas=0

# Start
kubectl -n argocd scale deploy --all --replicas=1
kubectl -n argocd scale statefulset argocd-application-controller --replicas=1
```

Note the StatefulSet scale is a separate command — `scale deploy --all` doesn't catch it.

### Tail Argo logs

```bash
kubectl -n argocd logs -f deploy/argocd-server                    # UI / API
kubectl -n argocd logs -f deploy/argocd-repo-server               # git clone / kustomize build
kubectl -n argocd logs -f statefulset/argocd-application-controller  # reconcile loop
kubectl -n argocd logs -f deploy/argocd-image-updater             # image polling
```

### Onboard a new app (template)

1. Create the app's manifest directory + manifests:
   ```bash
   mkdir -p ~/homelab/k8s/myapp
   # add 00-namespace.yaml, deployment, service, etc.
   ```

2. Create the Application YAML in `apps/`:

```yaml
# ~/homelab/k8s/argocd/apps/myapp.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/Yakshith15/homelab.git
    targetRevision: main
    path: k8s/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: false        # or true if stateless
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

3. Commit + push.

4. Within ~3 min, `root` reconciles, creates the `myapp` Application, and Argo onboards the new app. No manual `kubectl apply` needed.

### Onboard image auto-updates for a new app

1. Convert the app's source to Kustomize (if not already): add `kustomization.yaml` with `resources:` + `images:` blocks.

2. Add annotations to the Application:
   ```yaml
   metadata:
     annotations:
       argocd-image-updater.argoproj.io/image-list: alias=registry/image:tag
       argocd-image-updater.argoproj.io/alias.update-strategy: digest
       argocd-image-updater.argoproj.io/write-back-method: git:secret:argocd/argocd-image-updater-secret
       argocd-image-updater.argoproj.io/git-branch: main
   ```

3. Add the Application name to the `ImageUpdater` CR's `applicationRefs`:
   ```yaml
   spec:
     applicationRefs:
       - namePattern: vault
         useAnnotations: true
       - namePattern: myapp        # NEW
         useAnnotations: true
   ```

4. Commit + push. The PAT secret is already set up — no per-app credential work needed.

---

## 12. Reference

- Argo CD docs: <https://argo-cd.readthedocs.io/en/stable/>
- App-of-Apps pattern: <https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/>
- Sync policy reference: <https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/>
- Server-side apply: <https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/#server-side-apply>
- Argo CD Image Updater: <https://argocd-image-updater.readthedocs.io/en/stable/>
- Image Updater write-back methods: <https://argocd-image-updater.readthedocs.io/en/stable/basics/update-methods/>
- Image Updater update strategies: <https://argocd-image-updater.readthedocs.io/en/stable/basics/update-strategies/>
- Kustomize images field: <https://kubectl.docs.kubernetes.io/references/kustomize/builtins/#_imagetagtransformer_>

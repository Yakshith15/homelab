# Headlamp on k3s

In-cluster Kubernetes dashboard. Used as the unified UI to start/stop deployments without typing `kubectl` every time.

URL: <http://homelab/headlamp/>

## Layout

| File | Purpose |
|---|---|
| `00-namespace.yaml` | `headlamp` namespace |
| `01-rbac.yaml` | `headlamp` ServiceAccount + cluster-admin ClusterRoleBinding |
| `02-deployment.yaml` | Deployment + ClusterIP Service on port 4466 |
| `03-ingress.yaml` | Traefik ingress routing `/headlamp/*` → service |

## First-time deploy

```bash
cd k8s/headlamp
kubectl apply -f .
kubectl -n headlamp get pods -w   # wait for 1/1 Running
```

## Generate a login token

Headlamp uses Bearer-token auth. Generate one for the `headlamp` ServiceAccount (valid 1 year):

```bash
kubectl -n headlamp create token headlamp --duration=8760h
```

Copy the entire `eyJ...` string and paste into Headlamp's login screen.

Regenerate annually (or whenever you want a fresh token).

## Daily use

| Task | How |
|---|---|
| Stop a deployment | Workloads → Deployments → click deployment → Scale → 0 |
| Start a deployment | same path → Scale → 1 |
| View logs | Workloads → Pods → click pod → Logs tab |
| Exec into pod | Pods → click pod → Terminal tab |
| Edit a manifest live | Right-click resource → Edit |

## Permissions

Currently bound to `cluster-admin` — full read/write on the entire cluster. Fine for a single-user homelab. If more users are ever added, scope down with separate ServiceAccounts and Roles per namespace.

## Notes

- `-base-url=/headlamp` arg makes Headlamp generate prefix-aware URLs so the ingress works without a stripPrefix middleware.
- Image pinned to `:latest` with `imagePullPolicy: Always` — `kubectl rollout restart deploy/headlamp` picks up new versions.
- No persistent storage needed; Headlamp is stateless (auth is per-browser-session via the token you paste).

## Tear down

```bash
kubectl delete namespace headlamp
kubectl delete clusterrolebinding headlamp-admin
```

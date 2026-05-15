# MinIO on k3s

Self-hosted S3-compatible object storage. Used as the homelab's general-purpose blob store (media, backups, documents, anything you'd otherwise stick in S3).

| Endpoint | URL |
|---|---|
| S3 API (for SDKs, `aws cli`, `mc`, rclone, etc.) | <http://homelab:9000> |
| Web console (browser admin UI) | <http://homelab:9001> |

Object data is stored at `D:\minio-data\` on the Windows host (mounted into the pod via `hostPath: /mnt/d/minio-data`). Living on the HDD, not the C: SSD, so we don't burn through SSD space.

## Layout

| File | Purpose |
|---|---|
| `00-namespace.yaml` | `minio` namespace |
| `01-secret.template.yaml` | Template — **do not commit real credentials**. Create the real secret directly. |
| `02-deployment.yaml` | Deployment + LoadBalancer Service (ports 9000/9001) |

## First-time deploy

```bash
# 1. Make sure /mnt/d/minio-data exists and is owned by uid 1000
sudo mkdir -p /mnt/d/minio-data
sudo chown 1000:1000 /mnt/d/minio-data

# 2. Namespace
kubectl apply -f 00-namespace.yaml

# 3. Real secret (do NOT use the template)
kubectl -n minio create secret generic minio-secrets \
  --from-literal=MINIO_ROOT_USER='admin' \
  --from-literal=MINIO_ROOT_PASSWORD='<long-random-password>'
# Generate a password with: openssl rand -base64 24

# 4. Deployment + Service
kubectl apply -f 02-deployment.yaml

# 5. Wait for Ready
kubectl -n minio rollout status deploy/minio
```

## Verify

```bash
kubectl -n minio get pods
kubectl -n minio get svc
kubectl -n minio logs deploy/minio
```

Then in browser: <http://homelab:9001> — log in with the credentials you set.

## Daily use

### Web console
Browser → <http://homelab:9001> → log in. Create buckets, drag-drop files, set policies.

### CLI (`mc`) from Mac

Install once:
```bash
brew install minio-mc
```

Configure alias to your MinIO instance:
```bash
mc alias set homelab http://homelab:9000 admin '<your-password>'
```

Common ops:
```bash
mc ls homelab                              # list buckets
mc mb homelab/photos                       # make bucket
mc cp photo.jpg homelab/photos/            # upload
mc cp homelab/photos/photo.jpg ./          # download
mc cp -r ~/Documents homelab/docs/         # recursive upload
mc rm homelab/photos/photo.jpg             # delete
mc share download homelab/photos/photo.jpg # presigned URL (good for ~7 days)
```

### `aws cli` from Mac (works because S3 API is compatible)

```bash
aws configure --profile homelab
# Access Key: <MINIO_ROOT_USER>
# Secret Key: <MINIO_ROOT_PASSWORD>
# Region: us-east-1 (just a placeholder)
# Output: json

aws --profile homelab --endpoint-url http://homelab:9000 s3 ls
aws --profile homelab --endpoint-url http://homelab:9000 s3 cp file.txt s3://docs/
```

### Python (boto3)

```python
import boto3
s3 = boto3.client(
    's3',
    endpoint_url='http://homelab:9000',
    aws_access_key_id='admin',
    aws_secret_access_key='<password>',
)
s3.upload_file('local.txt', 'docs', 'remote.txt')
```

## Updating the admin password

```bash
kubectl -n minio delete secret minio-secrets
kubectl -n minio create secret generic minio-secrets \
  --from-literal=MINIO_ROOT_USER='admin' \
  --from-literal=MINIO_ROOT_PASSWORD='<new-password>'
kubectl -n minio rollout restart deploy/minio
```

## Backing up

The entire MinIO state (objects + metadata) lives in `D:\minio-data\`. Backup is a copy of that folder:

```bash
# Snapshot to another drive / NAS
sudo cp -r /mnt/d/minio-data /mnt/e/backups/minio-$(date +%Y%m%d)
```

Or `rclone sync` to another S3 / cloud destination.

## Notes

- **Single-drive mode**: this is a single-node, single-drive deployment — no erasure coding, no replication. If `D:\minio-data\` is corrupted/lost, the data is gone. Worth setting up rsync to E: or external NAS for anything important.
- **Don't touch `D:\minio-data\.minio.sys\`**: that's MinIO's internal metadata (IAM, bucket configs). Editing it directly can corrupt the install.
- **Adding files via Windows Explorer**: technically possible (you can drag a file into `D:\minio-data\photos\`), but bypasses MinIO's metadata layer. Prefer uploading via API/CLI/console so MinIO tracks it properly.
- **Image pinned to `:latest`**: `kubectl rollout restart deploy/minio` picks up new versions. Pin to a specific `RELEASE.YYYY-MM-DD` tag if you want stability.
- **Expansion**: if D: fills up, you can add E: as an additional pool — needs a redeploy with a new MinIO server pool (out of scope of this initial setup).

## Tear down

```bash
kubectl delete namespace minio
```

Object data on `D:\minio-data\` is **NOT** deleted (it's outside the cluster). Remove manually if desired:
```bash
sudo rm -rf /mnt/d/minio-data
```

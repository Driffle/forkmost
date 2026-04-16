# Forkmost Production Deployment Runbook

This document captures the exact deployment flow used for this repository so the stack can be recreated later.

## 0) Current deployment shape (as implemented)

- Namespace: `services`
- App deployment name: `be-wiki`
- Redis deployment name: `be-wiki-redis`
- Service account: `be-wiki` (IRSA role `arn:aws:iam::400472617168:role/prod-be-wiki-role`)
- Image: `400472617168.dkr.ecr.eu-north-1.amazonaws.com/forkmost:latest`
- Storage mode: `STORAGE_DRIVER=s3`
- External Secret name: `be-wiki` (from `driffle-parameter-store`)
- Ingress currently active: `internal-alb` with host `wiki.prod.driffle.com`

## 1) Prerequisites

- `kubectl`, `aws`, `docker` (with `buildx`) installed
- kubecontext set to the correct cluster
- AWS profile with ECR + EKS + parameter store access
- ECR repository exists: `forkmost`
- S3 bucket exists and is writable by the IRSA role
- External Secrets Operator and `ClusterSecretStore` (`driffle-parameter-store`) are present

## 2) Create / update required secrets in Parameter Store

The Kubernetes `ClusterExternalSecret` reads:

- `/conf/prod/forkmost/APP_SECRET`
- `/conf/prod/forkmost/DATABASE_URL`
- `/conf/prod/forkmost/AWS_S3_REGION`
- `/conf/prod/forkmost/AWS_S3_BUCKET`

Example values:

- `APP_SECRET`: long random string (64+ chars)
- `DATABASE_URL`: `postgresql://<user>:<password>@<rds-host>:5432/forkmost`
- `AWS_S3_REGION`: `eu-north-1`
- `AWS_S3_BUCKET`: `<your-bucket-name>`

## 3) Build and push Docker image (amd64)

Important: cluster nodes are `amd64`, so always push `linux/amd64`.

```bash
aws ecr get-login-password --region eu-north-1 --profile AdministratorAccess-400472617168 \
| docker login --username AWS --password-stdin 400472617168.dkr.ecr.eu-north-1.amazonaws.com

docker buildx build \
  --platform linux/amd64 \
  -t 400472617168.dkr.ecr.eu-north-1.amazonaws.com/forkmost:latest \
  -f Dockerfile . \
  --push
```

Verify image architecture:

```bash
docker buildx imagetools inspect 400472617168.dkr.ecr.eu-north-1.amazonaws.com/forkmost:latest
```

Expected in output: `linux/amd64`.

## 4) Apply Kubernetes manifests

From repo root:

```bash
kubectl apply -f .k8s/prod/secrets.yml
kubectl apply -f .k8s/prod/server.yml
kubectl apply -f .k8s/prod/gateway.yml
```

Rollout and health checks:

```bash
kubectl -n services rollout restart deploy/be-wiki
kubectl -n services rollout status deploy/be-wiki
kubectl -n services get pods -l app.kubernetes.io/name=be-wiki -o wide
kubectl -n services logs deploy/be-wiki --tail=150
```

## 5) Validate service wiring

```bash
kubectl -n services get ingress -o wide
kubectl -n services describe ingress be-wiki-internal
kubectl -n services get svc be-wiki be-wiki-redis -o wide
kubectl -n services get endpoints be-wiki be-wiki-redis -o wide
```

## 6) DNS setup

For current manifest (`internal-alb` ingress):

- Create DNS record for `wiki.prod.driffle.com` -> ALB address shown in ingress output.

If you switch back to Kong ingress with host `wiki.driffle.com`:

- Create `wiki.driffle.com` -> Kong public load balancer DNS name.

Quick check:

```bash
dig +short wiki.prod.driffle.com
curl -I https://wiki.prod.driffle.com
```

## 7) Database setup (RDS PostgreSQL)

Connect as admin:

```bash
psql "postgresql://<admin-user>:<admin-password>@<rds-host>:5432/postgres"
```

Run:

```sql
CREATE USER forkmost_app WITH PASSWORD '<strong-password>';
CREATE DATABASE forkmost OWNER forkmost_app;
\c forkmost
CREATE EXTENSION IF NOT EXISTS unaccent;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
GRANT ALL PRIVILEGES ON DATABASE forkmost TO forkmost_app;
```

## 8) Restore SQL backup into RDS

If your dump is a plain SQL file:

```bash
psql "postgresql://forkmost_app:<password>@<rds-host>:5432/forkmost" \
  -f dump_on_2026-04-13_12-00-01.sql
```

If it contains role/database creation statements and fails on permissions, strip those sections or import as admin once, then re-run app user grants.

## 9) Migrate attachment files to S3 (if moving from local storage)

If you have local files from old Docker volume, sync them:

```bash
aws s3 sync /path/to/local/storage s3://<your-bucket-name>/ --profile AdministratorAccess-400472617168 --region eu-north-1
```

Use the same bucket configured in `/conf/prod/forkmost/AWS_S3_BUCKET`.

## 10) Update Google OIDC provider credentials in DB

Check current provider:

```sql
SELECT id, name, type, oidc_issuer, oidc_client_id, is_enabled, updated_at
FROM auth_providers
WHERE deleted_at IS NULL
ORDER BY updated_at DESC;
```

Update existing provider row:

```sql
UPDATE auth_providers
SET
  oidc_client_id = '<new-client-id>',
  oidc_client_secret = '<new-client-secret>',
  oidc_issuer = 'https://accounts.google.com',
  is_enabled = true,
  updated_at = NOW()
WHERE id = '<provider-id>'
  AND deleted_at IS NULL;
```

## 11) Post-deploy smoke checklist

- App pod ready: `kubectl -n services get pods -l app.kubernetes.io/name=be-wiki`
- Health endpoint: `GET /api/health` returns 200 through ingress
- Login page loads
- Google SSO callback succeeds
- Page create/edit works
- Attachment upload works (verifies S3 path + IAM)
- No repeating errors in app logs

## 12) Known pitfalls (already encountered)

- `exec format error`: pushed image was `arm64`; fix by building `--platform linux/amd64`.
- Domain not reachable even when pods are healthy: DNS record missing/mispointed.
- OIDC login breaks after client rotation: DB `auth_providers` values not updated.
- Redis `MISCONF` / `ENOSPC` in local Docker: Docker Desktop disk full.


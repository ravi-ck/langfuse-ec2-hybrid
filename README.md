# Langfuse — Hybrid EC2 Deployment

Run the **stateless Langfuse app** (web + worker) in Docker while keeping the
**stateful data services native on the EC2 host**:

| Component        | Where it runs        |
| ---------------- | -------------------- |
| langfuse-web     | Docker container     |
| langfuse-worker  | Docker container     |
| PostgreSQL 17    | Native host (systemd)|
| Redis 7          | Native host (systemd)|
| ClickHouse       | Native host (systemd)|
| Blob storage     | **AWS S3** (managed) |

Containers reach host services via `host.docker.internal`, which Docker maps to
the host gateway. Host services listen on the docker bridge IP `172.17.0.1`.

> 📔 **Already deployed this and hit errors?** See
> [`DEPLOYMENT_RUNBOOK.md`](DEPLOYMENT_RUNBOOK.md) — a chronological log of every
> error encountered during a real Amazon Linux bring-up (Postgres `P1001`/`P1010`,
> ClickHouse install/auth, Redis `ECONNREFUSED`/`EPIPE`/protected-mode) with
> root causes and fixes, plus a generic debugging recipe.
>
> **Service-name note (Amazon Linux):** the Redis package is **`redis6`** (client
> `redis6-cli`, config `/etc/redis6/redis6.conf`), not `redis`/`redis-server`.
> Substitute accordingly in the commands below. ClickHouse installed via the
> standalone binary needs a custom systemd unit (see the runbook).

## Sizing

| Tier            | vCPU | RAM   | Disk (gp3) | Example EC2   |
| --------------- | ---- | ----- | ---------- | ------------- |
| Minimum / PoC   | 4    | 16 GB | 100 GB     | `m6i.xlarge`  |
| Recommended     | 8    | 32 GB | 200–500 GB | `m6i.2xlarge` |
| Heavy ingestion | 16   | 64 GB | 500 GB+    | `m6i.4xlarge` |

Use `m`-family (not burstable `t3`) for sustained load. ClickHouse is the
dominant RAM consumer — its limit is capped in `clickhouse/listen.xml`.

## Files

| File | Install location |
| ---- | ---------------- |
| `docker-compose.yml` | deployment dir (e.g. `/opt/langfuse-ec2-hybrid/`) |
| `.env.example` → `.env` | same dir as compose |
| `postgres.conf.d.langfuse.conf` | `/etc/postgresql/17/main/conf.d/langfuse.conf` |
| `redis.conf.d.langfuse.conf` | merge into `/etc/redis/redis.conf` |
| `clickhouse/listen.xml` | `/etc/clickhouse-server/config.d/listen.xml` |
| `clickhouse/users.langfuse.xml` | `/etc/clickhouse-server/users.d/langfuse.xml` |
| `langfuse-compose.service` | `/etc/systemd/system/` |
| `nginx.langfuse.conf` | `/etc/nginx/conf.d/langfuse.conf` |

## Setup steps

### 1. Install host services
```bash
# PostgreSQL 17, Redis 7, ClickHouse via your distro packages, e.g. Ubuntu:
sudo apt-get install -y postgresql-17 redis-server
curl https://clickhouse.com/ | sh && sudo ./clickhouse install
```

### 2. Configure host services
Copy the config files to the locations in the table above, replacing every
`<PLACEHOLDER>` with your real passwords. Then create the Postgres password and
ClickHouse DB, and restart each service:
```bash
sudo -u postgres psql -c "ALTER USER postgres PASSWORD '<PG_PASSWORD>';"
sudo systemctl restart postgresql redis-server clickhouse-server
```
Langfuse creates its own ClickHouse tables on first boot via migrations — you
only need the `clickhouse` user and the `default` database to exist.

### 3. AWS S3 + IAM
Create a bucket and grant the EC2 instance role access to it:
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject","s3:PutObject","s3:ListBucket","s3:DeleteObject"],
  "Resource": ["arn:aws:s3:::<S3_BUCKET>","arn:aws:s3:::<S3_BUCKET>/*"]
}
```
**IMDS hop limit:** to let containers read the instance role via IMDSv2, set the
metadata hop limit to 2:
```bash
aws ec2 modify-instance-metadata-options --instance-id <ID> \
  --http-put-response-hop-limit 2 --http-tokens required
```
If you prefer not to touch IMDS, set static IAM-user keys in `.env` instead.

### 4. Deploy the app
```bash
sudo mkdir -p /opt/langfuse-ec2-hybrid
sudo cp docker-compose.yml /opt/langfuse-ec2-hybrid/
cp .env.example /opt/langfuse-ec2-hybrid/.env   # then fill in every <PLACEHOLDER>
sudo cp langfuse-compose.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now langfuse-compose.service
```
Postgres + ClickHouse migrations run automatically when the containers start.

### 5. TLS reverse proxy
```bash
sudo cp nginx.langfuse.conf /etc/nginx/conf.d/langfuse.conf  # edit server_name
sudo certbot --nginx -d langfuse.yourdomain.com
sudo systemctl reload nginx
```

## Security checklist
- Host services bind to `172.17.0.1` / `127.0.0.1` only — never `0.0.0.0`.
- Security group exposes **443** (and 80→443) only; not 3000/5432/6379/8123/9000.
- Strong, unique passwords for Postgres, Redis, ClickHouse.
- The three Langfuse secrets (`NEXTAUTH_SECRET`, `SALT`, `ENCRYPTION_KEY`) are
  rotated off their defaults. `ENCRYPTION_KEY` must be exactly 64 hex chars.

## Backups (your responsibility — no Docker volumes here)
- **Postgres:** nightly `pg_dump` → S3.
- **ClickHouse:** `clickhouse-backup` → S3.
- **S3 blob data:** durable by default; enable versioning + lifecycle rules.
- **Safety net:** scheduled EBS snapshots of the data volume.

## Caveats
- Single node = single point of failure; no HA. Backups are mandatory.
- App and databases share CPU/RAM — memory limits are set on both the
  containers (compose) and ClickHouse (config) to prevent one starving another.

# Langfuse Hybrid EC2 Deployment — Runbook & Troubleshooting Log

This documents the **actual** bring-up of Langfuse on a single EC2 instance where
the app runs in Docker and the data tier (Postgres, Redis, ClickHouse) runs
natively on the host, with AWS S3 for blob storage. It includes every error we
hit, the root cause, and the fix — in the order they occurred.

- **Host:** Amazon Linux (user `ec2-user`, private IP `10.0.30.57`)
- **Docker bridge (`docker0`):** `172.17.0.1`
- **Compose project network:** `172.21.0.0/16` (containers get `172.21.0.x`)
- **App containers:** `langfuse-web`, `langfuse-worker` (Docker)
- **Data tier (native/systemd):** PostgreSQL 17, `redis6`, ClickHouse
- **Blob storage:** AWS S3

---

## The one rule that explains 90% of the errors

Three different config knobs look interchangeable but are **not**:

| Knob | Accepts | Example | Wrong |
| ---- | ------- | ------- | ----- |
| Postgres `listen_addresses` | **IP address(es)** to bind | `172.17.0.1` | `172.17.0.0/16` ❌ |
| ClickHouse `<listen_host>` | **IP address(es)** to bind | `172.17.0.1` | `172.17.0.0/16` ❌ |
| Redis `bind` | **IP address(es)** to bind | `172.17.0.1 127.0.0.1` | `172.17.0.0/16` ❌ |
| Postgres `pg_hba.conf` | **CIDR** of allowed clients | `172.16.0.0/12` | `172.17.0.1` (too narrow) |
| ClickHouse `<networks><ip>` | **CIDR** of allowed clients | `172.16.0.0/12` | — |

**Bind = plain IP. Allow-list = CIDR.** Putting a CIDR in a bind directive makes
the service bind nothing (or fail to start) → "connection refused".

And the corollary: **a config drop-in only takes effect if it lives in the file
the service actually loads.** Several errors below were "edited the wrong file."

### Why the CIDR must be `172.16.0.0/12`, not `172.17.0.0/16`

Docker's default bridge (`docker0`) is `172.17.0.0/16`, but **Docker Compose
creates its own project network** on a different subnet — here `172.21.0.0/16`.
The container's source IP is therefore `172.21.0.x`, which is **not** inside
`172.17.0.0/16`. `172.16.0.0/12` spans `172.16.x`–`172.31.x` and covers every
Docker bridge, present and future. Use it in all client allow-lists.

> Containers still reach host services at `172.17.0.1` (the `host-gateway` that
> `host.docker.internal` resolves to), but they *arrive from* `172.21.0.x`. Bind
> target = `172.17.0.1`; allowed source = `172.16.0.0/12`. Two different things.

---

## Error log (chronological)

### 1. Postgres — `P1001: Can't reach database server at host.docker.internal:5432`

```
langfuse-web-1 | Error: P1001
langfuse-web-1 | Can't reach database server at `host.docker.internal:5432`
```

**Meaning:** TCP-level failure — the container's SYN was never answered.
(Distinct from auth errors, which come later as `P1010`.)

**Diagnosis:**
```bash
sudo ss -ltnp | grep 5432          # what is Postgres bound to?
ip -4 addr show docker0            # confirm docker0 == 172.17.0.1
psql "postgresql://postgres:<PW>@172.17.0.1:5432/postgres" -c 'select 1;'  # from host
```
The host test returned **`Connection refused`**, proving Postgres was not bound
to `172.17.0.1` — it was on `localhost` only.

### 2. Postgres — invalid `listen_addresses` (CIDR mistake)

```sql
SHOW listen_addresses;
-- listen_addresses
-- localhost, 172.17.0.0/16
```

**Root cause:** `listen_addresses` was set to a **CIDR** (`172.17.0.0/16`).
That directive only accepts bindable **IP addresses** or `*`, so Postgres
ignored it and bound `localhost` only.

**Fix** (`postgresql.conf`):
```conf
listen_addresses = 'localhost,172.17.0.1'
# or, to avoid depending on the exact bridge IP:
# listen_addresses = '*'
```
`listen_addresses` needs a **full restart**, not reload:
```bash
sudo systemctl restart postgresql
sudo ss -ltnp | grep 5432          # now shows 172.17.0.1:5432
```

### 3. Postgres — `P1010: User was denied access on the database 172.21.0.3`

```
langfuse-web-1 | Error: P1010: User was denied access on the database `172.21.0.3`
```

**Meaning:** TCP now connects (progress!), but `pg_hba.conf` has no rule for the
client IP. Note the IP — **`172.21.0.3`** — the Compose bridge, not `172.17.x`.

We confirmed via the host (which connects as `10.0.30.57`):
```
psql: FATAL: no pg_hba.conf entry for host "10.0.30.57", user "postgres", ...
```
The active `pg_hba.conf` only had:
```
host all all 127.0.0.1/32   md5
host all all 172.17.0.0/16  md5     # too narrow — excludes 172.21.x
```

**Fix:** widen the rule to cover all Docker subnets, keeping the existing `md5`
method (the password was stored md5-compatible):
```
host all all 172.16.0.0/12 md5
```
`pg_hba` changes only need a reload:
```bash
sudo systemctl reload postgresql
sudo -u postgres psql -c \
  "SELECT user_name,address,auth_method FROM pg_hba_file_rules WHERE address IS NOT NULL;"
```
→ Postgres migrations then applied successfully (all 400).

### 4. ClickHouse — `dial tcp 172.17.0.1:9000: connect: connection refused`

```
langfuse-web-1 | error: failed to open database: dial tcp 172.17.0.1:9000: connect: connection refused
langfuse-web-1 | Applying clickhouse migrations failed.
```

**Root cause:** ClickHouse wasn't installed/running at all (only Postgres and
Redis had been set up). Port 9000 is ClickHouse's **native** protocol, used by
`CLICKHOUSE_MIGRATION_URL`; 8123 is HTTP, used by `CLICKHOUSE_URL`.

### 5. ClickHouse — "service is not available" (no systemd unit)

`systemctl status clickhouse-server` → unit not found, but `which clickhouse`
→ `/usr/bin/clickhouse`.

**Root cause:** ClickHouse was installed via the **standalone binary**
(`curl https://clickhouse.com/ | sh`), which does **not** register a
`clickhouse-server` systemd unit. The binary is multi-call (`clickhouse server`,
`clickhouse client`, …) and is managed with `clickhouse start` / `stop`.

**Fix (start now):**
```bash
sudo clickhouse install     # sets up /etc/clickhouse-server, clickhouse user, dirs
sudo clickhouse start
```

**Fix (reboot persistence) — wrap it in systemd:**
```bash
sudo clickhouse stop        # free the ports first
sudo tee /etc/systemd/system/clickhouse-server.service >/dev/null <<'EOF'
[Unit]
Description=ClickHouse Server
After=network-online.target
Wants=network-online.target
[Service]
Type=simple
User=clickhouse
Group=clickhouse
ExecStart=/usr/bin/clickhouse server --config-file=/etc/clickhouse-server/config.xml
Restart=always
RestartSec=3
LimitNOFILE=500000
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now clickhouse-server
```
> Do **not** also install the `clickhouse-server` package — two installs (binary
> + package) conflict. Pick one; we used the binary + the unit above.

During `clickhouse install` it prompts:
`Allow server to accept connections from the network (default is localhost only) [y/N]`
→ we answered **`y`**, which binds `0.0.0.0`. Safe here because the security
group blocks 8123/9000 externally **and** the user ACL restricts auth. (The
alternative — `N` plus a `config.d/listen.xml` with `<listen_host>172.17.0.1`
— also works but is more fragile.)

### 6. ClickHouse — `AUTHENTICATION_FAILED` for user `clickhouse`

```
curl "http://172.17.0.1:8123/?user=clickhouse&password=admin4" --data-binary "SELECT 1"
Code: 516. DB::Exception: clickhouse: Authentication failed ... (AUTHENTICATION_FAILED)
```

Then:
```
sudo cat /etc/clickhouse-server/users.d/langfuse.xml
cat: ... No such file or directory
```

**Root cause:** the `clickhouse` user was never defined — only the default user
existed. The users drop-in didn't exist.

**Fix:** create the user drop-in (note: `<networks>` is where the CIDR is valid),
matching `CLICKHOUSE_PASSWORD` in `.env`:
```bash
sudo mkdir -p /etc/clickhouse-server/users.d
sudo tee /etc/clickhouse-server/users.d/langfuse.xml >/dev/null <<'EOF'
<clickhouse>
    <users>
        <clickhouse>
            <password>admin4</password>
            <networks>
                <ip>172.16.0.0/12</ip>
                <ip>127.0.0.1</ip>
                <ip>::1</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
            <access_management>1</access_management>
        </clickhouse>
    </users>
</clickhouse>
EOF
sudo chown clickhouse:clickhouse /etc/clickhouse-server/users.d/langfuse.xml
sudo chmod 640 /etc/clickhouse-server/users.d/langfuse.xml
```

### 7. ClickHouse — `systemctl restart clickhouse-server` → "Unit not found"

The running instance had been started with `clickhouse start` (standalone
watchdog), so there was no systemd unit yet. To reload the new user file
immediately:
```bash
sudo clickhouse restart
```
Verify:
```bash
curl "http://172.17.0.1:8123/?user=clickhouse&password=admin4" --data-binary "SELECT 1"   # -> 1
```
(Then later switched to the systemd unit from step 5 for reboot persistence.)
→ ClickHouse migrations applied; **web server reached `✓ Ready` on :3000**.

### 8. Redis — `ECONNREFUSED 172.17.0.1:6379`

```
langfuse-web-1 | Redis error connect ECONNREFUSED 172.17.0.1:6379
```

**Diagnosis:**
```bash
sudo systemctl status redis6
#   CGroup: ... redis6-server 127.0.0.1:6379     <-- bound to loopback only
```

**Root cause #1 (wrong file):** Amazon Linux's package is **`redis6`**, and it
loads `/etc/redis6/redis6.conf` — not the file we'd been editing. The edits
(`bind 172.17.0.1`, `requirepass`) were never loaded.

**Root cause #2 (data-loss bug):** the config we *had* edited contained
`maxmemory-policy allkeys-lru`. Langfuse uses Redis as a **BullMQ job queue**,
not a cache — `allkeys-lru` lets Redis **evict queued jobs** (silent data loss).
It **must** be `noeviction`.

Find the real config:
```bash
systemctl cat redis6 | grep ExecStart    # path after redis6-server is the loaded file
```

### 9. Redis — `write EPIPE` (protected mode, no password)

```
langfuse-web-1 | Redis error write EPIPE
```
Connection now *accepted* then *closed* by the server. Direct test revealed why:
```
redis6-cli -h 172.17.0.1 -p 6379 -a admin4 ping
AUTH failed: DENIED Redis is running in protected mode ... no authentication
password is requested to clients ...
```

**Root cause:** the loaded config had **no `requirepass` and no `bind`**. With no
password set, Redis **protected mode** accepts the TCP connection but refuses
commands from non-loopback clients → the client sees `EPIPE`.

**Fix (live + persisted):** set the password from loopback (allowed under
protected mode); once a password exists, authenticated external clients are
accepted:
```bash
redis6-cli -h 127.0.0.1 -p 6379 CONFIG SET requirepass admin4
redis6-cli -h 127.0.0.1 -p 6379 -a admin4 CONFIG SET maxmemory-policy noeviction
redis6-cli -h 127.0.0.1 -p 6379 -a admin4 CONFIG REWRITE   # persists to the loaded config file
```
Verify over the bridge:
```bash
redis6-cli -h 172.17.0.1 -p 6379 -a admin4 ping                  # -> PONG
redis6-cli -h 172.17.0.1 -a admin4 config get maxmemory-policy   # -> noeviction
```
> `CONFIG REWRITE` writes back to whichever file Redis was started with — this
> both persists the change and reveals the real config path.

### Result

```
curl -I http://127.0.0.1:3000
HTTP/1.1 200 OK
```
All four dependencies connected; web + worker green.

---

## Quick reference — the working end state

| Service | Bind (plain IP) | Client allow-list (CIDR) | Auth | Boot unit |
| ------- | --------------- | ------------------------ | ---- | --------- |
| Postgres | `localhost,172.17.0.1` (or `*`) | `pg_hba`: `172.16.0.0/12 md5` | password in `DATABASE_URL` | `postgresql` |
| ClickHouse | `0.0.0.0` (installer `y`) | `<networks>`: `172.16.0.0/12` | `clickhouse` user + password | `clickhouse-server` (custom unit) |
| `redis6` | `172.17.0.1 127.0.0.1` (or unset) | n/a (password only) | `requirepass` | `redis6` |

Container → host addressing:
- `.env` uses `host.docker.internal` (mapped via `extra_hosts: host-gateway`).
- That resolves to `172.17.0.1`; containers connect **to** `172.17.0.1` but
  arrive **from** `172.21.0.x` → allow-lists must be `172.16.0.0/12`.

---

## Post-bring-up checklist (production hardening)

1. **Worker clean:** `docker compose logs --tail=50 langfuse-worker` — no errors.
2. **Boot persistence (critical — host services don't auto-start otherwise):**
   ```bash
   sudo systemctl enable redis6 postgresql clickhouse-server
   sudo systemctl is-enabled redis6 postgresql clickhouse-server   # all "enabled"
   ```
   Install `langfuse-compose.service` (uses `redis6`/`clickhouse-server` in its
   `After=`/`Wants=`). Reboot-test: `sudo reboot`, then
   `curl -I http://127.0.0.1:3000` should be 200 with zero manual steps.
3. **TLS:** install `nginx.langfuse.conf`, run `certbot`, set
   `NEXTAUTH_URL=https://...` in `.env`, `docker compose up -d`.
4. **Firewall:** security group open on **443** (and 80 redirect) only;
   **closed** on 3000 / 5432 / 6379 / 8123 / 9000.
5. **S3 / IAM:** exercise a media upload. On `AccessDenied`, raise the IMDS hop
   limit so containers can read the instance role over IMDSv2:
   ```bash
   aws ec2 modify-instance-metadata-options --instance-id <ID> \
     --http-put-response-hop-limit 2 --http-tokens required
   ```
   (or set static IAM keys in `.env`).
6. **Backups (no Docker volumes — your responsibility):** nightly `pg_dump`;
   `clickhouse-backup`; S3 versioning + lifecycle; scheduled EBS snapshots.

---

## Generic debugging recipe (for the next service that won't connect)

```bash
# 1. Is it running and on which address?
sudo ss -ltnp | grep <port>
# 2. Reachable from the host over the bridge IP? (refused = bind/run problem)
nc -zv 172.17.0.1 <port>
# 3. Auth works locally?  (service-specific client)
#    psql ... / clickhouse-client / redis6-cli -a <pw> ping
# 4. What does the SERVICE log say when the container connects?
sudo tail -n 40 /var/log/<service>/...log
# 5. Is the config you edited the one actually loaded?
systemctl cat <unit> | grep ExecStart      # or `SHOW config_file` / `CONFIG REWRITE`
```
Decision tree:
- **connection refused** → not bound to `172.17.0.1` (bind = plain IP!) or not running.
- **denied / no entry / auth failed** → client allow-list too narrow (use
  `172.16.0.0/12`) or password mismatch.
- **EPIPE / connection closed** → protected mode w/ no password, or TLS-vs-plaintext
  mismatch (`REDIS_TLS_ENABLED`, `rediss://`).
